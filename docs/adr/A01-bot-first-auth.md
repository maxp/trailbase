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
   Все token-bearing confirmation/success/invalid/expired/`503` responses используют
   `Cache-Control: no-store`, `Referrer-Policy: no-referrer` и только same-origin static
   assets. Caddy/app access/security logs пишут route без query/form body; raw
   token/digest отсутствуют в errors, analytics и traces.
   Initial token-query `GET /auth` не consume-ит token и всегда redirect-only. Valid
   branch создаёт existing short-lived auth-flow record/cookie+nonce и отвечает `303`
   на clean `/auth`. Malformed/unknown/expired/superseded branch получает `303` на
   token-free `/auth` со stable non-secret `result=invalid`; target GET показывает
   generic-invalid без lookup/изменения existing flow, plain `/auth` продолжает current
   confirmation. Marker не добавляет state/cookie. Raw token отсутствует в
   body/redirect/history; JavaScript, `history.replaceState`, client storage и новый
   credential namespace не используются.
   Auth-flow record, flow-cookie и form nonce разделяют exact original `web_session`
   token deadline без нового interval или sliding от confirmation GET, retry/`503`.
   Expired/missing component показывает generic-invalid page, idempotently очищает
   remaining flow record/cookie и не consume-ит/recover-ит/reissue-ит source token.
   Auth-flow record хранит только source token digest, allowlisted `return_to`, original
   deadline и SHA-256 form-nonce hash. Independent 128-bit flow-cookie ID адресует
   Valkey record по SHA-256 cookie value; raw source token после redirect не хранится.
   Raw flow ID/nonce остаются только в `HttpOnly` cookie/hidden POST field, bindings
   читаются из token record, raw/digest values отсутствуют в telemetry/DLQ.
   Flow cookie имеет exact name `__Host-trailbase_auth_flow`, `Secure`, `HttpOnly`,
   `SameSite=Lax`, `Path=/`, без `Domain`; `Max-Age`/`Expires` capped remaining source
   deadline. Success, terminal invalid/expiry и explicit cancel очищают её
   `Max-Age=0`; `__Host-trailbase_session` остаётся отдельной cookie.
   Новый valid initial auth GET с existing flow-cookie атомарно удаляет previous record,
   создаёт replacement и оставляет один active browser flow. Old tab/form получает
   generic-invalid без side effects; previous source token остаётся unconsumed/unrevoked
   и его link до deadline может снова заменить current flow.
   Failed flow-cookie/form-nonce validation на `POST /auth` не consume-ит source token
   и не удаляет valid flow. Missing/unknown cookie и nonce mismatch дают одну
   generic-invalid response без token/pointer/session mutations. Unknown cookie
   очищается у browser через `Max-Age=0`; valid record при wrong/stale nonce и current
   cookie сохраняются. Общий `/auth` rate limit — 10/min на IP без отдельного
   nonce-attempt counter.
   Non-transient invalid `POST /auth` использует PRG: `303` на token-free `/auth` со
   stable query marker `result=invalid`, затем generic-invalid GET без lookup current
   flow. Cleanup следует принятому error class; wrong/stale nonce сохраняет valid
   flow/cookie. Transient PostgreSQL/Valkey failure остаётся retryable `503` с
   `Retry-After` без redirect, terminal marker или credential mutations и сохраняет
   retryable flow/token/cookie.
   После successful PostgreSQL active identity/account check одна atomic Valkey
   function валидирует flow/nonce, source token и active pointer, создаёт либо
   rotate-ит browser session, consume-ит token/pointer и удаляет flow. Все credential
   mutations имеют одну linearization point; distributed PostgreSQL transaction нет.
   Два concurrent POST одной form создают ровно одну session, а проигравший terminal
   invalid и не revoke-ит успешную session.
   Token record и новая rotated session сохраняют исходный `fresh_authenticated_at`;
   ordinary activity и sliding TTL не продлевают absolute 10-minute freshness. Token
   непосредственно выпускает explicit user-initiated «Подтвердить вход»
   command/callback в private one-to-one chat, bound к
   provider/user/chat/message/requester. Injected application UTC clock фиксирует
   `fresh_authenticated_at` при server validation/acceptance event; provider-supplied
   timestamp отбрасывается после payload validation, а event dedupe/binding
   проверяются независимо. Этот validated event является единственным
   confirmation; recent messages/notifications/background/browser activity, второй
   prompt и PIN/password/TOTP freshness не создают.
   Provider timestamp не сохраняется в `web_session` token/session auth records; там
   остаются только server `fresh_authenticated_at` и opaque event/identity bindings
   для dedupe/consume. В MVP timestamp не пишется также в operational events, logs или
   metrics; observability ограничена server-time counters по provider/validation result
   без raw provider payload. Delivery-delay signal требует отдельного будущего
   retention contract.
   Единственная fresh-auth metric —
   `fresh_auth_confirmation_total{provider,result}`: `provider=telegram|max`,
   `result=accepted|duplicate|invalid_event|invalid_binding|account_unavailable|internal_error`.
   Identity/event/request/timestamp и дополнительные labels отсутствуют; HTTP/webhook
   latency/status покрываются общими metrics.
   `duplicate` означает exact provider replay только после committed acceptance:
   provider получает `2xx`, но новый token/link/message/session state не создаётся.
   Первый commit один раз выпускает link и increment-ит `accepted`; `internal_error` до
   него остаётся retryable через общий worker/DLQ и duplicate не является.
   Existing 7-day provider-event dedupe record содержит fresh-auth state
   `processing|accepted`, без новой table/key namespace. Ingress атомарно claim-ит
   processing вместе со Stream enqueue; replay этого state только получает `2xx`, пока
   worker retry/DLQ продолжает original event. После token/link issuance worker ставит
   accepted, replay которого increment-ит duplicate. Оба state имеют тот же TTL.
   Worker одной atomic Lua/Valkey operation проверяет processing bindings, создаёт
   единственный 128-bit 10-minute `web_session` token record и переводит state в
   accepted до любого Bot API send. Crash не разделяет token/accepted, а delivery retry
   переиспользует exact link; external network request не входит в commit.
   При новом confirmation той же provider identity эта issuance атомарно заменяет
   per-identity active-token pointer, удаляет previous token record и его ещё
   существующий raw delivery field. Previous provider-event marker остаётся accepted;
   отдельные bot message, revoke audit и notification не создаются.
   Гонку `/auth` consume old link и new issuance линейризуют atomic Valkey operations
   над pointer. Consume-first удаляет matching token record/pointer и может завершить
   auth; issuance-first заменяет pointer и делает old link terminal invalid. Поздняя
   issuance не отзывает создаваемую/созданную browser session; distributed transaction
   с PostgreSQL отсутствует.
   Active-token pointer хранится отдельным non-secret Valkey key по internal identity
   UUID, содержит только SHA-256 token digest и получает `PEXPIREAT` exact token
   deadline. Atomic consume удаляет его с token record, replacement заменяет;
   missing/expired pointer или target terminal invalid и очищается idempotently.
   Delivery data, sliding TTL и janitor отсутствуют.
   `web_session` token record адресуется только по `SHA-256(raw_token)`, тому же digest,
   что pointer, без raw token в record/key. При 128-bit CSPRNG entropy используется
   unsalted SHA-256 без custom HMAC/salt. Raw token остаётся лишь в link/browser
   request и short-lived delivery field; raw value и digest запрещены в
   logs/metrics/traces/DLQ.
   Raw link token для crash-safe retry хранится только как short-lived delivery field
   existing accepted dedupe record до successful send или token expiry. Field удаляется
   после delivery/expiry; seven-day marker остаётся non-secret. Raw token не попадает в
   logs/metrics/traces/DLQ; отдельного namespace, deterministic derivation и re-issuance
   нет.
   Runtime требует Valkey 9.x minimum: atomic issuance задаёт delivery field
   `HPEXPIREAT` на exact token deadline, successful send выполняет idempotent `HDEL`.
   Seven-day accepted key TTL не меняется; janitor и отдельный TTL key отсутствуют.
   Если successful Bot API delivery не завершилась до token/delivery-field expiry,
   worker прекращает её terminally без stale send, re-issuance, изменения accepted
   marker или отдельного user notification. Exact replay остаётся `duplicate`; только
   новое explicit «Подтвердить вход» action создаёт новый provider event, freshness,
   token и link. Failure отражается только общими Bot API delivery metrics/alert без
   raw token.
   Post-issuance delivery делает максимум пять total attempts с exponential
   backoff/jitter. Retry разрешён только для timeout/network errors, `429` и `5xx` и
   только если следующая attempt с `Retry-After` укладывается в original token
   deadline. Остальные `4xx`, exhausted budget и retry за deadline terminal сразу:
   idempotent `HDEL`, accepted marker сохраняется, DLQ/late replay отсутствуют.
   Pre-issuance `internal_error` остаётся в общем worker retry/DLQ flow.
   Любая active linked Telegram/Max
   identity может выполнить re-auth: primary является delivery preference, а не более
   сильной identity. Token bound к exact internal identity/provider/user/event;
   consume re-check-ит active membership, terminally отклоняет unlinked/foreign token
   и после success создаёт account-level freshness независимо от provider. При valid
   current browser session того же user rotation является credential refresh:
   token/CSRF/freshness заменяются, old current session revoke-ится, other sessions не
   меняются и `new_session` notification не создаётся. Без valid same-user session
   successful `/auth` является ordinary new login с locked-on notification.
   Same-browser record сохраняет original `created_at`, обновляет token hash/CSRF,
   bot-derived freshness, current `last_seen_at`, safe short User-Agent и one-year TTL;
   отдельного `reauthenticated_at` нет.
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
