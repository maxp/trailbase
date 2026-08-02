# TrailBase — Roadmap (vertical slices)

Декомпозиция дизайна (см. `architecture.md` + `docs/adr/`) на независимо доставляемые вертикальные срезы (tracer bullets). Каждый срез — **runnable end-to-end**, observable через конкретное поведение, а не горизонтальный слой.

Точные security, data-model, runtime и API-инварианты находятся в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). Он уточняет этот roadmap и
имеет приоритет над старыми детальными формулировками ниже.

M01 — prerequisite всех остальных. M02 → M03 → M04 — backbone. M05/M06/M07/M08 — independent extensions поверх backbone (могут разрабатываться параллельно после M04).

```
M01 Foundation
   │
   ▼
M02 Identity ──▶ M03 Upload ──▶ M04 Catalog Render
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼             ▼
             M05 POI        M06 Search    M07 Class      M08 Photo/Elev
```

---

## M01 — Foundation

**Объём**: проект, БД-схема core-таблиц, HTTP-skeleton, базовый render.

**Файлы/компоненты**:
- `deps.edn` (Clojure deps + aliases: dev/test/migrate).
- `bb.edn` tasks: `migrate`, `rollback`, `run-dev`, `repl`.
- `resources/migrations/` (migratus): `001-init.up.sql/down.sql` — core-схема.
- `src/trailbase/core.clj` — Ring/reitit app skeleton, `/health` endpoint.
- `src/trailbase/db.clj` — next.jdbc datasource + hugsql loader.
- `src/trailbase/render.clj` — Hiccup → HTML, базовый layout + partials.

**Core data model (M01 закладывает основу; M02/M03/M05 дополняют)**:
- `users`, `user_identities`, append-only `user_agreements`
- `tracks` как stable identity и immutable `track_revisions` для versioned content
- `tags`, а revision-tag связи добавляются в M07
- `locations` как stable identity и `location_revisions`, annotations добавляются в M05
- Valkey для browser sessions, web-session/link tokens, ephemeral chat interaction
  state, rate limits и async streams

**Acceptance**:
- `bb migrate` создаёт PostGIS-расширение и все core-таблицы.
- `bb run-dev` поднимает HTTP-сервер; `/health/live` и внутренний `/health/ready`
  выполняют разные проверки.
- `GET /` рендерит Hiccup-страницу с layoutом; partial добавляется через `hx-get`.
- next.jdbc datasource доступен в REPL; hugsql queries загружаются.
- Перед этим Docker Compose поднимает PostgreSQL/PostGIS, MinIO и Valkey одной командой.

**Non-goal**: auth, реальные треки, бот.

---

## M02 — Messenger Identity + Optional Web Session

**Объём**: Telegram/Max webhook identity → account и chat operations без обязательного
web activation; optional deep-link → безопасный GET/POST flow → Valkey browser session;
explicit identity linking, roles и active-session management.

**Файлы/компоненты**:
- `src/trailbase/bot/telegram.clj` — raw Bot API через hato (setWebhook/sendMessage);
  polling не реализуется.
- `src/trailbase/bot/max.clj` — raw Bot API (Max/NetM), аналогично Telegram.
- `src/trailbase/bot/core.clj` — общая проверка messenger identity, command routing и
  обработка обычного `/start`, `/start <link-token>` и optional browser-session
  deep-link.
- Plain `/start` атомарно создаёт account/identity и минимальный agreement record,
  затем показывает описание TrailBase/bot, главное меню, ссылки на актуальные rules и
  CC BY 4.0 и уведомление, что дальнейшее использование означает автоматическое
  согласие, включая будущие изменения правил. Кнопки «Согласен» и pending state нет;
  linking payload использует отдельную fail-closed ветку.
- Главное меню обоих providers содержит «Поиск», «Загрузить GPX», «Мои
  треки/черновики», «Настройки» и «Помощь» в одном порядке. «Правила» и «Лицензия»
  остаются вторичными ссылками вне главного меню.
- «Настройки» содержит «Профиль» (public display name/language), «Мессенджеры»
  (linked identities/primary provider), «Уведомления», «Сессии» (active web sessions,
  revoke/logout-all) и «Аккаунт» (деактивация). Email/phone/password/recovery
  settings отсутствуют.
- Все settings operations доступны в private chat без web session. Profile и
  notifications меняются в chat; provider linking завершается через target
  `/start <link-token>`; sessions доступны для list/revoke/logout-all; deactivation
  выполняется после fresh auth и confirmation. Web UI — optional mirror.
- Public `users.display_name` меняется сразу без fresh auth/pre-moderation и
  audit-ится. После NFC/trim значение должно быть непустым, без control/newline
  characters и не длиннее 200 Unicode code points; abuse идёт через общую moderation
  policy.
- Attribution на public track pages всегда использует текущее `users.display_name`,
  включая старые published tracks после реактивации и смены имени; отдельного
  revision-level author-name snapshot для web нет. Уже созданный GPX export остаётся
  immutable со старым именем, а новый export использует имя на момент генерации.
- UI locale выбирается только из `ru`/`en` и применяется к bot и web UI.
  Поддерживаемый provider language служит initial hint, но не перезаписывает ручной
  выбор. `simple` остаётся техническим content fallback, не user locale.
- Security notifications и action-required moderation (`changes_requested`) locked-on
  и всегда доставляются в web inbox/primary bot. Catalog, informational и остальные
  moderation results являются configurable preferences.
- `changes_requested` notification показывает локализованный actionable label
  сохранённого correction code и optional owner-visible note; delivery не принимает
  отдельный free-form reason, note не логируется.
- Moderator removal/hide также locked-on для active account, включая full-lock
  containment: lifecycle transaction атомарно создаёт moderation audit, web-inbox
  record и primary-bot outbox. Решение использует `reason_code text NOT NULL` с DB
  CHECK на `policy_violation|privacy_or_safety|legal_request|spam|duplicate|technical_containment|other`,
  без PostgreSQL enum. Optional audit-only `reason_note` обязателен для `other`, после
  trim содержит 1–1000 Unicode code points без control/newline и не показывается
  owner-у или в logs. Notification содержит только removal fact, локализованный
  public-safe reason label, short track ID, `TRAILBASE_SUPPORT_URL` и 90-day
  purge/appeal deadline;
  `technical_containment` показывается как generic «Техническая недоступность».
  Full lock, `track_issues.code/detail`, snapshot и internal data не копируются.
  Deactivated-account suppression действует без backlog/replay.
- Первый committed appeal outcome также locked-on: та же transaction сохраняет
  web-inbox record и primary-bot outbox. Uphold показывает confirmation fact, short
  track ID и purge deadline без second-appeal action. Restore даёт canonical link для
  published либо edit/resubmit action для private branch. Admin reason/note, actor,
  audit ID/internal context скрыты; failure/conflict/retry не создают новую
  notification, deactivated-account suppression действует без backlog/replay.
- Изменения `track_issues` не являются notification category: detection, resolution
  и recurrence не создают web-inbox records или messenger deliveries. Owner видит
  только текущий generic `has_problems` в track views.
- Пока account deactivated, доставляются только locked-on security/account-lifecycle
  notifications через web inbox и primary linked messenger. Все moderation, catalog
  и informational notifications подавляются без backlog/replay; domain events и
  audit сохраняются.
- Reactivation notification показывает только факт/time, `TRAILBASE_SUPPORT_URL` и
  инструкцию снова открыть bot. Admin identity, internal reason, audit ID и tokens не
  раскрываются; reason остаётся только в audit.
- Deactivation transaction отменяет `pending` non-security notification outbox
  delivery intents и suppress-ит связанные unread web-inbox records с обычным
  retention. Уже claimed `sending` delivery может завершиться;
  security/account-lifecycle records не отменяются.
- New-account defaults: optional moderation results, включая `published`, enabled;
  catalog и informational disabled. Locked-on categories не имеют configurable
  default.
- Один account-level optional toggle применяется одновременно к web inbox и primary
  bot; per-channel preferences отсутствуют. Disabled category не создаёт notification
  и bot delivery, но не подавляет domain state/event/audit. Для active account
  locked-on delivery неизменна.
- Preference snapshot применяется в transaction создания notification/outbox intent.
  Позднее изменение влияет только на будущие events и не отменяет queued delivery;
  dispatcher не перечитывает settings.
- «Мои треки/черновики» показывает в chat единый список owned tracks и
  upload/draft flows со статусами `draft`, `processing`, `pending_review`,
  `changes_requested`, `published`; отдельного track state `rejected` нет. Требующие
  действия пользователя элементы идут первыми, остальные — по последнему изменению;
  web session не нужна.
- Archived tracks скрыты из default list и доступны через вторичный filter «Архив».
  Карточка показывает оставшийся срок до 30-дневного purge и действие
  «Восстановить»; early permanent delete в MVP отсутствует.
- Дополнительных status filters в MVP нет: list views ограничены default «Все» и
  «Архив». Требующие действия entries остаются в начале «Все».
- User archive сохраняет pre-archive status/current revision. Restore до `purge_at`
  атомарно возвращает то же состояние: прежний approved snapshot публикуется без
  повторной moderation, private track остаётся private. Moderator removal не
  восстанавливается этим flow.
- «Восстановить» не требует второго confirmation: valid owner callback атомарно
  проверяет `archived`/`purge_at` и выполняет idempotent restore. Foreign, stale или
  expired callback state не меняет.
- Карточка показывает только допустимые для текущего статуса действия:
  редактирование для `draft`/`changes_requested`, прогресс/cancel для `processing`,
  read-only просмотр для `pending_review`, просмотр/download/new revision/archive
  для `published`. Delete/archive требуют отдельного подтверждения.
- Список показывает 10 элементов на страницу с controls «Назад»/«Далее» и
  deterministic server-side keyset cursor. Provider callback содержит только opaque
  ID; cursor и provider/chat/message/requester binding хранятся в Valkey.
- List controls истекают через 15 минут от открытия без продления. Expired callback
  или потеря Valkey state не редактирует старое сообщение и предлагает заново открыть
  «Мои треки/черновики».
- «Правила»/«Лицензия» возвращают обычные HTTPS-ссылки; полные документы и их chat
  copies не хранятся. Rules link всегда открывает актуальный текст.
- Автоматическое agreement первого plain `/start` служит standing consent для CC BY
  4.0 contributions. `user_agreements` хранит только `user_id`, `accepted_at`,
  `source = /start`, notice SHA-256 и rules/license URLs; unique `user_id` исключает
  повторную запись. IP, raw tokens и callback payload не сохраняются.
- Повторный plain `/start` не изменяет `accepted_at`, notice hash или URLs; он только
  снова показывает notice/menu и обновляет provider profile snapshot.
- Деактивация и административная реактивация account сохраняют исходный
  `user_agreements` record неизменным и не требуют нового agreement. Физическое
  удаление account остаётся отдельной data-retention/privacy policy вне MVP.
- Published tracks деактивированного account остаются в public catalog с прежним
  TrailBase author display name и CC BY 4.0 attribution. Деактивация блокирует новые
  login/mutations и сохраняет private content private; скрытие или удаление track
  остаётся отдельной lifecycle operation.
- Self-deactivation после fresh auth требует только explicit confirmation: reason
  code/free text/feedback не запрашиваются. Audit сохраняет actor=user, action,
  interface/provider, request ID и timestamp.
- Final confirmation без dynamic counts перечисляет session/token revocation,
  блокировку account-specific operations, сохранение public CC BY 4.0 tracks/private
  content, private-only parse completion, removal pending review из queue и
  admin-only reactivation через `TRAILBASE_SUPPORT_URL`. Actions:
  «Деактивировать аккаунт»/«Отмена».
- Confirmation expires exactly at `fresh_authenticated_at + 10 minutes` без sliding.
  Chat использует 128-bit opaque ID с server-side
  user/provider/chat/message/requester/purpose binding; web — active session, CSRF и
  user/purpose binding. Confirm/Cancel single-use; invalid/stale/replay ничего не
  меняет и требует нового fresh auth.
- `sensitive_operation_confirmations` находится в PostgreSQL и хранит только SHA-256
  opaque ID, bindings/timestamps и `consumed_at`; raw ID остаётся в control/form.
  Confirm под row lock атомарно consume-ит state, деактивирует account и пишет
  audit/outbox; Cancel только consume-ит state. Cleanup consumed/expired rows — 24
  часа.
- PostgreSQL active status проверяется при каждой session validation и каждом
  auth/link token consume, немедленно закрывая доступ после deactivation commit.
  Confirm transaction пишет idempotent cleanup command без raw tokens; worker удаляет
  account Valkey keys с retry/DLQ. Failure не открывает доступ, backlog/DLQ alert-ит
  operator.
- Дополнительный credential epoch/version отсутствует: deactivation flow удаляет
  account sessions и outstanding auth/link tokens непосредственно из Valkey через
  этот cleanup command.
- Если PostgreSQL недоступен, session/token authorization работает fail-closed без
  cached-active fallback: account-specific HTTP flow возвращает `503` с
  `Retry-After` без session/mutation, а bot event проходит обычный transient
  retry/DLQ без domain changes. Valkey credential сам по себе доступ не даёт.
- Session-validation `503` содержит `Cache-Control: no-store`, не очищает cookie или
  Valkey session и не продлевает sliding TTL/`last_seen_at`. После восстановления
  PostgreSQL ещё не expired session снова пригодна; очистка cookie и revoke
  выполняются только для подтверждённого invalid, expired или disabled состояния.
- Public catalog не показывает badge, причину или иной marker деактивации автора.
  Disabled status видят только сам пользователь через нейтральный auth response и
  администраторы через management/audit.
- In-flight parse деактивированного owner может завершиться только private draft.
  `pending_review` сохраняет status, но исключается из moderator queue; publish
  fail-closed re-check-ит active owner. Реактивация возвращает pending item в очередь
  без resubmit после `export_state = ready`, а draft пользователь отправляет сам.
  Новый `suspended` status не вводится.
- Deactivation и approve/publish используют owner row lock и единый порядок
  `user -> track -> revision`. Первый commit определяет результат: approval-first
  оставляет опубликованный track public после деактивации, deactivation-first не
  допускает публикацию.
- Private chat disabled identity не создаёт новый account. Ему доступны только
  `/start` с нейтральной инструкцией, help/rules/license и read-only public search под
  stateless principal; upload, owned data/settings и auth/link tokens запрещены.
- Встроенного reactivation-request flow нет. Disabled `/start` показывает обязательный
  production `TRAILBASE_SUPPORT_URL`; admin после out-of-band проверки реактивирует
  account в management UI с fresh auth, обязательной причиной и audit. Bot не
  создаёт ticket и не пересылает free-form сообщения.
- Reactivation reason code обязателен:
  `support_request_verified`, `administrative_correction` или `other`. Audit note
  optional, но обязательна для `other`; после trim она должна быть непустой, без
  control/newline characters и не длиннее 1 000 Unicode code points. Code/note
  остаются только в append-only audit и не логируются.
- Reactivation только возвращает account в active: linked identities, roles,
  settings, private content и pending moderation снова доступны. Revoked sessions,
  invalidated auth/link tokens и suppressed notifications не восстанавливаются;
  agreement не меняется, а новый web login инициируется отдельно через bot.
- Изменение правил по ссылке не создаёт agreement records, notifications или domain
  state, не ограничивает account/session capabilities и не требует повторного
  consent.
- Agreement является account-level состоянием по `user_id`. После explicit linking
  новая messenger identity сразу использует capabilities target account; отдельного
  notice/agreement record нет, исходное evidence первого `/start` не меняется.
- `src/trailbase/auth.clj` — token.issue/verify, session cookie set/get, identity bind/link.
- Web-session/link tokens и sessions находятся в Valkey; PostgreSQL хранит users,
  messenger identities, roles и audit.
- `GET /auth` показывает confirmation; `POST /auth` атомарно расходует token и выдаёт
  `__Host-trailbase_session`, но не активирует account.
- Все token-bearing GET/POST outcomes, включая confirmation, success, invalid/expired
  и `503`, имеют `Cache-Control: no-store`, `Referrer-Policy: no-referrer` и только
  same-origin static assets. Caddy/app access/security logs пишут `/auth` без query и
  не пишут form body; raw token/digest отсутствуют в errors, analytics и traces.
- Initial `GET /auth` с token query всегда redirect-only и token не consume-ит. Valid
  branch создаёт existing short-lived auth-flow record/cookie+nonce и отвечает `303`
  на clean `/auth`. Malformed/unknown/expired/superseded branch отвечает `303` на
  token-free `/auth` со stable non-secret `result=invalid`; target GET показывает одну
  generic-invalid page без lookup/изменения existing flow. Plain `/auth` продолжает
  current confirmation; marker не добавляет state/cookie. Body/redirect/history не
  содержат raw token; JavaScript, `history.replaceState`, client storage и новый
  credential namespace отсутствуют.
- Auth-flow record, cookie и form nonce истекают ровно в original `web_session` token
  deadline без нового ten-minute interval или sliding extension от confirmation GET,
  retry/`503`. Expired/missing component показывает generic-invalid page, idempotently
  очищает remaining flow record/cookie и не consume-ит/recover-ит/reissue-ит source
  token.
- Flow record содержит только source token digest, allowlisted `return_to`, original
  deadline и SHA-256 nonce hash. Independent 128-bit flow-cookie ID адресует record по
  SHA-256 cookie value; raw source token после redirect не хранится. Raw flow ID
  остаётся только в `HttpOnly` cookie, nonce — в hidden POST field; bindings читаются
  из source token record, а raw/hash values запрещены в telemetry/DLQ.
- Flow cookie имеет exact name `__Host-trailbase_auth_flow`, attributes `Secure`,
  `HttpOnly`, `SameSite=Lax`, `Path=/`, без `Domain`; `Max-Age`/`Expires` capped remaining
  source-token deadline. Success, terminal invalid/expiry и explicit cancel очищают её
  `Max-Age=0`; `__Host-trailbase_session` остаётся отдельной cookie.
- Новый valid initial auth GET с existing flow-cookie одной atomic Valkey operation
  удаляет previous flow record, создаёт новую и заменяет cookie, оставляя один active
  browser flow. Old tab/form generic-invalid без side effects; previous source token
  остаётся unconsumed/unrevoked и его link можно снова открыть до deadline, снова
  заменив current flow.
- Failed flow-cookie/form-nonce validation на `POST /auth` не consume-ит source token и
  не удаляет valid flow. Missing/unknown cookie и nonce mismatch дают одну
  generic-invalid response без token/pointer/session mutations. Unknown cookie
  очищается у browser через `Max-Age=0`; valid record при wrong/stale nonce и current
  cookie сохраняются. Используется общий `/auth` rate limit 10/min на IP без отдельного
  nonce-attempt counter.
- `/auth` rate-limit reject — retryable `429` с `Retry-After`, не terminal invalid.
  Один budget 10/min на normalized client IP из trusted Caddy forwarding охватывает
  initial token GET, clean confirmation GET и POST. Limiter работает до credential
  lookup, сохраняет flow/token/cookie/nonce/session и отвечает generic `no-store`/
  `no-referrer` body без redirect, automatic retry, terminal marker или отдельных
  per-flow/per-nonce budgets.
- Non-transient invalid `POST /auth` использует PRG: `303` на token-free `/auth` со
  stable query marker `result=invalid`, затем generic-invalid GET без lookup current
  flow. Cleanup зависит от уже принятого error class, поэтому wrong/stale nonce для
  valid record сохраняет flow/cookie. Transient PostgreSQL/Valkey failure остаётся
  retryable `503` с `Retry-After` без redirect, terminal marker или credential
  mutations и сохраняет retryable flow/token/cookie.
- После successful PostgreSQL active identity/account check одна atomic Valkey function
  валидирует flow/nonce, source token и active pointer, promote-ит flow-ID digest в
  session namespace, создаёт либо rotate-ит browser session, consume-ит token/pointer и
  заменяет flow completion receipt как одну credential linearization point; distributed
  PostgreSQL transaction нет.
- Receipt до original flow deadline хранит nonce hash, allowlisted `return_to`, session
  digest/status и expiry без raw ID. Success/matching retry ставит session cookie в тот
  же raw flow-ID value и очищает flow cookie. Concurrent/repeated POST возвращает
  identical success по flow digest, не создаёт вторую и не revoke-ит committed session;
  прежний terminal-invalid concurrent-loser outcome отменён.
- Commit создаёт provisional session и receipt с `PEXPIREAT` original flow deadline.
  Первый request с новой session cookie после PostgreSQL account validation атомарно
  помечает session claimed, удаляет receipt и включает one-year sliding TTL. Без cookie
  оба provisional keys истекают без orphan/janitor; expiry claimed session не revoke-ит,
  а validation `503` сохраняет provisional state до deadline.
