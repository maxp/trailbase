# Research: payment mechanisms for topping up Stamina

Resolves ticket `05-research-stamina-payment-mechanism.md`. Findings only — no
recommendation or decision is made here; that belongs to the blocked ticket
"Stamina monetization — pricing & policy."

Research date: 2026-08-04.

---

## 1. Telegram — Bot Payments API and Telegram Stars

Telegram actually exposes **two separate payment mechanisms** for bots, and
they behave very differently for a Russian-hosted project. Both stay fully
in-chat: the bot sends an invoice message, the user taps Pay, a native
Telegram payment sheet opens, and the result comes back to the bot as a
`successful_payment` update. Neither requires a browser/web checkout step
from the user's point of view ([core.telegram.org/bots/payments](https://core.telegram.org/bots/payments)).

### 1a. Classic Bot Payments API (real-currency, third-party provider)

- Telegram itself charges **no commission**: "Telegram does not charge any
  commission for using the Payments API." The connected payment provider
  charges its own fee — the docs give Stripe's 2.9% + 30¢ as an example
  ([core.telegram.org/bots/payments](https://core.telegram.org/bots/payments)).
- The provider is chosen once, in BotFather (`/mybots` → the bot →
  **Payments**), and issues a `provider_token` used in `sendInvoice`. Flow
  stays entirely inside Telegram's UI — the "Pay" button opens Telegram's own
  payment sheet, not an external browser (confirmed by both the official docs
  and a working implementation walkthrough at
  [habr.com/ru/articles/855824](https://habr.com/ru/articles/855824), which
  shows `bot.send_invoice(provider_token=...)` with the whole flow handled via
  `PreCheckoutQuery` / `successful_payment` callbacks, no redirect).
- **Critically for this project's Russian hosting: YooKassa (ЮKassa) is one
  of the selectable providers in BotFather's payments provider list**, and
  YooKassa's own marketing page confirms this integration exists and works
  fully in-chat ("нажимает «Заплатить» прямо в чате" — pays directly in
  chat) ([yookassa.ru/telegram](https://yookassa.ru/telegram/)). YooKassa
  explicitly serves **legal entities, individual entrepreneurs (ИП), and
  self-employed (самозанятые)**, with a commission "от 2,8%" (from 2.8%, ex
  VAT) across card/YooMoney/SberPay methods
  ([yookassa.ru/telegram](https://yookassa.ru/telegram/)). CloudPayments is
  also connectable as a Telegram bot payment method, though CloudPayments'
  own support surface indicates it targets ИП/ООО rather than self-employed
  by default (per secondary sources — not independently verified against a
  CloudPayments primary doc page).
- This means the widely-repeated assumption that "Telegram bot payments
  require Stripe, which won't onboard a Russian entity" is **not the full
  picture**: Stripe itself does exclude Russia — Stripe's own support page
  states Stripe does not operate in Russia, Ukraine, or Belarus and prohibits
  use of its products by parties located in Russia
  ([support.stripe.com/questions/sanctions-on-russia-and-belarus](https://support.stripe.com/questions/sanctions-on-russia-and-belarus)) —
  but Telegram's Bot Payments API is provider-pluggable, and a
  Russia-domestic provider (YooKassa, and likely CloudPayments/Robokassa/
  PayMaster) can be selected instead, settling in RUB to a Russian merchant
  account with no cross-border/sanctions exposure.
- **Caveat found but not fully closed**: Telegram's docs advertise "200+
  countries, 20+ providers" generically; there is no single official page
  enumerating which providers are live for which country in 2026. The
  YooKassa-as-Telegram-provider claim is corroborated by two independent
  primary-ish sources (YooKassa's own site, and a working code sample), which
  is good but not as authoritative as a Telegram-published provider list —
  worth a quick live BotFather check (`/mybots` → Payments) before the
  decision ticket commits to this path, since provider availability can
  change without a docs update.

### 1b. Telegram Stars (XTR) — digital goods only

- Stars are a separate, mandatory rail for **digital goods and services**:
  "Payments for digital goods and services must be carried out exclusively
  in Telegram Stars" — currency tag `XTR`
  ([core.telegram.org/bots/payments-stars](https://core.telegram.org/bots/payments-stars)).
  A Stamina top-up (virtual in-app currency spent inside the bot) plausibly
  counts as a "digital good/service" under this rule, which would make Stars
  **mandatory**, not optional, for that flow if Telegram enforces the
  distinction the way it does for other bots (this classification question is
  worth a direct confirmation with @BotSupport before relying on it).
- Users acquire Stars via Apple/Google IAP or `@PremiumBot`, then spend them
  in the bot — this leg is fully in-chat, no web checkout.
- **Withdrawal is the hard constraint.** There is no direct fiat payout.
  Developers withdraw Stars by converting them to **Toncoin (TON)** via
  **Fragment** (Telegram's affiliated marketplace), not to a bank account.
  Getting from TON to RUB in a Russian entity's bank account requires an
  additional crypto-to-fiat exchange leg, which:
  - requires **KYC at the exchange step** (self-custody wallet transfer from
    Fragment doesn't require KYC, but cashing out to fiat does) — this is
    corroborated by multiple secondary/community sources but the exact
    KYC/AML flow is not documented on a Telegram-owned primary page; I could
    not locate an official Telegram/Fragment page enumerating
    country-restricted withdrawal — treat this as **not fully verified**.
  - has a **minimum withdrawal threshold** (commonly cited as 1,000 Stars)
    and a **~21-day settlement delay** before Stars become withdrawable
    (secondary sources; not confirmed on a primary Telegram page — the
    official ToS/Stars pages fetched during this research
    ([telegram.org/tos/stars](https://telegram.org/tos/stars),
    [core.telegram.org/bots/payments-stars](https://core.telegram.org/bots/payments-stars))
    describe the existence of a developer balance but not the withdrawal
    mechanics or timing in the fetched content).
  - means the proceeds arrive as **crypto income**, which for a Russian
    resident (individual or entity) triggers Russia's crypto taxation regime
    (see §4) rather than ordinary payment-processor settlement — a materially
    different operational and tax posture than a RUB card payment.
- **Net assessment**: the classic Bot Payments API + a Russian provider
  (YooKassa/CloudPayments) is the more directly usable rail for a Russian
  entity/individual wanting predictable RUB settlement. Stars work fully
  in-chat but convert the settlement problem into "manage a TON wallet and
  cash out crypto," which is a materially different (and less proven, from
  primary sources) operational commitment than it first appears.

---

## 2. Max messenger — bot payments capability

**Max's Bot API has no payment, invoice, purchase, wallet, or billing
endpoints.** Reviewing the official developer documentation entry point
([dev.max.ru/docs-api](https://dev.max.ru/docs-api)), the documented API
surface covers only: bot info/commands, chat/group/channel operations,
webhook subscriptions, media upload, and messaging. There is no
`payments`/`invoice`/`purchase` section analogous to Telegram's.

Two secondary findings worth flagging (found via search, not confirmed
against a Max-owned primary source, so treat as leads rather than
established fact):
- A third-party product called "MAX PayBot" exists and reportedly handles
  in-messenger payments for some merchants, but this appears to be a
  **third-party bot product**, not a first-party Max Bot API payments
  capability — i.e. it's built the same way any bot would integrate an
  external payment gateway and message links/inline confirmations, not a
  `sendInvoice`-equivalent platform primitive.
- Since August 2025, only Russian legal entities may create and publish bots
  on Max at all (sole proprietors, self-employed, and private individuals are
  reportedly excluded) — this is a broader Max platform constraint (not
  specific to payments) that would need separate verification against a Max
  primary source before it affects any decision, since it also bears on
  whether TrailBase's own bot registration on Max is even viable, independent
  of Stamina.

**Conclusion: Max cannot support an in-chat Stamina top-up today.** Any
Max-originated top-up would have to fall back to a standalone gateway with a
web checkout (§3), breaking the "stays in-chat" property that Telegram's
classic Payments API offers.

---

## 3. Standalone payment gateways (fallback / web-session path)

For users without an active bot chat context (e.g. an optional web session,
per ADR A01), or as the only option for Max users, a standalone Russian
gateway is the fallback. Two were checked against primary docs:

- **YooKassa** ([yookassa.ru/developers/payment-acceptance](https://yookassa.ru/developers/payment-acceptance/getting-started/quick-start)):
  REST API, merchant credentials (`shopId` + secret key) issued after
  registering a "Merchant Profile." Two integration shapes: **redirect-based
  checkout** (server creates a payment, redirects the user to
  `confirmation_url` hosted by YooKassa) and a lower-level API/SDK path for
  custom UI. Same YooKassa account/contract used for the Telegram bot
  integration (§1a) can also serve the standalone web checkout — i.e. one
  merchant relationship can cover both surfaces. Explicitly open to legal
  entities, ИП, and self-employed.
- **CloudPayments** ([cloudpayments.ru](https://cloudpayments.ru/blog/kak-podklyuchit-cloudpayments-k-telegram-botu-/), secondary-corroborated):
  Also usable for both a Telegram bot and a standalone checkout; commission
  cited around 1–3.9% depending on turnover/MCC, individually negotiated.
  Per secondary sources, CloudPayments' default onboarding targets ИП/ООО
  rather than self-employed — worth confirming directly with CloudPayments if
  the eventual TrailBase legal vehicle is a self-employed individual rather
  than a registered business.
- Other Russian gateways surfaced in search but not independently verified
  against primary docs in this pass: Robokassa, PayMaster. Listed for
  completeness only — do not treat as vetted.

**Conclusion**: a standalone gateway (YooKassa is the best-evidenced option)
is a viable, low-risk fallback, and notably can share one merchant contract
with the Telegram in-chat integration, avoiding duplicated payment-provider
onboarding.

---

## 4. Compliance/tax surface (high-level, not a legal opinion)

This section is scoping information for the later pricing/policy ticket, not
a substitute for actual legal advice before launch.

- **Cash register law (54-FZ)**: Russian law requires use of a fiscal cash
  register (ККТ/онлайн-касса) for essentially all payments from individuals,
  including card and other electronic payments — this was extended to cover
  "any electronic means of payment," not just cards, per the 2016 amendment
  to 54-FZ (secondary legal-summary sources corroborate this; I could not
  pull the primary consultant.ru/pravo.gov.ru statute text directly in this
  pass — flagged as a gap). A payment aggregator like YooKassa/CloudPayments
  typically handles fiscal-receipt generation on the merchant's behalf as
  part of the integration, but the underlying obligation sits with the
  merchant of record.
- **Self-employed regime (NPD — Налог на профессиональный доход)** is the
  lightest-weight vehicle for an individual maintainer to legally receive
  payments in Russia, per the official FNS portal
  ([npd.nalog.ru](https://npd.nalog.ru/)):
  - Available to individuals/ИП with no employees, annual income capped at
    **2.4 million RUB**.
  - Tax rate **4%** on receipts from individuals, **6%** from legal entities.
  - **No separate ККТ purchase needed** — the "Мой налог" mobile app itself
    generates the fiscal receipt ("Не надо покупать ККТ. Чек можно
    сформировать в мобильном приложении"), which meaningfully lowers the
    operational bar versus a full ИП/ООО cash-register setup.
  - This regime is a plausible fit if the entity taking payment for Stamina
    top-ups is an individual (e.g. project maintainer) rather than a
    registered company — but note CloudPayments and possibly other gateways
    may not onboard self-employed merchants (see §3), which would push
    toward YooKassa specifically, or toward ИП/ООО registration, if that
    route is chosen.
  - Above the 2.4M RUB/year cap, NPD stops applying and a different regime
    (ИП on a simplified tax system, or a full company) is required.
- **Crypto income (relevant only if the Stars/TON path in §1b is used)**:
  Russian individuals owe **13% NDFL up to 2.4M RUB/year of income, 15%
  above that**, on gains from crypto disposal (sale/exchange), computed as
  proceeds minus acquisition/holding/disposal costs, self-reported via a
  3-NDFL declaration due **April 30** of the following year, tax due by
  **July 15** (per secondary tax-guide sources summarizing current Tax Code
  treatment of digital currency; not independently verified against the
  primary Tax Code article in this pass — flagged as a gap, and made largely
  moot if the classic-Payments-API + YooKassa path from §1a is used instead
  of Stars, since that settles in RUB with no crypto leg).
- **Personal data (152-FZ)**: not researched in depth this pass — flagged as
  a likely-relevant adjacent obligation (any payment flow collects some
  identifying/transactional data) that the pricing/policy ticket or a
  dedicated compliance ticket should pick up explicitly, since this ticket's
  scope was payment-rail feasibility rather than a full compliance sweep.

**Net takeaway for the next ticket**: the compliance floor for a
YooKassa-based flow (either in-chat via Telegram's classic Payments API, or
web-checkout fallback) is manageable at small scale — self-employed (NPD)
registration plus a payment aggregator that handles fiscal-receipt generation
covers the common case up to 2.4M RUB/year. The Stars/TON path adds a
materially different tax and settlement surface (crypto disposal income,
self-reported) that the policy ticket should weigh against its "stays fully
in-chat" convenience.

---

## Gaps / follow-ups flagged during this research

These were surfaced but not closed against a fully primary source — worth a
cheap confirmation pass before the policy ticket locks in a rail:

1. Whether Telegram actually classifies a Stamina top-up as a "digital good"
   requiring mandatory Stars (vs. eligible for the classic Payments API) —
   confirm with @BotSupport or by testing BotFather's payment-provider setup
   flow for the bot.
2. Exact Stars→TON withdrawal minimum, delay, and KYC requirements — no
   Telegram/Fragment-owned primary page was found enumerating this; only
   community/secondary sources.
3. Whether YooKassa (and/or CloudPayments) is still live as a selectable
   BotFather payment provider at implementation time — provider lists can
   change; verify live in BotFather rather than trusting docs alone.
4. CloudPayments' actual self-employed (самозанятый) onboarding policy,
   direct from CloudPayments rather than secondary summaries.
5. 152-FZ personal-data obligations attached to a payment flow — not
   researched here.
