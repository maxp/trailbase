# ADR A01 — Bot-first Auth Model

**Status:** Accepted
**Date:** 2026-07-25

## Context

TrailBase serves people through web and Telegram/Max chats. Requiring a browser
activation before useful chat operations would make the primary interaction harder
without adding an identity signal.

## Decision

- Telegram and Max are the only MVP identity providers; email, phone and passwords
  are not identity channels.
- A validated private-chat event is sufficient for chat operations and may create the
  account and identity.
- Browser access is optional and uses a separate protected session flow.
- Bot and web adapters call shared domain services rather than duplicating business
  logic.
- Losing every linked messenger identity has no self-service recovery path in the MVP.

The normative rules are in:

- [Identities and account creation](../contract/auth/identity.md)
- [Browser sessions, linking and delivery](../contract/auth/browser-delivery.md)
- [Valkey sessions and account lifecycle](../contract/auth/sessions-lifecycle.md)
- [Webhooks, bot workers and notifications](../contract/bots.md)

## Alternatives considered

- Email or phone login/recovery — rejected because identity providers are deliberately
  limited to Telegram and Max.
- Mandatory web activation — rejected because the messenger already authenticates the
  private-chat operation.
- Bot-only application — rejected because web remains a first-class catalog and map UI.
- Separate bot business logic — rejected because permissions and domain behavior would
  drift.

## Consequences

- Chat workflows must handle multi-step interaction state.
- Browser sessions remain optional and CSRF-protected.
- There is no email/phone PII or a separate account-recovery channel.