- Provisional session скрыта из active-session list и создаёт только low-card auth
  metric. Ordinary-login claim ровно один раз создаёт audit, web-inbox record и
  locked-on primary-bot `new_session` outbox intent, idempotently keyed session digest.
  Same-browser re-auth notification не создаёт; delivery failure не rollback-ит claimed
  session и retry-ится независимо.
- Та же atomic claim function одной Valkey operation делает `XADD` durable
  `session_claimed` event в Stream session Valkey. Event содержит random UUID,
  `user_id`, provider, safe device summary, claim time и internal session digest только
  для idempotency, без raw token, IP и cookie. Worker одной PostgreSQL transaction
  idempotently создаёт audit/web-inbox/outbox с UNIQUE guards по session digest и event
  ID и делает `XACK` только после commit; pending/retry не создают duplicates, delivery
  остаётся независимой outbox-задачей.
- Для `session_claimed` общий terminal DLQ не применяется: transient и deterministic
  failures с operational alert retry-ятся бессрочно с capped backoff/jitter, а unacked
  entry остаётся в PEL и не trim/ack/drop-ится. После PostgreSQL commit worker делает
  `XACK`, затем `XDEL`; cleanup acknowledged остатка можно повторить. Метрики содержат
  только pending count, oldest age и retry count без event/session/user identifiers.
- `session_claimed` использует dedicated Stream и consumer group в session Valkey,
  отдельно от webhook ordering/retention/retry/DLQ. Существующий worker process
  запускает отдельный consumer loop без нового service/container; readiness и
  low-cardinality lag/PEL metrics проверяются отдельно, а общий shutdown lifecycle и
  `XAUTOCLAIM` после 60 секунд обеспечивают failover.
- Projector lag/unhealthy readiness не блокирует новые claims, пока atomic claim +
  `XADD` commit-ятся. Web не ждёт/не poll-ит worker и не ставит PEL age/size gate;
  задержка видна в worker readiness/alerts. Невозможность commit-ить claim + event не
  даёт success и использует ambiguous-outcome recovery без обхода `XADD`.
- Recipient snapshot/target identity в event отсутствуют. Projector под
  user/identity lock order выбирает current primary и одной PostgreSQL transaction
  создаёт audit/web-inbox/locked-on outbox. Порядок commit с primary change определяет
  old/new target; recipient PII/stale snapshot в Valkey не хранится, а deactivated
  account использует обязательную security-notification policy.
- Server-generated UTC `claimed_at` из event сохраняется как immutable `occurred_at`,
  PostgreSQL transaction time — отдельно как `recorded_at`. Inbox/message показывает
  время факта; audit/inbox сортируются по `occurred_at`, затем event ID. `recorded_at`
  служит только projection-lag diagnostics и не подменяет login time.
- Event обязан содержать integer `schema_version = 1` и полный payload strict closed
  Malli schema. Missing/unsupported version или malformed field до DB mutations
  остаётся pending, alert-ит и делает group unready без default/coercion, `XACK` или
  DLQ. Breaking rollout временно поддерживает переходные versions только до drain
  старого PEL, без постоянного compatibility layer.
- Safe device summary — immutable closed map `browser_family|os_family|device_class`
  из maintained server-side UA parser. Catalogs ограничены browser
  `chrome|safari|firefox|edge|webview|other|unknown`, OS
  `android|ios|windows|macos|linux|other|unknown`, class
  `mobile|tablet|desktop|other|unknown`; parse failure даёт `unknown`. Session/event/
  audit/notification используют одну map; raw UA, versions, model, language и IP не
  сохраняются и не логируются.
- `auth_provider = telegram|max` означает immutable source consumed token из validated
  binding, а не request/current primary. Audit/inbox/message показывают этот
  локализованный source; outbox target независимо выбирает current primary. Provider
  user/identity ID отсутствует, unlink/primary change historical fact не меняют.
- Append-only auth audit row — единый PostgreSQL idempotency root с independent UNIQUE
  по `event_id` и `session_digest`; связанные inbox/outbox создаются после неё той же
  transaction. Exact matching replay переиспользует rows. One-guard или payload
  mismatch rollback-ит transaction как integrity incident, оставляет event pending и
  alert-ит без `XACK`; partial independently-deduped projection запрещена.
- `event_id` — application-generated UUIDv7, созданный до first dispatch и сохранённый
  first claim в session claim state/`XADD` для stable retry/read-back. Он является audit
  root primary/idempotency key и FK target inbox/outbox. Stream ID используется только
  transport bookkeeping PEL/`XACK`/`XDEL`, не сохраняется как domain ID; replacement
  DB ID и UUIDv4 отсутствуют.
- Failed/pending event не создаёт global/per-user head-of-line block: bounded pool fair
  schedule-ит новые и due-retry entries, не допуская двух in-flight owners одного
  Stream ID. Failure остаётся в PEL/backoff, alert-ит и держит group unready, но later
  valid events продолжаются. Same-user races сериализует DB user lock; audit/inbox
  order — `occurred_at,event_id`, strict messenger delivery order отсутствует.
- Revoke/logout/expiry до projection/send не подавляют immutable login audit,
  web-inbox или pending `new_session` delivery; projector/dispatcher не перечитывают
  current session. Message показывает occurred time, auth provider, safe device и
  generic «Управление сессиями», без direct target или заявления об active state.
- Timeout/network error после dispatch этой mutating function — ambiguous commit, не
  mutation-free `503`. Handler делает bounded retry/read-back по opaque commit ID;
  success или сохраняющий flow `503` допустим только при подтверждённом результате, а
  прежний `503` — только до dispatch либо при proven non-commit. Для still-unknown
  outcome требуется recoverable branch без terminal cleanup и blind second session.
- Browser re-auth переиспользует тот же bot-issued `web_session` token и `/auth`
  GET/POST без отдельного credential/table/cookie/consume endpoint. Token выдаётся
  только после fresh bot authentication и хранит её `fresh_authenticated_at`; новая
  session rotate-ит текущую session браузера и переносит этот timestamp. Обычные
  requests, `last_seen_at` и sliding TTL не продлевают absolute 10-minute freshness.
- Fresh bot authentication для web-session link — explicit user-initiated
  «Подтвердить вход» command/callback в private one-to-one chat, bound к
  provider/user/chat/message/requester. После validation/acceptance server сразу
  выпускает link и фиксирует `fresh_authenticated_at` injected application UTC clock;
  provider event timestamp не влияет на freshness и отбрасывается после payload
  validation. Event dedupe/binding проверяются отдельно. Ordinary message/
  notification/background/browser activity freshness не создают. Дополнительных
  PIN/password/TOTP и второй confirmation-кнопки нет.
- `web_session` token/session auth state не сохраняет provider event timestamp. В нём
  остаются server `fresh_authenticated_at` и opaque event/identity bindings для
  dedupe/consume validation. В MVP timestamp отсутствует также в operational events,
  logs и metrics; observability использует только server-time counters по provider и
  validation result без отдельной retention policy.
- Fresh-auth telemetry — один counter
  `fresh_auth_confirmation_total{provider,result}`: `provider=telegram|max`,
  `result=accepted|duplicate|invalid_event|invalid_binding|account_unavailable|internal_error`.
  User/chat/event/request IDs, timestamps и дополнительные labels отсутствуют; общие
  HTTP/webhook metrics уже покрывают latency/status.
- `duplicate` ставится только exact replay после committed action acceptance и отвечает
  provider-у `2xx` без нового token/link, bot message edit/send или session/auth
  mutation. Первый commit один раз increment-ит `accepted` и выпускает link;
  `internal_error` до него остаётся worker retry/DLQ, не duplicate.
- Existing 7-day provider-event dedupe record для fresh-auth содержит только
  `processing|accepted`; новой table/key namespace нет. Ingress атомарно claim-ит
  `processing` вместе со Stream enqueue. Replay processing получает только `2xx` без
  metric/side effects, а исходный worker продолжает retry/DLQ; после token/link issuance
  worker ставит `accepted`, и только его replay increment-ит `duplicate`.
- Worker одной atomic Lua/Valkey operation проверяет processing bindings, создаёт ровно
  один 128-bit 10-minute `web_session` token record и ставит accepted до любого Bot API
  send. Crash не разделяет token/accepted; delivery retry использует тот же link и не
  выдаёт новый credential, network call не входит в commit.
- Для той же provider identity эта issuance атомарно заменяет per-identity active-token
  pointer, удаляет previous token record и его ещё существующий raw delivery field.
  Previous provider-event record сохраняет accepted marker; отдельные bot message,
  revoke audit и notification не создаются.
- Previous-link `/auth` consume и new issuance линейризуются atomic Valkey operations
  над pointer. Consume-first проверяет matching token и удаляет record/pointer, после
  чего auth может завершиться; issuance-first заменяет pointer и делает old link
  terminal invalid. Поздняя issuance не отзывает уже создаваемую/созданную browser
  session; distributed Valkey/PostgreSQL transaction отсутствует.
- Active-token pointer хранится отдельным non-secret Valkey key по internal identity
  UUID, содержит только SHA-256 token digest и получает `PEXPIREAT` exact token
  deadline в atomic issuance. Consume удаляет его с token record, replacement заменяет;
  missing/expired pointer или target terminal invalid и очищается idempotently.
  Delivery data, sliding TTL и janitor отсутствуют.
- `web_session` token record адресуется только по `SHA-256(raw_token)`, тому же digest,
  что pointer; raw token в record/key не хранится. При 128-bit CSPRNG entropy
  используется unsalted SHA-256 без custom HMAC/salt. Raw value остаётся только в
  user-facing link/browser request и short-lived delivery field; raw token и digest
  запрещены в logs/metrics/traces/DLQ.
- Raw link token для crash-safe retry существует только как short-lived delivery field
  existing accepted dedupe record до successful send или exact token expiry. Success
  удаляет field атомарно; seven-day accepted marker затем содержит лишь non-secret
  outcome/bindings. Raw token отсутствует в logs/metrics/traces/DLQ, отдельного
  namespace и deterministic token derivation нет.
- Runtime minimum — Valkey 9.x. Atomic issuance ставит delivery field
  `HPEXPIREAT` на absolute token deadline; successful send выполняет idempotent `HDEL`.
  Accepted marker сохраняет key-level seven-day TTL; janitor и отдельный TTL key не
  создаются.
- Если Bot API delivery не завершилась до token/delivery-field expiry, worker
  прекращает её terminally без stale send, re-issuance, изменения accepted marker или
  отдельного user notification. Exact replay остаётся `duplicate`; только новое
  explicit «Подтвердить вход» action создаёт новый provider event/freshness/token/link.
  Failure отражается только общими Bot API delivery metrics/alert без raw token.
- Post-issuance delivery имеет максимум пять total attempts с exponential
  backoff/jitter. Retry разрешён только для timeout/network, `429` и `5xx`; следующая
  attempt и `Retry-After` должны укладываться в original token deadline. Остальные
  `4xx`, exhausted budget и retry за deadline terminal сразу: idempotent `HDEL`,
  accepted marker сохраняется, DLQ/late replay отсутствуют. Pre-issuance
  `internal_error` остаётся в общем worker retry/DLQ flow.
- Любая active linked Telegram/Max identity может выпустить re-auth link; primary
  остаётся только delivery route. Token bound к internal identity UUID/provider/user/
  event, а `/auth` consume re-check-ит active membership и account status. Unlinked
  или foreign identity terminally инвалидирует token без primary fallback; successful
  freshness account-level.
- Если `/auth` видит valid current browser session того же user, rotation является
  credential refresh: новые token/CSRF/freshness заменяют current session, остальные
  sessions не меняются, `new_session` web-inbox/bot notification не создаётся. Без
  valid same-user current session commit создаёт provisional ordinary new login, а
  locked-on security notification появляется только при claim.
- Same-browser rotated session сохраняет original `created_at`, обновляет token hash,
  CSRF, bot-derived `fresh_authenticated_at`, current `last_seen_at` и safe short
  User-Agent summary; one-year sliding TTL начинается после provisional claim.
  Отдельного `reauthenticated_at` нет. Ordinary new login получает новый `created_at`.

**Acceptance**:
- Первый валидированный Telegram `/start` в private one-to-one chat для неизвестной
  identity атомарно создаёт active account и identity; повторные или конкурентные
  deliveries идемпотентно resolve в тот же account.
- Первый plain `/start` атомарно создаёт account, identity и один agreement record;
  повторный или конкурентный delivery не создаёт дубликат.
- Повторный `/start` оставляет agreement record byte-for-byte неизменным, но может
  обновить provider profile snapshot и заново показать notice/menu.
- После деактивации и административной реактивации исходный agreement record остаётся
  неизменным; повторное agreement не требуется.
- Деактивация не скрывает и не удаляет published tracks: они остаются public с
  прежним author display name и CC BY 4.0 attribution; private content не становится
  публичным, а track removal выполняется отдельной lifecycle operation.
- Self-deactivation UI не содержит reason или feedback field; audit record не
  содержит пользовательский free text.
- Confirmation показывает все принятые последствия и exact primary/secondary
  actions, но не вычисляет counts, способные устареть до mutation.
- Confirmation callback после original fresh-auth deadline, с неверным binding или
  после первого Confirm/Cancel не меняет account. Web mutation без valid CSRF/state
  также fail-closed.
- Concurrent/replayed Confirm не может дважды выполнить domain mutation: PostgreSQL
  row lock и `consumed_at` дают единственный winner; confirmation rows старше 24
  часов после terminal/expiry удаляются janitor.
- Даже до физического Valkey cleanup session/token deactivated account не
  авторизуется. Cleanup command повторяем и не содержит credentials; DLQ не меняет
  committed account state.
- При недоступном PostgreSQL Valkey session или token не авторизует operation:
  HTTP получает `503` с `Retry-After` без session/mutation, bot event повторяется как
  transient failure и при исчерпании попыток попадает в DLQ без domain changes.
- Session-validation `503` не выглядит как logout: cookie и Valkey session
  сохраняются, `last_seen_at`/sliding TTL не обновляются, ответ имеет
  `Cache-Control: no-store`, а после восстановления PostgreSQL та же неистёкшая
  session снова работает.
- Public response для такого track не раскрывает деактивацию автора; account status
  доступен только самому пользователю и администраторам.
- Деактивация не допускает новую публикацию: parse может завершить private draft,
  `pending_review` временно отсутствует в moderator queue, а approve/publish
  отклоняется до реактивации.
- Concurrent deactivation/approval детерминирован row lock: только approval,
  committed до deactivation, может опубликовать revision; обратный порядок не меняет
  public state.
- Disabled identity в private chat resolve-ится в прежний account, видит ограниченное
  static/read-only меню и не получает account-specific данные или mutation controls.
- Production без valid absolute HTTPS `TRAILBASE_SUPPORT_URL` не стартует. Disabled
  `/start` показывает эту ссылку; единственная reactivation mutation остаётся в admin
  management UI и создаёт locked-on lifecycle notification.
- Reactivation с unknown/missing reason code, missing `other` note или note длиннее
  1 000 Unicode code points отклоняется без изменения account; accepted code/note
  находятся только в audit.
- После reactivation следующий validated private-chat event показывает обычное меню
  и сохранённые account capabilities. Ни одна прежняя web session/token/notification
  не воскресает; lifecycle notification не содержит auth token.
- Reactivation notification не раскрывает admin identity/reason/audit ID и содержит
  только timestamp, support URL и безопасную инструкцию вернуться в bot.
- После реактивации смена `users.display_name` сразу меняет attribution всех старых
  public track pages. Ранее сгенерированный GPX не переписывается; новый export
  содержит имя, актуальное на момент генерации.
- Ответ на plain `/start` показывает информацию о боте, главное меню, rules/CC BY 4.0
  links и явно сообщает, что использование бота означает автоматическое согласие.
  Кнопки «Согласен» и отдельного consent callback нет.
- Главное меню Telegram и Max показывает ровно пять основных действий: «Поиск»,
  «Загрузить GPX», «Мои треки/черновики», «Настройки» и «Помощь»; правила и лицензия
  не добавляют отдельные пункты в это меню.
- «Настройки» показывает ровно пять разделов: «Профиль», «Мессенджеры»,
  «Уведомления», «Сессии», «Аккаунт»; recovery credentials/channels не предлагаются.
- Каждый раздел «Настроек» выполняет разрешённые операции без web session в private
  chat; sensitive operations сохраняют fresh-auth/confirmation guards, а web UI не
  является обязательным переходом.
- Valid display name применяется сразу и не перезаписывается provider snapshot;
  invalid/empty/over-200 input отклоняется, каждое изменение audit-ится.
- Ручной выбор `ru`/`en` одинаково действует в Telegram/Max и web и сохраняется при
  последующих provider events; `simple` отсутствует в locale selector.
- «Уведомления» не позволяет отключить security или action-required moderation, но
  позволяет изменить catalog/informational/other moderation preferences.
- Новый account сразу показывает moderation-result notifications как enabled, а
  catalog/informational как disabled.
- Изменение optional toggle одинаково управляет будущими web-inbox и primary-bot
  notifications; отдельных channel toggles UI не показывает.
- Уже созданная notification/outbox delivery завершается по transaction-time
  preference snapshot, даже если пользователь затем изменил toggle; deactivation
  suppression является отдельным lifecycle exception.
- Для deactivated account создаются только security/account-lifecycle notifications;
  moderation/catalog/informational delivery отсутствует и не воспроизводится после
  реактивации, но соответствующие domain events/audit не теряются.
- Pending non-security delivery при деактивации становится cancelled, связанный unread
  inbox record — suppressed и невидимым после реактивации. Уже `sending` delivery
  разрешено завершиться; отзыва отправленного сообщения нет.
- Chat session card показывает safe device/browser summary, `created_at`,
  `last_seen_at`, current expiry и «Завершить». IP/geolocation/full User-Agent/session
  ID/token отсутствуют; target хранится только в bound server-side control state.
  Метки «текущая» в chat нет.
- Revoke одной session требует short confirmation с тем же summary без fresh auth.
  Confirm атомарно re-check-ит owner/target; already-ended result идемпотентен.
  `logout-all` остаётся отдельной fresh-auth operation.
- Chat `logout-all` после fresh auth/confirmation атомарно завершает все active web
  sessions account и сообщает actual revoked count. Messenger identities, chat
  access и account state не меняются.
- «Мои треки/черновики» без web session возвращает единый список owned entries,
  показывает их нормализованный статус и сортирует требующие действия пользователя
  раньше остальных.
- Default list не содержит archived tracks; filter «Архив» показывает срок до purge и
  позволяет восстановление в пределах 30 дней, но не досрочный permanent delete.
- UI не показывает дополнительных status filters кроме «Все» и «Архив».
- Restore user-archived track сохраняет `track_id`, не создаёт новую revision и
  возвращает pre-archive status/current revision; moderator-removed track через
  «Архив» восстановить нельзя.
- Одно valid нажатие «Восстановить» завершает restore без дополнительного prompt;
  повторная доставка идемпотентна, а invalid/expired callback не меняет track.
- Карточка owned entry показывает status-dependent actions; `pending_review` остаётся
  read-only, а delete/archive mutation невозможна без отдельного confirmation.
- Переходы «Назад»/«Далее» показывают не более 10 entries, используют keyset
  pagination и не принимают callback с несовпадающим
  provider/chat/message/requester binding.
- Перелистывание не продлевает исходный 15-минутный TTL; после expiry старое
  list message остаётся неизменным, а bot предлагает открыть новый список.
- В Telegram/Max «Правила»/«Лицензия» являются обычными HTTPS-ссылками; rules URL
  показывает актуальный текст без version-specific account behavior.
- Durable agreement record фиксирует notice hash, показанные URLs, `accepted_at` и
  `/start` source без ephemeral callback state.
- Любое изменение текста правил не требует нового agreement, не создаёт notification
  и не блокирует существующие browser или chat capabilities.
- Созданная Telegram identity может выполнять разрешённые chat commands без web
  session.
- Telegram bot `/start` может предложить optional browser deep-link; confirmation POST
  создаёт browser session, но не является условием account/chat access.
- `/start <link-token>` от второго provider атомарно привязывает candidate identity к
  target account и не создаёт отдельный account; browser completion не требуется.
- Привязанная identity не создаёт вторую запись `user_agreements`; agreement и
  capabilities принадлежат target `user_id`.
- Identity, уже принадлежащая другому account, не переносится; автоматического merge
  accounts нет.
- Невалидный, просроченный, использованный или provider-mismatched link token не
  создаёт account и не fallback-ит на обычный `/start`; plain `/start` без payload
  остаётся отдельным явным созданием account.
