# HPC — The Proof Loop on a National Supercomputer

The setting of a theorem-proving experiment on **Jean Zay**, France's national
supercomputer (IDRIS, CNRS). An open-weight LLM, served with
[SGLang](https://github.com/sgl-project/sglang) on H100 nodes, proves theorems
in Rocq (in some arms, Lean 4) through a bash-tool harness, against the proof
assistant's kernel, in unattended Slurm jobs.

> The loop is the one in [`Formal_maths/`](../Formal_maths/) (the agent
> proposes, the kernel accepts or rejects), and so is the idea: **don't trust
> the model's word — wire its output to something that can reject it.** Here
> open-weight models drive it through a bash tool (live proof-state tools
> optional), **at scale on a shared cluster**: unattended batch jobs, hours-long
> attempts, records that must be trusted weeks later. Only the setting is
> shared: no speeds, start-up times, GPU-hours, energy, costs or solve rates.

---

## The pattern

```
  login node ── sbatch ──▶   Slurm job on whole, exclusive nodes
                           ┌────────────────────────────────────────────────┐
                           │ model server: SGLang on 1 or 4 nodes of 4 H100 │
                           │       ▲▼  OpenAI-compatible chat, streamed     │
                           │ lane 1 · lane 2 · … · lane N  (one agent each) │
                           │       ▲▼  bash, completed(), session tools     │
                           │ sandbox per attempt (bubblewrap, no network):  │
                           │   Rocq tools (MCP) ◀──▶ Rocq kernel            │
                           │ monitor on every node, every 15 s              │
                           └──────────┬─────────────────────────────────────┘
                                      ▼ records, checksum manifests
                            flash work filesystem ──▶ archive filesystem
                                  │                   (raw logs, compressed, verified)
                                  ▼ saved proofs
  independent recheck of every accepted proof, outside the harness
```

One Slurm job runs **one model server and its lanes** (agent workers), sized on
the **KV-cache pool**, not the GPU count. Every tool call runs in a **sandbox**,
a checker judges in the loop, and an **independent recheck** decides what
counts.

### When it fits

- **It fits** when you hold cluster GPU-hours, want fixed open weights to serve,
  inspect and measure, and run unattended hours-long attempts that must be
  logged in full.
- **It does not fit** closed frontier models (call their APIs) or quick
  iteration: every change waits in a shared queue.

---

## The setting on Jean Zay

### Serving

Each job takes **whole, exclusive nodes** of 4 × H100 80 GB (InfiniBand between
them) and runs one SGLang 0.5.20 server, one process per node in one Slurm step.
Its **lanes** are agent workers, each an 8-CPU Slurm step taking problems from a
file-locked queue, one attempt and one request at a time. Compute nodes have no
internet: CPU jobs stage the weights on the scratch filesystem with SHA-256
checks; SGLang and Rocq run from a prebuilt Python environment and opam switch.

**Tensor parallelism** (TP) splits each layer over GPUs that work on every token
together; **pipeline parallelism** (PP) gives each GPU group a block of layers.
The **KV cache** holds the attention keys and values of the requests in flight;
SGLang's prefix cache, on in every server, keeps a conversation's prefix there
between turns while room lasts, so a turn processes only its new tokens.

| Model (open weights) | Nodes (GPUs) | Parallelism | Weights | KV cache | Context (tokens) | Lanes |
|---|---|---|---|---|---|---|
| [DeepSeek-V4-Flash-0731](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash-0731) ("V4-Flash") | 1 (4) | TP4 | MXFP4 experts, the rest mostly FP8 | FP8 | 1,048,576 | 3 |
| [DeepSeek-V4-Pro](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro) (preview checkpoint) | 4 (16) | TP8 × PP2 | MXFP4 experts, the rest mostly FP8 | FP8 | 1,048,576 | 3 |
| [GLM-5.3](https://huggingface.co/zai-org/GLM-5.3) | 4 (16) | TP4 × PP4 | mostly FP8 | FP8 | 688,128 | 2 |

| Rule | How and why |
|---|---|
| **The KV cache, not only the weights, sets the layout** | These models keep one compressed (latent) KV head, which TP cannot split: under TP every GPU stores the whole cache of the layers it serves, under PP each stage only its own layers' cache. V4-Pro's weights need four nodes; on those 16 GPUs, two pipeline stages of 8 each hold the cache of half the layers, where 16-way TP would put the cache of every layer on every GPU, and that is what makes room for three lanes. GLM-5.3 needs its four nodes for the cache, not the weights. |
| **Native low precision** | DeepSeek-V4 runs as released: routed experts in MXFP4 (4-bit); attention, shared experts and the other dense weights in FP8, with embeddings, LM head and norms in BF16; activations in FP8, so experts run as W4A8. That fits V4-Flash on one node. GLM-5.3's FP8 cache (two lanes, against one with a BF16 cache) came after a numerics check. |
| **KV-aware admission** | Each lane reserves room for its longest possible conversation: 524,288 tokens, where the agent must compact its history (see *Stopping* below), plus one maximal reply of 32,768. Lanes start only if all reservations fit in 90% of the pool, and the server's limit on running requests is the number of lanes. GLM-5.3's context is its pool divided by its two lanes, rounded down to a multiple of 32,768, so a conversation that fails to compact is refused at once instead of growing into the other lane's share. |
| **Readiness and requests** | Before the lanes start, smoke tests check that tool calls parse, and a probe checks that the chat template passes the requested reasoning effort. Requests: temperature 0.7, thinking on, effort high (max for V4-Pro), at most 32,768 new tokens. |

### One attempt (the agent loop)

The benchmark is [Lean-Rocq-bench](https://github.com/LLM4Rocq/lean-rocq-bench)
(public since v1.0.0): **46 problems**, numbered in a random order, each stated
in Lean 4 (Mathlib) and Rocq 9.2 (MathComp). In Rocq each has a **minimal**
variant (only the statement's imports) and an **imports** variant (every library
a known proof uses). An **arm** gives one model one attempt at each of the 46 in
one language, variant, tool set and protocol version. Each model also ran a
bash-only Lean 4 arm, with Lean's toolchain and checks; this README follows the
Rocq side.

The task: replace `Admitted` (Rocq's placeholder for a missing proof) with a
proof, adding imports, definitions or lemmas at will; the theorem and its
definitions must stay as given after elaboration, not just as text. The
**harness**, grown from an open-source bash-tool agent for Lean proofs, stages a
fresh copy of the file and runs the model's tool calls:

| Piece | Rule |
|---|---|
| **Tools** | `bash`; `completed()`, the in-loop check, which ends the attempt on a pass (in Rocq the judge, rocq-comparator; in Lean a stricter textual approximation, with Lean's official checker deciding in the recheck); `disproved()`, a falsity claim for human review. A pilot and then a larger round compared arms with and without Rocq **session tools** from rocq-mcp-evolve, an MCP (Model Context Protocol) server running Rocq in process, one per attempt: open, step, try, check, rollback, state, auto_close, and build from protocol v2. A small proxy checks each call against the tool's schema and times out stuck calls; from v2 it also kills a server whose memory passes 8 GB and tells the model, which must reopen the session (the file is kept, the session's steps are lost). |
| **Stopping** | The model ends an attempt with a passing `completed()` or a `disproved()`; otherwise it runs to the **4-hour wall-clock cap**, where the file is checked once more (a pass there counts). There is no turn cap. A model request that still fails after retries (an over-long one, say) ends the attempt as a failure; an infrastructure fault voids it. A turn without a tool call gets a nudge; past 524,288 tokens the model summarizes its history and continues from the summary (**compaction**). |
| **Protocol v2** | From the larger round on, every tool result shows the time left, a bash command times out after 600 s (v1: 3,600 s), and the prompt asks for a skeleton first: state the main theorem early, prove it from lemmas left `Admitted`, then fill them in. A later option appends to a failed compile's error the list of every failing proof (Rocq reports only the first). |
| **Sandbox** | Commands, checks and proof servers run in [bubblewrap](https://github.com/containers/bubblewrap), an unprivileged sandbox already installed on the nodes that wraps each command, with no image to build: no network, a read-only root, only the attempt's directory and a private temporary directory writable; the harness, the queue, other attempts and any key hidden. A failing sandbox stops the lane; it is never worked around. The first three arms ran before the sandbox existed. |

### Jobs and lanes

| Aspect | Rule |
|---|---|
| **Split** | From the pilot on, a job holds at most four problems per lane, so its lanes finish within the time limit even if every attempt runs to the cap: 4 jobs of 11–12 problems for a V4-Flash arm (3 lanes), 6 of up to 8 for GLM-5.3 (2 lanes). The V4-Pro arms ran as 3 or 4 jobs; the first V4-Flash arms were split while they ran. |
| **Time limit** | Production jobs ask for 20 hours, the most the standard H100 QoS (a Slurm quality-of-service class) allows. A job must hold the start-up cap (1 h on one node, 1.5 h on four), one full attempt and a margin, and a lane takes a new problem only while a full attempt still fits before the limit. |
| **Job flow** | Record the settings and check the time limit → test the sandbox → start the server and wait for health → smoke tests and probes → check the KV pool → start the lanes → write the summary, accounting and checksums. |
| **Release** | No Slurm dependencies: an arm's later jobs are submitted on hold and released by hand once its first job's first attempts look clean, so a human sees real attempts before more GPU-hours are committed. |
| **Voids** | Only infrastructure failures void an attempt: a failed start, a node or job failure, a server crash that its own request did not cause, a sandbox error, the time limit. Failures the model causes count. Void and unstarted problems rerun in a follow-up job whose settings, printed by a dry run before submission, must match the original job's key by key, except the run tag and the problem list. |

### Checking and pre-registration

**In the loop** (Rocq arms), `completed()` has the judge, rocq-comparator,
compare the agent's file with the original. Within 900 s:

- the file compiles, and the theorem and every definition it uses are unchanged
  after elaboration;
- nothing is admitted, and no forbidden command appears (plugins, `Load`,
  extraction and the like);
- only eight axioms are allowed, standard classical ones the libraries assume
  (excluded middle, extensionality, indefinite description and kin): a looser
  bar than the axiom-free proofs of [`Formal_maths/`](../Formal_maths/);
- rocqchk, Rocq's standalone kernel checker, accepts the result.

| Step | Rule |
|---|---|
| **Rejections** | A rejection returns its exact reason; infrastructure errors are retried, never scored. |
| **Independent recheck** | After the runs, the official checkers (rocq-comparator for Rocq, Lean's comparator for Lean) judge every accepted proof again, from the saved files and outside the harness. **A solve counts only if both the loop and the recheck accept.** |
| **Contamination audit** | It checks that no solving attempt saw or copied a proof of its own problem: what its sandbox let it see, every path its commands named or its results showed, and how close its proof is to other proofs of the same problem. |
| **Pre-registration** | For the pilot and for the larger round, the question, arms, differences, what counts as solved, the primary analysis (an exact McNemar test on paired problems) and what may change were frozen before the first production job and checksummed (MD5: a guard against silent edits, not a public timestamp). Later arms came as frozen **addenda**. |

### Records and accounting

| Item | What and how |
|---|---|
| **Per lane** | An SQLite trajectory database (exact prompts, tool schemas, limits and file hashes, then every message, tool call and token count) and a raw HTTP log of every request and streamed chunk, credentials excluded. |
| **Per job** | An event timeline (readiness, smoke tests, KV pool, lanes, a status line every 60 s, failures); 15-second samples on every node (per-GPU utilization, memory, power, temperature; CPU, memory, disk, network; cgroup counters with out-of-memory kills); server logs; Slurm accounting; an archive per attempt; SHA-256 manifests that every copy is checked against. |
| **Per submission** | A ledger record with every setting, the code checksums and the new job ids. |
| **Accounting** | GPU-hours and energy, not money. GPU-hours are allocated GPUs × elapsed time; GPU energy integrates the sampled power; CPU and DRAM energy is Slurm's RAPL counter (the processors' own meters). Network, fans and cooling are not covered, and GPU utilization (the share of time any kernel runs) is never read as efficiency. |
| **Close-out** | Raw logs are archived compressed, each decompressed copy is checked against its run's SHA-256 manifest, and the originals are deleted only after a separate human approval. |

---

## Design decisions that generalize

| Decision | Why it matters |
|---|---|
| **Self-host open weights at a pinned version** | The model cannot change mid-campaign, compute comes from GPU-hours granted by a national allocation instead of a per-token bill, and memory and power can be read at the GPU. |
| **One job = one model server and its lanes** | Server and agents start, fail and end together, and each job leaves one self-contained record. |
| **Size lanes on the KV-cache pool; refuse what does not fit** | Requests beyond the pool are preempted and recomputed, or time out just before finishing: GPU time paid for nothing. |
| **Buy KV room with pipeline stages** | When the KV head cannot be split, more tensor parallelism copies the cache; more pipeline stages divide it. |
| **A wall-clock cap per attempt, a time-limit guard per job** | Every attempt gets the same budget in the unit that is paid for, and none is cut short by its job's end. |
| **Sandbox every tool call, and fail closed** | The agent runs shell commands for hours; a broken sandbox stops the work instead of being bypassed. |
| **Check in the loop, recheck independently** | The loop gives the model exact reasons to act on; the judge, run again outside the harness, decides what counts. |
| **Freeze before running** (pre-registration, checksums, addenda) | Choices are fixed before the production jobs they govern start, and every later change is visible. |
| **Gate production** (a CPU-only verification job, then short smoke jobs) | A failure caught on a CPU node or in a smoke test saves a 20-hour, 16-GPU job. |

---

## Reusable lessons

- **Keep long-lived work in the Slurm job.** What must outlive an ssh session on
  a login node (a long copy, say) is detached with `setsid` and checked from a
  fresh connection, not left to a bare `nohup`.
- **Read Slurm's answers carefully.** Call a job gone only when the queue
  command succeeds without listing it: a transient Slurm failure returns an
  empty reply, which means "unknown". Match alarm words against the event-name
  field only: a search of whole log lines for FAIL or TIMEOUT fired at every
  job start, on the settings line's timeout value.
- **Find the quota that actually binds** (per project, inodes too) and point
  every cache at a filesystem you watch: SGLang's just-in-time GPU-kernel
  builds, cached under the home directory by default, once filled a shared inode
  quota and killed jobs at start-up.
- **Tell "the check failed" from "the check could not launch".** A Slurm
  controller fault, read as a failed check, tore down a healthy 16-GPU server:
  retry a step that cannot launch, and log the two cases apart.
- **Watch memory where it is written.** Read out-of-memory kills live from the
  job's cgroup counters, not Slurm's end-of-step message, and check what each
  one hit.
- **A limit the prompt states must be one the machine enforces.** A review
  caught the new prompt promising per-command memory caps the cluster never
  applied; older prompts did too, and the bash tool's description still does.
- **Know the cluster's traps.** Scratch deletes files unread for 30 days,
  silently breaking Python environments (sources go, caches stay). Compute nodes
  lack internet and some tools: test on the job's partition (without git, Lean's
  build tool deleted Mathlib). The ssh name rotates across login nodes: pin one
  for long copies.

---

## Where the material lives

Not published here: the job scripts, the modified harness and the logs. What
is public:

- [`lean-rocq-bench`](https://github.com/LLM4Rocq/lean-rocq-bench) — the 46
  problems, their Lean and Rocq statements and the official checkers. These runs
  were judged with the checker as it was before v1.0.0, which ships a rewritten
  one.
- [`rocq-mcp-evolve`](https://github.com/LLM4Rocq/rocq-mcp-evolve) — the OCaml
  MCP server behind the session tools, at a pinned commit of its `rocq-9.2`
  branch (not the `rocq-mcp` of [`Formal_maths/`](../Formal_maths/)).
- [`rocq-comparator`](https://github.com/LLM4Rocq/rocq-comparator) — the Rocq
  judge, in the loop and the recheck, modeled on
  [`comparator`](https://github.com/leanprover/comparator), Lean's own checker.

---

## Caveats

- **Training data.** The contamination audit covers what attempts saw, not
  whether the models met these problems or their solutions in training.
- **Jean Zay specifics** (four-GPU nodes, the traps above, project quotas): the
  patterns travel, the numbers may not.
- **Gaps.** GPU topology, InfiniBand link rates and GPU power limits went
  unrecorded; on the cluster, memory was capped per lane, not per command.
