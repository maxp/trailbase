---
title: Reputation decay formula
status: closed
type: wayfinder:grilling
assignee: maxp
blocked_by: []
---

## Question

Reputation "decreases logarithmically over time" — resolve the exact shape:

- Decays relative to what: time since the account's last qualifying activity, or
  continuous decay of the whole score regardless of activity?
- What does the logarithmic curve mean concretely here — decay amount per elapsed
  interval shrinks over time (fast at first, slow later), or the *score* approaches
  some floor logarithmically? Needs a concrete formula, not just the word
  "logarithmic."
- Is there a floor (can Reputation go to zero, negative, or stop at some minimum
  above zero)?
- Does decay pause while the account is actively using the service, or run
  continuously regardless?

## Answer

- **Relative to what**: inactivity-based. Decay only runs once an Account stops
  earning the daily-use tick (see "Reputation accrual formula"); it pauses
  entirely for as long as the Account keeps having qualifying activity, and
  resets (`t` back to 0, reference score reset to whatever Reputation was at that
  moment) each time inactivity resumes after a run of activity.
- **Formula**: linear, not logarithmic —
  `R(t_weeks) = max(floor, R_start − w · t_weeks)`, where `t_weeks` = whole weeks
  of continuous inactivity, `w` = a small tunable weekly decrement (same kind of
  tunable constant as the accrual amounts), `R_start` = the Reputation value at
  the moment the current inactivity streak began.
- **Floor**: not a flat 0 — grows with diminishing returns from lifetime
  publication history: `floor(Account) = c · ln(1 + published_track_count)`. A
  logarithmic curve is the right shape here (unlike the decay-over-time curve,
  which was deliberately kept linear): the input only grows across a whole
  contributor career, so slow, unbounded, ever-flattening growth is intentional
  rather than something to cap.