- В group/channel contexts `/start`, выпуск tokens и linking не создают и не изменяют
  account/identity state; bot без auth token направляет пользователя в private chat.
- Max bot поддерживает тот же identity/chat contract и optional `/auth` endpoint,
  identity `max:*`.
- Session cookie проверяется middleware; запрос на web UI без cookie предлагает вход
  через Telegram/Max.
- Logout текущей сессии, logout-all и active sessions management работают в web UI и
  private chat; claimed sessions имеют sliding TTL один год, provisional session —
  exact auth-flow deadline до первого validated request.
- Session list в chat не раскрывает IP или identifiers и позволяет выбрать конкретную
  session по безопасному summary через opaque bound control.
- Одна session не revoke-ится первым нажатием: confirmation требуется, fresh auth —
  нет; stale/already-ended target не затрагивает другие sessions.
- Подтверждённый `logout-all` завершает все web sessions, существующие в момент
  mutation, но не unlink-ит messenger identity и не деактивирует account.
- Email, phone, password и recovery flow отсутствуют.

**Non-goal**: фактическая загрузка и поиск треков — M03 и M06.

---

## M03 — GPX Upload + Parse

**Объём**: web/bot upload → exact original private raw в S3 → async parse job → permanent
private draft → moderated immutable revision.

**Файлы/компоненты**:
- `src/trailbase/storage/s3.clj` — aws-simple-sign подпись + hato put/get-object + presigned URL.
- `src/trailbase/parse/gpx.clj` — hardened pull-parser → 2D MultiLineString,
  elevation profile, duration variants, metrics и warnings.
- `src/trailbase/tracks.clj` — upload flow (write S3 + insert DB), simplify-geometry (compute z11/13/15 в PostGIS-side helper или Clojure-side `simplify`).
- M03 migration добавляет `raw_objects` как durable owner/storage entity и
  `raw_object_id` foreign keys в `upload_jobs` и `track_revisions`; snapshots не
  дублируют S3 storage fields.
- Та же migration добавляет nullable `track_revisions.base_revision_id`. First
  publication использует `NULL`; update published track сохраняет current revision
  при создании. Composite same-track FK `(track_id, base_revision_id)` ссылается на
  `(track_id, id)`, DB CHECK запрещает self-reference. Approval под track lock требует
  совпадения `tracks.current_revision_id` с baseline; mismatch оставляет revision
  stale без mutation/audit или automatic rebase.
- Та же migration добавляет nullable `track_revisions.correction_of_revision_id` с
  composite same-track FK на `(track_id, id)` и запретом self-reference. Поле
  обязательно только для авторского resubmit после `changes_requested`, указывает на
  непосредственно возвращённую immutable revision и равно `NULL` для остальных
  revisions; resubmit сохраняет её `base_revision_id`.
- Assignment schema для обычной track moderation не создаётся: у revision/queue нет
  `assigned_to`, claim lease или отдельной claim table.
- Та же migration добавляет
  `track_revisions.submitted_for_review_at timestamptz NOT NULL`, атомарно
  устанавливаемый при submit immutable revision и неизменяемый после него. Queue
  index поддерживает порядок `(submitted_for_review_at, id)`.
- Та же migration добавляет `track_issues` с `track_id`, `code text NOT NULL`,
  `subject_type text NOT NULL`, `subject_id UUID NOT NULL`, admin-only
  `detail text NOT NULL`, `detected_at`, `last_seen_at` и `resolved_at`;
  `char_length(detail) BETWEEN 1 AND 1000` защищён DB CHECK, а application пишет
  только code-specific безопасные шаблоны без raw exceptions, HTTP
  headers/provider body, GPX content или credentials. Detail не использует `jsonb`;
  машинная семантика остаётся в `code`. Шаблоны образуют закрытый каталог с явным
  whitelist подстановок; в MVP разрешены только тип mismatch и точно известные
  expected/observed byte counts. Oversized read, остановленный на `expected + 1`,
  сообщает только lower bound observed size. `subject_id`, object key/bucket,
  filename, digest, provider response/status, exception и operator free text не
  подставляются; ручной текст остаётся append-only audit note. Rendered detail
  хранится только на canonical English, не зависит от `ru`/`en` UI locale и не
  переписывается при её смене; admin UI локализует labels/actions, но показывает
  stored diagnostic text. `code` имеет DB CHECK только на `raw_object_missing`,
  `raw_integrity_mismatch`,
  `sanitized_export_missing` и `snapshot_integrity_unknown`, без PostgreSQL enum.
  Отдельной scope column нет,
  blocked capabilities выводятся из закрытого mapping:
  `raw_object_missing|raw_integrity_mismatch -> reparse`,
  `sanitized_export_missing -> download+publication_switch`,
  `snapshot_integrity_unknown -> full_track_lock`. Unknown code отклоняется, новый
  требует migration существующего CHECK, checker, subject constraint и mapping.
  `subject_type` имеет CHECK только на `track|raw_object|revision`, без PostgreSQL
  enum. Compound CHECK допускает только
  `raw_object_missing|raw_integrity_mismatch` с `raw_object` и
  `sanitized_export_missing|snapshot_integrity_unknown` с `revision`; начальный
  набор не допускает `track`, а первый track-wide code расширяет code CHECK и pair
  CHECK.
  Partial unique index по
  `(track_id, code, subject_type, subject_id)` для active rows не допускает duplicate
  incident; `has_problems` вычисляется через active rows, а не хранится mutable
  boolean в `tracks`. Track-wide issue использует `track`/`track_id`, raw —
  `raw_object`/`raw_objects.id`, revision/export — `revision`/`track_revisions.id`;
  nullable subject/sentinel не используются. `track_id` имеет FK, polymorphic
  `subject_id` FK не имеет, не pin-ит subject и не считается storage reference;
  membership проверяется checker-ом под track lock. Пока track существует, purge
  raw/revision может оставить historical subject UUID; physical purge track удаляет
  все active/resolved issue rows. CHECK для `track` требует
  `subject_id = track_id`; новый type требует migration CHECK и application keyword.
- `track_revisions.raw_object_id` для GPX-derived snapshots использует
  `ON DELETE RESTRICT`; nullable `upload_jobs.raw_object_id` — `ON DELETE SET NULL`.
  Job FK existence само по себе не является retention pin.
- Та же migration добавляет partial covering quota index по
  `raw_objects(owner_id, state)` для `pending|ready` с included
  `byte_size`; counter columns в `users` отсутствуют.
- PostGIS-функция в `003-tracks-geom.up.sql` (migratus) —
  `populate_simplified_geometries(track_revision_id)` заполняет три revision columns.
- Activity-auto-suggestion: рудиментарный classifier показывает до трёх ranked
  вариантов с confidence; пользователь обязан выбрать activity явно.
- Canonical duration default выбирается как GPX moving, затем GPX elapsed, затем
  unknown. Summary показывает оба рассчитанных значения и выбранный source; manual
  override или explicit unknown не удаляет auto-derived values.
- Manual duration — целые секунды `1..31_536_000`; unknown хранится как `NULL`, а не
  zero. Outlier comparison использует moving, иначе elapsed; симметричный ratio
  строго больше 10 требует warning confirmation, но остаётся допустимым. Без auto
  candidate warning отсутствует.
- Durable outlier acknowledgement в PostgreSQL привязано к manual value, comparison
  source/value и algorithm version. Mutation проверяет lease generation; resume и
  takeover сохраняют acknowledgement, а duration change/reparse/version change
  инвалидируют. Valkey содержит только prompt/control token.
- Track moderation summary показывает acknowledged duration outlier: manual value,
  comparison auto value/source и ratio. Flag не вызывает automatic moderation
  decision, не меняет `submitted_for_review_at` или FIFO position.
- Moderator не редактирует pending track revision: approve as-is либо
  `changes_requested` с одним `reason_code text NOT NULL`, DB CHECK на
  `metadata_correction|geometry_correction|privacy_correction|classification_correction|license_or_attribution|other`
  и optional owner-visible `reason_note`. Note обязателен для `other`, после trim
  содержит 1–1000 Unicode code points без control/newline и не логируется. Array,
  checklist relation и PostgreSQL enum отсутствуют; несколько проблем перечисляются
  в note. Авторская правка создаёт новый immutable revision, повторно запускает
  validation/outlier acknowledgement и может переиспользовать raw GPX;
  administrative repair остаётся отдельной audited operation.
- Первая публикация получает full-snapshot moderator review. Для новой revision уже
  опубликованного track review по умолчанию показывает field diff и overlay старой и
  новой geometry относительно сохранённого `base_revision_id`; полный pending
  snapshot открывается отдельным действием. Любой moderation outcome применяется ко
  всей immutable revision, partial approval нет.
- Review авторского resubmit после `changes_requested` вместо этого по умолчанию
  показывает correction diff относительно `correction_of_revision_id` и сохранённые
  correction code/note. Total diff относительно `base_revision_id` и полный pending
  snapshot доступны как вторичные views; решение по-прежнему применяется ко всей
  revision.
- Raw GPX сохраняется без application encryption или преобразования: private S3
  object содержит exact original upload bytes. Data/master keys, crypto envelope,
  keyring, rewrap и decrypt path для raw отсутствуют.
- Transparent provider-side SSE или disk encryption raw storage допустимы как
  deployment control: TrailBase не управляет provider keys или key IDs и после GET
  получает те же exact original bytes без application decrypt path.
- Raw доступен только backend workers для parse/re-parse. Public и owner-only
  download routes, presigned raw URL и HTTP streaming endpoint в MVP отсутствуют;
  пользователь скачивает только опубликованный sanitized GPX.
- После успешного parse raw хранится без отдельного TTL до физической очистки
  связанного draft/track, включая published и archive/appeal retention, и всё время
  учитывается в user raw-storage quota. Подтверждённое удаление draft или purge track
  удаляет raw; 24-hour janitor касается только incomplete/orphan objects.
- Каждый новый upload создаёт новый immutable exact raw object без content-based,
  cross-track или cross-user dedup. Re-parse/metadata revision того же track без
  нового upload переиспользует durable reference на существующий object, не копирует
  source bytes и не увеличивает quota; object удаляется после последней reference.
- `raw_objects` row владеет `owner_id`, opaque S3 key, exact `byte_size`, private
  SHA-256, lifecycle state и timestamps. Digest вычисляется при streaming upload,
  проверяется при последующих чтениях/re-parse и не участвует в dedup или API.
- Mismatch уже `ready` raw завершает текущий parse/re-parse permanent
  `raw_integrity_mismatch` без automatic retry и без revision/publication mutation.
  Новый raw state не добавляется; row/object сохраняется по обычному retention,
  operator alert содержит только `raw_object_id`, а следующая попытка снова
  проверяет digest. Если восстановить object нельзя, пользователь загружает source
  заново.
- Application/admin repair flow отсутствует. Infrastructure operator может
  восстановить под тем же opaque key только bytes с исходным SHA-256 из MinIO
  versioning/off-host backup; digest не меняется. Без совпадающей копии новый upload
  создаёт новый `raw_object_id`, а другой source нельзя подставить в старую row.
- После primary purge raw сохраняется только в encrypted operator-only off-host
  backup максимум 30 дней, недоступен приложению и не может быть восстановлен в
  active service. После disaster restore при остановленных web/workers operator
  запускает CLI reconciliation до открытия traffic.
- CLI перечисляет все TrailBase-managed S3 keys и durable S3 references
  восстановленной PostgreSQL. Key без единой DB reference является orphan и
  удаляется со всеми versions/delete markers; referenced key не удаляется.
  Manifest, pending approval state и reconciliation rows не хранятся.
- CLI default выполняет dry-run и показывает counts/total bytes orphan keys по
  storage class; DB/S3 scan error возвращает nonzero exit. Удаление требует explicit
  destructive flag и нового полного DB/S3 scan при всё ещё остановленных
  web/workers; результат dry-run не сохраняется.
- Dry-run не пишет markers. Destructive run удаляет orphan keys и durable помечает
  все referenced-but-missing objects. Полный успешный scan/cleanup с однозначными
  track markers разрешает открыть traffic в degraded mode; scan error или unmappable
  violation возвращает nonzero и оставляет services закрытыми.
- Missing referenced S3 key не удаляет DB reference. CLI устанавливает на связанном
  track durable problem marker с admin-only reason code/пояснением. Этот же marker
  используется для других track data-integrity problems: administrator видит точную
  причину, owner — только нейтральный факт проблемы с записью.
- Marker блокирует reason-scoped capabilities, а не весь track: missing raw запрещает
  re-parse, missing sanitized export — download/publication switch, но исправная
  geometry и metadata view остаются доступны. Full lock используется только для
  snapshot-wide integrity reason. Он блокирует normal content serving,
  owner/content и publication mutations, restore, approve и content-dependent
  moderation. Allowlist: minimal status с generic owner warning, owner archive,
  moderator removal/hide, scheduled purge, admin issue/history read,
  detector-specific recheck/resolution и append-only audit; containment не читает и
  не публикует snapshot.
- Full-lock owner projection содержит только `track_id`, lifecycle `status`,
  `has_problems = true`, `purge_at` для archived track, localized generic warning и
  доступное действие «Архивировать». Карточка использует generic title/short track ID
  без revision read. Name/descriptions, geometry, metrics, POI, classification,
  author, `current_revision_id`, export/raw IDs, issue semantics и blocked capability
  list отсутствуют; archived card скрывает restore и показывает purge deadline.
- Blocked owner mutation получает terminal `409 Conflict`, `Cache-Control: no-store`
  и JSON `{"error":"track_temporarily_unavailable"}` либо localized generic HTML
  partial. Bot даёт neutral acknowledgement, не меняет domain state и обновляет
  minimal card; domain rejection не retry-ится/DLQ-ится. `423`, `Retry-After`, issue
  details и capability list отсутствуют; `503` остаётся public/infrastructure status.
- Moderator без admin issue permissions видит full-locked track в существующем
  moderation surface как snapshot-free placeholder без новой queue: только
  `track_id`, lifecycle `status`, `has_problems = true`, generic warning и
  removal/hide с обязательной причиной. Preview/approve/changes-requested,
  export retry/download и content/revision fields скрыты. Issue history/recheck
  требуют отдельных admin permissions, которые не обходят content lock.
- Stale/direct moderator content mutation после auth/permission/visibility checks
  получает тот же terminal `409`/`no-store`/safe JSON error либо generic HTML, не
  создавая domain mutation или audit decision; UI обновляется до placeholder. `403`
  означает missing permission, `404` — invisible/removed resource, `503` — infra
  failure. Capability rejection не retry-ится/DLQ-ится и не раскрывает issue.
- Успешный removal/hide как containment атомарно создаёт обычный moderation audit и
  locked-on owner notification/outbox; это lifecycle notification без full-lock или
  issue details, а detector/recheck notification не создают.
- Начальный mapping закрыт:
  `raw_object_missing|raw_integrity_mismatch -> reparse`,
  `sanitized_export_missing -> download+publication_switch`,
  `snapshot_integrity_unknown -> full_track_lock`. Unknown code не принимается.
- `code` — `text NOT NULL` с закрытым DB CHECK
  на `raw_object_missing`, `raw_integrity_mismatch`, `sanitized_export_missing` и
  `snapshot_integrity_unknown`, не PostgreSQL enum. Новый code добавляется
  migration-ой существующего CHECK одновременно с checker, subject constraint и
  capability mapping.
- Несколько active `track_issues` не перезаписывают друг друга; effective blocked
  capabilities являются union code mappings; отдельной mutable scope column нет.
  Из issue-related данных owner получает только derived `has_problems` и generic
  warning, administrator — все codes/details/timestamps; full-lock projection ниже
  дополнительно ограничивает весь field set.
- `subject_id` всегда UUID NOT NULL: track-wide issue повторяет `track_id`, raw
  ссылается идентификатором raw object, а revision/export — revision ID. Поэтому
  partial unique active identity не имеет NULL loophole. Subject UUID является
  historical identity без FK/retention semantics; checker до issue mutation
  проверяет существование и принадлежность track.
- `subject_type` — text с закрытым DB CHECK
  `track|raw_object|revision`, не PostgreSQL enum. Unknown value отклоняется; track
  type дополнительно требует `subject_id = track_id`.
- Compound DB CHECK допускает только пары
  `raw_object_missing|raw_integrity_mismatch`/`raw_object` и
  `sanitized_export_missing|snapshot_integrity_unknown`/`revision`. Ни один
  начальный code не допускает `track`; первый track-wide code требует одной
  migration code CHECK и pair CHECK.
- Admin-only `detail` — `text NOT NULL` с DB CHECK
  `char_length(detail) BETWEEN 1 AND 1000`, не `jsonb`. Application формирует его
  только из code-specific безопасных шаблонов; raw exceptions, HTTP
  headers/provider body, GPX content и credentials не сохраняются. Машинная
  семантика остаётся в `code`. Каталог шаблонов закрыт; каждый шаблон имеет явный
  whitelist. В MVP подставляются только тип mismatch и точно известные
  expected/observed byte counts; остановка oversized read на `expected + 1`
  сохраняет lower bound, не точный observed size. `subject_id`, object key/bucket,
  filename, digest, provider response/status, exception и operator free text не
  подставляются. Ручные пояснения администратора остаются append-only audit notes.
  Rendered detail хранится только на canonical English, независимо от `ru`/`en` UI
  locale, и не переписывается при её смене. Admin UI локализует labels/actions, но
  показывает stored diagnostic text.
- Issue закрывает только detector-specific successful recheck: storage checker
  проверяет object/integrity metadata, другие codes — свой validator. Administrator
  может запустить check и оставить append-only audit note, но не force-set
  `resolved_at`; resolved rows сохраняются до physical purge owning track. Purge
  удаляет active/resolved issues без fake resolution; issue rows не являются
  retention pin и не блокируют purge. Tombstone track UUID и существующий audit
  остаются, но codes/details/subject UUID в новый audit event не копируются.
  Отсутствие user notifications не меняет это правило.
- Resolution последней active full-lock issue автоматически пересчитывает effective
  block, меняя только `track_issues.resolved_at`. `tracks.status`,
  `current_revision_id` и moderation records не меняются; если lifecycle state всё
  ещё разрешает доступ, прежний immutable snapshot снова доступен без новой
  moderation/publication operation. Archive/moderator removal сохраняются, а другие
  active issues продолжают свои блокировки.
- Повторный detection active identity обновляет `last_seen_at`/admin detail,
  сохраняя первый `detected_at`; после resolution recurrence создаёт новый incident
  ID. Изменения issue state не создают web-inbox records или messenger deliveries.
  Из issue-related данных owner получает только текущее derived `has_problems` и
  generic warning в track views; codes/details/history остаются admin-only.
- Все issue-state transactions сериализуются `SELECT ... FOR UPDATE` на `tracks`
  row. Standalone operation блокирует только track; если owner lock нужен по другой
  причине, применяется порядок `user -> track`, без advisory locks. Partial unique
  active-identity index остаётся DB backstop.
- Issue commit linearize-ит активацию full lock. Canonical single-track
  detail/download держат конфликтующий `FOR SHARE` до response decision/signing,
  mutations — `FOR UPDATE` до commit; оба повторно проверяют issues под lock.
  Победившая до detector transaction operation завершается как pre-lock, после issue
  commit новые operations видят block. Collection queries используют statement
  snapshot без per-track locks и могут завершить начатый до commit response; уже
  начатые HTTP/S3 responses не отменяются.
- Preliminary scan только выбирает candidate. Финальный detector-specific check
  повторяется после `tracks FOR UPDATE` и меняет issue в той же transaction;
  timeout/error выполняет rollback без изменения issue state. Под lock разрешена
  одна attempt с hard deadline 10 секунд; retry/backoff внутри transaction
  отсутствует, повторная operation заново берёт lock. Отдельные retry rows, stream и
  scheduler не создаются: CLI/admin сообщают safe failure, operator/administrator
  повторяет operation явно, а attempt остаётся только в logs/metrics.
- Checker имеет три результата: conclusive `problem_present` создаёт issue либо
  обновляет active `last_seen_at`/safe admin detail; `healthy` ставит `resolved_at`;
  `inconclusive` откатывает transaction без issue mutation. User notifications не
  создаются.
- S3 `404`/`NoSuchKey` и completed-read SHA-256/size mismatch дают
  `problem_present`; совпавшие invariants — `healthy`. Auth/permission/`429`,
  DNS/TLS/network/timeout, `5xx`, truncated/unexpected response дают `inconclusive`,
  operational alert и rollback без track issue mutation.
