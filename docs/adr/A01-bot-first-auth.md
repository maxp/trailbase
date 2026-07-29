# ADR A01 — Bot-first Auth Model

**Status**: Accepted
**Date**: 2026-07-25

**Уточнение 2026-07-29:** Telegram и Max — единственные identities; browser activation
не обязательна, а upload/search доступны через chat. Browser session остаётся
опциональным двухшаговым GET/POST flow, provider linking только явный. Полный контракт:
[Implementation Contract](../IMPLEMENTATION-CONTRACT.md#4-bot-first-authentication).

## Контекст

TrailBase — публичный каталог GPX-треков. Web и Telegram/Max chats являются
first-class interfaces к одному account/domain model. Messenger identity должна быть
достаточна для работы без предварительной web activation; отдельных email/phone
identity или recovery channels нет.

## Решение

1. **Telegram и Max — закрытый список identity providers.** Email, phone, passwords и
   отдельный recovery flow отсутствуют.
2. **Messenger identity достаточна для account/chat access.** Валидированный private
   webhook event аутентифицирует chat operation; browser session не является
   активацией account. Первый валидированный `/start` неизвестной identity в private
   one-to-one chat атомарно создаёт active account и provider identity; повторная
   доставка идемпотентна. Identity и token flows вне private chat запрещены.
3. **Web session опциональна.** Bot может выдать одноразовый deep-link
   `https://catalog/auth?token=...`; безопасный GET/POST flow создаёт server-side
   session cookie только для web UI.
4. **Единый account с explicit provider linking.** Один `user_id`, к нему привязано не
   более одной `telegram:*` и одной `max:*` identity в `user_identities`.
   `/start <link-token>` второго provider после fresh authentication существующей
   identity привязывает candidate identity прямо к target account и не создаёт новый
   account; browser completion и автоматического merge нет. Ошибка token обрабатывается
   fail-closed без fallback на создание account.
5. **Chat — first-class application interface.** Upload и search могут завершаться в
   Telegram/Max без web session. Bot и web adapters используют общие domain services,
   permission checks и async pipelines; web остаётся полноценным интерфейсом. Upload
   доступен только в private chat, а group/channel search — только stateless read-only
   по public published catalog.

## Альтернативы рассмотренные

- **Email/phone login или recovery.** Отвергнуто: identities ограничены Telegram и Max.
- **Обязательная web activation перед chat operations.** Отвергнуто: messenger
  identity уже аутентифицирует пользователя.
- **Bot-only application, web read-only.** Отвергнуто: web остаётся first-class UI для
  карты, каталога и управления.
- **Отдельная bot business logic.** Отвергнуто: chat и web должны вызывать общие domain
  services, иначе permission и search/upload semantics расходятся.
- **Telegram Login Widget / Max OAuth Widget (OAuth-like на сайте).** Отвергнуто: два разных виджета вместо одного паттерна; Telegram-Login-Widget требует публичного домена, плохо для self-hosted без DNS; не покрывает Max единообразно.
- **Two parallel identity systems (bot-first + email-first в разных таблицах).** Отвергнуто: множит двойные аккаунты и рассинхрон.

## Последствия

- Положительные: chat работает без browser round-trip; web session остаётся
  CSRF-protected и опциональной; upload/search имеют одну domain semantics во всех
  интерфейсах; нет email/phone PII и delivery stack.
- Отрицательные: bot conversations требуют state machine для многошагового upload и
  pagination/search filters; Telegram и Max adapters имеют разные UI capabilities.
- Потеря доступа ко всем linked messenger identities означает отсутствие self-service
  recovery; изменение этого ограничения требует отдельного решения.
