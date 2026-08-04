---
title: Reputation display & moderation-priority mechanics
status: open
type: wayfinder:grilling
assignee: unassigned
blocked_by: []
---

## Question

Reputation must be "visible in the catalog" and "a priority in moderation" — both
confirmed, mechanics not yet defined:

- Display: shown as a raw numeric score, a tier/badge (e.g. bronze/silver/gold),
  or both? Shown where exactly — on the Contributor's tracks in catalog listings,
  on a profile, both?
- Moderation priority: does higher Reputation change the review-queue *order*
  (reordering the existing FIFO queue ADR A06 already defines), skip review
  entirely past some threshold, or just surface as a visible flag/hint to the
  Moderator without changing queue order or decisions? ADR A06 is explicit that
  duration-comparison flags stay "informational... without automatic moderation
  decision or changing submitted_for_review_at/FIFO position" — decide whether
  Reputation priority follows that same informational-only pattern or actually
  changes queue order/FIFO position.
