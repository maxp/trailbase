# ADR A01 — Bot-first Auth Model

**Status**: Accepted
**Date**: 2026-07-25

**Уточнение 2026-07-30:** Telegram и Max — единственные identities; browser activation
не обязательна, а upload/search доступны через chat. Browser session остаётся
опциональным двухшаговым GET/POST flow, provider linking только явный. Plain `/start`
показывает описание сервиса, правила и ссылку на CC BY 4.0 и включает user agreement
flow. Полный контракт:
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
   one-to-one chat атомарно создаёт active account, provider identity и минимальный
   `user_agreements` record; повторная доставка идемпотентна. Plain `/start`
   показывает описание TrailBase/bot, главное меню, rules и CC BY 4.0 links и
   уведомляет, что использование бота означает автоматическое согласие с правилами,
   включая будущие изменения. Кнопки «Согласен» и pending state нет.
   Главное меню Telegram/Max одинаково: «Поиск», «Загрузить GPX», «Мои
   треки/черновики», «Настройки» и «Помощь»; rules/license остаются вторичными
   ссылками. «Настройки» содержит «Профиль», «Мессенджеры», «Уведомления», «Сессии»
   и «Аккаунт» без email/phone/password/recovery settings. Все эти operations
   завершаются в private chat без web session; sensitive actions сохраняют
   fresh-auth/confirmation guards, а web UI является optional mirror. Session cards
   показывают только short device/browser summary и timestamps без IP/full
   UA/session identifiers; revoke target находится в bound opaque state. Single
   revoke требует confirmation без fresh auth; logout-all после fresh-auth
   confirmation revoke-ит только все web sessions, не затрагивая identities/chat/account.
   Public display name применяется сразу без fresh auth/pre-moderation после
   validation и audit;
   provider snapshots его не перезаписывают. Attribution всех public track pages
   разрешается через текущее account display name, а уже сгенерированный GPX остаётся
   immutable с именем на момент export; новый export использует актуальное имя.
   UI locale выбирается из `ru`/`en`,
   применяется к bot/web и не перезаписывается provider language; `simple` остаётся
   technical content fallback. Security и action-required moderation notifications
   locked-on; прочие categories configurable. Optional moderation results default
   enabled, catalog/informational default disabled. Один account-level toggle
   управляет одновременно web inbox и primary bot без per-channel preferences;
   domain state/audit независимы. Preference snapshot фиксируется при создании
   notification/outbox; последующие settings действуют только на future events.
   Пока account deactivated, доставляются только locked-on security/account-lifecycle
   notifications; moderation/catalog/informational delivery подавляется без replay,
   но domain events и audit сохраняются. Deactivation transaction отменяет pending
   non-security notification outbox delivery intents и suppress-ит связанные unread
   inbox records; уже `sending` delivery может завершиться,
   security/account-lifecycle records не отменяются.
   «Мои треки/черновики»
   показывает без web session единый список owned
   tracks и upload/draft flows со статусом каждого; требующие действия пользователя
   элементы выводятся первыми. Действия карточки зависят от durable status;
   `pending_review` доступен только для просмотра, а delete/archive требует отдельного
   подтверждения. Список листается по 10 entries через server-side keyset state;
   provider callback содержит только opaque ID, привязанный к requester/chat/message.
   Controls имеют абсолютный TTL 15 минут без sliding refresh; expired state не
   меняет старое сообщение. User-archived tracks доступны только через вторичный
   filter «Архив», где до 30-дневного purge показываются срок и restore action;
   досрочного permanent delete нет. «Все» и «Архив» — единственные list views;
   дополнительных status filters в MVP нет. Restore возвращает durable pre-archive
   status/current revision того же track без повторной moderation approved snapshot;
   moderator removal этим flow не отменяется. Одного valid owner callback достаточно:
   confirmation prompt нет, операция атомарно проверяет `archived`/`purge_at` и
   идемпотентна.
   Bot отдаёт обычные HTTPS-ссылки на актуальные rules и CC BY 4.0; документы в chat
   не копируются. Изменения rules по ссылке не версионируются в account model и не
   влияют на agreement, account, sessions или notifications.
   Автоматическое agreement является standing consent для CC BY 4.0 contributions.
   Append-only PostgreSQL `user_agreements` хранит только user ID, notice hash,
   показанные URLs, acceptance time и `/start` source с unique user, без IP, raw
   tokens или callback state. Повторный plain `/start` agreement record не меняет:
   он только обновляет provider profile snapshot и снова показывает notice/menu.
   Деактивация и административная реактивация account сохраняют исходный agreement
   record неизменным и не требуют повторного agreement; физическое удаление
   регулируется отдельной data-retention/privacy policy вне MVP. Published tracks
   деактивированного account остаются public с прежним TrailBase author display name
   и CC BY 4.0 attribution; private content остаётся private, а скрытие/удаление
   track выполняется отдельной lifecycle operation. Self-deactivation собирает
   только confirmation без reason/free text/feedback; audit сохраняет actor/action/
   interface/provider/request ID/timestamp. Final prompt без dynamic counts объясняет
   session/token revocation, access/content/moderation consequences и admin-only
   возврат через support; actions — «Деактивировать аккаунт»/«Отмена». State истекает
   в original fresh-auth deadline без sliding; chat использует bound 128-bit opaque
   ID, web — session/CSRF и bound purpose state, Confirm/Cancel single-use. Durable
   state — PostgreSQL `sensitive_operation_confirmations` с SHA-256 ID; Confirm row
   lock атомарно consume-ит state с account/audit/outbox mutation, Cancel только
   consume-ит, cleanup — 24 часа. PostgreSQL active status проверяется при session
   validation и auth/link token consume; idempotent outbox command без raw
   credentials чистит Valkey с retry/DLQ, а cleanup failure не возвращает доступ.
   Дополнительный credential epoch/version не вводится: deactivation flow удаляет
   account sessions и outstanding auth/link tokens непосредственно из Valkey.
   При недоступном PostgreSQL authorization работает fail-closed без cached-active
   fallback: HTTP получает `503` с `Retry-After` без session/mutation, bot event
   повторяется как transient failure и не выполняет domain changes; Valkey
   credential сам по себе недостаточен. Session-validation `503` не очищает cookie
   или Valkey session и не обновляет `last_seen_at`/sliding TTL; ответ содержит
   `Cache-Control: no-store`, а неистёкшая session снова работает после
   восстановления PostgreSQL. Revoke выполняется только для подтверждённого invalid,
   expired или disabled состояния.
   Public catalog не показывает
   badge, причину или иной marker деактивации автора; status доступен только самому
   пользователю и администраторам. In-flight parse может завершиться только private
   draft; `pending_review` deactivated owner сохраняет status, но исключается из
   moderator queue, а approve/publish re-check-ит active owner. Реактивация возвращает
   pending item в очередь при `export_state = ready`, без отдельного `suspended`
   status. Deactivation и
   approve/publish сериализуются owner row lock и lock order
   `user -> track -> revision`: approval-first остаётся public, deactivation-first
   блокирует публикацию. Linked disabled identity никогда не создаёт новый account:
   private chat оставляет только `/start`, help/rules/license и stateless read-only
   public search без account-specific data, tokens или mutations. Встроенного bot
   reactivation request нет: `/start` показывает обязательный production
   `TRAILBASE_SUPPORT_URL`, а admin после out-of-band проверки использует management
   UI с fresh auth, обязательной причиной и audit. Reactivation возвращает тот же
   account в active с сохранёнными identities/roles/settings/content/pending
   moderation, но не восстанавливает sessions, auth/link tokens или suppressed
   notifications и не создаёт agreement. Lifecycle notification содержит только
   факт/time, `TRAILBASE_SUPPORT_URL` и инструкцию открыть bot; admin identity,
   internal reason, audit ID и tokens не раскрываются. Internal reason — required
   enum `support_request_verified`, `administrative_correction`, `other` плюс optional
   validated audit note до 1 000 code points, обязательная для `other`; code/note не
   логируются.
   Identity и token flows вне private chat запрещены.