- Raw `healthy` требует полного streaming GET с пересчётом SHA-256 и size против
  `raw_objects`, без full-body memory buffer. `HEAD` только выбирает missing
  candidate; `ETag`/custom metadata не resolve-ят integrity issue и не заменяют
  private digest. Reader останавливается на stored `byte_size + 1`: extra byte
  conclusive size mismatch, остаток object не скачивается; SHA сравнивается только
  после clean EOF ровно на expected size. Clean shorter body — conclusive size
  mismatch; transport abort/broken framing/premature EOF — `inconclusive`.
  `upload_jobs` и `track_revisions` хранят только `raw_object_id`; quota и cleanup
  работают по этой durable entity.
- Retained revision всегда pin-ит raw. Upload job pin-ит его только во время
  продолжимого upload/parse либо explicit transient retry до 24-hour incomplete
  deadline. Successful job передаёт pin revision; cancel/permanent failure снимает
  его немедленно. Историческая terminal job row не блокирует cleanup.
- Reuse и last-reference cleanup берут locks `users -> raw_objects FOR UPDATE`.
  Reuse проверяет owner, same track lineage и `ready`; cleanup под lock повторяет pin
  query перед `delete_pending`+outbox. State не оживляется: cleanup-first означает
  reject позднего reuse и новый upload.
- Lifecycle raw row ограничен `pending`, `ready`, `delete_pending`: row создаётся до
  PUT, revision принимает только checksum-validated `ready`, а потеря последней
  reference либо 24-hour incomplete cleanup атомарно ставит `delete_pending` и
  cleanup outbox command. После успешного S3 delete row удаляется; upload errors
  остаются в `upload_jobs`, delete retries/DLQ — в cleanup command.
- Quota contribution: новый `pending` атомарно резервирует 10 MiB независимо от
  provider/HTTP size metadata; `ready` считает actual byte size;
  `delete_pending` — zero с commit transition. Terminal upload failure ставит
  `delete_pending` сразу, 24-hour janitor остаётся fallback. Cleanup lag/DLQ не
  расходует user quota.
- Raw cleanup outbox payload содержит только `raw_object_id`. Worker загружает
  `delete_pending` row, берёт opaque S3 key, пагинированно перечисляет и удаляет по
  version ID все versions/delete markers exact key. Prefix matches других keys не
  удаляются. Только empty exact-key listing разрешает удалить row; missing
  version/object/row при replay — success. Wrong state или retained revision
  fail-closed проходят retry/DLQ с alert; crash закрывается повторной enumeration.
- Quota-changing transaction берёт owner `users` row `FOR UPDATE`, выполняет indexed
  SQL sum contributions из `raw_objects` и вставляет `pending` в той же transaction.
  `users.raw_bytes_*` counters и reconciliation отсутствуют.
- Private sanitized export pre-generate-ится для immutable pending revision.
  Approve доступен только при durable `export_state = ready` и в одной transaction
  переключает revision/track в published/current. Export failure оставляет
  `pending_review`, не меняет старый current и проходит retry/DLQ; public
  `publishing` status отсутствует.
- До approval private sanitized object доступен только backend workers. Owner и
  moderator работают через draft/review UI без presigned URL или отдельного
  authenticated download route. Download появляется только после publication через
  canonical route.
- Обычная moderator queue показывает только `pending_review` revisions active owners
  с `export_state = ready`. `pending|error` items, export badges и retry action из неё
  исключены. Pending остаётся technical progress; exhausted error попадает в
  отдельный operator/admin operations view с idempotent retry и alert. Owner видит
  обычный `pending_review` без infrastructure details. Snapshot-free full-lock
  containment остаётся отдельным surface только с removal/hide и не зависит от
  queue eligibility.
- Queue общая для всех moderators с permission и не имеет assignment, claim lease,
  heartbeat, hand-off или состояния «в работе». Decision transaction повторно
  проверяет eligibility/invariants под row locks; первый commit сохраняет actual
  actor в audit, а конкурентный submit получает deterministic conflict без второго
  decision, audit или notification.
- Currently eligible items идут строго FIFO по
  `submitted_for_review_at ASC, track_revisions.id ASC` с keyset pagination по той же
  паре. Priority/escalation field, manual bump, SLA buckets и resubmit priority не
  вводятся; containment и export-error operations остаются отдельными surfaces.
- Moderation UI принимает решение только по одной revision: без checkboxes, select
  all, bulk endpoints, массивов IDs или batch jobs для approve,
  `changes_requested`/removal. Успешный decision и конкурентный conflict возвращают
  moderator-а в обновлённую общую FIFO queue; audit/outbox остаются per-revision.
- Approve выполняется явной submit-кнопкой без confirmation modal. Для
  `changes_requested` final submit заполненной correction code/note form является
  единственным подтверждением. Controls disabled на время request, backend всё равно
  повторно проверяет eligibility/invariants. Отдельный confirmation остаётся только у
  moderator removal с немедленным hide и retention/appeal side effects.
- Ordinary review постоянно показывает approve primary action и
  `changes_requested` secondary action с inline correction form. Moderator removal
  доступен отдельно в destructive danger-секции и не стоит рядом с обычными
  outcomes. Full-lock containment остаётся removal-only exception.
- Durable defer/snooze/second-review state для ordinary track item отсутствует.
  Закрытие review не создаёт mutation/audit/notification и не меняет
  `pending_review`, `submitted_for_review_at` или FIFO position; item остаётся
  доступным всем moderators. Owner action использует `changes_requested`, terminal
  case — removal.
- Internal comments для undecided ordinary item не реализуются: нет
  `moderation_comments`, drafts/replies, mentions, attachments или unread state.
  Moderator data появляется только в outcome audit; correction/removal notes
  принадлежат соответствующим decisions. Admin `track_issues` audit notes остаются
  отдельным operations flow.
- Ordinary queue — одна FIFO list eligible `pending_review` items с keyset
  «Назад»/«Далее». Filters по revision kind/author/activity/reason, full-text search,
  saved views/queries и per-filter counts не реализуются. Containment и export-error
  operations остаются отдельными surfaces.
- Queue row содержит только track name, localized `first|update|correction` kind и
  submission time/age; действие одно — открыть review. Map/diff/correction/hint
  preview и outcome controls в list отсутствуют. List query не загружает geometry или
  иные тяжёлые snapshot data.
- Queue/review не имеют WebSocket, SSE, auto-polling timer или presence transport.
  Refresh выполняется при navigation/manual reload и после decision/conflict.
  Открытая stale review допустима: authoritative transaction re-check возвращает
  conflict и свежую queue; background row removal/re-render отсутствует.
- Competing decision loser получает `409`, `no-store` и stable
  `moderation_item_already_decided`. Web показывает neutral localized notice и свежую
  FIFO queue без winner actor/outcome/reason или retry; второй audit/notification/
  outbox не создаётся.
- Stale direct HTML GET уже обработанного ordinary item после auth/permission checks
  делает `303` в FIFO queue с neutral one-time notice. Historical read-only mode и
  decision details в ordinary moderation отсутствуют; audit/history остаётся
  отдельным permissioned admin view.
- Approve, `changes_requested` и moderator removal сохраняются как append-only
  immutable outcomes без undo, reopen, edit или delete decision в ordinary
  moderation. Исправление после approve идёт новой owner revision либо отдельным
  removal, после `changes_requested` — новым author resubmit, а пересмотр removal —
  отдельным appeal/admin lifecycle. Корректировка всегда создаёт новый audit event и
  не меняет исходный outcome или reason/note.
- In-product appeal workflow отсутствует: нет owner form, `appeals` table, appeal
  status, attachments/chat thread или отдельной appeal queue. Removal notification
  направляет owner-а по `TRAILBASE_SUPPORT_URL` с short track ID и 90-дневным
  deadline; support verification проходит out-of-band, после чего отдельная
  permissioned admin operation пишет новый append-only audit event.
- Appeal management UI начинается с единственного exact lookup field по полному track
  UUID либо short track ID из notification/support case. Recent cases, queue, filters,
  fuzzy search по названию/owner-у и saved cases отсутствуют. Только unique exact match
  после permission/fresh-auth checks загружает sensitive context; collision short ID
  требует полный UUID без automatic selection, а unknown/ambiguous lookup возвращает
  одинаково нейтральный результат.
- Short track ID вычисляется из последних 12 lowercase hex UUIDv7 `tracks.id` и
  отображается как `trk-xxxx-xxxx-xxxx`; отдельные column/sequence/generator и UNIQUE
  constraint не создаются. Lookup нормализует ASCII case, optional exact `trk-` и group
  hyphens, затем принимает ровно 12 hex либо полный canonical UUID. Non-unique
  expression index обслуживает short lookup, collision требует full UUID. Первые 8 hex
  в GPX filename остаются отдельным display-only suffix и lookup key не являются.
- Appeal case screen после lookup — одна read-only removal summary, переиспользующая
  historical snapshot renderer, а не вторая moderation workspace. Она показывает
  short/full ID, lifecycle state, original removal time, исходные reason code/note,
  purge deadline, immutable decision snapshot, точный restore target и computed
  eligibility со stable blocking class. Live owner/provider identity, editable fields,
  support text/attachments и full audit timeline не копируются; support остаётся
  out-of-band, отдельный case record не создаётся.
- Server case projection отдаёт один derived/non-persisted `action_state`:
  `decision_ready`, `restore_blocked_full_track_lock`,
  `restore_blocked_export_unavailable`, `window_closed`, `not_current`,
  `already_decided`.
  Precedence: decided, not-current, closed window, full lock, missing export, ready.
  Ready показывает обе кнопки, restore-blocked states — uphold и disabled restore с
  coarse label, terminal states скрывают form, а decided показывает saved outcome.
  Issue details не раскрываются; backend recompute под lock и refresh summary после
  conflict являются source of truth, client eligibility logic отсутствует.
- Appeal flow имеет отдельный HTML namespace `/admin/track-appeals`, не использующий
  ordinary `/moderation/...`. GET root показывает exact `track_ref` lookup/summary;
  обе кнопки POST-ят closed outcome/reason/idempotency payload в
  `/admin/track-appeals/:removal_decision_id/decision`. Mutation требует admin, fresh
  auth и session-bound CSRF. Htmx имеет full-page fallback; JSON/bot endpoints и route
  aliases отсутствуют, все responses — `Cache-Control: no-store`.
- Actionable form получает server-generated 128-bit CSPRNG base64url
  `idempotency_key` в hidden field, общий для обеих outcome buttons и отдельный от
  CSRF. Outcome хранит только UNIQUE SHA-256 `idempotency_key_hash`; raw key,
  idempotency table, TTL/cleanup отсутствуют и key/hash не логируются. Same hash с
  exact normalized removal/outcome/reason payload возвращает result; payload mismatch
  даёт `409`/`no-store`/`idempotency_key_reused`, новый key после commit —
  `appeal_already_decided`.
- Successful commit и exact idempotent retry завершаются на одном canonical GET summary
  с full track UUID: full-page POST использует `303 See Other`, htmx — client navigation
  к тому же URL без отдельного success fragment. Summary показывает
  `already_decided`, committed outcome и одинаковый статус «Решение сохранено».
  Refresh не повторяет POST, short ID в canonical success URL не используется;
  validation остаётся на form, conflict загружает fresh authoritative summary.
- Decision POST error matrix едина для htmx/full-page: `422` сохраняет form values и
  same idempotency key с field errors; `409` отбрасывает stale form/key и возвращает
  fresh summary/action_state со stable code; `503` сохраняет values/key для manual
  same-key retry без automation и не утверждает commit result. `422/409` не создают
  outcome/audit/outbox; uncertain COMMIT разрешается idempotent retry. Все responses
  `no-store`, internal stack/issue/provider/storage details скрыты.
- Appeal/admin operation допускает только `uphold_removal` без lifecycle mutation и
  state-changing `restore_after_appeal`. Каждый outcome содержит ссылку на исходный
  removal decision, требует admin reason и создаёт новый append-only audit event, не
  меняя исходный removal. Generic `resolved`, partial и custom/free-form outcomes не
  вводятся.
- Одна permission `track_appeal_decide` mapped только к фиксированной роли `admin` и
  защищает оба outcome; отдельной uphold/restore permission нет. Management UI требует
  existing fresh auth до sensitive audit context. Роль `moderator`, ordinary
  moderation endpoints и bot action permission не получают; audit хранит actual
  admin actor.
- Final submit заполненной outcome+reason form является единственным confirmation для
  appeal decision. Fresh-authenticated management form до submit показывает short
  track ID, current state, выбранный outcome, его последствия и обязательную reason.
  Две визуально разделённые кнопки называются «Подтвердить удаление» и «Восстановить
  трек»; generic «Сохранить», второй modal и отдельный confirm screen отсутствуют.
  Controls disabled in-flight; backend повторно проверяет permission, fresh auth,
  single-shot и lifecycle invariants, а retry остаётся idempotent.
- Appeal form показывает static localized hint о 10-minute fresh-auth window без exact
  timestamp или live countdown. JavaScript timer, polling, auto-refresh/submit
  отсутствуют; backend POST остаётся authoritative, expiry идёт в принятый re-auth
  flow с discarded form и fresh summary.
- Actionable GET и Decision POST используют один exact server-side predicate
  `now < fresh_authenticated_at + 10 minutes`: equality уже expired, скрытого UX
  buffer или второго threshold нет. Если deadline проходит во время заполнения, POST
  следует принятому discard/re-auth flow.
- Decision POST один раз фиксирует server `authorization_now` в authoritative mutation
  guard перед preconditions/первой mutation. Успешная freshness check действует до
  конца transaction, даже если physical commit позже deadline; повторной wall-clock
  check у commit boundary и expiry из-за последующей queue/lock latency нет.
- Appeal GET/POST используют один injected application-level UTC clock с
  `java.time.Instant` semantics. Client clock и PostgreSQL `now()` не участвуют в
  freshness authorization; app instances требуют синхронизированных system clocks,
  а DB timestamps не образуют второй freshness authority.
- `fresh_authenticated_at` создаёт тот же application clock при server acceptance
  validated explicit bot webhook/callback. Provider event timestamp отбрасывается
  после payload validation; dedupe и event/identity binding от него не зависят.
- Provider timestamp отсутствует в `web_session` token/session records; там остаются
  только server freshness и opaque event/identity bindings для dedupe/consume. В MVP
  он не пишется также в operational events/logs/metrics; остаются server-time counters
  по provider/result без raw payload. Delivery-delay event требует отдельного будущего
  решения и retention contract.
- Единственная fresh-auth metric —
  `fresh_auth_confirmation_total{provider,result}` с provider `telegram|max` и result
  `accepted|duplicate|invalid_event|invalid_binding|account_unavailable|internal_error`.
  Identity/request/event/timestamp и дополнительные labels запрещены.
- `duplicate` относится только к exact provider replay после committed acceptance:
  response `2xx`, никаких новых token/link/message/session side effects. Pre-commit
  `internal_error` продолжает общий worker retry/DLQ и duplicate не increment-ит.
- Та же 7-day dedupe record хранит fresh-auth state `processing|accepted`; ingress
  атомарно делает processing claim+Stream enqueue. Processing replay только получает
  `2xx`, accepted replay increment-ит `duplicate`; отдельной table/namespace/TTL нет.
- Atomic worker Lua/Valkey commit создаёт единственный 10-minute token record и меняет
  processing на accepted до Bot API send. Subsequent delivery retry обязан использовать
  тот же link; network delivery не является частью atomic operation.
- При новом explicit confirmation той же provider identity atomic issuance заменяет
  active-token pointer, удаляет previous token record и его ещё существующий raw
  delivery field. Previous provider-event record остаётся accepted; отдельные bot
  message, revoke audit и notification отсутствуют.
- Гонку `/auth` consume old link/new issuance разрешает первая atomic Valkey operation:
  consume-first удаляет matching token/pointer и может завершить auth; issuance-first
  заменяет pointer и делает old link terminal invalid. Поздняя issuance не revoke-ит
  создаваемую/созданную browser session; cross-store transaction не добавляется.
- Pointer — отдельный non-secret per-identity Valkey key только с SHA-256 token digest
  и `PEXPIREAT` на original token deadline. Atomic consume удаляет его вместе с token
  record, replacement заменяет. Missing/expired pointer или target terminal invalid и
  idempotently очищается; delivery data, sliding TTL и janitor отсутствуют.
- Token record lookup использует тот же `SHA-256(raw_token)`, что pointer, без raw
  token в key/record. 128-bit CSPRNG entropy позволяет unsalted SHA-256 без custom
  HMAC/salt. Raw value существует только в link/browser request и short-lived delivery
  field; raw token/digest не попадают в logs/metrics/traces/DLQ.
- Same-link retry читает raw token только из short-lived delivery field accepted dedupe
  record; field очищается после success или token expiry, marker остаётся семь дней.
  Secret не логируется/метрится и не попадает в trace/DLQ; нового delivery storage нет.
- Valkey 9.x minimum предоставляет `HPEXPIREAT` field expiry; success использует
  idempotent `HDEL`. Отдельный TTL key, scan/polling janitor и изменение seven-day
  marker TTL отсутствуют.
- Expiry до successful Bot API delivery terminally останавливает fresh-auth delivery:
  stale link не отправляется, token не перевыпускается, accepted marker не меняется и
  user notification не создаётся. Replay остаётся `duplicate`; новый link требует
  нового explicit «Подтвердить вход» provider event. Failure остаётся только в общих
  Bot API delivery metrics/alert без raw token.
- Post-issuance delivery ограничена пятью total attempts с exponential backoff/jitter;
  retry допускается только для timeout/network, `429` и `5xx` и только внутри token
  deadline с учётом `Retry-After`. Остальные `4xx`, exhausted budget или retry за
  deadline terminal сразу с idempotent `HDEL`, сохранением accepted marker и без
  DLQ/late replay. Pre-issuance `internal_error` сохраняет общий retry/DLQ flow.
- Appeal outcome использует тот же internal reason contract, что account reactivation:
  `reason_code text NOT NULL` с CHECK на
  `support_request_verified|administrative_correction|other`, без PostgreSQL enum.
  Optional audit-only `reason_note` обязательна для `other`, после trim содержит
  1–1000 Unicode code points без control/newline и не попадает в owner notification
  или logs. Validation/UI переиспользуются, отдельного appeal catalog нет.
- На `removal_decision_id` appeal outcome действует UNIQUE: первый commit является
  terminal. Идемпотентный retry того же outcome возвращает сохранённый результат без
  второго audit/outbox; competing или последующий другой outcome получает
  `409`/`no-store`/`appeal_already_decided` без side effects. Reopen и second appeal
  для того же removal отсутствуют; correction — новая permissioned lifecycle
  operation, не изменение outcome.
- `restore_after_appeal` возвращает прежний immutable approved `current_revision_id`
  в published без ordinary re-moderation. Если approved revision до removal не было,
  track возвращается только в private/editable state, а публикация требует явного
  owner resubmit с новой immutable revision. Decided removed revision и её discarded
  export никогда не возвращаются в ordinary queue.
- Moderator removal сохраняет private sanitized export последнего approved current
  snapshot до `uphold_removal` либо 90-day purge. В этот период canonical routes дают
  `404`, новые presigned URL не создаются, object доступен только backend.
  `restore_after_appeal` атомарно возвращает тот же snapshot в published и пишет
  audit/outbox без regeneration job или `restoring` state; uphold/purge ставят
  idempotent delete. Unapproved pending export по-прежнему сразу становится
  `discarded` и удаляется.
- Uphold не меняет original `purge_at`: export удаляется сразу, остальные retained
  data — только в исходный 90-day deadline; immediate purge, extension и новый timer
  отсутствуют. Restore атомарно ставит `purge_at = NULL`. Failure/conflict/retry срок
  не меняют. Outcome и purge worker используют один track lock: first commit wins,
  worker после restore видит NULL и не очищает track.
- Первый новый uphold или restore под track lock требует current active removal,
  отсутствующий outcome и `now() < purge_at`. Expiration закрывает обе операции даже
  при lag purge worker-а, support request не продлевает окно; newer lifecycle state
  также закрывает оба outcome. До deadline full-track lock и unavailable retained
  export блокируют только restore, audit-only uphold остаётся доступным. Уже committed
  outcome по-прежнему возвращается идемпотентным retry.
- Restore transaction под track lock проверяет текущий `moderator_removed`, совпадение
  active removal с `removal_decision_id`, deadline, отсутствие newer lifecycle event и
  full-track lock, а для published branch — retained export readiness. Stale/blocked
  request получает `409`/`no-store`/`appeal_not_restorable` без outcome, audit/outbox
  или delete и может быть повторён после temporary resolution до deadline. Full lock
  не запрещает audit-only uphold.
