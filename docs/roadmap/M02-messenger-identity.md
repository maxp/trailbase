# M02 — Messenger Identity + Optional Web Session

## Goal

Make Telegram and Max identities usable for private chat operations and support the
optional browser-session flow without introducing email or phone identities.

## Acceptance

- A validated private-chat event creates or resumes the correct account and identity.
- The optional browser flow creates a protected session.
- Linking, unlinking, notifications and delivery-health behavior follow the contract.

## Contract

- [Identities и account creation](../contract/auth/identity.md)
- [Browser sessions, linking и delivery](../contract/auth/browser-delivery.md)
- [Valkey sessions и account lifecycle](../contract/auth/sessions-lifecycle.md)
- [Webhooks, bot workers и notifications](../contract/bots.md)
