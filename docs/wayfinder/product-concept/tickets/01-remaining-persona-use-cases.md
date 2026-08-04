---
title: Confirm remaining persona use cases
status: open
type: wayfinder:grilling
assignee: unassigned
blocked_by: []
---

## Question

The draft use case list covers, per persona:

- **Contributor**: upload via bot/web, get parse/validation feedback, track
  submission status + notifications, view/manage own tracks and revisions.
- **Seeker**: browse/search the public catalog, view a track with POIs and
  elevation, discover POIs independent of a track, export public GPX, search via
  bot chat.
- **Moderator**: review pending revisions, handle integrity findings (track
  issues), handle removal appeals.

Are there use cases missing in these areas, and if so what are they:

- **Identity/account management** — linking/unlinking Telegram and Max identities,
  activating an optional browser session (per `docs/adr/A01-bot-first-auth.md` and
  `docs/contract/auth/`).
- **POI curation** — beyond the existing "contributor sees auto-detected POI
  read-only, can request a new one" flow in `docs/adr/A04-poi-gazetteer.md`, is
  there a distinct curation use case (for a Moderator, or a separate persona)?
- **Social/engagement features** — favorites, comments, following a contributor,
  or anything similar. If none of these exist, that itself is worth stating
  explicitly in the product concept (what TrailBase deliberately is *not*).

Resolve to a final, confirmed per-persona use case list.

**Cross-reference**: [TrailBase user engagement & motivation
system](../../engagement/MAP.md)'s ticket "Reputation accrual formula" (closed)
has already settled that **comments on tracks do exist**, at least as the vehicle
for a confirmed-useful-comment Reputation trigger (author or ≥3 reputable Accounts
confirm a comment as useful). Take that as given when resolving the
Social/engagement bullet above, rather than re-deciding whether comments exist —
this ticket should instead cover *use cases* (who leaves/reads comments, why),
while comment storage/moderation/display mechanics belong to the engagement map's
"Comment feature: storage, moderation & display mechanics" ticket.
