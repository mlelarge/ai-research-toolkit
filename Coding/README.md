# Coding — Claude's Features for Research Engineering

Notes on the Claude Code features that the other folders in this toolkit are
built on. The literature survey, the formal-proof loop, and the course server
are all the *same* coding agent, configured differently. This folder collects
the configuration vocabulary so you can assemble your own setup.

> Treat this as the **shared toolbox**. The use-case folders
> ([`Informal_maths/`](../Informal_maths/),
> [`Formal_maths/`](../Formal_maths/), [`Teaching/`](../Teaching/)) each pick a
> few of these primitives and wire them together. Read here to learn the
> primitives; read there to see them combined.

---

## The primitives

| Feature | What it is | Where it shows up in this toolkit |
|---|---|---|
| **Permissions / allowed-tools** | Restrict an agent to a named set of tools (`--allowed-tools "WebSearch WebFetch"`). Smaller blast radius, more reproducible runs. | The survey agent can search and fetch but not write files or run shell. |
| **MCP servers** | Give the agent live, typed access to an external system (a database, a proof kernel, a course API) as callable tools. | `rocq-mcp` hands the agent live proof state; a course MCP server answers content queries. |
| **Skills** | Reusable, host-agnostic workflows + domain knowledge the agent loads on demand (a style guide, a prove/review loop). | The Rocq style-guide and prove/golf skills; the spatial-edit and review skills. |
| **Hooks** | Code the *harness* runs automatically on events (before/after a tool call, on stop). For "always do X" rules the model can't be trusted to remember. | Auto-formatting, guardrails, post-run validation. |
| **Subagents / agent teams** | Spawn specialized agents for independent sub-tasks and run them in parallel; the parent keeps only the conclusions. | Fan-out exploration; one reviewer per item. |
| **Dynamic workflows** | A deterministic script that orchestrates many agents (loops, fan-out, verify-then-synthesize) rather than leaving control flow to the model. | The propose→verify→synthesize attack loops; multi-agent review. |
| **`claude -p` (non-interactive)** | Run the agent as a CLI batch job on a flat-rate plan — no API key, predictable cost for long runs. | The engine under *every* batch pipeline here: one invocation per item, resumable. |
| **Containers (Docker / Apptainer)** | Reproducible, isolated environments. Apptainer in particular runs unprivileged on HPC clusters where Docker can't. | Pinned toolchains for proof checking and batch runs on shared compute. |

---

## Reusable lessons


- **MCP when the agent needs to *interact* with a system; a skill when it needs
  to *know* something.** They compose — an MCP server plus a style-guide skill is
  the common pair.
- **Hooks for invariants, prompts for judgement.** Anything that must happen
  *every time* belongs in a hook (the harness enforces it), not in a prompt (the
  model may forget).
- **A deterministic workflow beats a clever mega-prompt** once control flow has
  loops, fan-out, or verify-then-decide structure — encode the structure, let
  agents fill the slots.
- **`claude -p` + one-file-per-item is the batch primitive.** Idempotent,
  resumable, `git diff`-able, parallel across terminals. Every pipeline in this
  toolkit is a variation on it.


---

## Docker / Apptainer containers

Example: [`LLM4Rocq/mathcomp-eulerian/Dockerfile`](https://github.com/LLM4Rocq/mathcomp-eulerian/blob/main/Dockerfile)
— a self-contained image that runs Claude Code **autonomously** on a Rocq proof
refactor. It pins the toolchain (Rocq 9.1.1, OCaml 5.2, mathcomp 2.5.0, coq-lsp)
and bakes in the [`rocq-mcp`](https://github.com/LLM4Rocq/rocq-mcp) server, so the
agent gets live proof-state tools with zero host setup — `docker run` and it
proves, with the container as the isolation boundary for `--dangerously-skip-permissions`.
