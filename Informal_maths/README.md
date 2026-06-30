# Informal Maths

Two complementary, reusable pipelines for doing mathematics with LLM agents — the
breadth-first one to triage a whole field, the depth-first one to push on a single
open problem:

| Pipeline | Shape | Question it answers | Output |
|---|---|---|---|
| **[Survey a corpus](#survey-a-corpus-breadth-first)** | breadth — many items, one pass each | "For each of these N items, what's its status?" | a uniform, cited verdict per item |
| **[Attack one conjecture](#attack-one-conjecture-depth-first)** | depth — one problem, many rounds | "Can we push *this* open problem, and where exactly is it stuck?" | a living `ledger.json` + an honest hand-back when only human insight remains |

Both run on the flat-rate CLI, keep all state in plain files (resumable,
`git diff`-able), and are engineered to **hand back honestly** rather than
fabricate progress.

---

# Survey a corpus (breadth-first)

A reusable pipeline for surveying a **corpus of items** with an LLM agent: for
each item, the agent searches the web, verifies its sources, and returns a
**structured, validated verdict**. The worked example is a status review of
~1,000 graph-theory conjectures
([graph-conjectures](https://github.com/mlelarge/graph-conjectures)) — *is each
conjecture still open, solved, disproved, or partially resolved?* — but nothing
below is specific to graph theory.


---

## When this pipeline fits

Reach for it when **all** of these hold:

- You have **many items** (dozens to thousands) that each need the *same kind*
  of judgement.
- Each judgement requires **looking things up** (the web, a database, a set of
  papers) — it is not answerable from the model's memory alone.
- You want the output **machine-readable and uniform** (so you can aggregate,
  filter, render, or diff it), not a pile of prose.
- **Citations / sources must be real.** A fabricated reference is worse than no
  answer.

If your task is one-off, or needs no lookups, or wants free-form prose, this is
overkill — just ask the model directly.

---

## The pipeline at a glance

```
  corpus.json            one record per item to review
        │
        ▼
  ┌───────────────┐      partition the work-list into N buckets
  │  partition    │      (skip items already done — idempotent)
  └───────────────┘
        │   bucket-01 … bucket-NN
        ▼
  ┌───────────────┐      N terminals, each: for each item in its bucket,
  │  workers      │      run one agent with web tools + a fixed system prompt,
  │  (claude -p)  │      write one JSON file per item
  └───────────────┘
        │   reviews/<id>.json   (idempotent, resumable, one file per item)
        ▼
  ┌───────────────┐      validate against the schema; dedup; cross-reference;
  │  aggregate    │      attach to whatever consumes it
  └───────────────┘
        │
        ▼
  reviews.json  →  site / report / dataset
```

The whole thing is **five stages**. Each is independently re-runnable.

---

## The five stages (generic)

### 1. Build the corpus — *the "what to review" list*

A flat list of records, one per item, each carrying everything the agent needs
to start: a title/identifier, the statement or content under review, an anchor
date ("what happened *since* this?"), and any **seed references** you already
have. Seeds matter — they give the agent a backward-search starting point
instead of a cold start.

> In the example, the corpus came from two sources: a crawl of an open-problem
> database, *and* a second agent pass that **extracted** new conjectures from
> recent papers. That extraction step (paper → list of conjectures, with a
> schema and an attribution rule) is the same pipeline pointed at a different
> question — proof that the pattern composes.

### 2. Define the output schema — *the contract*

Decide the **exact JSON shape** the agent must return *before* writing the
prompt. The schema is what every downstream step depends on, so it is the real
interface. Keep a small, closed set of verdict values — e.g.
`open | solved | disproved | partial | unclear` — and **always include an
explicit "I couldn't decide" value**. Carry `confidence`, a short human summary,
a list of cited sources (each with a URL you can check), and a `notes` field for
caveats.

A useful trick: also require the agent to return the **list of URLs it actually
verified**, separately from the list it cites. Then a downstream validator can
*reject any citation whose URL was not verified* — turning "please don't
hallucinate" into a mechanical, enforced invariant (see §"Verify before cite").

### 3. Write the system prompt — *the reviewer's job description*

One file (~1–2 K tokens), reused verbatim across every item (so the backend
**prompt-caches** it for free). It should pin down:

- **Role & task** in one line ("you review one item and return its status").
- **Method**: which tools, in what order, search by *content* not by *name*
  (distinctive technical terms beat un-searchable labels like "Conjecture 5.10").
- **A hard cap on tool calls** (e.g. "at most 5 searches") so runtime is bounded.
- **Parallelism instruction**: "issue independent searches/fetches in a *single*
  turn" — batched tool calls are the single biggest wall-clock win.
- **The anti-guessing rule**: if the search comes up empty, return the
  "unclear"/"open" value with low confidence — *that is the correct answer, not
  a failure.* Spell this out or the model will invent progress to seem helpful.
- **The schema**, inline, with an example.

### 4. Run the reviews — *orchestration*

The mechanics that make a long, flaky, rate-limited run actually finish:

- **One agent invocation per item** via the CLI in non-interactive mode
  (`claude -p`), restricted to read-only web tools.
- **One output file per item.** This is the whole resumability story: a run that
  dies at item 700 restarts and skips the 700 files already on disk. No
  database, no checkpoints.
- **Terminal-per-bucket workers** instead of one big async program. Partition
  the to-do list into N buckets; open N terminals; each runs its bucket
  sequentially. Simple, observable (you watch N logs), trivially resumable, and
  it sidesteps async-framework complexity. Concurrency = number of terminals.
- **A hard per-item timeout** that kills the *whole process tree* on expiry. The
  CLI spawns child processes that escape a naive kill of the parent; the
  example ships a tiny `start_new_session` + `killpg` wrapper for exactly this.
  Without it, one stalled item hangs a worker forever.
- **Rate-limit back-off**: detect the limit message, sleep until the window
  resets, resume. With per-item files this is free.

### 5. Aggregate & integrate

Collect the per-item files, **validate each against the schema** (retry or quarantine
the malformed ones), then do whatever your use case needs: dedup near-duplicates,
cross-reference items against each other, and render to a site / report /
dataset. Keep this stage **pure and cheap** (no agent calls) so you can re-run it
freely as you tweak presentation.

---

## Design decisions that generalize

These are the choices that made the run cheap, finishable, and trustworthy —
the transferable lessons:

| Decision | Why it matters elsewhere |
|---|---|
| **CLI on a flat-rate plan, not the metered API** | Predictable cost for a long batch run; no per-token meter anxiety while you iterate the prompt. The orchestrator just shells out per item. |
| **One JSON file per item** | Idempotent + resumable + `git diff`-able for free. The cache *is* the progress tracker. |
| **Restrict tools** (`--allowed-tools "WebSearch WebFetch"`) | The agent can't write files, run shell, or wander. Smaller blast radius, more reproducible. |
| **Schema as the contract** | Everything downstream depends on shape, not prose. Validate it; the model retries on mismatch. |
| **Verify before cite** | Require a `verified_urls` list; reject any citation not in it. Mechanically eliminates fabricated references — the #1 failure mode of survey tasks. |
| **Terminal-per-bucket over in-process concurrency** | Observable, dead-simple, resumable. You scale by opening more terminals, not by debugging an event loop. |
| **Process-group hard timeout** | One stalled call must not wedge a worker. Kill the whole tree, mark the item failed, move on. |
| **An explicit "unclear" verdict + "don't guess" rule** | Lets the agent be *correctly* silent. Without it, models fabricate to appear useful. |
| **Cheap model by default, escalate selectively** | Run the bulk on the cost-effective model (e.g. Sonnet); re-run only the low-confidence / unclear minority on a stronger one (e.g. Opus). |

---

## Adapting it to your own use case — a checklist

1. **Name the item and the question.** "For each ⟨X⟩, decide ⟨Y⟩ with cited
   evidence." If you can't fill that blank in one sentence, stop here.
2. **Produce `corpus.json`** — one record per item, with an anchor date and any
   seed references. (Generating this list may itself be an agent pass.)
3. **Write the schema** — closed verdict set + an explicit "unclear" + a
   `verified_sources` list. This is your interface; freeze it early.
4. **Write one system prompt** — role, method, tool cap, parallelism, the
   don't-guess rule, schema inline. Reuse it verbatim for cache hits.
5. **Pilot on ~10 hand-picked items** — include a couple of *known* answers and
   one adversarial item. Read every output by hand; iterate the prompt until the
   tone and schema are right *before* spending the full run.
6. **Partition → run N terminal workers → one file per item.** Add the
   process-group timeout and rate-limit back-off.
7. **Validate, then spot-check ~10%** of cited sources by hand. If the
   bad-citation rate is non-trivial, tighten the verify-before-cite rule.
8. **Aggregate and integrate** — keep it agent-free so it's cheap to re-run.

---

## The worked example

[`graph-conjectures`](https://github.com/mlelarge/graph-conjectures) applies
exactly this to ~989 graph-theory conjectures from two corpora (a crawled
open-problem database + conjectures mined from recent arXiv papers), producing a
browseable, status-annotated site. Files worth reading there if you want the
concrete version:

- `LIT_REVIEW.md` / `PLAN.md` — the design rationale (schema, model, rate
  limits, quality safeguards).
- `scraper/*_system_prompt.md` — the actual reviewer/extractor prompts.
- `scraper/review.py` — a single-process orchestrator (schema validation,
  process-group timeout, rate-limit resume).
- `scripts/partition.py`, `scripts/run_worker.sh`, `scripts/timeout_claude.py` —
  the terminal-per-bucket worker harness and the hard-timeout wrapper.

---

## Caveats

- **The output is advisory.** An agent literature survey is a strong first pass
  and a finding aid — *not* a referee. Spot-check before relying on any single
  verdict for a research decision.
- **Coverage is bounded by search.** "No progress found" means *the agent's
  searches found none*, not that none exists. The verdict encodes that honestly
  (low confidence), but don't read absence of evidence as evidence of absence.
- **Reproducibility is approximate.** Web results and model outputs drift over
  time; record the model, date, and the exact queries in each output so a re-run
  is comparable, not identical.

---

# Attack one conjecture (depth-first)

A small **autonomous multi-agent engine** that attacks a *single* open conjecture
without babysitting. It encodes the loop a strong mathematician runs in a long
chain of thought — *propose an idea, ground it, ruthlessly try to kill it, keep the
thread, pivot when dry* — as a deterministic agent workflow, so both the
*mathematical* direction and the *operational* monitoring move off the human. It
runs round after round until it either makes progress or hits a wall it can name,
then hands that wall back to you.

> Like the survey, the point is the **pattern**, not the data. The worked example
> attacks graph / tournament conjectures, but nothing in the engine is
> domain-specific: all the mathematics lives in **two files you supply**.

---

## When it fits

Reach for it when **all** of these hold:

- You have **one** open problem you actually want to move (not a corpus to triage).
- The problem has a **finite, checkable core**: explicit objects (constructions,
  bounds, small cases) whose relevant invariants an **exact program** can compute.
- You want the machine to do the **cheap, mechanical, soul-crushing part** —
  enumerate, push a bound, refute a tempting-but-false lemma — and to **stop and
  ask** the moment only genuinely new mathematics would help.

If there is no computable oracle (the conjecture is purely about objects nothing
can enumerate or check), the engine has no ground truth — wrong tool.

---

## What you must provide — the ledger contract

The engine (`research-attack`) is **problem-agnostic**. To point it at a new
problem you supply exactly **two things** in the problem folder; no code changes.

**1. `ledger.json` — the single source of truth.** The engine's entire state,
rewritten every round and safe for a human to read or correct between runs:

| key | role |
|---|---|
| `central_question` | one-paragraph statement of what we're settling |
| `benchmark` | the fixed yardstick every proposal is scored against — **includes how to call the oracle** (`oracle_cmd` / `oracle_api`) and known `ground_truth_landmarks` |
| `proved[]` | claims that survived independent verification (never empirical-only), each tagged with its `claim_form` and stated **at the scope it was actually verified** |
| `open_crux` | the single load-bearing hole the engine drives at — rewritten as the frontier moves |
| `live_hypotheses[]` | candidate moves, each with a falsifiable prediction |
| `graveyard[]` | killed ideas + a one-line reason (never re-proposed) |
| `decision_log[]` | append-only, D-numbered history |
| `next_action`, `needs_human`, `recommend_handback` | control / hand-back signals |
| `discipline_gates` | the hard rules below |

**2. An oracle — sound, exact, computational ground truth.** A script (by
convention `<problem>/scripts/oracle.py` in a `<problem>/.venv`) that, given an
explicit finite object, returns the **exact** invariants the conjecture is about —
no heuristics. This is the engine's **anti-hallucination spine**: a proposal that
contradicts the oracle is killed instantly and cheaply.

---

## One round

```
startup gate ─▶ refuse to relaunch a ledger already handed back with nothing queued
   │            (clears if a concrete next_action is queued)
   ▼   each round, until a stop condition:
execute + propose ─▶ ground ─▶ skeptic ─▶ verify ─▶ synthesize ─▶ decide
(run the queued      (run the  (try to   (re-derive   (one lead     (continue /
 next_action; N       oracle,   refute,   from         rewrites      pivot /
 diverse-lens         kill      kill by   scratch;     ledger.json)  hand back)
 proposers)           fast)     default)  gate proof)
```

- **Startup gate.** Refuses to relaunch a ledger that already records a standing
  hand-back (`needs_human`, or a `recommend_handback` in its latest
  `decision_log`) with no concrete `next_action` queued — so a relaunch never
  burns a round re-deriving the same wall. The ledger is authoritative; a stale
  `docs/STATUS.md` mirror may hint but never blocks.
- **Execute + propose.** A first-class **executor** runs whatever concrete
  experiment the last round queued, so the engine's own priority never sits idle.
  In parallel, **N proposers** cycle four generic lenses —
  *explicit-construction, asymptotic-argument, literature-reduction, dual-attack* —
  each proposing one concrete, oracle-testable move that isn't already buried.
- **Ground → skeptic → verify** (a pipeline; each proposal flows independently):
  *ground* runs the oracle and reads the real output; *skeptic* tries to
  **refute, defaulting to killed**; *verify* independently re-derives the
  survivors and applies the proof gates.
- **Synthesize.** One **lead** rewrites `ledger.json`: promote / graveyard,
  rewrite the `open_crux` to the new frontier, append a decision, set the next
  `next_action` and any hand-back; it also mirrors a short human-readable
  `docs/STATUS.md`.
- **Decide / stop.** The engine hands back when synthesis sets
  `needs_human` / `recommend_handback`, after `dry_limit` rounds with no ledger
  change, or after `drought_limit` rounds with no *new* frontier result (only
  re-killing variants of dead families) — that last one is the "you need a
  genuinely new idea" signal.

---

## The discipline gates — why you can trust the output

The engine is built to **not fool itself**. Four hard gates, enforced by the
skeptic and the synthesis writer:

- **empirical ≠ proof.** Finite-`n` oracle evidence never promotes an asymptotic
  ("infinitely many …") claim to *proved* — that needs a symbolic argument.
- **universal needs a generic census.** A "for **all** `X` in class `C`" claim can
  **never** be supported by examples from a *structured* sub-family (circulants,
  the construction the conjecture came from): those are exactly where special
  identities hold — the worst possible sample. It must survive an **exhaustive
  census over generic** small members of the full class (`nauty gentourng` /
  all-objects). One counterexample kills it; structured confirmations prove
  nothing. *(This gate is a scar: a real run recorded `ω̄(S[H]) = ω̄(S)+ω̄(H)−1`
  as universal when it held only for circulant factors — it is false in general.
  Auto-scoping now records every claim at the scope it was actually checked.)*
- **quantifier-trap check.** An argument must not hand an *induced* sub-object its
  *optimal / min-over-orders* invariant when it really inherits an *arbitrary* one
  (a transitive triple has tournament-clique-number 1, yet its reverse order has a
  3-clique).
- **citations verified.** A cited theorem stays flagged needs-verification until
  checked.

A separate **scan discipline** keeps runs bounded and honest: every oracle
computation is a *single foreground* command wrapped in `timeout`; nothing is
backgrounded or split across turns, and a search too big to finish in budget is
reported **infeasible** rather than faked.

---

## Design decisions that generalize

| Decision | Why it matters |
|---|---|
| **State = one `ledger.json`, rewritten each round** | Resumable, inspectable, `git diff`-able. A human can correct the maths between launches; relaunching *continues the thread*, it doesn't restart. |
| **A sound oracle as ground truth** | Turns "please don't hallucinate" into a mechanical kill: contradict the oracle and you're dead. |
| **Kill-by-default skeptic + independent verify** | Two adversarial passes before anything is believed; survivors are re-derived from scratch, not trusted. |
| **Diverse lenses + a graveyard** | Forces breadth of attack each round while guaranteeing dead ideas are never re-proposed. |
| **Model split** | Cheap idea-generation proposers on **Fable 5**; the schema-heavy, load-bearing slots (executor, ground, skeptic, verify, synthesize) on **Opus**. |
| **Honest hand-back (`needs_human`, dry, drought)** | The engine quits *honestly* when the computational frontier is exhausted, instead of grinding out fake progress. |

---

## Launch

```
Workflow({ name: 'research-attack',
           args: { problem_dir: '<abs path to problems/X>', rounds: 4, proposers: 4 } })
```

`args`: `problem_dir` (required, absolute), `rounds` (default **1** — pass e.g.
`rounds: 4` to make real progress), `proposers` (default 3), `dry_limit`
(default 2), `drought_limit` (default 3). All state lives in
`<problem_dir>/ledger.json`, so a run is resumable and you can inspect or correct
the ledger between launches.

---

## The worked example

[`graph-conjectures`](https://github.com/mlelarge/graph-conjectures) ships the
engine and several attacked problems. Worth reading there:

- `engine/README.md` and `engine/LEDGER_CONTRACT.md` — the engine and the
  two-file contract above.
- `.claude/workflows/research-attack.js` — the actual workflow (tracked in-repo so
  a fresh clone can launch it by name).
- `problems/<name>/ledger.json` + `problems/<name>/scripts/oracle.py` — a real
  ledger and its oracle.

The headline run is the **tournament clique-number conjecture** (Aboulker–Aubian–
Charbit–Lopes, [arXiv:2310.04265](https://arxiv.org/abs/2310.04265), Conjecture
5.10): the engine reproduced the constructions behind the `k = 3, 4, 5` cases of
the existence of `k`-ω̄-critical tournaments and — just as usefully — *named the
wall* at `k ≥ 6`, where naive iterated substitution provably fails and a new
construction is needed.

---

## Caveats

- **It assists; it does not referee.** Every "proved" entry is engine-internal
  until a human checks the argument. The gates make fabrication hard, not
  impossible.
- **No oracle, no engine.** The whole guarantee rests on an exact, *sound* oracle.
  A buggy or heuristic oracle silently poisons everything downstream.
- **Hand-backs are the goal, not a failure.** The valuable output is often
  *"here is exactly where it's stuck, and why"* — not a finished proof.