- Первый committed appeal outcome атомарно пишет locked-on owner web-inbox и
  primary-bot outbox. Uphold показывает fact/short ID/purge deadline без повторной
  appeal action; restore — canonical link либо edit/resubmit. Internal admin fields не
  раскрываются, failed/conflicting/idempotent request notification не создаёт;
  deactivated-account suppression остаётся без backlog/replay.
- `changes_requested` или moderator removal pending revision атомарно переводит
  export state в `discarded` и пишет high-priority idempotent object delete. Export
  не переиспользуется новой revision; late worker completion не меняет discarded
  state и удаляет созданный object через retry/DLQ/alert.
- Обычная track moderation имеет только approve/publish и `changes_requested`.
  Отдельного track lifecycle state `rejected` нет; terminal policy, privacy/safety,
  legal, spam и duplicate cases используют moderator removal с confirmation и
  закрытым reason catalog. Статусы `rejected` других сущностей этим не меняются.
- Submit использует автоматическое standing consent первого `/start`, показывает
  CC BY 4.0 reminder/link и не имеет отдельного agreement gate.
- Sanitized GPX опубликованного track доступен анонимно через canonical download URL:
  endpoint не создаёт account/session, повторно проверяет publication status,
  до lookup/signing применяет 30 requests/min на normalized IP с burst 10 и выдаёт
  5-minute presigned redirect. `404` также расходует лимит; session его не повышает,
  S3 GET не учитывается. Private/archived/moderator-removed state отвечает одинаковым
  `404`; otherwise published full-locked track отвечает generic `503` без metadata,
  integrity details или `Retry-After`; bot не публикует signed URL.
- Sanitized object не шифруется приложением и остаётся в private bucket. Presigned
  HTTPS URL отдаёт exact object напрямую без backend decrypt proxy; transparent
  provider-side SSE/disk encryption допустимы как deployment control и не меняют
  bytes/API.
- Canonical download `302/404/429/5xx` всегда имеет `Cache-Control: no-store`; CDN
  cache и `ETag` для route отсутствуют. Каждый request повторяет limit/publication
  checks, а signed S3 URL живёт отдельно только пять минут.
- Full lock прекращает выдачу новых signed URLs, но сам не ставит sanitized object на
  deletion/rotation; ранее выданный URL живёт не более исходного five-minute TTL.
- Canonical `/tracks/:track-id/download` всегда разрешает только current published
  revision. Старые revision exports не имеют public route и eligible для storage
  retention cleanup; approval новой revision переключает тот же URL без изменения
  внешней ссылки.
- Sanitized S3 object задаёт `Content-Type: application/gpx+xml; charset=utf-8` и
  `Content-Disposition: attachment` с ASCII slug/UUID filename, поэтому эти headers
  действуют после presigned redirect.
- Sanitized object и final response содержат exact uncompressed XML bytes без
  `Content-Encoding`, gzip/ZIP или второго representation; filename заканчивается
  на `.gpx`.
- Download filename детерминированно строится из final Name: NFKD, lowercase ASCII
  alnum, collapse остальных runs в `-`, trim/max 64, fallback `track`, затем первые
  8 lowercase hex track UUID и `.gpx`. Transliteration и raw upload filename не
  используются.
- Переход current export в superseded/non-public пишет high-priority idempotent S3
  delete с retry/DLQ/alert, кроме private retention последнего approved export после
  moderator removal до appeal outcome/90-day purge. После delete ранее выданный
  signed URL не работает; при задержке доступ живёт не дольше исходных пяти минут.
  Отдельного download proxy нет.
- Htmx-форма `/tracks/new`: upload создаёт async job; polling открывает private draft
  preview; пользователь подтверждает final summary и отправляет revision на
  moderation. Submit требует непустой final name и явно выбранный primary activity;
  CC BY 4.0 показывается reminder/link без отдельного действия, description и
  остальные metadata optional.
- Любой GPX attachment в private one-to-one chat после quota/three-slot check
  автоматически начинает upload flow: backend скачивает file_id через provider API
  и использует тот же S3+parse+DB pipeline. `/upload` только предлагает прислать файл
  и не создаёт обязательный pending mode; attachment другого типа не создаёт job.
  Обязательные metadata и отправка на moderation завершаются в chat; web deep-link
  optional. Group/channel upload не создаёт job или draft.
- До трёх chat upload flows могут идти параллельно; каждый status/control/prompt
  связан с конкретным `upload_job_id`, неявного current upload нет.
- После parse metadata редактируются из одной draft summary card без обязательного
  порядка: Name/Activity отмечены required, actions открывают bound prompts для любых
  полей. Submit с пропусками ничего не меняет и перечисляет их; готовый draft
  переходит к final summary и CC BY 4.0 confirmation.
- Initial Name default выбирается из file-level `<metadata><name>`, затем из
  единственного distinct непустого `<trk><name>`, затем из безопасно
  нормализованного attachment filename stem. Несколько разных track names дают
  warning и не выбираются/не склеиваются. Любой default остаётся неподтверждённым
  draft field; raw filename отдельно не хранится и не публикуется. Без источников
  Name пустой.
- Initial Description default выбирается из file-level `<metadata><desc>`, затем из
  единственного distinct непустого `<trk><desc>`. Несколько разных track descriptions
  дают warning, не выбираются/не склеиваются и оставляют optional Description пустым.
  Выбранный default записывается только в owner UI locale на момент upload.
- Attachment caption не становится Description автоматически и не копируется в
  domain metadata. Description default берётся только по принятому GPX precedence или
  из текста, явно введённого через bound prompt; caption существует лишь в обычном
  raw-webhook retention.
- GPX `<desc>` без locale записывается в `description[owner_ui_locale_at_upload]` как
  неподтверждённый default. Autodetect/translation/duplication между `ru` и `en` нет;
  последующая смена UI locale не переносит существующий text.
- Presentation выбирает Description в UI locale, затем другую доступную ветвь с
  видимой language label; без обеих ветвей секция скрыта. JSON API возвращает полный
  `description` map, не переводя и не подменяя branches.
- Sanitized GPX сериализует обе непустые description branches в один deterministic
  plain-text `<desc>` блоками `[ru]`, затем `[en]`; пустой map не создаёт element.
  Viewer-specific exports, translation и custom XML extensions отсутствуют.
- Sanitized GPX использует только стандартные GPX 1.1 поля provenance/license:
  root `creator="TrailBase revision:<uuid>"`, author display name в
  `<metadata><author><name>`, CC BY 4.0 в
  `<metadata><copyright author="..."><license>`, canonical track URL в
  `<metadata><link href="...">`, а пользовательские Name/Description в
  `<trk><name>`/`<trk><desc>`. Provider identity, internal user ID, raw filename и
  application extensions отсутствуют.
- Sanitized export всегда содержит один `<trk>` на revision и один `<trkseg>` на
  каждый canonical `MultiLineString` component в deterministic order. Исходная
  multi-`<trk>` grouping не восстанавливается; route-only input также становится
  `<trk>`, а не `<rte>`.
- Sanitized `<trkpt>` содержит `<ele>` только при валидном исходном elevation этой
  сохранившейся после trimming point. Missing elevation остаётся missing; zero,
  interpolation и smoothed/LTTB values не экспортируются независимо от derived
  profile coverage.
- Export numeric format использует deterministic `HALF_EVEN`: lat/lon максимум
  7 decimal places, elevation максимум 2, decimal dot, plain notation без exponent
  и trailing zeros; negative zero нормализуется в `0`.
- Весь export одной revision byte-identical: UTF-8 без BOM, fixed XML declaration,
  GPX 1.1 namespaces и element/attribute order, compact XML без insignificant
  whitespace, один final LF и никаких runtime timestamps/random values.
- Export не добавляет durable SHA-256/byte-size columns. S3 PUT использует signed
  actual payload SHA-256 и только после checksum-validated success переводит state в
  ready; deterministic bytes покрываются golden tests и не становятся HTTP `ETag`.
- `<metadata><bounds>` не экспортируется: viewer получает bounds из track points, без
  дублирования geometry и неоднозначного antimeridian min/max.
- Activity, difficulty и tags не записываются в `<trk><type>` или
  `<metadata><keywords>`; их versioned taxonomy доступна по canonical URL/JSON API,
  а GPX mapping vocabulary в MVP отсутствует.
- Один attachment с одним или несколькими `<trk>` создаёт один draft, без
  automatic split. Каждый валидный `<trkseg>` становится отдельным компонентом
  canonical `MultiLineString` в document order. Segment manifest не создаётся;
  исходная `<trk>/<trkseg>` hierarchy остаётся только в original raw GPX.
- Parse result сохраняет только `source_track_count`, `source_route_count` и
  `valid_segment_count`. Несколько source elements выбранного track/route geometry
  kind дают неблокирующий warning с counts в draft/final summary рядом с map preview;
  отдельного confirmation до metadata editing нет, а final submit подтверждает весь
  snapshot.
- Route-only GPX следует тем же правилам: один attachment создаёт один draft, каждый
  `<rte>` становится component. Name выбирается из metadata, затем единственного
  distinct route name, затем filename stem; Description — из metadata, затем
  единственного distinct route description, иначе пусто. Разные values дают warning
  без выбора или concatenation.
- Для каждого принятого attachment bot сразу создаёт одну status card и best-effort
  редактирует её на стадиях download/validation/parse. Terminal success превращает
  card в draft summary/actions, failure — в безопасную причину и retry action. При
  provider edit failure отправляется одна replacement card без повторного domain job.
- После исчерпания automatic retries transient failure позволяет atomically
  reacquire-ить slot и повторно поставить тот же `upload_job_id` в очередь,
  переиспользуя сохранённый raw object. Permanent validation/limit/geometry error
  просит новый исправленный attachment; отсутствующий/очищенный raw также требует
  повторной отправки файла.
- Parsed draft, ожидающий required metadata или moderation submit, продолжает занимать
  один из трёх slots; slot освобождается после submit, cancel или terminal
  intake/parse failure.
- Cancel после parse закрывает chat flow, но сохраняет private draft/raw под quota;
  продолжение доступно через `/drafts` или web, delete — отдельное подтверждённое
  действие.
- `/drafts` resume атомарно занимает свободный slot; при полном лимите state не
  меняется и bot показывает три active flows.
- На draft действует один global lease для Telegram/Max/web. Явный takeover сохраняет
  slot, меняет lease generation и инвалидирует старые prompts/controls.
- Takeover требует authenticated owner и explicit confirmation; web проверяет CSRF,
  bot — private-chat identity, fresh bot auth не требуется.
- После takeover старый bot prompt best-effort теряет controls; edit failure не
  откатывает lease. Stale web mutation получает `409` и reload/resume action без
  realtime transport.
- Post-parse lease закрывается после 24 часов без успешной user mutation, освобождая
  slot без удаления draft/raw; reads/polling/worker events deadline не продлевают.
- PostgreSQL хранит durable lease generation/interface/activity и slot accounting;
  acquire/takeover/metadata/slot transitions транзакционны. Valkey содержит только
  ephemeral prompt/control tokens.
- Миграция upload flows добавляет `slot_no` 1..3, `closed_at` и partial unique indexes
  на active `(user_id, slot_no)` и active non-null `draft_id`.
- Upload/resume transaction перед acquire закрывает expired post-parse leases,
  увеличивает generation и только затем выделяет slot; janitor использует ту же
  operation как fallback.
- Slot-changing transactions сериализуются через `SELECT ... FOR UPDATE` на `users`
  row и lock order `user -> flow -> draft`; advisory locks не используются.
- Pre-parse cancel ставит `cancel_requested_at`, меняет generation и удерживает slot
  до terminal `cancelled`; worker проверяет cancel перед S3/parse/revision commit.
- Parse worker при deactivated owner может завершить started job только private draft;
  он не submit-ит revision и не создаёт publication side effects.
**Acceptance**:
- Web и bot принимают несжатый GPX 1.0/1.1 до 10 MiB и максимум 250 000 points.
- Успешный parse создаёт private revision snapshot с 2D MultiLineString, metrics,
  user-confirmed activity и S3 references.
- Raw object нельзя получить через public/owner API или presigned URL; его
  читают только backend workers. Единственный user-facing GPX download —
  опубликованный sanitized export.
- S3 raw object byte-identical исходному принятому attachment: application encryption,
  compression, normalization и иное преобразование bytes отсутствуют.
- Включённое provider-side SSE/disk encryption прозрачно для TrailBase: GET
  возвращает byte-identical raw, application keys/key IDs и decrypt path отсутствуют.
- SHA-256 mismatch уже `ready` raw не меняет revision/publication и не запускает
  automatic retry: job получает permanent `raw_integrity_mismatch`, operator —
  безопасный alert с `raw_object_id`, а object остаётся под обычным retention.
- Admin UI/API не умеет repair raw. Infrastructure restore принимает только exact
  bytes, совпадающие с stored SHA-256; иначе требуется новый upload/new
  `raw_object_id`, а старый digest не изменяется.
- Purged raw отсутствует в primary MinIO, может оставаться в encrypted off-host
  backup не более 30 дней и не возвращается в active service. После disaster restore
  orphan reconciliation завершается до открытия traffic.
- При остановленных web/workers CLI считает orphan любой TrailBase-managed S3 key без
  durable DB reference и удаляет все его versions/markers. Referenced key остаётся;
  отдельный manifest/approval state не нужен.
- Default invocation ничего не удаляет и показывает orphan counts/bytes; destructive
  invocation явно включается operator-ом, заново сканирует DB/S3 и fail-ится на
  любой scan error.
- Referenced-but-missing object отмечает затронутый track durable problem marker и
  сохраняет DB reference. Admin UI показывает точную безопасную причину, owner
  surfaces — только общий факт проблемы; storage key/internal detail owner-у не
  раскрываются.
- Успешно записанные markers не блокируют global startup; affected track operations
  работают fail-safe. Неполный scan или missing object без однозначного track target
  блокирует открытие traffic.
- Capability checks соответствуют reason: unrelated track operations проходят,
  dependent operation fail-closed. Missing raw/re-parse и missing
  export/download-or-publication-switch покрыты integration cases; full lock
  применяется только к snapshot-wide integrity code и блокирует normal content,
  owner/content/publication mutations, restore, approve и content-dependent
  moderation. Разрешены только minimal status/generic warning, owner archive,
  moderator removal/hide, scheduled purge, admin issue/history read,
  detector-specific recheck/resolution и audit. Containment не читает snapshot.
  Scope выводится из code, а не читается из отдельной issue column.
- Owner full-lock card строится без revision read и содержит только `track_id`,
  lifecycle `status`, `has_problems = true`, optional `purge_at`, generic warning и
  доступный archive action. Content/revision/issue fields отсутствуют; archived card
  показывает purge deadline без restore.
- Запрещённая full-lock owner mutation даёт terminal `409`/`no-store` и stable safe
  JSON error либо generic HTML/bot acknowledgement без retry/DLQ и internal details.
- Non-admin moderator получает в existing surfaces snapshot-free placeholder только
  с removal/hide; обычные moderation/content actions скрыты, admin issue permissions
  не дают bypass content lock.
- Stale/direct moderator content action после permission/visibility checks получает
  тот же terminal `409` без mutation/audit/retry; UI возвращается к placeholder.
- Full-lock containment removal создаёт normal locked-on moderator-removal
  notification для active owner с public-safe label закрытого removal reason code;
  audit-only note и `track_issues.code/detail` не копируются, deactivated suppression
  остаётся общей.
- Две одновременные issues сохраняются отдельными rows; resolution одной оставляет
  `has_problems = true` и блокировки второй, а resolution последней снимает derived
  flag. Ни одно изменение не создаёт user notification.
- Manual clear отсутствует: failed recheck оставляет issue active, successful
  detector-specific check заполняет `resolved_at`, а история row и admin audit
  сохраняются.
- Resolution последней full-lock issue автоматически снимает только её derived
  capability block. Lifecycle/moderation state и `current_revision_id` не меняются;
  доступ прежнего immutable snapshot возвращается без повторной moderation только
  если текущий lifecycle допускает его. Archive/removal и другие issues сохраняют
  effect.
- Search/map/catalog collection results исключают track под full lock. Прямые
  canonical detail/download requests для otherwise published track получают generic
  `503 Service Unavailable` с `Cache-Control: no-store`, без `Retry-After`, metadata
  или integrity details; private/archived/moderator-removed resources сохраняют
  одинаковый `404`. После resolution следующий request заново проверяет publication
  и effective capabilities без cached failure.
- Full lock не является non-public lifecycle transition и сам не удаляет/ротирует
  sanitized object. Уже выданный presigned URL может работать до исходных пяти минут,
  закэшированный GeoJSON — до 90 секунд; уже доставленный client content не
  отзывается. Новые responses, увидевшие lock, не создают новый content access. После
  resolution переиспользуется тот же immutable snapshot/export, если lifecycle это
  разрешает; download proxy не добавляется.
- Два одинаковых scan-а создают одну active issue; второй scan обновляет только
  `last_seen_at`. Resolution заполняет `resolved_at`, а последующий recurrence
  создаёт новую row. Ни один переход не создаёт user notification; track views
  показывают только текущее generic `has_problems`.
- Purge raw/revision не блокируется `track_issues.subject_id`: это не FK и не durable
  storage reference. Historical issue сохраняет UUID, пока существует track; новая
  issue с foreign или отсутствующим subject отклоняется checker-ом под track lock.
  Physical purge track удаляет все active/resolved issue rows вместе с derived data,
  не ставит `resolved_at` и не копирует codes/details/subject UUID в новый audit
  event. Tombstone track UUID и существующий append-only audit остаются.
- Concurrent detector/recheck одного track упорядочиваются lock на `tracks` row;
  transaction, которой также нужен owner, берёт locks только в порядке
  `user -> track`. Duplicate active identity дополнительно предотвращает partial
  unique index.
- Full-lock activation linearize-ится commit-ом issue transaction. Direct
  detail/download (`FOR SHARE`) и mutations (`FOR UPDATE`) повторно проверяют issues
  под тем же track-row lock; pre-lock winner завершается до detector commit, а
  post-commit operation видит block. Collection statement, начатый до commit, может
  вернуть прежний result без per-row locking; in-flight responses не отменяются.
- Stale result preliminary scan не применим: после track lock выполняется
  authoritative check. Его timeout/error оставляет issue неизменной; только
  successful in-transaction result создаёт/обновляет/resolve-ит row. Check имеет
  hard deadline 10 секунд и одну attempt; network/timeout/`5xx` не retry-ятся под
  lock. Durable retry queue отсутствует: CLI возвращает nonzero, admin UI показывает
  safe error, issue остаётся active, а повтор запускается явно.
- `problem_present`, `healthy` и `inconclusive` покрыты отдельно: первый
  создаёт/обновляет active issue, второй resolve-ит её, третий не меняет DB.
- S3 classification покрывает exact missing/mismatch как `problem_present`, exact
  match как `healthy`, а access/transport/provider failures как `inconclusive`;
  storage credentials failure не создаёт track issue.
- Raw integrity resolution покрыт full streaming GET: SHA-256 и size пересчитаны по
  body; HEAD/ETag/custom metadata не могут дать `healthy`. Object больше stored size
  обнаруживается на первом extra byte и не скачивается дальше. Clean short body и
  transport truncation дают соответственно `problem_present` и `inconclusive`.
- Parsed raw не истекает отдельно от draft/track, остаётся учтённым в raw quota и
  удаляется при физическом delete/purge owning entity. Janitor не удаляет referenced
  raw по возрасту; его 24-hour rule относится только к incomplete/orphan objects.
- Две revisions одного track, созданные из одного existing raw без нового upload,
  ссылаются на один immutable object. Quota считает его один раз, удаление одной
  revision не удаляет object при оставшейся durable reference; новый upload всегда
  создаёт отдельный object даже для byte-identical content.
- Storage metadata raw находится только в `raw_objects`; upload job и
  каждая derived revision ссылаются на одну row через `raw_object_id`. Quota не
  умножается на число revision references, cleanup не удаляет referenced object.
- Physical raw delete блокируется retained revision FK. Terminal job history не
  считается pin и после raw delete получает `raw_object_id = NULL`; retryable
  transient job удерживает raw только до incomplete deadline.
- Concurrent reparse-reference insert и cleanup имеют один winner: обе операции
  блокируют owner/raw rows, а reference insert разрешён только для `ready`.
  `delete_pending` нельзя вернуть в `ready`; FK `ON DELETE RESTRICT` ловит ошибочный
  physical delete referenced row.
- Raw row проходит только `pending -> ready -> delete_pending -> physical row
  delete`; abandoned `pending` может перейти прямо в `delete_pending`. Revision не
  может ссылаться на non-`ready` row. Отдельных raw `error`/`deleted` states нет:
  upload failure виден в job, cleanup failure — в outbox retry/DLQ.
