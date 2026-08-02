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
   `/auth` rate-limit reject — retryable `429` с `Retry-After`, не terminal invalid.
   Один budget 10/min на normalized client IP из trusted Caddy forwarding охватывает
   initial token GET, clean confirmation GET и POST. Limiter работает до credential
   lookup, сохраняет flow/token/cookie/nonce/session и отвечает generic `no-store`/
   `no-referrer` body без redirect, automatic retry, terminal marker или отдельных
   per-flow/per-nonce budgets.
   Non-transient invalid `POST /auth` использует PRG: `303` на token-free `/auth` со
   stable query marker `result=invalid`, затем generic-invalid GET без lookup current
   flow. Cleanup следует принятому error class; wrong/stale nonce сохраняет valid
   flow/cookie. Transient PostgreSQL/Valkey failure остаётся retryable `503` с
   `Retry-After` без redirect, terminal marker или credential mutations и сохраняет
   retryable flow/token/cookie.
   После successful PostgreSQL active identity/account check одна atomic Valkey
   function валидирует flow/nonce, source token и active pointer, promote-ит flow-ID
   digest в session namespace, создаёт либо rotate-ит browser session, consume-ит
   token/pointer и заменяет flow completion receipt как одну credential linearization
   point; distributed PostgreSQL transaction нет.
   Receipt до original flow deadline хранит nonce hash, allowlisted `return_to`, session
   digest/status и expiry без raw ID. Success/matching retry ставит session cookie в raw
   flow-ID value и очищает flow cookie. Concurrent/repeated POST возвращает identical
   success без второй/revoked session; terminal-invalid concurrent-loser outcome
   отменён.
   Commit создаёт provisional session и receipt с `PEXPIREAT` original flow deadline.
   Первый request с новой session cookie после PostgreSQL account validation атомарно
   помечает session claimed, удаляет receipt и включает one-year sliding TTL. Без cookie
   оба provisional keys истекают без orphan/janitor; expiry claimed session не revoke-ит,
   а validation `503` сохраняет provisional state до deadline.
   Provisional session скрыта из active-session list и создаёт только low-card auth
   metric. Ordinary-login claim ровно один раз создаёт audit, web-inbox record и
   locked-on primary-bot `new_session` outbox intent, idempotently keyed session digest.
   Same-browser re-auth notification не создаёт; delivery failure не rollback-ит
   claimed session и retry-ится независимо.
   Та же atomic claim function одной Valkey operation делает `XADD` durable
   `session_claimed` event в Stream session Valkey. Event содержит random UUID,
   `user_id`, provider, safe device summary, claim time и internal session digest только
   для idempotency, без raw token, IP и cookie. Worker одной PostgreSQL transaction
   idempotently создаёт audit/web-inbox/outbox с UNIQUE guards по session digest и event
   ID и делает `XACK` только после commit; pending/retry не создают duplicates, delivery
   остаётся независимой outbox-задачей.
   Для `session_claimed` общий terminal DLQ не применяется: transient и deterministic
   failures с operational alert retry-ятся бессрочно с capped backoff/jitter, а unacked
   entry остаётся в PEL и не trim/ack/drop-ится. После PostgreSQL commit worker делает
   `XACK`, затем `XDEL`; cleanup acknowledged остатка можно повторить. Метрики содержат
   только pending count, oldest age и retry count без event/session/user identifiers.
   `session_claimed` использует dedicated Stream и consumer group в session Valkey,
   отдельно от webhook ordering/retention/retry/DLQ. Существующий worker process
   запускает отдельный consumer loop без нового service/container; readiness и
   low-cardinality lag/PEL metrics проверяются отдельно, а общий shutdown lifecycle и
   `XAUTOCLAIM` после 60 секунд обеспечивают failover.
   Projector lag/unhealthy readiness не блокирует новые claims, пока atomic claim +
   `XADD` commit-ятся. Web не ждёт/не poll-ит worker и не ставит PEL age/size gate;
   задержка видна в worker readiness/alerts. Невозможность commit-ить claim + event не
   даёт success и использует ambiguous-outcome recovery без обхода `XADD`.
   Recipient snapshot/target identity в event отсутствуют. Projector под
   user/identity lock order выбирает current primary и одной PostgreSQL transaction
   создаёт audit/web-inbox/locked-on outbox. Порядок commit с primary change определяет
   old/new target; recipient PII/stale snapshot в Valkey не хранится, а deactivated
   account использует обязательную security-notification policy.
   Server-generated UTC `claimed_at` из event сохраняется как immutable `occurred_at`,
   PostgreSQL transaction time — отдельно как `recorded_at`. Inbox/message показывает
   время факта; audit/inbox сортируются по `occurred_at`, затем event ID. `recorded_at`
   служит только projection-lag diagnostics и не подменяет login time.
   Event обязан содержать integer `schema_version = 1` и полный payload strict closed
   Malli schema. Missing/unsupported version или malformed field до DB mutations
   остаётся pending, alert-ит и делает group unready без default/coercion, `XACK` или
   DLQ. Breaking rollout временно поддерживает переходные versions только до drain
   старого PEL, без постоянного compatibility layer.
   Safe device summary — immutable closed map `browser_family|os_family|device_class`
   из maintained server-side UA parser. Catalogs ограничены browser
   `chrome|safari|firefox|edge|webview|other|unknown`, OS
   `android|ios|windows|macos|linux|other|unknown`, class
   `mobile|tablet|desktop|other|unknown`; parse failure даёт `unknown`. Session/event/
   audit/notification используют одну map; raw UA, versions, model, language и IP не
   сохраняются и не логируются.
   `auth_provider = telegram|max` означает immutable source consumed token из validated
   binding, а не request/current primary. Audit/inbox/message показывают этот
   локализованный source; outbox target независимо выбирает current primary. Provider
   user/identity ID отсутствует, unlink/primary change historical fact не меняют.
   Append-only auth audit row — единый PostgreSQL idempotency root с independent UNIQUE
   по `event_id` и `session_digest`; связанные inbox/outbox создаются после неё той же
   transaction. Exact matching replay переиспользует rows. One-guard или payload
   mismatch rollback-ит transaction как integrity incident, оставляет event pending и
   alert-ит без `XACK`; partial independently-deduped projection запрещена.
   `event_id` — application-generated UUIDv7, созданный до first dispatch и сохранённый
   first claim в session claim state/`XADD` для stable retry/read-back. Он является audit
   root primary/idempotency key и FK target inbox/outbox. Stream ID используется только
   transport bookkeeping PEL/`XACK`/`XDEL`, не сохраняется как domain ID; replacement
   DB ID и UUIDv4 отсутствуют.
   Failed/pending event не создаёт global/per-user head-of-line block: bounded pool fair
   schedule-ит новые и due-retry entries, не допуская двух in-flight owners одного
   Stream ID. Failure остаётся в PEL/backoff, alert-ит и держит group unready, но later
   valid events продолжаются. Same-user races сериализует DB user lock; audit/inbox
   order — `occurred_at,event_id`, strict messenger delivery order отсутствует.
   Revoke/logout/expiry до projection/send не подавляют immutable login audit,
   web-inbox или pending `new_session` delivery; projector/dispatcher не перечитывают
   current session. Message показывает occurred time, auth provider, safe device и
   generic «Управление сессиями», без direct target или заявления об active state.
   `new_session` outbox фиксирует internal target identity при projection. Later primary
   change не делает retarget/duplicate и не переписывает delivery; linked target получает
   её и после потери primary status. Unlink exact target до send terminally завершает
   ordinary delivery без retry/retarget и detached-target snapshot. Snapshot запрещён
   для `new_session`; closed exception определён отдельно только для
   `identity_unlinked`. Audit/web-inbox сохраняются, target ID в метрике
   отсутствует.
   Projector transaction фиксирует `delivery_locale` (`ru|en`, fallback `ru`) в
   `new_session` outbox; retry не перечитывает locale, а его смена влияет только на
   будущие messenger notifications. Web-inbox остаётся semantic и локализуется текущим
   UI locale. Outbox содержит notification type, schema/template version и allowlisted
   semantic fields без rendered text или user-supplied free form.
   Dispatcher рендерит exact записанный `template_version`. Immutable bundled templates
   имеют обязательные `ru`/`en`, проверяемые при startup, и сохраняются минимум 90 дней
   после последнего producer deployment версии. Missing version не вызывает Bot API:
   это deterministic failure с обычным alert/DLQ и replay после возврата шаблона, без
   silent fallback на current version.
   Template catalog key — notification type, `template_version` и locale, без provider.
   Telegram/Max получают одинаковые wording, semantic fields и meaning; adapters меняют
   только escaping, markup, equivalent controls и transport limits. Если кнопка
   недоступна, тот же safe HTTPS URL включается в text без другого editorial template.
   Каждый security template с maximum allowlisted values в `ru`/`en` обязан помещаться
   в одно сообщение обоих providers: одна outbox delivery — один atomic send, без split,
   truncation или удаления fields. CI/startup проверяют worst case по более строгому
   Telegram/Max limit; invalid catalog останавливает dispatcher consumer и его readiness,
   а превышение требует новой короткой template version.
   Notification type + `schema_version` задают strict closed Malli payload schema;
   каждая `template_version` совместима ровно с одной schema version. Projector до
   insert валидирует required fields/types/no extras; failure rollback-ит transaction,
   оставляет `session_claimed` в PEL и alert-ит. Dispatcher повторно валидирует stored
   payload; mismatch не render-ится и не вызывает Bot API, а идёт в deterministic
   alert/DLQ. Coercion/defaults/ignored unknown fields отсутствуют.
   Notification schema/template pair включается двухрелизным expand/activate rollout:
   первый release добавляет её в catalog/dispatcher без смены producer, второй successful
   release переключает producer constant, когда previous rollback image уже знает обе
   versions. Старые versions сохраняются минимум 90 дней. Runtime registry, dual-write
   и silent downgrade отсутствуют; contract test проверяет emitted pair в current и
   rollback catalogs.
   Retire старой schema/template pair разрешён только после 90 дней с last emission и
   при zero replayable undelivered/DLQ references. Deploy preflight группирует references
   по notification type/schema/template version и блокирует removal; rewrite/drop/silent
   upgrade records ради gate запрещены. Delivered records, audit, semantic web-inbox и
   backup copies removal не блокируют.
   `new_session` outbox сохраняет closed `manage_sessions` action code без absolute URL,
   query, redirect target или action token. Catalog разрешает его в named internal route;
   dispatcher строит same-origin canonical HTTPS URL из current `PUBLIC_BASE_URL`, а
   startup проверяет HTTPS/allowlist. Смена base URL обновляет queued link, но не locale,
   template version или semantic fields.
   Generated `manage_sessions` link identifier-free: без user/session/notification/
   outbox IDs и provider/locale/tracking query. Canonical route при отсутствии session
   использует обычный auth flow с server-side allowlisted return target. Click-specific
   analytics/audit нет; разрешены aggregate route/status HTTP metrics без user/outbox
   labels, исключая leakage через provider/history/logs/referrer.
   Если link открыт под другим authenticated TrailBase account, route показывает current
   display name и только current account sessions. Navigation URL не bind-ит/switch-ит
   account и не раскрывает expected recipient; смена требует explicit logout/re-auth.
   Automatic switch/merge, target-account lookup и existence disclosure запрещены.
   Sessions-management authenticated GET, htmx partial, mutation, validation/error и
   redirect responses всегда ставят `Cache-Control: no-store` и
   `Referrer-Policy: no-referrer`; Caddy/CDN их не кэшируют. History сохраняет только
   identifier-free URL, не body. Cacheable summary и persistent client cache запрещены.
   `occurred_at` хранится как UTC Instant. Messenger всегда показывает explicit `UTC`;
   locale меняет format/wording, но не zone. Account timezone не хранится и не infer-ится
   из IP/provider/language. Web локализует Instant browser `Intl`, показывает zone/offset
   и UTC `datetime`, с explicit UTC fallback без client rendering.
   Messenger показывает только absolute `occurred_at` до минут с explicit `UTC`, без
   `recorded_at`, send/attempt time или relative age; retry/DLQ replay не меняют timestamp.
   Web relative age допустим только secondary рядом с обязательным absolute local time,
   вычисляется client-side и не хранится/outbox-ится.
   Каждый distinct `new_session` claim даёт отдельные audit/inbox/delivery; batching по
   minute/provider/device, fingerprint coalescing, replacement и collapsed inbox rows
   запрещены. Dedupe только exact `event_id` + `session_digest` replay. UI показывает
   count и отдельные occurred times; strict messenger order не вводится.
   Flood сохраняет каждую audit/inbox/outbox row; suppression/digest/coalescing нет.
   Dispatcher использует provider-wide budget, per-target pacing и fair target queues с
   provider `Retry-After`. Local throttle оставляет row pending без attempt/DLQ; один
   noisy account не блокирует других. Метрики queue depth/oldest age/throttled count
   имеют только provider label, без target IDs.
   `429 Retry-After` сохраняет в PostgreSQL provider-wide
   `cooldown_until=max(current,parsed)`; triggering call расходует attempt, waiting rows —
   нет. Все loops/restarts соблюдают cooldown, другой provider работает. Invalid/missing
   header получает capped jittered fallback; ingress/projection/inbox не блокируются,
   telemetry содержит только provider label.
   Adapters используют closed errors `provider_blocked|target_unreachable|rate_limited|
   transient|unclassified`. Global auth/config failure открывает durable provider circuit,
   останавливает calls, оставляет rows pending без attempts и делает readiness failed;
   другой provider работает. Target error terminal только для row; unclassified идёт в
   DLQ/alert без circuit. Raw body/target ID/credentials не логируются и не DLQ-ятся.
   `target_unreachable` оставляет identity linked/primary и не меняет future routing:
   exact delivery terminal без retry/retarget, audit/inbox остаются, а future events
   выбирают explicit current primary. Другой linked provider не получает fallback или
   duplicate. Route меняют только user unlink/change-primary; automatic account mutation
   запрещена.
   `target_unreachable` durable ставит identity `delivery_health=unreachable` с observed
   timestamp без raw reason; latest observed outcome под row lock побеждает stale race.
   Web показывает generic warning и change-primary/unlink. Successful ordinary send
   exact identity либо validated inbound user message от exact linked provider identity
   ставит `reachable`; inbound использует application acceptance time, не provider
   timestamp. Health не влияет на auth/linking/routing/queue.
   `unreachable` не имеет TTL/manual dismiss; time, restart, circuit close и primary
   change его не очищают. Recovery даёт только ordinary send success exact identity или
   validated inbound user message exact linked identity; unlink удаляет state, relink
   начинает с `unknown`. Non-primary warning остаётся только в identity card.
   Inbound recovery принимает только validated user-authored private-chat message event
   exact linked identity, включая commands, text и media. `callback_query`, inline query,
   membership/service events и delivery receipts health не меняют.
   Provider event dedupe выполняется до health mutation. Только первое accepted unique
   message ставит `reachable`/application acceptance timestamp; exact replay получает
   idempotent acknowledgement без DB update и не может перекрыть более свежий
   `target_unreachable` новым observed timestamp.
   Accepted inbound message exact linked identity обновляет health и для deactivated
   account, поскольку reachability не является authorization. Это не reactivate-ит
   account, не создаёт session/notification и не обходит command policy; retained health
   используется после будущей reactivation.
   Inbound recovery и unlink exact identity сериализуются identity row lock. Unlink-first
   не позволяет event обновить/recreate/relink identity; inbound-first может поставить
   `reachable`, после чего unlink удаляет row и health. Old provider identity не
   attach-ится к future relink.
   Concurrent inbound `reachable`/outbound `target_unreachable` используют application
   observation time до ожидания lock: accepted-event time после dedupe и normalized
   provider-result receipt соответственно. Под lock применяется только более новое;
   commit/provider timestamps и send-start не участвуют, exact tie выигрывает
   `unreachable`.
   `delivery_health_observed_at` не показывается owner-у. Web/private-chat card имеет
   actionable warning только для `unreachable`; `unknown|reachable` не получают badge/
   timestamp. Поле остаётся internal ordering/operations data, не last activity или last
   successful delivery.
   Warning — derived UI projection, не messenger delivery/web-inbox/outbox/notification.
   Web показывает его при owner view; private-chat settings — только через другую working
   linked identity. Message ранее blocked exact bot сначала делает inbound recovery, и
   его card уже без warning; без working identity bot warning невозможен, остаётся Web.
   Closed warning copy: RU «Доставка в этот мессенджер недоступна. Разблокируйте бота,
   если нужно, и отправьте ему сообщение.»; EN “Delivery to this messenger is
   unavailable. Unblock the bot if needed, then send it a message.” Он общий для
   Web/cross-identity card, без raw cause, рядом с change-primary/unlink.
   `unreachable -> reachable` только убирает warning при следующем card render/read.
   Recovery toast/flash/bot reply/notification/inbox/outbox/audit отсутствуют; normal
   response исходной message не объявляет channel recovery.
   Открытая Web card не refresh-ится автоматически после inbound recovery: polling,
   SSE/WebSocket отсутствуют. Она может быть stale до navigation/обычного htmx card
   refresh; следующий server render читает authoritative row. Bot card обновляется при
   следующем user request; `Cache-Control: no-store` сохраняется.
   `user_identities` хранит `delivery_health text NOT NULL DEFAULT 'unknown'` с CHECK
   `unknown|reachable|unreachable` и nullable `delivery_health_observed_at timestamptz`;
   compound CHECK связывает unknown/NULL и observed/non-NULL. Enum, raw reason, manual
   clear и отдельная health-history table отсутствуют.
   Отдельной delivery-check функции нет: Web/private-chat action, probe routes,
   `delivery_kind=health_probe`, outbox/status/polling/cooldown и test-message templates
   отсутствуют.
   Timeout/network error после dispatch этой mutating function — ambiguous commit, не
   mutation-free `503`. Handler делает bounded retry/read-back по opaque commit ID;
   success либо сохраняющий flow `503` допустим только при confirmed outcome, а прежний
   `503` — до dispatch или при proven non-commit. Still-unknown outcome использует
   recoverable branch без terminal cleanup и blind second session.
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
   commit создаёт provisional ordinary login, а locked-on notification — только claim.
   Same-browser record сохраняет original `created_at`, обновляет token hash/CSRF,
   bot-derived freshness, current `last_seen_at` и safe short User-Agent; one-year
   sliding TTL начинается после provisional claim, отдельного `reauthenticated_at` нет.
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
