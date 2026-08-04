---
title: "Amend ADR A06 activity_type model: moderated growable dictionary"
status: open
type: wayfinder:grilling
assignee: unassigned
blocked_by: []
---

## Question

Per the product-concept dialogue, `activity_type` stops being the closed enum ADR
A06 currently defines (`{hike, bike, run, ski, water, horse, motor, other}`) and
becomes extensible during data population — the answer given was "виды активности
должны быть дополняемыми в процессе наполнения базы." This also surfaced a concrete
new value, Коньки (skating), not present in the current enum.

Still open:

- Does `activity_type` become a moderated, growable dictionary using the *same*
  mechanism ADR A06 already defines for secondary tags (user submits a request →
  moderator approves/rejects → value appears in the dictionary), or a different
  mechanism?
- Can a contributor propose a new activity type via that request flow, or is adding
  one moderator-only?
- Does migration seed the dictionary from the current 8 enum values as-is, or are
  any of them (e.g. `run`, `horse`) reconsidered/dropped — the product-concept
  dialogue's example list omitted both, but that may have been informal shorthand
  rather than a decision to drop them (see the answer on the map's ticket about
  movement modes).
- Does the per-activity difficulty-standard lookup (see the difficulty-facet
  ticket) extend automatically to new dictionary entries, or do new entries always
  get the 3-level fallback until a standard is explicitly assigned?

Resolve, then add a dated "Уточнение" entry to `docs/adr/A06-classification.md` and
update `docs/contract/classification.md`, in the same session.