- Cleanup command не содержит object key/SHA-256 и восстанавливает их только
  по `raw_object_id`. Он удаляет все versions/delete markers exact opaque key и
  завершает physical row delete только после empty listing. Replayed command после
  уже удалённой row/version успешно завершается; prefix matches, unexpected state/FK
  не вызывают чужой delete.
- Concurrent upload quota нельзя обойти: до PUT каждый `pending` занимает 10 MiB,
  после validation contribution уменьшается до actual byte size, а
  `delete_pending` немедленно освобождает logical quota независимо от S3 cleanup lag.
- Два concurrent uploads одного owner сериализуются owner row lock: оба не могут
  пройти проверку одной старой суммы. Quota вычисляется из indexed `raw_objects`
  rows, не из denormalized user counter.
- GPX с несколькими `<trk>` создаёт один draft. Границы `<trkseg>` сохраняются как
  отдельные components в deterministic document order; отдельная manifest
  table/JSONB и вторая копия coordinates не создаются.
- Multi-track draft/final summary показывает source-track и valid-segment counts и
  map preview, но не блокирует редактирование отдельным confirmation. Cancel позволяет
  вместо него загрузить каждый catalog track отдельным файлом.
- Route-only GPX с несколькими `<rte>` также создаёт один draft и показывает
  неблокирующий source-route/valid-segment warning. Metadata defaults используют тот
  же distinct-value precedence, подставляя `<rte><name>/<desc>` вместо track fields.
- Sanitized GPX для track и route input имеет один `<trk>` с final submitted
  Name/Description и количеством `<trkseg>`, равным числу canonical geometry
  components; порядок segments совпадает с component order.
- Каждая exported point сохраняет только собственное валидное source elevation;
  missing `<ele>` не синтезируется, а 90% profile threshold не удаляет доступные
  per-point values и не добавляет отсутствующие.
- Повторная генерация той же revision создаёт byte-identical point numeric text:
  lat/lon не больше 7 decimals, elevation не больше 2, без exponent/trailing zeros
  или negative zero.
- Повторная generation одной revision создаёт byte-identical GPX object целиком;
  serializer не зависит от locale, platform newline, map iteration order, clock или
  randomness.
- PostgreSQL не хранит export hash/size. Checksum mismatch или PUT failure оставляет
  export не-ready и проходит обычный retry/DLQ; canonical route остаётся
  `no-store` без `ETag`.
- Final S3 GET отвечает GPX MIME type и attachment disposition с принятым ASCII
  filename; browser не пытается inline-render XML после redirect.
- Downloaded bytes совпадают с deterministic stored XML object; transparent/explicit
  compression и archive extraction в MVP отсутствуют.
- Sanitized export хранится как application-plaintext exact XML в private bucket и
  скачивается по presigned HTTPS URL без decrypt proxy. Provider-side SSE/disk
  encryption, если включено при deployment, прозрачно для bytes/API.
- Non-Latin-only Name получает `track-<8-hex-track-uuid>.gpx`; ASCII slug ограничен
  64 символами и не содержит header separators/control chars.
- Sanitized GPX не содержит `<metadata><bounds>`; корректность export не зависит от
  отдельного derived bbox или antimeridian convention.
- Sanitized GPX не содержит classification codes в `<trk><type>`/keywords; activity,
  difficulty и tags остаются доступны на track page и в JSON API.
- При наличии timestamps canonical duration по умолчанию использует moving с fallback
  на elapsed; manual/unknown override сохраняет исходные moving/elapsed metrics.
- Manual duration вне `1..31_536_000` отклоняется; более чем 10× расхождение с
  default auto candidate сохраняется только после explicit warning confirmation;
  ровно 10× и отсутствие auto candidate warning не вызывают.
- Подтверждённый duration outlier остаётся подтверждённым после resume/takeover, но
  stale control, изменённое manual value или reparse не могут переиспользовать
  acknowledgement для других comparison inputs.
- Moderator видит acknowledged duration outlier и comparison inputs; наличие flag
  само по себе не меняет moderation outcome, `submitted_for_review_at` или FIFO
  position.
- Moderator не может исправить duration pending revision; после
  `changes_requested` авторский resubmit создаёт новый immutable snapshot, а прежний
  остаётся неизменным.
- `changes_requested` сохраняет один correction code из закрытого DB CHECK и optional
  owner-visible note; `other` требует note. Notification показывает localized label и
  note, а removal/issue codes не переиспользуются.
- Авторский resubmit хранит обязательный `correction_of_revision_id` на
  непосредственно возвращённую same-track revision, сохраняет прежний
  `base_revision_id` и по умолчанию показывает moderator-у correction diff вместе с
  сохранёнными code/note. Total diff от baseline и полный snapshot остаются
  вторичными views.
- First-publication review показывает полный snapshot. Review изменения published
  track по умолчанию показывает field diff и old/new geometry overlay относительно
  сохранённого `base_revision_id`, позволяет открыть полный pending snapshot и не
  допускает field-level approval. Stale baseline не approve-ится и не rebase-ится
  автоматически.
- Новый revision submit использует одноразовый standing CC BY 4.0 consent и не требует
  повторного acceptance.
- Anonymous visitor без account/session может скачать только public sanitized GPX
  через canonical URL; signed redirect живёт пять минут, а непубличные states не
  различаются внешним `404`.
- Download limiter допускает 30 attempts/min на normalized client IP с burst 10,
  считает `404` до publication lookup, возвращает `429 Retry-After` и не повышается
  browser session; redirected S3 GET не входит в budget.
- Ни один canonical download response не кэшируется: `no-store`, без `ETag`/CDN.
  Повторный request после revision switch/removal не использует старый redirect.
- Revision-specific public download отсутствует: после смены `current_revision_id`
  canonical URL выдаёт новый export, а superseded/privacy-pre-trim/removed snapshot
  нельзя получить старым TrailBase URL.
- После supersede/removal старый sanitized object удаляется high-priority cleanup;
  failure виден в DLQ/alert, а остаточный signed доступ ограничен five-minute TTL.
- Approve не может создать published revision без готового private sanitized export.
  Export error оставляет item pending и прежний current public; успешный approve
  атомарно переключает DB pointers без S3 generation внутри transaction.
- До approval owner/moderator не могут скачать private sanitized export и не
  получают presigned URL; object доступен только backend workers. После approval
  скачивание идёт исключительно через canonical current-revision route.
- Pending/error export item отсутствует в обычной moderator queue; moderator не видит
  infrastructure badge и retry action. Exhausted error доступен operator/admin, а
  idempotent manual retry не создаёт второй export job. Owner-facing status не
  раскрывает S3/queue failure.
- Обычная moderation queue не хранит assignment/claim state. Любой moderator с
  permission может открыть item; из конкурентных решений сохраняется только первое
  committed вместе с actual actor, проигравший submit получает conflict и не создаёт
  второй audit или notification.
- Eligible revisions имеют immutable `submitted_for_review_at` и выдаются стабильным
  FIFO `(submitted_for_review_at ASC, id ASC)` через keyset pagination. Duration
  flags, resubmit и ручные действия не меняют порядок; priority/SLA schema нет.
- Approve, `changes_requested` и moderator removal принимают ровно один revision ID
  на request. Bulk controls/API/jobs отсутствуют; successful decision создаёт только
  per-revision audit/notifications и возвращает обновлённую FIFO queue.
- Approve и валидная `changes_requested` form выполняются без дополнительного modal;
  double submit не обходит backend re-check. Moderator removal без отдельного
  confirmation не выполняется.
- На ordinary review screen removal визуально и структурно отделён от approve/
  `changes_requested`; на full-lock containment screen approve/correction отсутствуют
  и остаётся только confirmed removal.
- Navigation away из undecided ordinary review оставляет item в исходном
  `pending_review` и FIFO position без durable defer flag, audit или notification;
  отдельного defer filter нет.
- До outcome ordinary review не сохраняет moderator-authored comments/drafts и не
  создаёт comment notification/unread state. Issue audit notes не отображаются как
  moderation thread.
- Ordinary moderation entrypoint показывает единственную FIFO list с keyset
  navigation; filter/search/saved-view controls и per-filter counts отсутствуют.
- Queue rows не содержат map/diff previews или decision buttons; opening review —
  единственное действие, а все decision context загружается на review page.
- Queue/review не обновляются realtime или polling-ом; stale page не может обойти
  transaction re-check и после conflict заменяется свежей FIFO queue.
- Race loser получает exact `409` error `moderation_item_already_decided`; response не
  раскрывает победившее решение и не создаёт domain/audit/notification side effects.
- Direct web link обработанного item не открывает historical review: moderator
  получает `303` и neutral queue notice; audit history требует отдельную admin
  permission.
- Outcome approve, `changes_requested` или moderator removal нельзя undo, reopen,
  edit или delete через ordinary moderation. Исправление представлено новой owner
  revision, новым removal либо отдельным appeal/admin action; исходный decision и
  reason/note остаются неизменными, а новый результат получает отдельный audit event.
- Removal notification содержит production `TRAILBASE_SUPPORT_URL`, short track ID и
  90-дневный deadline. Owner appeal form/record/status, attachments/thread и appeal
  queue отсутствуют; после out-of-band support verification результат фиксируется
  отдельной permissioned admin operation и append-only audit event.
- Appeal management entrypoint имеет только одно exact lookup field по full track UUID
  или short track ID. Списка/recent cases, queue, filters, fuzzy owner/name search и
  saved cases нет. Unique exact match загружает sensitive context только после
  permission/fresh-auth checks; collision short ID требует full UUID, а
  unknown/ambiguous result нейтрален и не раскрывает context.
- Short track ID имеет canonical display `trk-xxxx-xxxx-xxxx` из последних 12 lowercase
  hex UUIDv7 track ID, без отдельной column/sequence и без UNIQUE assumption. Lookup
  принимает normalized 12-hex reference либо canonical full UUID через non-unique
  expression index; collision требует full UUID. 8-hex GPX filename suffix не
  принимается как short lookup ID.
- После unique lookup показывается только read-only removal summary: short/full ID,
  lifecycle state, removal time, original reason code/note, purge deadline, immutable
  decision snapshot, restore target и computed eligibility/blocking class. Отдельной
  moderation workspace, live owner/provider identity, editable fields, support
  payload/attachments, full audit timeline или persisted appeal case нет.
- Summary содержит ровно один derived `action_state` из closed catalog
  `decision_ready`, `restore_blocked_full_track_lock`,
  `restore_blocked_export_unavailable`, `window_closed`, `not_current`,
  `already_decided`. Deterministic precedence и button mapping применяются server-side;
  payload не раскрывает issue details. UI не recompute-ит eligibility, а после
  conflict загружает свежую summary; mutation повторяет computation под lock.
- Admin appeal routes: `GET /admin/track-appeals` с optional `track_ref` query и
  `POST /admin/track-appeals/:removal_decision_id/decision` с closed outcome/reason и
  idempotency key. Обе outcome buttons используют один POST. HTML/htmx имеет full-page
  fallback; active admin/fresh auth/CSRF и `no-store` обязательны. JSON/bot endpoints,
  `/moderation/...` aliases и отдельные mutation routes для uphold/restore отсутствуют.
- Fresh-auth expiry между form render и POST обрабатывается до sensitive reload и
  mutation, вне `422/409/503`, без outcome/audit/outbox/notification. Re-auth сохраняет
  server-side только allowlisted return target на canonical appeal GET с full UUID в
  `track_ref`; submitted outcome/reason и прежний idempotency key не попадают в URL,
  auth-flow или session. После re-auth summary/`action_state` recompute-ятся, form
  получает новый key, а admin повторно выбирает outcome и reason.
- Successful re-auth кладёт в active session только generic one-time flash
  `appeal_form_discarded`, без identifiers/form data/key. Canonical GET атомарно
  consume-ит flash и показывает coarse localized notice, что предыдущая отправка не
  сохранена и decision нужно заполнить заново. Flash не находится в query или
  `409/503` catalog и исчезает после одного render; terminal authoritative state
  скрывает form независимо от notice.
- Expired fresh auth ведёт в один top-level fresh-auth flow: full-page POST отвечает
  `303` на server-generated start URL, htmx POST — `200`/`no-store` с пустым body и
  `HX-Redirect` на тот же URL. Оба используют один server-side `return_to` на
  canonical appeal GET; auth UI не рендерится fragment-ом. Invalid session/CSRF
  остаются отдельными fail-closed ветками.
- Appeal re-auth использует обычный bot-issued `web_session` token и существующий
  `/auth` GET/POST. Consume rotate-ит current browser session, переносит исходный
  `fresh_authenticated_at` из token record и возвращает по bound `return_to`; отдельной
  appeal/re-auth credential state или consume route нет.
- Этот общий `/auth` flow для всех token-bearing GET/POST outcomes использует
  `Cache-Control: no-store`, `Referrer-Policy: no-referrer` и только same-origin assets.
  Access/security logs пишут route без query/form body; raw token/digest отсутствуют в
  errors, analytics и traces.
- Initial token-query GET этого flow не consume-ит token и всегда отвечает `303` на
  clean `/auth`: valid branch сначала создаёт existing auth-flow record/cookie+nonce.
  Malformed/unknown/expired/superseded branch использует stable non-secret
  `result=invalid`; target GET показывает generic-invalid без lookup/изменения current
  flow, а plain `/auth` продолжает confirmation. Marker не добавляет state/cookie. Raw
  token отсутствует в body/redirect/history; JavaScript, client storage и новый
  credential state не нужны.
- Auth-flow record/cookie/form nonce разделяют original token absolute deadline без
  sliding от GET, retry или `503`. Expired/missing component даёт generic-invalid page,
  idempotently очищает remaining flow state/cookie и не consume-ит, не восстанавливает
  и не перевыпускает source token.
- Auth-flow record хранит только source token digest, allowlisted `return_to`, original
  deadline и SHA-256 nonce hash. Independent 128-bit flow-cookie ID использует hashed
  Valkey lookup; raw source token после redirect не сохраняется. Raw flow ID/nonce
  находятся только в `HttpOnly` cookie/hidden POST field, bindings читаются из token
  record, raw/digest values отсутствуют в telemetry/DLQ.
- Общая flow cookie — `__Host-trailbase_auth_flow`; `Secure`, `HttpOnly`,
  `SameSite=Lax`, `Path=/`, без `Domain`, expiry не позже remaining source deadline.
  Success, terminal invalid/expiry и cancel очищают её `Max-Age=0`; session cookie
  остаётся отдельной.
- Новый valid initial GET атомарно удаляет record, адресуемую existing flow-cookie,
  создаёт replacement и оставляет один active browser flow. Old tab/form generic-invalid;
  previous source token не consume/revoke-ится и его link может снова заменить flow до
  deadline.
- Failed flow-cookie/form-nonce validation на `POST /auth` не consume-ит token и не
  удаляет valid flow. Missing/unknown cookie и nonce mismatch дают generic-invalid без
  token/pointer/session mutations. Unknown cookie очищается у browser `Max-Age=0`, но
  wrong/stale nonce для valid record сохраняет record и current cookie. Общий `/auth`
  rate limit — 10/min на IP; отдельного nonce-attempt counter нет.
- `/auth` rate-limit reject — retryable `429` с `Retry-After`, не terminal invalid.
  Один budget 10/min на normalized trusted-proxy client IP охватывает initial token
  GET, clean confirmation GET и POST. До credential lookup limiter сохраняет весь
  credential state и отвечает generic `no-store`/`no-referrer` body без redirect,
  automatic retry или отдельного per-flow/per-nonce budget.
- Non-transient invalid `POST /auth` отвечает `303` на token-free `/auth` со stable
  `result=invalid`; target GET generic-invalid и не lookup-ит current flow. Cleanup
  следует принятому error class, поэтому wrong/stale nonce сохраняет valid flow/cookie.
  Transient PostgreSQL/Valkey failure остаётся retryable `503` с `Retry-After` без
  redirect, terminal marker и credential mutations, сохраняя flow/token/cookie.
- После successful PostgreSQL identity/account check одна atomic Valkey function
  валидирует flow/nonce, source token и active pointer, promote-ит flow-ID digest в
  session namespace, создаёт либо rotate-ит session, consume-ит token/pointer и
  заменяет flow receipt как одну credential linearization point; distributed
  PostgreSQL transaction нет.
- Receipt до original deadline хранит nonce hash, `return_to`, session digest/status и
  expiry без raw ID. Success/retry ставит session cookie в raw flow-ID value и очищает
  flow cookie; duplicate POST даёт identical success без второй/revoked session. Это
  заменяет прежний terminal-invalid concurrent-loser outcome.
- Commit создаёт provisional session/receipt с `PEXPIREAT` original deadline. Первый
  validated request с session cookie атомарно claim-ит session, удаляет receipt и
  включает one-year sliding TTL. Если cookie не доставлена, оба keys истекают без
  orphan/janitor; claimed session receipt expiry не revoke-ит.
- Provisional session скрыта из active-session list и даёт только low-card auth metric.
  Ordinary-login claim ровно один раз создаёт audit/web-inbox/locked-on primary-bot
  outbox, idempotently keyed session digest. Same-browser re-auth notification нет;
  delivery failure не rollback-ит claim и retry-ится независимо.
- Та же atomic claim function делает `XADD` durable `session_claimed` event в Stream
  session Valkey с random UUID, `user_id`, provider, safe device summary, claim time и
  internal session digest, без raw token, IP и cookie. Worker одной PostgreSQL
  transaction idempotently создаёт audit/web-inbox/outbox с UNIQUE guards по session
  digest и event ID и делает `XACK` только после commit; pending/retry не дублируют
  записи, delivery остаётся независимой outbox-задачей.
- `session_claimed` не использует terminal DLQ: transient/deterministic failures
  бессрочно retry-ятся с capped backoff/jitter и alert, оставаясь в PEL без trim,
  ack или drop. После PostgreSQL commit следуют `XACK` и `XDEL`; cleanup acknowledged
  остатка повторяем. Метрики low-card: pending count, oldest age и retry count без IDs.
- Для него выделены отдельные Stream и consumer group в session Valkey. Existing worker
  process запускает отдельный loop без нового deployment unit; webhook policy не
  переиспользуется, readiness/lag/PEL проверяются отдельно, failover использует общий
  60-second `XAUTOCLAIM` lifecycle.
- Projector lag не блокирует claim после successful atomic claim + `XADD`: synchronous
  wait/poll и PEL age/size gate отсутствуют, worker readiness/alerts показывают delay.
  Failure commit этой Valkey operation не может вернуть success и идёт в принятую
  ambiguous-outcome recovery.
- Recipient target в event не snapshot-ится: projector под existing user/identity
  locks выбирает current primary и transactionally создаёт audit/web-inbox/outbox.
  Primary-change race определяется commit order; Valkey не хранит recipient PII, а
  deactivated-account security exception сохраняется.
- Server UTC `claimed_at` становится immutable `occurred_at`, а PostgreSQL transaction
  time — отдельным `recorded_at`. UI показывает occurred time; audit/inbox order —
  `occurred_at`, event ID. Recorded time нужен только для projection-lag diagnostics.
- Exact integer `schema_version = 1` и strict closed Malli payload обязательны.
  Missing/unsupported/malformed event остаётся в PEL, alert-ит и делает group unready
  без mutation/default/coercion/ack/DLQ; переходная multi-version support живёт только
  до drain старого PEL.
- Safe device summary — одна immutable closed map browser/OS/device-class из maintained
  parser с bounded catalogs и `unknown` fallback. Raw UA, exact versions, model,
  language и IP не сохраняются; parse failure claim не блокирует.
- `auth_provider=telegram|max` берётся только из consumed-token binding и остаётся
  historical auth source; current primary определяет лишь delivery target. External
  provider identity ID отсутствует, unlink/primary change source не переписывают.
- Одна append-only audit row с independent UNIQUE `event_id`/`session_digest` является
  idempotency root для связанных inbox/outbox. Exact replay успешен; guard/payload
  mismatch rollback-ится, alert-ит и остаётся pending без `XACK`, partial projection
  невозможна.
- `event_id` — generated-before-dispatch UUIDv7, сохранённый в claim state/`XADD` и
  переиспользуемый retry/read-back как audit root/FK key. Stream ID — только PEL/
  `XACK`/`XDEL` transport offset, не domain ID; DB replacement/UUIDv4 нет.
- Failed event остаётся PEL/backoff/unready, но bounded fair worker без global/per-user
  HOL продолжает later valid events; один Stream ID имеет один in-flight owner.
  Same-user DB lock защищает races, order отображается по occurred time/event ID без
  strict messenger delivery guarantee.
