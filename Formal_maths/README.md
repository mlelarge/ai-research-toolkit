# Formal Maths — Proofs an Agent Can't Fake

Using an LLM agent to produce **mathematics verified by a proof kernel**. The
agent writes [Rocq](https://rocq-prover.org/) (formerly Coq); the Rocq kernel
checks it. If the proof type-checks and uses no axioms, it is *correct* — full
stop. There is nothing to spot-check and nothing to hallucinate.

> This is the mirror image of [`Informal_maths/`](../Informal_maths/). There the
> "oracle" is web search plus human review, and the output is *advisory*. Here
> the oracle is a **mechanical checker that cannot be fooled**, and the output is
> *certain*. The transferable idea is the same in both: **don't trust the
> model's word — wire its output to something that can reject it.** Whenever your
> task has a ground-truth checker (a kernel, a compiler, a test suite, a
> simulator), put it in the loop and let the agent iterate against it.

---

## The pattern

```
  goal (a statement to prove)
        │
        ▼
  ┌──────────────┐   agent drafts a proof, then asks the checker:
  │  agent       │◀─┐  "what's the current proof state? does this step close it?"
  │              │
  └──────────────┘  │  live proof state, errors, available lemmas
        │           │
        ▼           │
  ┌──────────────┐  │
  │  Rocq kernel │──┘   type-checks the proof; rejects anything unsound
  └──────────────┘
        │
        ▼
  verified theorem  (axiom-free `Definition … : Prop` + a proof that compiles)
```

The agent and the kernel form a **closed feedback loop**: every step the model
proposes is immediately accepted or rejected with a precise error, so the model
repairs its own mistakes instead of confidently emitting a wrong proof. The
loop is the whole point.

Two things make it work:

- **The checker is reachable from inside the agent.** An MCP server exposes live
  proof state, goal inspection, library search, and axiom checking *as tools the
  agent can call mid-proof* — not a batch compile at the end.
- **The agent has the right conventions loaded.** Skills bundle a
  prove/review/golf workflow and a library style guide, so the agent writes
  *idiomatic* proofs that fit the existing development, not just compiling ones.


---

## Reusable lessons


- **Expose the checker as live tools (MCP), not a final gate.** Mid-proof state
  is what lets the agent self-correct; a pass/fail at the end does not.
- **Encode house style as a skill.** Compiling ≠ idiomatic. Conventions loaded
  up front save a long tail of review churn.

---

## Where the material lives

The tooling is developed in the open under
[**LLM4Rocq**](https://github.com/LLM4Rocq):

- [`rocq-mcp`](https://github.com/LLM4Rocq/rocq-mcp) — MCP server giving the
  agent live proof state, goal/notation inspection, axiom checking, compilation.
- [`rocq-skills`](https://github.com/LLM4Rocq/rocq-skills) — host-agnostic
  prove / review / golf workflows, library search, safety guardrails.
- [`mathcomp-rocq-guide`](https://github.com/LLM4Rocq/mathcomp-rocq-guide) — a
  style-guide skill for MathComp / MathComp-Analysis.

