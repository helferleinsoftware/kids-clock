# Vendored skills

These skills are vendored from **[mattpocock/skills](https://github.com/mattpocock/skills)**
so that any agent working in this repo has them without a network fetch or a
plugin install.

- Source revision: `8b78b531ab965735c5dc74f6f7a219e1e37326df` (2026-08-13)
- Licence: MIT, © 2026 Matt Pocock — see [LICENSE](./LICENSE)

Only the `.md` files were copied. The `agents/openai.yaml` files that ship
alongside each skill upstream are for other harnesses and were left out.

## What is here and why

The set is the **wayfinder chain** — the planning flow this repo's v1 was
charted with, plus everything downstream of a cleared map:

| Skill | Role in this repo |
| --- | --- |
| `wayfinder` | Charts and works the map in `.scratch/v1-farbuhr/`. The entry point. |
| `grilling` | Resolves the HITL decision tickets. Interview in rounds, never answer for the human. |
| `domain-modeling` | Keeps [CONTEXT.md](../../CONTEXT.md) honest. Consult it before inventing a term. |
| `research` | Runs the AFK research tickets as a subagent. |
| `prototype` | Resolves the prototype tickets (06, 07) — cheap throwaway artifacts to react to. |
| `handoff` | Bridge into or out of a map when a session outgrows itself. |
| `to-spec` | **Next after the map clears.** Collapses the linked decisions into one spec. |
| `to-tickets` | Slices that spec into implementation tickets. |
| `implement` | Builds a ticket. |
| `tdd` | Ticket 02 delivered 31 test cases for the pure time function — this is how they get written. |
| `codebase-design` | Vocabulary reference that `tdd` points at. Not a session to run. |

## Two things worth knowing before using these

**Wayfinder plans, it does not build.** Every ticket holds a question whose
resolution is a decision, not a slice of the build. This is the rule agents
break most often. The map's `## Notes` block can override it — so read the
Notes on a map you did not chart yourself, and treat any `task` ticket that
looks like a slice of the build as mis-typed.

**The tracker is local markdown, not GitHub Issues.** Upstream these skills
assume a real tracker with native blocking. The conventions this repo uses
instead are in [.agents/issue-tracker.md](../../.agents/issue-tracker.md);
its "Wayfinding operations" section is what `wayfinder` reads. Skills that say
"run `/setup-matt-pocock-skills`" have already had that done — that setup skill
is deliberately not vendored, since re-running it would only re-derive files
that already exist.