- Session revoke/logout/expiry не отменяет historical audit/inbox/pending delivery.
  Dispatcher не re-read-ит session; message показывает occurred time/source/safe device
  и generic session-management action без active-state claim или direct revoke target.
- `new_session` outbox фиксирует internal target identity при projection. Later primary
  change не делает retarget/duplicate и не переписывает delivery; linked target получает
  её и после потери primary status. Unlink exact target до send terminally завершает
  ordinary delivery без retry/retarget и detached-target snapshot. Snapshot запрещён для
  `new_session`; closed exception определён отдельно только для `identity_unlinked`.
  Audit/web-inbox сохраняются, target ID в метрике
  отсутствует.
- Projector transaction фиксирует `delivery_locale` (`ru|en`, fallback `ru`) в
  `new_session` outbox; retry не перечитывает locale, а его смена влияет только на
  будущие messenger notifications. Web-inbox остаётся semantic и локализуется текущим
  UI locale. Outbox содержит notification type, schema/template version и allowlisted
  semantic fields без rendered text или user-supplied free form.
- Dispatcher рендерит exact записанный `template_version`. Immutable bundled templates
  имеют обязательные `ru`/`en`, проверяемые при startup, и сохраняются минимум 90 дней
  после последнего producer deployment версии. Missing version не вызывает Bot API:
  это deterministic failure с обычным alert/DLQ и replay после возврата шаблона, без
  silent fallback на current version.
- Template catalog key — notification type, `template_version` и locale, без provider.
  Telegram/Max получают одинаковые wording, semantic fields и meaning; adapters меняют
  только escaping, markup, equivalent controls и transport limits. Если кнопка
  недоступна, тот же safe HTTPS URL включается в text без другого editorial template.
- Каждый security template с maximum allowlisted values в `ru`/`en` обязан помещаться
  в одно сообщение обоих providers: одна outbox delivery — один atomic send, без split,
  truncation или удаления fields. CI/startup проверяют worst case по более строгому
  Telegram/Max limit; invalid catalog останавливает dispatcher consumer и его readiness,
  а превышение требует новой короткой template version.
- Notification type + `schema_version` задают strict closed Malli payload schema;
  каждая `template_version` совместима ровно с одной schema version. Projector до
  insert валидирует required fields/types/no extras; failure rollback-ит transaction,
  оставляет `session_claimed` в PEL и alert-ит. Dispatcher повторно валидирует stored
  payload; mismatch не render-ится и не вызывает Bot API, а идёт в deterministic
  alert/DLQ. Coercion/defaults/ignored unknown fields отсутствуют.
- Notification schema/template pair включается двухрелизным expand/activate rollout:
  первый release добавляет её в catalog/dispatcher без смены producer, второй successful
  release переключает producer constant, когда previous rollback image уже знает обе
  versions. Старые versions сохраняются минимум 90 дней. Runtime registry, dual-write
  и silent downgrade отсутствуют; contract test проверяет emitted pair в current и
  rollback catalogs.
- Retire старой schema/template pair разрешён только после 90 дней с last emission и
  при zero replayable undelivered/DLQ references. Deploy preflight группирует references
  по notification type/schema/template version и блокирует removal; rewrite/drop/silent
  upgrade records ради gate запрещены. Delivered records, audit, semantic web-inbox и
  backup copies removal не блокируют.
- `new_session` outbox сохраняет closed `manage_sessions` action code без absolute URL,
  query, redirect target или action token. Catalog разрешает его в named internal route;
  dispatcher строит same-origin canonical HTTPS URL из current `PUBLIC_BASE_URL`, а
  startup проверяет HTTPS/allowlist. Смена base URL обновляет queued link, но не locale,
  template version или semantic fields.
- Generated `manage_sessions` link identifier-free: без user/session/notification/
  outbox IDs и provider/locale/tracking query. Canonical route при отсутствии session
  использует обычный auth flow с server-side allowlisted return target. Click-specific
  analytics/audit нет; разрешены aggregate route/status HTTP metrics без user/outbox
  labels, исключая leakage через provider/history/logs/referrer.
- Если link открыт под другим authenticated TrailBase account, route показывает current
  display name и только current account sessions. Navigation URL не bind-ит/switch-ит
  account и не раскрывает expected recipient; смена требует explicit logout/re-auth.
  Automatic switch/merge, target-account lookup и existence disclosure запрещены.
- Sessions-management authenticated GET, htmx partial, mutation, validation/error и
  redirect responses всегда ставят `Cache-Control: no-store` и
  `Referrer-Policy: no-referrer`; Caddy/CDN их не кэшируют. History сохраняет только
  identifier-free URL, не body. Cacheable summary и persistent client cache запрещены.
- `occurred_at` хранится как UTC Instant. Messenger всегда показывает explicit `UTC`;
  locale меняет format/wording, но не zone. Account timezone не хранится и не infer-ится
  из IP/provider/language. Web локализует Instant browser `Intl`, показывает zone/offset
  и UTC `datetime`, с explicit UTC fallback без client rendering.
- Messenger показывает только absolute `occurred_at` до минут с explicit `UTC`, без
  `recorded_at`, send/attempt time или relative age; retry/DLQ replay не меняют timestamp.
  Web relative age допустим только secondary рядом с обязательным absolute local time,
  вычисляется client-side и не хранится/outbox-ится.
- Каждый distinct `new_session` claim даёт отдельные audit/inbox/delivery; batching по
  minute/provider/device, fingerprint coalescing, replacement и collapsed inbox rows
  запрещены. Dedupe только exact `event_id` + `session_digest` replay. UI показывает
  count и отдельные occurred times; strict messenger order не вводится.
- Flood сохраняет каждую audit/inbox/outbox row; suppression/digest/coalescing нет.
  Dispatcher использует provider-wide budget, per-target pacing и fair target queues с
  provider `Retry-After`. Local throttle оставляет row pending без attempt/DLQ; один
  noisy account не блокирует других. Метрики queue depth/oldest age/throttled count
  имеют только provider label, без target IDs.
- `429 Retry-After` сохраняет в PostgreSQL provider-wide
  `cooldown_until=max(current,parsed)`; triggering call расходует attempt, waiting rows —
  нет. Все loops/restarts соблюдают cooldown, другой provider работает. Invalid/missing
  header получает capped jittered fallback; ingress/projection/inbox не блокируются,
  telemetry содержит только provider label.
- Adapters используют closed errors `provider_blocked|target_unreachable|rate_limited|
  transient|unclassified`. Global auth/config failure открывает durable provider circuit,
  останавливает calls, оставляет rows pending без attempts и делает readiness failed;
  другой provider работает. Target error terminal только для row; unclassified идёт в
  DLQ/alert без circuit. Raw body/target ID/credentials не логируются и не DLQ-ятся.
- `target_unreachable` оставляет identity linked/primary и не меняет future routing:
  exact delivery terminal без retry/retarget, audit/inbox остаются, а future events
  выбирают explicit current primary. Другой linked provider не получает fallback или
  duplicate. Route меняют только user unlink/change-primary; automatic account mutation
  запрещена.
- `target_unreachable` durable ставит identity `delivery_health=unreachable` с observed
  timestamp без raw reason; latest observed outcome под row lock побеждает stale race.
  Web показывает generic warning и change-primary/unlink. Successful ordinary send
  exact identity либо validated inbound user message от exact linked provider identity
  ставит `reachable`; inbound использует application acceptance time, не provider
  timestamp. Health не влияет на auth/linking/routing/queue.
- `unreachable` не имеет TTL/manual dismiss; time, restart, circuit close и primary
  change его не очищают. Recovery даёт только ordinary send success exact identity или
  validated inbound user message exact linked identity; unlink удаляет state, relink
  начинает с `unknown`. Non-primary warning остаётся только в identity card.
- Inbound recovery принимает только validated user-authored private-chat message event
  exact linked identity, включая commands, text и media. `callback_query`, inline query,
  membership/service events и delivery receipts health не меняют.
- Provider event dedupe выполняется до health mutation. Только первое accepted unique
  message ставит `reachable`/application acceptance timestamp; exact replay получает
  idempotent acknowledgement без DB update и не может перекрыть более свежий
  `target_unreachable` новым observed timestamp.
- Accepted inbound message exact linked identity обновляет health и для deactivated
  account, поскольку reachability не является authorization. Это не reactivate-ит
  account, не создаёт session/notification и не обходит command policy; retained health
  используется после будущей reactivation.
- Inbound recovery и unlink exact identity сериализуются identity row lock. Unlink-first
  не позволяет event обновить/recreate/relink identity; inbound-first может поставить
  `reachable`, после чего unlink удаляет row и health. Old provider identity не
  attach-ится к future relink.
- Concurrent inbound `reachable`/outbound `target_unreachable` используют application
  observation time до ожидания lock: accepted-event time после dedupe и normalized
  provider-result receipt соответственно. Под lock применяется только более новое;
  commit/provider timestamps и send-start не участвуют, exact tie выигрывает
  `unreachable`.
- `delivery_health_observed_at` не показывается owner-у. Web/private-chat card имеет
  actionable warning только для `unreachable`; `unknown|reachable` не получают badge/
  timestamp. Поле остаётся internal ordering/operations data, не last activity или last
  successful delivery.
- Warning — derived UI projection, не messenger delivery/web-inbox/outbox/notification.
  Web показывает его при owner view; private-chat settings — только через другую working
  linked identity. Message ранее blocked exact bot сначала делает inbound recovery, и
  его card уже без warning; без working identity bot warning невозможен, остаётся Web.
- Closed warning copy: RU «Доставка в этот мессенджер недоступна. Разблокируйте бота,
  если нужно, и отправьте ему сообщение.»; EN “Delivery to this messenger is
  unavailable. Unblock the bot if needed, then send it a message.” Он общий для
  Web/cross-identity card, без raw cause, рядом с change-primary/unlink.
- `unreachable -> reachable` только убирает warning при следующем card render/read.
  Recovery toast/flash/bot reply/notification/inbox/outbox/audit отсутствуют; normal
  response исходной message не объявляет channel recovery.
- Открытая Web card не refresh-ится автоматически после inbound recovery: polling,
  SSE/WebSocket отсутствуют. Она может быть stale до navigation/обычного htmx card
  refresh; следующий server render читает authoritative row. Bot card обновляется при
  следующем user request; `Cache-Control: no-store` сохраняется.
- `user_identities` хранит `delivery_health text NOT NULL DEFAULT 'unknown'` с CHECK
  `unknown|reachable|unreachable` и nullable `delivery_health_observed_at timestamptz`;
  compound CHECK связывает unknown/NULL и observed/non-NULL. Enum, raw reason, manual
  clear и отдельная health-history table отсутствуют.
- Отдельной delivery-check функции нет: Web/private-chat action, probe routes,
  `delivery_kind=health_probe`, outbox/status/polling/cooldown и test-message templates
  отсутствуют.
- Post-dispatch timeout/network error — ambiguous commit, не mutation-free `503`.
  Bounded retry/read-back по opaque commit ID должен подтвердить success/non-commit;
  только pre-dispatch или proven non-commit получает сохраняющий flow `503`.
  Still-unknown outcome идёт в recoverable branch без cleanup или blind second session.
- Freshness для appeal link создаёт только bound explicit private-chat
  «Подтвердить вход», сразу выпускающий token; recent messages, notifications,
  background events и browser activity не считаются fresh auth. Второго prompt/PIN нет.
- Link можно получить через любую active linked identity, не только primary. Consume
  re-check-ит exact identity/account binding; invalid membership terminal, а после
  success freshness применяется к account независимо от provider.
- Same-browser re-auth с valid same-user current session не создаёт ложный
  `new_session` notification: current credential/CSRF заменяются, other sessions
  неизменны. Без такой session commit создаёт provisional ordinary login, а обычное
  locked-on security notification появляется только при claim.
- Rotated same-browser record сохраняет original `created_at`, обновляет
  credential/CSRF/freshness/last-seen/safe-UA; sliding TTL на один год начинается после
  provisional claim, отдельного `reauthenticated_at` нет.
- Server-rendered form содержит один 128-bit base64url `idempotency_key` для обеих
  buttons. Committed outcome сохраняет только UNIQUE SHA-256 hash без raw key или
  отдельной idempotency table/TTL. Same hash и exact normalized removal/outcome/reason
  payload идемпотентны; payload mismatch получает `idempotency_key_reused`, новый key
  после commit — `appeal_already_decided`. Key/hash не логируются и не заменяют CSRF.
- Commit/idempotent retry выполняют PRG на canonical GET с full UUID: full-page POST
  отвечает `303`, htmx navigates к тому же GET, отдельного success fragment нет.
  Final `already_decided` summary показывает committed outcome и «Решение сохранено»;
  browser refresh безопасен. Validation остаётся на form, conflict reload-ит summary.
- Error matrix: `422` rerender-ит same form/values/key с field errors; `409` заменяет
  stale form/key fresh authoritative summary и stable code; `503` сохраняет values/key
  для manual retry и не заявляет, committed ли outcome. Automatic retry/polling нет;
  `422/409` side-effect-free, uncertain COMMIT разрешается same-key retry. Htmx и
  full-page semantics одинаковы, все responses `no-store` без internal details.
- Closed appeal POST codes: `409` допускает только `appeal_already_decided`,
  `appeal_window_closed`, `appeal_not_current`, `appeal_not_restorable` и
  `idempotency_key_reused`, а `503` — только `appeal_temporarily_unavailable`.
  `409` deterministically выбирает `already_decided`, `window_closed`, `not_current`,
  recomputed restore-blocked/ready либо fresh same-case summary. `503` не утверждает
  authoritative state/commit result. Template выбирается по status+code и не принимает
  free-form backend/internal error details.
- Admin appeal action принимает только `uphold_removal` или `restore_after_appeal`,
  ссылку на исходный removal decision и обязательную admin reason. Оба outcome
  добавляют audit event без изменения исходного removal; uphold не меняет lifecycle,
  restore является отдельной state mutation. Generic/partial/custom outcome
  отклоняется.
- Без `track_appeal_decide` либо fresh auth admin appeal action не загружает sensitive
  context и не создаёт outcome/audit/outbox. Permission mapped только к `admin`; через
  moderator UI/endpoints или bot операция недоступна.
- Заполненная appeal outcome+reason form завершается одним final submit без второго
  modal/confirm screen. Перед submit видимы short track ID, current state, выбранный
  outcome, его последствия и обязательная reason; используются только явно
  разделённые «Подтвердить удаление» и «Восстановить трек», без generic «Сохранить».
  Пока request выполняется, controls disabled; backend повторно проверяет permission,
  fresh auth, single-shot и lifecycle invariants, retry не создаёт второй result.
- Form содержит только static localized 10-minute fresh-auth hint без exact expiry или
  live timer. Polling/auto-refresh/auto-submit отсутствуют; POST проверяет freshness
  server-side и при expiry использует принятый discard/re-auth/fresh-summary flow.
- GET render и POST mutation применяют один predicate
  `now < fresh_authenticated_at + 10 minutes`; при equality freshness expired. UX
  safety margin и отдельный render threshold отсутствуют, поэтому expiry во время
  заполнения обрабатывается существующим discard/re-auth flow.
- POST фиксирует один server `authorization_now` в authoritative mutation guard перед
  preconditions/первой mutation. Если check успешна, transaction завершается без
  второй wall-clock проверки у commit boundary, даже когда physical commit произошёл
  после deadline; последующая queue/lock latency freshness не отзывает.
- `now` для appeal GET/POST предоставляет один injected application UTC clock с
  `java.time.Instant` semantics. PostgreSQL/client clocks freshness не решают;
  application instances синхронизируют UTC system clocks, отдельного DB time query нет.
- Тот же application clock фиксирует `fresh_authenticated_at` при server-side
  validation/acceptance explicit bot webhook/callback. Provider timestamp не задаёт
  freshness и отбрасывается после payload validation; dedupe/binding используют
  provider event независимо от его времени.
- Token/session auth records не хранят provider timestamp: только server
  `fresh_authenticated_at` и opaque event/identity bindings для dedupe/consume.
  Operational events/logs/metrics в MVP его также не получают; observability
  ограничена server-time counters по provider/validation result. Отдельный
  delivery-delay signal может появиться только будущим явным решением с retention.
- Appeal re-auth использует общий единственный counter
  `fresh_auth_confirmation_total{provider,result}` с закрытыми provider
  `telegram|max` и result
  `accepted|duplicate|invalid_event|invalid_binding|account_unavailable|internal_error`;
  identifiers, timestamps и дополнительные labels отсутствуют.
- Его `duplicate` означает exact event replay только после committed acceptance и даёт
  `2xx` без token/link/message/session mutation; pre-commit `internal_error` остаётся
  retryable в общем worker/DLQ flow.
- Existing provider-event dedupe record для appeal re-auth хранит
  `processing|accepted` семь дней. Atomic ingress claim+enqueue создаёт processing;
  worker после token/link issuance ставит accepted. Processing replay side-effect-free
  acknowledge-ится, accepted replay increment-ит duplicate; нового storage нет.
- Worker одной atomic Valkey operation создаёт единственный 128-bit 10-minute
  `web_session` token record и переводит processing в accepted до Bot API send. Crash
  не разделяет state/token, delivery retries переиспользуют exact link.
- Для той же provider identity atomic issuance заменяет active-token pointer, удаляет
  previous token record и его ещё существующий raw delivery field. Previous event
  marker остаётся accepted; отдельные bot message, revoke audit и notification не
  создаются.
- `/auth` consume previous link и new issuance используют pointer как Valkey
  linearization point. Consume-first удаляет matching token/pointer и может завершить
  auth; issuance-first заменяет pointer и делает old link terminal invalid. Поздняя
  issuance не revoke-ит создаваемую/созданную browser session; distributed
  Valkey/PostgreSQL transaction отсутствует.
- Active-token pointer хранится отдельным non-secret Valkey key по internal identity
  UUID, только как SHA-256 token digest, с `PEXPIREAT` exact token deadline. Consume
  удаляет его с token record, replacement заменяет; missing/expired pointer или target
  terminal invalid и очищается idempotently. Delivery data, sliding TTL и janitor не
  добавляются.
- `web_session` record адресуется тем же `SHA-256(raw_token)`, что pointer; raw token
  отсутствует в record/key. Unsalted SHA-256 достаточен для 128-bit CSPRNG token без
  custom HMAC/salt. Raw value остаётся лишь в link/browser request и short-lived
  delivery field; raw token и digest запрещены в logs/metrics/traces/DLQ.
- Raw token временно находится только в delivery field accepted dedupe record до
  successful send/10-minute expiry. После очистки seven-day marker non-secret; raw
  value отсутствует в logs/metrics/traces/DLQ и отдельном namespace.
- Valkey 9.x minimum: worker задаёт `HPEXPIREAT` field на token deadline, success делает
  `HDEL`; key-level accepted TTL остаётся seven days без janitor/второго expiry key.
- Не доставленный до expiry appeal re-auth link завершается terminally без stale send,
  re-issuance, изменения accepted marker или отдельного user notification. Existing
  event replay остаётся `duplicate`; только новое explicit «Подтвердить вход» action
  создаёт новый provider event/freshness/token/link. Failure отражается только общими
  Bot API delivery metrics/alert без raw token.
- Его post-issuance delivery имеет максимум пять total attempts с exponential
  backoff/jitter. Retry разрешён только для timeout/network, `429` и `5xx`, если
  следующая attempt с `Retry-After` укладывается в original token deadline. Остальные
  `4xx`, exhausted budget и retry за deadline terminal сразу: raw field удаляется
  idempotent `HDEL`, accepted marker остаётся, DLQ/late replay не создаются.
  Pre-issuance `internal_error` остаётся в общем worker retry/DLQ flow.
- Appeal reason принимает только `support_request_verified`,
  `administrative_correction` или `other`; для `other` требуется validated audit-only
  note 1–1000 code points. Unknown code, invalid/missing note отклоняются без outcome;
  code/note не показываются owner-у и не логируются.
- Appeal outcome имеет UNIQUE `removal_decision_id`. Same-key retry того же outcome
  возвращает прежний результат без нового audit/outbox; второй либо competing outcome
  получает exact `409` error `appeal_already_decided` и не меняет state. Reopen/second
  appeal нет; correction выполняется отдельным append-only lifecycle action.
- Restore прежнего published track использует тот же approved immutable snapshot без
  новой moderation. Track без прежней approved revision становится private/editable и
  требует нового owner resubmit. Ни одна already-decided removed revision повторно не
  становится queue-eligible; её export остаётся discarded.
