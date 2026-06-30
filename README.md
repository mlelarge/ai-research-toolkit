# AI Research Toolkit

A collection of prompts, workflows, and pointers for using AI assistants (LLMs and
agents) in research. 

## Introduction


A field report on using AI for everyday mathematical research on a modest budget: [**"Vibe Proving on a €200 Budget"**](https://docs.google.com/presentation/d/1jdWGvQrfkBJCZc4AVt9WcY4XVWYSU13Slf06xVwKoiU/edit?usp=sharing) — Marc Lelarge (INRIA – ENS), 29/06/2026. 

## Use cases

| Folder | Use case | What's here |
| --- | --- | --- |
| [`Teaching/`](Teaching/) | Teaching | Expose a course (notes, notebooks, slides) to agents and students through three surfaces over the *same* content — CLI / skill, REST API, MCP server (`dataflowr-tools`). |
| [`Review/`](Review/) | Refereeing papers | [`harsh_review.md`](Review/harsh_review.md) — a prompt for an uncompromising, no-flattery referee report, plus the review workflow. |
| [`Informal_maths/`](Informal_maths/) | Informal mathematics | Two agent pipelines: a breadth-first **survey** of a whole corpus (status of ~1,000 conjectures) and a depth-first **engine** that attacks a single open conjecture against an oracle (see [graph-conjectures](https://github.com/mlelarge/graph-conjectures)). |
| [`Formal_maths/`](Formal_maths/) | Formal mathematics | Proofs an agent can't fake: it writes [Rocq](https://rocq-prover.org/), the kernel checks it. Uses a dedicated MCP server for live proof interaction (see [LLM4Rocq](https://github.com/LLM4Rocq/)). |
| [`Coding/`](Coding/) | Coding | The **shared toolbox** the other folders build on — the Claude Code primitives: permissions, MCP, Skills, Hooks, subagents, dynamic workflows, `claude -p`, and Docker / Apptainer containers. |

The recurring idea across all of them: **don't trust the model's word — wire its
output to something that can reject it** (a proof kernel, a compiler, verified
citations, an exact oracle).

> Each folder's README explains the reusable *pattern*; the runnable code lives in
> the linked repositories. This is a living collection, still being filled in.


## Feedback

Anonymous feedback (shadow AI welcome): <https://www.admonymous.co/marc-lelarge>


## License

[Apache License 2.0](LICENSE).
