# Teaching — Exposing a Course to AI Agents

Take an existing course — lecture notes, notebooks, slides — and make it
**queryable by AI agents** (and by the students who use them), through three
access surfaces over the *same* content:

- **CLI / skill** — a local command an agent invokes to look things up in the
  course while working alongside a student.
- **REST API** — programmatic access for notebooks, autograders, or a chatbot
  front-end.
- **MCP server** — agentic access: the course becomes a set of typed tools an
  assistant can call mid-conversation ("find the lecture on X", "give the
  notebook for Y").

The worked instance is `dataflowr-tools`, built on the
[dataflowr](https://dataflowr.github.io/website/) deep-learning course.

---

## The pattern

```
                    ┌──────────── CLI / skill   (agent at the student's side)
  course content ───┼──────────── REST API      (notebooks, autograder, chatbot)
  (notes, nbs)      └──────────── MCP server     (assistant calls it as tools)
        │
        └─ one indexed source of truth, three surfaces
```

The lesson is **one source, many surfaces**. The course content is indexed once;
each surface is a thin adapter over the same index. You don't fork the material
per channel — you add an adapter when you need a new way in.

Why three? Because the consumers differ:

- a **student's agent** wants a *skill* it can call locally without network setup;
- a **notebook or autograder** wants a stable *REST* endpoint;
- a **conversational assistant** wants *MCP* tools it can discover and invoke.

Picking the right surface per consumer is the whole design decision.

---

## Reusable lessons

- **Index once, adapt per surface.** The expensive part is structuring the
  content; the surfaces are cheap shells over it. Don't duplicate content per
  channel.
- **MCP turns "course material" into "agent capabilities."** Once the course is
  a set of tools, any assistant can teach *from* it without you building a bot.
- **REST keeps the door open for non-agent consumers** — autograders, dashboards,
  LMS integrations — that won't speak MCP.
- **A CLI/skill is the lowest-friction entry point** for a student already
  working with a coding agent: no server to run, no key to manage.


