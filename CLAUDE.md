# kids-clock

A fullscreen colour clock for kids: the whole screen shows one colour, and the
colour changes at configured times of day.

## Domain language

[CONTEXT.md](./CONTEXT.md) is the glossary. Read it before naming anything —
in particular, a **Abschnitt** begins a stretch of the day, it is not an event,
and "Zeitpunkt" is deliberately retired.

## Issue tracker

This repo uses a **local markdown issue tracker**. See
[.agents/issue-tracker.md](./.agents/issue-tracker.md) for the conventions —
including the "Wayfinding operations" section that `/wayfinder` needs.

Planning artifacts live in `.scratch/`. The v1 plan is
[.scratch/v1-farbuhr/map.md](./.scratch/v1-farbuhr/map.md) — start there.

## Skills

The planning and implementation skills this repo works with are vendored into
[.claude/skills/](./.claude/skills/) so no fetch or plugin install is needed.
See [.claude/skills/README.md](./.claude/skills/README.md) for what each one is
for. `/wayfinder` is the entry point while the map is still open; once it
clears, the chain continues `/to-spec` → `/to-tickets` → `/implement`.
