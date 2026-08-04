---
title: "Amend ADR A06 difficulty model: add optional simplified difficulty facet"
status: open
type: wayfinder:grilling
assignee: unassigned
blocked_by: []
---

## Question

Per the product-concept dialogue, difficulty becomes two independent, both-optional
layers instead of the single per-activity-standard scale ADR A06 currently defines:

1. A simplified 4-level facet: легкий / нормальный / трудный / экстремальный
   (easy / normal / hard / extreme).
2. The existing per-activity standard (SAC Scale T1–T6 for hike, MTB-Trail-Difficulty
   S0–S5 for bike, 3-level `easy/moderate/hard` fallback otherwise).

Both are optional (nullable), per the answer already given. Still open:

- Is the simplified facet set manually only, or auto-suggested at upload time like
  `activity_type` currently is (see ADR A06 point 3)?
- Does the simplified facet apply uniformly across *all* activity types — including
  `water` and `motor`, which today have no per-activity standard at all — making it
  their only difficulty signal, or does the fallback 3-level scale still apply to
  them alongside it?
- How does this interact with the existing "difficulty allows unknown" rule (see the
  2026-07-29 clarification in ADR A06)?
- Does catalog search/filtering use the simplified facet, the per-activity standard,
  or both, and if both, how do they combine in a single filter?

Resolve, then add a dated "Уточнение" entry to `docs/adr/A06-classification.md` and
update `docs/contract/classification.md` to match, in the same session.
