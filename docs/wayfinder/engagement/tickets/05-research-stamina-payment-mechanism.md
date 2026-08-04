---
title: "Research: payment mechanisms for topping up Stamina"
status: closed
type: wayfinder:research
assignee: unassigned
blocked_by: []
context_pointer: "docs/wayfinder/engagement/research/stamina-payment-mechanism.md"
---

## Question

Stamina can be topped up with real money when it runs low. TrailBase is bot-first
(Telegram and Max are the only identity providers, per ADR A01) and appears to run
on Russian-hosted infrastructure (gitflic.ru/gitverse.ru/sourcecraft.dev remotes),
which constrains which payment rails are actually usable. Research, against
primary sources:

- Telegram's native Payments API (Bot Payments / Telegram Stars) — feasibility,
  fees, currency/settlement constraints, and whether it works for a bot-first flow
  without requiring a web checkout.
- Max messenger's equivalent payment capability, if any (Max is a Russian
  messenger — check whether it exposes a bot payments API at all).
- Standalone payment gateway options viable for a Russian-hosted open-source
  project (e.g. YooKassa, CloudPayments, or others) as a fallback if messenger-
  native payments don't cover the need (e.g. web-session users without a bot
  context).
- High-level compliance/tax surface for taking real-money payments as an
  open-source catalog project (not a full legal opinion — just what obligations
  typically attach, so the pricing/policy ticket can scope around them).

Findings feed directly into "Stamina monetization — pricing & policy," which is
blocked on this ticket.

## Answer

Full findings: [research/stamina-payment-mechanism.md](../research/stamina-payment-mechanism.md).

- **Telegram** exposes two separate rails: the classic Bot Payments API
  (provider-pluggable, no Telegram commission) and Telegram Stars (XTR,
  mandatory for "digital goods," settles only via Fragment→TON crypto
  conversion). **YooKassa is a selectable BotFather provider**, open to
  self-employed/ИП/legal entities, settling in RUB with no sanctions exposure —
  contradicting the common "Telegram payments require Stripe" assumption
  (Stripe explicitly excludes Russia). Stars work fully in-chat too, but shift
  the settlement/tax posture to crypto disposal — a materially different
  commitment.
- **Max messenger's Bot API has no payment/invoice/wallet endpoints at all** —
  any Max-originated top-up needs a standalone gateway with a web checkout.
- **Standalone gateways**: YooKassa (best-evidenced, self-employed-friendly,
  can share one merchant contract with the Telegram integration) is the
  strongest fallback; CloudPayments works but reportedly favors ИП/ООО over
  self-employed onboarding.
- **Compliance**: NPD (self-employed regime) is the lightest-weight fit for an
  individual maintainer — 4%/6% tax, no separate cash-register purchase, capped
  at 2.4M RUB/year — and pairs with an aggregator that auto-generates fiscal
  receipts. The Stars/TON path instead triggers Russia's crypto-disposal NDFL
  regime (13%/15%, self-reported).
- Five gaps flagged as not fully closed against a primary source (see the
  findings file's "Gaps / follow-ups" section) — notably whether Telegram would
  actually classify a Stamina top-up as a mandatory-Stars "digital good," and
  Stars withdrawal timing/KYC specifics. The pricing/policy ticket should do a
  cheap confirmation pass on these before committing to a rail.