- После moderator removal export прежнего approved current snapshot остаётся private
  до uphold или 90-day purge, но canonical routes отвечают `404` и signed URL не
  выдаются. Restore атомарно возвращает его в published без regeneration/restoring;
  uphold/purge удаляют object. Это исключение не применяется к unapproved pending
  export, который остаётся discarded и удаляется сразу.
- `uphold_removal` сохраняет исходный purge deadline, `restore_after_appeal` ставит
  `purge_at = NULL`; ни один outcome не продлевает и не перезапускает retention. Race с
  purge worker сериализован track lock: purge-first блокирует restore, restore-first
  отменяет eligibility worker-а.
- Для первого нового appeal outcome общая eligibility требует current active removal,
  отсутствующий outcome и `now() < purge_at`. После deadline или newer lifecycle state
  недоступны обе операции независимо от lag purge worker-а; out-of-band case срок не
  продлевает. До deadline full-track lock/unavailable retained export отключают только
  restore, uphold остаётся доступным. Same-outcome idempotent retry committed result
  продолжает работать.
- `restore_after_appeal` succeeds только для current active removal до purge, без newer
  lifecycle event/full lock и с ready retained export, когда восстанавливается
  published snapshot. Иначе exact `409` error `appeal_not_restorable` не создаёт
  outcome/audit/outbox/delete; temporary block можно устранить и повторить request.
- Successful first appeal commit создаёт ровно одну locked-on notification/outbox:
  uphold содержит fact/short ID/deadline, restore — published link либо private
  edit/resubmit action. Internal admin context отсутствует; retry/conflict/failure не
  создаёт duplicate, disabled owner следует общей moderation suppression.
- После `changes_requested` или moderator removal pending revision export становится
  discarded и private object удаляется. Новый immutable resubmit после
  `changes_requested` создаёт собственный export; late completion не может вернуть
  discarded export в ready.
- `geometry_simplified_z11/13/15` populated для revision с tolerances 40/10/2 м.
- GPX attachment в private chat без предварительного `/upload` после quota/slot
  checks создаёт upload flow и draft-track с распарсенными auto-derived. `/upload`
  только показывает инструкцию, а attachment другого типа не создаёт job. User
  подтверждает final name и primary activity и отправляет revision на moderation без
  web session. Summary показывает CC BY 4.0 reminder/link; description optional,
  auto-derived metrics не требуют отдельных подтверждений.
- Один bot upload использует одну status card на `upload_job_id`: переходы стадий
  редактируют её best-effort, success показывает draft actions, failure — безопасную
  причину/retry. Ошибка provider edit может создать replacement card, но не второй
  upload job.
- Explicit retry после исчерпания transient attempts повторно занимает slot и
  requeue-ит тот же job не более одного concurrent attempt; сохранённый raw не
  загружается повторно. Permanent file error и отсутствующий raw требуют новый
  attachment.
- Metadata text применяется только как reply на prompt конкретного upload job; иначе
  bot просит выбрать один из active uploads.
- Name, Activity и optional metadata можно менять в любом порядке из draft summary
  card. Submit без Name/Activity не создаёт moderation revision и возвращает точный
  список пропусков; заполненный draft показывает final summary/license confirmation.
- Name precedence детерминирован: file-level metadata name, затем единственное
  distinct непустое track name, затем нормализованный filename stem. Разные track
  names не выбираются и не concatenated; warning виден пользователю. Default требует
  final confirmation, raw filename отсутствует в storage metadata/public export.
- Description precedence детерминирован: file-level metadata description, затем
  единственное distinct непустое track description. Несколько разных values дают
  warning и пустой optional field без concatenation; выбранный default попадает
  только в upload-time owner locale.
- Caption входного attachment не появляется в draft/public Description. Только GPX
  `<desc>` и explicit bound Description input могут заполнить это поле.
- GPX `<desc>` без language marker попадает только в сохранённый UI locale owner на
  момент upload; смена UI locale не меняет эту ветвь, а другой перевод добавляется
  explicit metadata action.
- Web/bot detail показывает Description fallback на вторую locale с явной меткой
  языка; JSON response сохраняет обе branches, а пустой map не создаёт секцию.
- GPX export с двумя descriptions содержит один XML-escaped `<desc>` с deterministic
  `[ru]`/`[en]` blocks; один approved revision не порождает варианты по viewer locale.
- GPX export использует root `creator="TrailBase revision:<uuid>"` и стандартные
  metadata author/copyright-license/link для immutable author display name, CC BY 4.0
  и canonical track URL; provider/internal identity, raw filename и application
  extensions отсутствуют.
- Четвёртый upload не принимается, пока один из трёх flows не освободит slot.
- Cancelled-after-parse draft остаётся доступным owner и не удаляется автоматически.
- Resume не позволяет превысить три active upload flows.
- Metadata mutation со stale lease generation отклоняется и не перезаписывает более
  свежие изменения из другого interface.
- Подтверждённый takeover из web/chat работает без дополнительного bot round-trip.
- Старый interface не может изменить metadata после takeover; provider notification
  failure не влияет на committed generation.
- Idle-expired draft можно позднее снова открыть через обычный slot-checked resume.
- Потеря Valkey инвалидирует prompts/controls, но не меняет lease ownership или число
  занятых slots.
- Деактивация во время parse не оставляет orphan data: job завершается private draft
  либо обычным terminal failure/cleanup, но не переходит в moderation автоматически.
- Конкурентные upload/resume не могут создать четвёртый active flow или два active
  leases одного draft даже без предварительного `COUNT`.
- Expired lease не вызывает ложный отказ по полному лимиту, даже если janitor ещё не
  запускался.
- Конкурентные lifecycle operations одного user сериализованы, но не блокируют flows
  других users.
- Cancel-wins не создаёт draft и чистит temporary raw; commit-wins применяет
  недеструктивный post-parse cancel.
- Reparse не переписывает опубликованный snapshot; он создаёт новую revision с
  versioned algorithms.

**Non-goal**: классификация/difficulty/season (M07), POI autodetect (M05), фото (M08).

---

## M04 — Catalog Render (Map Island)

**Объём**: ключевой срез. MapLibre-остров + OSM-raster basemap + adaptive GeoJSON delivery + bridge glue.

**Файлы/компоненты**:
- `src/trailbase/render/map.clj` — Hiccup partial для контейнера MapLibre (с `hx-disable` / помечен вне swap-области).
- `resources/public/js/map.js` — MapLibre init, raster source (OSM tiles), POI-cluster source + track-polyline source, event handlers (`moveend`, `click`).
- `resources/public/js/bridge.js` — alpine store `Alpine.store('map')`, htmx.ajax из map events, setData на track source.
- Сервер: `/api/tracks.geojson?bbox=&zoom=` → honeysql query: `ST_AsGeoJSON(ST_Force2D(geometry_simplified_zNN))` по zoom-aware колонке + `(WHERE ST_Intersects(geometry, bbox))` GIST.
- Сервер: `/api/locations/clusters.geojson?bbox=&zoom=` → server-side cluster для low-zoom (см. M05; в M04 — заглушка, возвращает по одному центроиду на track как POI-замену).
- Sidebar (htmx-зона): `/tracks` → partial; альпin-bridge: hover трек в списке → `map.setFeatureState({highlight: true})`.
- Track detail view: `/tracks/:id` → partial с мини-картой (на full zoom) + метаданные.

**Acceptance**:
- Карта показывает OSM-raster basemap.
- Full-locked tracks отсутствуют в GeoJSON/sidebar/catalog results, а прямой detail
  URL otherwise published track отвечает generic `503` с `Cache-Control: no-store`.
- Уже закэшированный GeoJSON может сохранять track до 90 секунд согласно
  `max-age=30, stale-while-revalidate=60`; lock не запускает cache purge.
- z0–12 получает server-side POI clusters; z13/z14/z15 используют соответственно
  z11/z13/z15 simplified geometry.
- На z16+ z15 остаётся context layer, а full geometry загружается только для
  выбранного track.
- Hover трека в sidebar → полилиния на карте стилизована (highlight).
- Performance: 5000 треков в тестовой выборке — pan/zoom без lag (>30 FPS) в Chrome.

**Non-goal**: POI синтаксис (M05), real search (M06), классификация (M07).

---

## M05 — POI Gazetteer

**Объём**: versioned location catalog, semantic categories, revision annotations,
autodetect, moderated OSM import и deterministic cluster endpoint.

**Файлы/компоненты**:
- Миграции создают `locations`, `location_revisions`, category dictionary,
  revision-location links и multiple ordered occurrences.
- `src/trailbase/locations.clj` — CRUD для модератора, list/get, autodetect-along-track (honeysql with `ST_DWithin` / `ST_Intersects` radius-aware), cluster endpoint.
- ОSM-import pipeline: `src/trailbase/import/osm.clj` — Overpass API fetch by osm_id/type, кэш name/geometry в `locations`, provenance record.
- Autodetect создаёт revision-location annotations с confidence/status. High-confidence
  links к approved locations одобряются автоматически; остальные идут в moderation.
  Новая POI всегда создаётся через заявку.
- Low-zoom cluster endpoint использует global hex-grid в EPSG:3857; MapLibre
  client-side clustering отключён.
- Bot уведомляет о завершении autodetect и ведёт deep-link в web preview; POI CRUD в
  боте не дублируется.
- Мoderator UI: `/moderation/locations` — список заявок, approve → creates location → re-runs autodetect binding.

**Acceptance**:
- После загрузки трека backend autodetect-ит POI вдоль трека (radius-aware), показывает их в загрузчик-форме.
- Загрузчик может предложить новое POI через форму (имя, тип, координаты или клик на карте) — заявка в модерацию.
- Модератор одобряет заявку → location revision публикуется → annotations
  пересчитываются.
- `/api/locations/clusters.geojson?zoom=10` возвращает ≤500 кластер-точек для bbox.
- Поиск «треки через локацию X» работает по approved revision-location annotations.
- OSM-import: модератор вводит osm_id+type → backend fetches Overpass → создаёт location с cached name/geometry.

**Non-goal**: модерация тегов (M07), фото POI.

---

## M06 — Search

**Объём**: единый PostgreSQL search engine — текст × гео × фасеты × POI-join. Instant-search.

**Файлы/компоненты**:
- Миграции `006-search.up.sql`:
  - GENERATE ALWAYS columns: `ts_description_ru to_tsvector('russian', description->>'ru')`, `ts_description_en to_tsvector('english', description->>'en')`, GIN индексы.
  - `pg_trgm` extension для name fuzzy matching, триграм-индекс на `name`.
  - compound indexes на `(activity_type)`, `(difficulty)`, `(season)`, `(duration_s)` buckets.
- `src/trailbase/search.clj` — honeysql composable query builder: combine text
  (`tsvector` + `pg_trgm` fallback), geo bbox, facets и approved POI annotations.
- `/search` отдаёт HTML partial, `/api/v1/search` — JSON; оба используют opaque
  HMAC-protected keyset cursor и server-side disjunctive facet counts.
- Telegram/Max `/search` использует тот же domain search service, filters и keyset
  pagination, адаптированные к chat controls; browser session не требуется.
- Group/channel `/search` использует public principal: только published public tracks,
  без account creation, персональных данных и сохранения history/settings.
- Private-chat `/search` linked deactivated identity использует тот же stateless
  public principal без private results, account-specific facets или history/settings.
- Group-search controls связаны с provider/chat/message/requester; только инициатор
  запроса может менять filters или page общего результата.
- Channel search без стабильной requester identity возвращает статическую первую
  страницу без controls и предлагает продолжить в private chat.
- Search controls в private/group chats истекают через 15 минут от создания без
  продления; expired callback не редактирует старое сообщение.
- Chat-search callback содержит только случайный 128-bit opaque ID; binding,
  query/filters/cursor и absolute expiry хранятся в Valkey.
- Каждый успешный control callback атомарно заменяет текущий opaque ID новым с тем же
  исходным `expires_at`; только победитель ротации редактирует message.
- Transient/ambiguous provider edit повторяется с тем же новым ID; terminal failure
  удаляет новый state без rollback старого.
- Search-result edit имеет максимум пять total attempts с backoff/jitter; retry только
  для network/timeout, `429` и `5xx`, не позже исходного `expires_at`.
- Callback acknowledgement отправляется после binding/expiry validation и atomic
  rotation, до query/edit; `bot-worker` обновляет result асинхронно.
- Query timeout/transient failure после ACK удаляет новый state без rollback,
  сохраняет прежний result content и terminal edit убирает controls.
- Instant-search: `hx-get` on `keyup changed delay:300ms` → htmx partial swap `#results` и `#facets`.
- Location-constraint: `location_id=N` → join approved revision annotations.
- Geography-aware: bbox в segmented radio — server postgis geometry comparison.

**Acceptance**:
- Free-text поиск по track name/description: tsvector + триграмы, опечатки 1-2 символа учитываются.
- Full-locked tracks отсутствуют в search results и facet counts для любого principal.
- Фильтр `activity=hike`, `difficulty=T3`, `season=winter`, `duration_min=3600 & duration_max=21600` (1-6h) — composable AND.
- Комбинированный: текст + фасеты + bbox одновременно — один SQL-запрос через CTE.
- POI-filter: `location_id=5` → список треков через локацию 5.
- Instant-search: ввод 3+ символов → partial swap без перезагрузки страницы; сервер откликивается <200ms на индексированных данных.
- `#facets` обновляется server-side aggregation (activity counts, difficulty distribution) при каждом изменении фильтра.
- Мультиязычность: `description` jsonb с языковыми ветвями; tsvector собирается из всех языков, ranking по combined.
- Bot `/search` возвращает те же результаты и permission-filtered facets, что web/API,
  с постраничной навигацией в chat.
- Group/channel search никогда не возвращает private/unlisted tracks; bot upload и
  state-changing commands остаются private-chat-only.
- Search от deactivated identity возвращает только public results и не создаёт новый
  account или persistent personal history/settings.
- Нажатие group-search control другим участником не меняет результат и предлагает ему
  запустить отдельный `/search`.
- Channel search без requester identity не создаёт callback state; ссылка для
  продолжения в private chat не содержит auth token.
- Через 15 минут search control предлагает повторить `/search` и не меняет старый
  result message.
- Потеря Valkey callback state безопасно инвалидирует controls; query и requester
  identity отсутствуют в provider callback payload.
- Повторный или конкурентный callback со старым ID считается stale и не может
  перезаписать новый search result.
- После исчерпания provider-edit retries result становится неинтерактивным и предлагает
  повторить `/search`.
- Permanent `4xx` не retry-ится; исчерпанный ephemeral edit не попадает в DLQ/replay.
- Stale/foreign/expired callback получает нейтральный acknowledgement, не ротирует
  state и не изменяет общее сообщение.
- При query failure старый result остаётся видимым без controls и предлагает повторить
  `/search`; terminal edit использует общий provider retry policy.

**Non-goal**: external search engine (Meilisearch), semantic search (pgvector).

---

## M07 — Classification & Moderation

**Объём**: tag dictionary + очереди заявок тегов; difficulty/season/duration facet wiring; модераторский API.

**Файлы/компоненты**:
- Миграции `007-tags.up.sql`:
  - `tags(id, key, label_i18n jsonb, parent_id, status enum approved/pending/rejected, created_by, approved_by)`.
  - `revision_tags(track_revision_id, tag_id, source enum user/moderator/derived)`.
  - `tag_requests(id, requested_tag_label, requested_by, status, declined_reason, created_at, reviewed_by, reviewed_at)`.
- `src/trailbase/tags.clj` — словарь (list, CRUD модератор), запрос нового тега, одобрение, привязка к треку.
- `src/trailbase/walks/facets.clj` — обёртка для difficulty (per-activity lookup: SAC для hike, MTB-scale для bike, 3-level fallback), season bitmask, duration handling.
- UI: `/tracks/:id/edit` — частичная форма с тегами (autocomplete из approved-словаря) + фасетами (difficulty по current activity_type, season checkboxes, duration manual override if `duration_source=manual`).
- Moderation queue: `/moderation/tags` — список pending tag_requests; approve → создаёт `tags` row; assign to track (если request был из track edit).
- M03 auto-suggestion теперь стersistие tags suggestions помимо activity_type (basic heuristic — `alpine` if high elev/hike, `loop` if start≈end point).

**Acceptance**:
-Approved-теги видны как autocomplete при редактировании трека.
- Юзер без модераторской роли не может добавить новый тег из ниоткуда; может только предложить → queue.
- Модератор видит очередь, approve/reject с reason; approve создаёт tag в словаре.
- Difficulty widget меняется по activity_type (hike → SAC-T1..T6 selector; bike → S0..S5; fallback → 3-level).
- Season bitmask хранится и фильтруется по multiple bits (`season & 1`, `season & 2`, ... — winter=8 etc.).
- M06 search facet-агрегация теперь использует настоящие tags/difficulty/season — end-to-end работает.

**Non-goal**: ML-based tag suggestion (basic heuristic only).

---

## M08 — Photo + Elevation

**Объём**: фото трека (S3, presigned), профиль высоты (alpine island), gallery lightbox.

**Файлы/компоненты**:
- Миграции `008-photos.up.sql` — `track_photos(id, track_id, s3_uri, caption, taken_at?, seq)`; optionally geometry from EXIF for map EXIF-plots.
- `src/trailbase/photos.clj` — upload flow (multipart → S3 via aws-simple-sign+hato → insert row), presigned URLs (rotating expiry), delete.
- Htmx / alpine integration: `/tracks/:id/photos` partial-ul — список + upload-form (`hx-post` with progress indicator).
- Alpine island for gallery lightbox: `x-data="{open, index}"` + keyboard nav; lazy-swipe for many photos.
- Elevation profile chart использует LTTB-profile до 2 000 samples из track revision;
  полный point array в PostgreSQL не хранится.
- Map EXIF-plots: photos with `geometry` from EXIF shown as markers on MapLibre (separate source).
- Presigned URL rotation: photo URLs expire; client refetches через htmx when needed.

**Acceptance**:
- Upload фото к треку → S3 → row в таблице; превью в галерее.
- Lightbox: click → fullscreen; ← → keyboard nav; Esc-close.
- Elevation profile: drawn from parsed GPX altitudes (if `<ele>` present); hover on chart → marker on map at corresponding track point; brush selection → zoom on chart for section.
- EXIF photos appear as markers on map (if EXIF contains GPS).
- Presigned URLs expire but no visible broken images — URLs rotate via alpine/htmx refetch.

**Non-goal**: video, live-tracking, social features.

---

## Cross-cutting concerns (投融资 не по срезам)

- **telemere** логирование: с M01; structured logs в каждом handler.
- **Malli schemas** для API валидации: с M01 (health schema) и расширяется каждый срез.
- **reitit coercion** полной цепочки middleware: с M01.
- **bot module** (raw Bot API over hato): Telegram и Max входят в M02; конец M02 —
  работающие identity и разрешённые chat operations через оба provider.
- **tests**: по одному integration test на каждый срез (smoke-level: upload fixture GPX, render map page, search returns expected track). Не TDD по слоям, smoke на end-to-end.
- **I18n**: user-selectable UI locales — `ru`, `en`; `simple` используется только как
  technical fallback для tags/POI names и full-text processing.
- **Backup retention**: удалённый из primary raw хранится в encrypted operator-only
  off-host backup не более 30 дней и не восстанавливается в active service.
- **Restore safety**: после disaster restore при остановленных web/workers operator
  запускает CLI, удаляющую все S3 keys без DB references; traffic закрыт до её
  успешного завершения.
  CLI по умолчанию dry-run; destructive mode всегда делает fresh scan.
  Полностью промаркированные track-scoped нарушения допускают degraded startup;
  incomplete/unmappable result — нет.

## Orders & Dependencies

- M01 — prerequisite всех.
- M02 — после M01 (moderate подмамины — bot, auth).
- M03 — после M02 (ownership требует messenger identity/account).
- M04 — после M03 (карта рендерит реальные tracks).
- M05, M06, M07, M08 — после M04; ** могут разрабатываться параллельно** (independent extensions; не блокируют друг друга кроме небольших hooks):
  - M06 search facet-aggregation нормально работает до M07 (tags NULL = no facets), а после M07 facets оживает.
  - M05 POI может отставать — M04 тогда использует fallback as-centroids; при готовом M05 cluster endpoint просто переключается.
  - M08 фото не зависит от M06/M07.

## Definition of Done (per slice)

- Acceptance criteria выполнены и продемонстрированы.
- Smoke-test green: запуск `bb run-dev` + fixture-data (через миграции/seed) → end-to-end scenario runnable.
- Миграции — мигration + rollback оба проходят на empty-DB.
- telemere logs хотя бы в ключевых handlers; Malli schemas на endpoint inputs.
- ADR-ссылки в PR-описании; новые ADR если design отклонился от roadmap.
