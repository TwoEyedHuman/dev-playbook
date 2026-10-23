# Lessons Learned

A living document. Append after every project or epic. Periodically rasterize into skeleton READMEs.

---

## Format

Each entry has:
- **Project/Context** — what you were building
- **Lesson** — what went wrong or what worked well
- **Pattern** — the durable rule to bake into future skeletons

---

2026-06-20 — revenant-card-game (Story 9.3 implementation)

Lesson: each implementation session burns a lot of time/context re-discovering
the existing codebase (component tree, hooks, animation conventions) from
scratch, because the story README only documents WHAT to build, not HOW the
current code is structured. A research pass spent ~20 tool calls just mapping
the animation system before any code was written.

Template fix: add a setup step (once per repo, not per story) to maintain a
living architecture doc (CLAUDE.md) covering: component/file map, state shape,
established patterns (e.g. "animation = CSS keyframe + diff-hook, gated by
prefers-reduced-motion in both JS and CSS"), and tooling gotchas (lint rules
that reject obvious code). Story template should say: read it before starting,
append one short paragraph to it after finishing. Turns "re-derive everything"
into "read one file, append one paragraph."

Secondary lesson: story numbering tied to an epic prefix (9.x = "Motion &
Feedback") gets awkward when a new story doesn't fit the epic's theme — a
layout/viewport-fit story got forced in as 9.4 just to keep numbering
sequential. Template should call out the choice explicitly: open a new epic
when the theme shifts, or accept numbering-continuity over thematic purity —
either is fine, but it should be a decision, not a default.