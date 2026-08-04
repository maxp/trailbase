---
title: "Comment feature: storage, moderation & display mechanics"
status: open
type: wayfinder:grilling
assignee: unassigned
blocked_by: []
---

## Question

Resolving "Reputation accrual formula" introduced a new domain concept: comments
on a published track, used as the vehicle for the confirmed-useful-comment
Reputation trigger. That ticket settled the *confirmation/reward* mechanic only;
the comment feature itself still needs:

- **Moderation**: does a comment need moderator approval before becoming visible
  (matching TrailBase's moderation-heavy pattern for track content), or is it
  visible immediately, with moderation only reactive (e.g. on report)?
- **Storage/lifecycle**: can a comment be edited or deleted after posting? If
  deleted, does an already-paid-out Reputation reward (to the commenter or its
  confirmers) get clawed back, or stand?
- **Display**: where do comments appear — on the track page only, or elsewhere
  (Contributor profile, catalog listings)?
- **Who can comment**: any authenticated Account (Contributor or Seeker), or
  restricted somehow (e.g. must have interacted with the track first)?
- **Scope boundary**: is this a v1 feature launched alongside Reputation, or can
  Reputation ship first with comments as a fast-follow — given comments are net-new
  domain surface, not a detail of something already built.

**Cross-reference**: [TrailBase product concept — personas & use
cases](../../product-concept/MAP.md)'s ticket "Confirm remaining persona use
cases" was already asking whether social/feedback features exist at all — this
ticket confirms comments do exist (at least for Reputation's purposes), so
resolve that sibling ticket with this in mind rather than independently.
