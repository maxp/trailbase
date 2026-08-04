---
title: Stamina pool, spend & regen mechanics
status: open
type: wayfinder:grilling
assignee: unassigned
blocked_by: []
---

## Question

Stamina is spent on uploading a track, regenerates over time, and its regen rate
depends on Reputation. Resolve:

- Initial and maximum pool size for a new Account.
- Cost per upload: flat, or variable (e.g. by file size, track length, or
  something else)?
- Regen formula: rate per unit time, and precisely how Reputation scales it
  (linear? tiered thresholds? a multiplier?).
- Behavior at zero Stamina: is the upload hard-blocked until regen/purchase, does
  it queue, or something else?
- Does Stamina apply to any action besides track upload (the concept dialogue only
  named upload), or is upload the sole sink for now?
- Confirm this lives on the Account, same as Reputation (ticket "Reputation
  accrual formula" makes the same assumption — keep them consistent).