3. **Web session опциональна.** Bot может выдать одноразовый deep-link
   `https://catalog/auth?token=...`; безопасный GET/POST flow создаёт server-side
   session cookie только для web UI. Browser re-auth переиспользует тот же
   `web_session` token и `/auth` flow без отдельного credential или consume endpoint.
   Token record и новая rotated session сохраняют исходный `fresh_authenticated_at`;
   ordinary activity и sliding TTL не продлевают absolute 10-minute freshness.
4. **Единый account с explicit provider linking.** Один `user_id`, к нему привязано не
   более одной `telegram:*` и одной `max:*` identity в `user_identities`.
   `/start <link-token>` второго provider после fresh authentication существующей
   identity привязывает candidate identity прямо к target account и не создаёт новый
   account; browser completion и автоматического merge нет. Ошибка token обрабатывается
   fail-closed без fallback на создание account. Agreement принадлежит `user_id`;
   linked identity не создаёт новый agreement record и сразу использует capabilities
   target account.
5. **Chat — first-class application interface.** Upload, search и account settings
   могут завершаться в Telegram/Max без web session. Bot и web adapters используют
   общие domain services, permission checks и async pipelines; web остаётся
   полноценным optional интерфейсом. Upload и settings доступны только в private
   chat, а group/channel search — только stateless read-only по public published
   catalog. Любой GPX attachment в private chat после quota/slot checks начинает
   upload flow без предварительного `/upload`; команда только показывает подсказку,
   а attachment другого типа job не создаёт.

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
  интерфейсах; нет email/phone PII и delivery stack; нет отдельного consent state
  machine.
- Отрицательные: bot conversations требуют state machine для многошагового upload и
  pagination/search filters; Telegram и Max adapters имеют разные UI capabilities.
- Потеря доступа ко всем linked messenger identities означает отсутствие self-service
  recovery; изменение этого ограничения требует отдельного решения.
