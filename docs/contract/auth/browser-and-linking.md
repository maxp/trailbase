# Контракт: browser session и identity linking

Часть [Implementation Contract](../../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

### 4.2 Опциональная browser session и identity linking

- Web-session и identity-link tokens живут в Valkey 10 минут и расходуются атомарным
  `GETDEL`.
- Каждый web-auth/link token consume дополнительно читает PostgreSQL account status.
  Только active account может завершить flow; Valkey token сам по себе не является
  достаточным authorization state.
- Если PostgreSQL status прочитать нельзя, token authorization работает fail-closed
  без cached-active fallback. HTTP flow возвращает `503` с `Retry-After` и не создаёт
  session/link/mutation; bot flow завершается transient failure без domain changes и
  проходит обычный retry/DLQ. После восстановления PostgreSQL исходный token можно
  использовать повторно, только если он всё ещё действителен.
- Для каждой provider identity действительна только последняя web-session ссылка.
  Atomic issuance нового `web_session` token заменяет per-identity active-token
  pointer, удаляет previous token record и очищает его raw delivery field, если оно ещё
  существует. Previous provider-event dedupe record сохраняет accepted marker; новые
  bot messages, отдельный revoke audit и notification не создаются.
- Гонка `/auth` consume previous link и новой issuance линейризуется их atomic Valkey
  operations. Consume проверяет совпадение active-token pointer с предъявленным token
  и одной operation удаляет token record и pointer. Если consume linearize-ился
  первым, auth flow может завершиться, а более поздняя issuance не отзывает уже
  созданную или создаваемую browser session. Если первой linearize-илась issuance,
  previous link terminal invalid. Distributed transaction между Valkey и PostgreSQL
  для этой гонки отсутствует.
- Active-token pointer хранится отдельным non-secret Valkey key, scoped к internal
  identity UUID, и содержит только SHA-256 token digest. Issuance ставит pointer и
  `PEXPIREAT` ровно на absolute token deadline в той же atomic operation; consume
  удаляет его вместе с token record, replacement заменяет. Missing/expired pointer или
  missing target делает token terminal invalid и очищается idempotently. Pointer —
  token index, не delivery storage; sliding TTL и janitor отсутствуют.
- `web_session` token record адресуется только по `SHA-256(raw_token)` — тому же digest,
  который хранит active-token pointer. Raw token отсутствует в record и Valkey key;
  unsalted SHA-256 достаточен при 128-bit CSPRNG entropy, custom HMAC/salt scheme не
  вводится. Raw token существует только в user-facing link/browser request и
  short-lived delivery field. Ни raw token, ни digest не попадают в logs, metrics,
  traces или DLQ.
- `web_session` и `link_identity` — разные token namespaces/purposes. Link token
  содержит `target_user_id` и ожидаемый candidate provider, но не candidate
  `provider_user_id`.
- Для browser re-auth отдельного token namespace нет. Новая `web_session` link
  выдаётся только в private chat после fresh bot authentication; server-side token
  record сохраняет исходный `fresh_authenticated_at`. Тот же `/auth` GET/POST flow
  атомарно consume-ит token, создаёт новую browser session и использует bound
  `return_to`; отдельные re-auth token/table/cookie/consume endpoint отсутствуют.
- Fresh bot authentication для выпуска `web_session` link — только explicit
  user-initiated action «Подтвердить вход» в private one-to-one chat. Validated
  command/callback bound к provider/user/chat/message/requester сразу выпускает link,
  а `fresh_authenticated_at` фиксируется injected application UTC clock в момент,
  когда server валидировал и принял исходный webhook event. Provider-supplied event
  timestamp не участвует в freshness и отбрасывается после validation входного
  payload; provider event dedupe/binding остаются отдельными обязательными checks.
  Обычное недавнее сообщение, notification click, background event или existing
  browser session не создают freshness. Дополнительные PIN/password/TOTP и вторая
  confirmation-кнопка отсутствуют.
- Provider-supplied event timestamp не сохраняется в `web_session` token record или
  browser session. Auth state содержит только server `fresh_authenticated_at` и уже
  принятые opaque provider event/identity bindings, необходимые для dedupe и consume
  validation. В MVP этот timestamp также не пишется в operational events, logs или
  metrics; observability ограничена server-time counters по provider и validation
  result без raw provider payload и отдельной retention policy.
- Единственная special-purpose fresh-auth metric — counter
  `fresh_auth_confirmation_total{provider,result}`. Label `provider` закрыт значениями
  `telegram|max`, `result` —
  `accepted|duplicate|invalid_event|invalid_binding|account_unavailable|internal_error`.
  User/chat/event/request identifiers, timestamps, routes, error details и иные labels
  отсутствуют; latency/status покрывают общие HTTP/webhook metrics.
- Fresh-auth `duplicate` означает только повтор exact provider event, для которого
  action-level acceptance уже committed. Provider получает success/`2xx`, но новый
  token/link не выпускается, bot message не редактируется и не отправляется, session/
  auth state не меняется; increment-ится только `duplicate`. Первый committed
  acceptance один раз выпускает link и increment-ит `accepted`. `internal_error` до
  commit не становится `duplicate` и продолжает общий worker retry/DLQ flow.
- Для fresh-auth существующая 7-day provider-event dedupe record имеет только state
  `processing|accepted`; отдельной table или второго key namespace нет. Ingress одной
  атомарной Valkey operation создаёт `processing` и enqueue-ит event. Replay при
  `processing` получает `2xx` через общую webhook dedupe без fresh-auth result
  increment или side effects; исходный event продолжает worker retry/DLQ. После
  выпуска token/link worker переводит record в `accepted`; replay этого state
  increment-ит `duplicate`. Оба state используют исходный seven-day TTL.
- Worker генерирует ровно один 128-bit `web_session` token и одной atomic Lua/Valkey
  operation проверяет processing event/owner bindings, создаёт его 10-minute token
  record и переводит dedupe state `processing -> accepted`. Operation выполняется до
  любого Bot API request; crash не оставляет accepted без usable token или live token
  при processing. External delivery не входит в atomic boundary и при retry обязана
  использовать тот же link, не выпуская новый credential.
- Та же atomic issuance заменяет per-identity active-token pointer. Если previous
  `web_session` token ещё не истёк, operation удаляет его token record и raw delivery
  field в прежней accepted dedupe record, но не меняет сам accepted marker. Новое bot
  message сверх обычной доставки нового link, revoke audit и notification для этой
  замены отсутствуют.
- `/auth` consume и issuance используют active-token pointer как общий linearization
  point. Consume-first атомарно проверяет pointer и удаляет token record/pointer, после
  чего может завершить auth даже при более поздней issuance; issuance-first заменяет
  pointer и делает previous link terminal invalid. Более поздняя issuance не отзывает
  уже созданную/создаваемую browser session; cross-store distributed transaction нет.
- Pointer является отдельным non-secret per-identity Valkey key только с SHA-256 token
  digest. Atomic issuance задаёт ему `PEXPIREAT` на тот же absolute token deadline,
  consume удаляет с token record, replacement заменяет. Missing/expired pointer или
  missing target terminal invalid и очищаются idempotently; raw token, sliding TTL,
  janitor и delivery data в pointer отсутствуют.
- `web_session` record lookup key — только `SHA-256(raw_token)`, совпадающий с pointer
  digest; raw token в record/key отсутствует. При 128-bit CSPRNG entropy используется
  unsalted SHA-256 без custom HMAC/salt. Raw value остаётся лишь в link/browser request
  и short-lived delivery field; raw token и digest запрещены в logs/metrics/traces/DLQ.
- Для crash-safe delivery retry raw link token хранится только как short-lived delivery
  field той же accepted dedupe record. Field доступно не дольше token expiry
  (`fresh_authenticated_at + 10 minutes`), не попадает в logs, metrics, traces или DLQ
  payload и атомарно удаляется после successful Bot API delivery. После delivery/expiry
  seven-day record сохраняет только non-secret accepted outcome/event/identity
  bindings для replay classification. Deterministic/custom token derivation, второй
  delivery namespace и повторная выдача token отсутствуют.
- Minimum runtime Valkey 9.x. Worker atomic script задаёт delivery field absolute
  expiry через `HPEXPIREAT` ровно на token deadline; successful Bot API send выполняет
  idempotent `HDEL`. После logical field expiry `HGET` не возвращает raw token, тогда
  как key-level TTL non-secret accepted marker остаётся seven days. Janitor, polling и
  отдельный TTL key отсутствуют.
- Если successful Bot API delivery не завершилась до token/delivery-field expiry,
  worker прекращает delivery terminally: stale link не отправляется, token не
  перевыпускается, accepted record не меняется и отдельное user notification не
  создаётся. Exact replay исходного provider event остаётся `duplicate`; только новое
  explicit «Подтвердить вход» action создаёт новый provider event, freshness, token и
  link. Failure отражается только в общих Bot API delivery metrics/alert без raw token.
- Post-issuance fresh-auth Bot API delivery выполняет максимум пять total attempts с
  exponential backoff и jitter. Retry разрешён только для timeout/network errors,
  `429` и `5xx`; `Retry-After` соблюдается, только если следующая attempt укладывается
  в исходный token deadline. Остальные `4xx`, исчерпание пяти attempts и невозможность
  уложить следующую attempt в deadline terminal сразу: worker делает idempotent
  `HDEL` raw delivery field, сохраняет accepted marker и не создаёт DLQ/late replay.
  Pre-issuance `internal_error` по-прежнему проходит общий worker retry/DLQ flow.
- Re-auth link можно выпустить через любую active linked Telegram/Max identity, не
  только через primary provider: primary управляет delivery, но не trust level.
  Token record bound к exact internal identity UUID, provider, user и issuance event.
  Перед session rotation `/auth` повторно проверяет в PostgreSQL, что identity всё ещё
  active и принадлежит тому же active account. Unlinked/foreign identity делает token
  terminal invalid без fallback на primary; successful consume создаёт account-level
  freshness без provider restriction для новой session.
- Если `POST /auth` одновременно видит valid current browser session того же `user_id`,
  re-auth rotation считается credential refresh той же browser/device continuity, а
  не новым login. Она выдаёт новые session token и CSRF state, переносит новый
  `fresh_authenticated_at`, отзывает прежнюю current session и не меняет другие
  sessions, но не создаёт web-inbox/primary-bot `new_session` notification. Если
  current session отсутствует, invalid либо принадлежит другому user, auth commit
  создаёт provisional ordinary new login, а принятое locked-on security notification
  появляется только при claim этой session.
- Same-browser rotated session наследует original `created_at`, получает новый session
  token hash и CSRF state, `fresh_authenticated_at` из web-session token record,
  `last_seen_at` равный времени successful `POST /auth` и заново вычисленный safe short
  User-Agent summary текущего request. После provisional claim Valkey key получает
  стандартный one-year sliding TTL; до claim он capped original flow deadline.
  Отдельное `reauthenticated_at` не хранится; без valid same-user current session
  ordinary new session получает новый `created_at`.
- Link flow инициируется из authenticated account context. До выдачи link token
  требуется fresh bot authentication не старше 10 минут через уже привязанную
  identity; browser request дополнительно требует session-bound CSRF.
- Валидированный provider webhook `/start <link-token>` из private one-to-one chat
  доказывает контроль candidate identity. Если provider совпадает и identity ещё не
  связана, backend атомарно расходует token и добавляет `user_identities` к target
  account. Новый `users` row в linking mode не создаётся, browser session для
  completion не требуется.
- Если candidate identity уже принадлежит другому account, link отклоняется;
  автоматического merge нет.
- Просроченный, использованный, повреждённый или предназначенный для другого provider
  link token обрабатывается fail-closed: backend не создаёт `users`, не добавляет
  identity и не fallback-ит на обычный `/start`. Ответ не раскрывает причину или target
  account и предлагает получить новую ссылку либо отдельно отправить plain `/start`
  без payload для создания собственного account.
- Initial `GET /auth` с query parameter `token` всегда redirect-only и не расходует
  web-session token: preview/scanner не должен сжечь вход. Для valid token GET создаёт
  существующую short-lived auth-flow запись в Valkey, выдаёт
  `__Host-trailbase_auth_flow` cookie с одноразовым form nonce и отвечает
  `303 See Other` на чистый `GET /auth`. Malformed/unknown/expired/superseded token
  получает `303` на token-free `GET /auth` со stable non-secret `result=invalid`
  marker. Target GET не lookup-ит и не меняет existing flow-cookie/record, показывает
  одинаковую generic-invalid page без reason/identifier; plain `/auth` без marker
  продолжает existing confirmation. Marker не добавляет server-side state/cookie.
  Initial response не рендерит body, raw token отсутствует в redirect target и
  обычной browser history. JavaScript, `history.replaceState`, local/session storage и
  новый credential namespace не используются.
- Auth-flow record, flow-cookie и form nonce имеют один absolute expiry, равный
  original `web_session` token deadline. Initial redirect не выдаёт новый ten-minute
  interval; confirmation GET, retry и `503` не продлевают срок. Expired/missing любой
  component показывает ту же generic-invalid page, idempotently удаляет оставшиеся
  flow record/cookie и не consume-ит, не восстанавливает и не перевыпускает source
  token. Sliding expiry отсутствует.
- Auth-flow record хранит только source `web_session` token digest, allowlisted
  `return_to`, original deadline и SHA-256 form-nonce hash. Flow-cookie — независимый
  128-bit CSPRNG opaque ID; Valkey lookup key строится из SHA-256 cookie value. После
  initial redirect raw source token не хранится. Raw flow ID остаётся только в
  `HttpOnly` cookie, raw nonce — только в hidden POST field; provider/user bindings
  читаются из source token record. Raw flow ID, nonce и их digests отсутствуют в logs,
  metrics, traces и DLQ.
- Flow-cookie contract: exact name `__Host-trailbase_auth_flow`, `Secure`, `HttpOnly`,
  `SameSite=Lax`, `Path=/`, без `Domain`. `Max-Age`/`Expires` не превышают remaining
  source-token deadline. Cookie очищается через `Max-Age=0` после success, terminal
  invalid/expiry или explicit cancel и остаётся отдельной от
  `__Host-trailbase_session`.
- Новый valid initial auth GET при existing flow-cookie одной atomic Valkey operation
  удаляет previous flow record, создаёт новую и заменяет cookie: на browser остаётся
  ровно один active auth-flow. Previous tab/form получает generic-invalid без side
  effects. Её source token не consume-ится и не revoke-ится; исходную link можно снова
  открыть до deadline, что, в свою очередь, заменит текущий browser flow.
- Failed flow-cookie/form-nonce validation на `POST /auth` никогда не consume-ит source
  token и не удаляет valid flow. Missing/unknown cookie и nonce mismatch дают одну
  generic-invalid response без token/pointer/session mutations. Unknown cookie
  очищается у browser через `Max-Age=0`; если cookie адресует valid record, wrong или
  stale nonce сохраняет record и не очищает и не заменяет cookie, поэтому old tab не
  может сбросить current valid form. Действует общий `/auth` rate limit 10 запросов в
  минуту на IP; отдельного nonce-attempt counter нет.
- Rate-limit reject на `/auth` — retryable `429 Too Many Requests` с `Retry-After`, а
  не terminal invalid. Один budget 10/min на normalized client IP охватывает initial
  token GET, clean confirmation GET и POST; client IP принимается только из trusted
  Caddy forwarding contract. Limiter срабатывает до credential lookup и не очищает и
  не меняет flow/token/cookie/nonce/session. Ответ использует `Cache-Control: no-store`,
  `Referrer-Policy: no-referrer` и generic body без redirect на `result=invalid`;
  отдельных per-flow/per-nonce budgets, automatic retry и terminal marker нет.
- Все non-transient invalid `POST /auth` используют Post/Redirect/Get: POST не рендерит
  body и отвечает `303` на token-free `/auth` со stable non-secret query marker
  `result=invalid`; target GET показывает generic-invalid без lookup или изменения
  current flow. Cleanup выполняется до redirect только по принятому классу ошибки:
  wrong/stale nonce для valid record сохраняет flow/cookie. Transient PostgreSQL или
  Valkey failure остаётся retryable `503` с `Retry-After` без redirect, terminal marker
  и token/flow/session mutations; retryable flow, token и cookie сохраняются.
- После successful PostgreSQL active identity/account check `POST /auth` имеет одну
  Valkey linearization point. Одна atomic function валидирует flow/nonce, source token
  record и active-token pointer, promote-ит digest текущего 128-bit flow ID в session
  namespace, создаёт новую либо same-browser rotated session, consume-ит source token
  и pointer и заменяет active flow record completion receipt; все credential mutations
  выполняются вместе либо не выполняются. Distributed transaction с PostgreSQL нет, а
  failure его предварительной проверки возвращает принятый mutation-free `503`.
- Completion receipt живёт до original flow deadline и содержит только nonce hash,
  allowlisted `return_to`, committed session digest/status и expiry; raw flow/session ID
  остаётся только в browser cookie. Success и matching retry ставят
  `__Host-trailbase_session` в то же raw value и очищают
  `__Host-trailbase_auth_flow`. До receipt expiry concurrent/повторный POST той же form
  возвращает identical success по existing flow-ID digest как opaque commit identifier:
  вторая session не создаётся и committed session не revoke-ится. Это заменяет прежний
  terminal-invalid outcome для проигравшего concurrent POST.
- Auth commit создаёт provisional session и completion receipt с `PEXPIREAT` ровно на
  original flow deadline. Первый request с новой session cookie после обычной
  PostgreSQL active-account validation одной atomic Valkey operation помечает session
  claimed, удаляет receipt и устанавливает standard one-year sliding TTL. Если cookie
  не была доставлена, оба provisional keys истекают без orphan и cleanup job; receipt
  expiry не revoke-ит уже claimed session. PostgreSQL validation failure сохраняет
  provisional state и использует обычный mutation-free `503` до deadline.
- Provisional session не входит в active-session list и создаёт только low-cardinality
  auth metric. Claim является user-visible login point: ordinary new login ровно один
  раз создаёт audit, web-inbox record и locked-on primary-bot `new_session` outbox
  intent, idempotently keyed по session digest. Same-browser re-auth notification
  по-прежнему не создаёт. Durable outbox delivery failure не откатывает claimed
  session и retry-ится независимо.
- Та же atomic claim function одной Valkey operation после claim state transition
  выполняет `XADD` durable `session_claimed` event в Stream session Valkey. Event
  содержит random UUID, `user_id`, provider, safe device summary, claim time и internal
  session digest только для idempotency; raw token, IP и cookie отсутствуют. Worker
  одной PostgreSQL transaction idempotently создаёт audit, web-inbox record и
  notification outbox с UNIQUE guards по session digest и event ID, после commit
  выполняет `XACK`. Pending/retry не создают duplicates; delivery остаётся обычной
  outbox-задачей и не откатывает claimed session.
- `session_claimed` является явным исключением из общего retry/DLQ policy: transient и
  internal schema/deterministic failures retry-ятся бессрочно с capped exponential
  backoff и jitter, остаются pending в PEL и поднимают operational alert; terminal
  DLQ, ack/drop и trim unacked entries запрещены. Только после PostgreSQL commit worker
  выполняет `XACK`, затем `XDEL`; acknowledged entry при сбое cleanup остаётся
  безвредной и удаляется повторной очисткой. Наблюдаемость ограничена low-cardinality
  pending count, oldest age и retry count без event, session или user identifiers в
  logs и metrics.
- `session_claimed` использует dedicated Stream и consumer group в session Valkey,
  отдельно от webhook ordering, retention и пяти-попыточного retry/DLQ policy. Новый
  service/container не создаётся: существующий worker process запускает отдельный
  consumer loop. Для этой group отдельно проверяются readiness и low-cardinality
  lag/PEL metrics; общий shutdown lifecycle и `XAUTOCLAIM` после 60 секунд без
  heartbeat применяются без отдельного механизма failover.
- Unhealthy или lagging `session_claimed` projector не блокирует новые claims, пока
  atomic claim + `XADD` успешно commit-ятся. Эта durable Valkey operation является
  request-path commit boundary: web не ждёт PostgreSQL projection, не poll-ит worker и
  не вводит fail-closed gate по PEL age/size; lag отражается через worker
  readiness/alerts. Если session Valkey не может commit-ить claim + event, success
  запрещён и применяется уже определённая ambiguous-outcome recovery без обхода
  `XADD`.
- `session_claimed` event не содержит recipient snapshot или target identity.
  Projector под существующим user/identity lock order выбирает current primary linked
  identity и в той же PostgreSQL transaction создаёт audit, web-inbox и locked-on
  outbox target. Если primary change commit-нулся раньше, используется новый target;
  если projector commit-нулся раньше, outbox сохраняет прежний. Так Valkey не хранит
  recipient PII или stale snapshot; для deactivated account применяется существующее
  исключение обязательных security notifications.
- Atomic claim function получает server-generated UTC `claimed_at` и сохраняет его в
  `session_claimed`. Projector записывает это immutable время факта как `occurred_at`,
  а PostgreSQL transaction clock — отдельно как `recorded_at`; processing time не
  подменяет login time. User-visible inbox/message показывает `occurred_at`, audit и
  inbox сортируются по `occurred_at`, затем event ID для deterministic ties.
  `recorded_at` используется для projection-lag diagnostics и не переопределяет время
  события.
- `session_claimed` обязан содержать exact integer `schema_version = 1` и полный payload
  strict closed Malli schema. Consumer валидирует version, required fields и types до
  любых PostgreSQL mutations; missing/unsupported version или malformed field остаётся
  pending в PEL, поднимает alert и делает group unready без default/coercion, `XACK`
  или DLQ. Breaking change использует coordinated rollout с временной поддержкой
  переходных versions только до drain старого PEL; постоянный compatibility layer не
  сохраняется.
- `safe device summary` — closed normalized map только из `browser_family`,
  `os_family` и `device_class`, вычисленная server-side maintained User-Agent parser.
  Browser catalog: `chrome|safari|firefox|edge|webview|other|unknown`; OS:
  `android|ios|windows|macos|linux|other|unknown`; device class:
  `mobile|tablet|desktop|other|unknown`. Parse failure даёт `unknown` и не блокирует
  claim. Одна immutable map используется в session, event, audit и notification; raw
  User-Agent существует только в памяти request, а точные versions, device model,
  language и IP не сохраняются и не попадают в telemetry.
- Поле `auth_provider` в `session_claimed` содержит immutable source provider
  `telegram|max` consumed `web_session` token и берётся только из validated token
  binding, не из request parameter, browser или current primary. Audit/inbox/message
  показывают локализованный источник подтверждения входа; delivery target независимо
  выбирается по current primary в projector transaction. Provider user/identity ID в
  event отсутствует, а последующие unlink или primary change не переписывают
  historical fact.
- Одна append-only auth audit row является PostgreSQL idempotency root с двумя
  независимыми UNIQUE guards по `event_id` и `session_digest`, не composite. Та же
  transaction после root создаёт связанные web-inbox и outbox rows. Exact replay с
  совпадающими identifiers и immutable payload использует существующие rows как
  idempotent success. Conflict только по одному guard либо с другим payload считается
  integrity incident: transaction rollback, event остаётся pending в PEL, alert и no
  `XACK`. Independently deduped partial projections запрещены.
- `event_id` — application-generated UUIDv7, созданный один раз до первого dispatch
  atomic claim function. First claim сохраняет его в session claim state и `XADD`, а
  retry/read-back переиспользует то же значение. Projector использует `event_id` как
  primary/idempotency key audit root и FK target inbox/outbox. Valkey Stream ID остаётся
  только transport offset для PEL, `XACK` и `XDEL`, не сохраняется как domain ID;
  DB-generated replacement ID и UUIDv4 отсутствуют.
- Failed/pending `session_claimed` не создаёт global или per-user head-of-line block.
  Consumer использует bounded pool и fair scheduling новых/due-retry entries; для
  одного Stream ID одновременно существует не более одного in-flight owner. Malformed
  или transient event остаётся в PEL со своим backoff/alert и держит group unready, но
  чтение later valid entries продолжается. Same-user races сериализует существующий
  PostgreSQL user lock; audit/inbox сортируются по `occurred_at,event_id`. Strict
  messenger delivery order не гарантируется, сообщение всегда показывает occurred
  time.
- Audit, web-inbox и pending `new_session` delivery не подавляются и не изменяются,
  если session revoked, logout-нута или expired до projection/send. Это immutable
  security fact произошедшего claim, поэтому projector/dispatcher не перечитывают
  current Valkey session для suppression. Message содержит `occurred_at`,
  `auth_provider`, safe device summary и generic action «Управление сессиями», но не
  direct revoke target и не утверждение, что session всё ещё active; поздняя доставка
  остаётся truthful.
- `new_session` outbox фиксирует internal target identity в projector transaction.
  Последующая смена primary не делает retarget/duplicate и не переписывает delivery:
  пока selected identity linked, сообщение идёт ей, даже если она уже не primary. Если
  exact target identity физически unlinked до send, ordinary delivery завершается
  terminal без retry/retarget и без detached-target snapshot. Такой snapshot запрещён
  для `new_session`; closed exception определён отдельно только для
  `identity_unlinked`. Audit и web-inbox сохраняются, а метрика terminal outcome
  не содержит target ID.
- В той же projector transaction `new_session` outbox фиксирует `delivery_locale` из
  сохранённого account locale (`ru|en`) с fallback `ru`. Dispatcher и его retry не
  перечитывают позднейшую настройку; её изменение влияет только на будущие messenger
  notifications. Web-inbox хранит semantic record и локализуется при чтении в текущем
  UI locale. Outbox хранит notification type, schema/template version и только
  allowlisted semantic fields, без заранее rendered text и user-supplied free form.
- Dispatcher рендерит только exact записанный `template_version`; versioned templates
  immutable и bundled с приложением, а наличие `ru` и `en` для каждой версии
  валидируется при startup. Версия сохраняется минимум 90 дней после последнего
  producer deployment этой версии, покрывая undelivered/DLQ replay window. Missing
  version — deterministic delivery failure без Bot API call: обычные alert/DLQ и replay
  после возврата шаблона применяются, silent fallback на current version запрещён.
- Каталог шаблонов индексируется по notification type, `template_version` и locale, но
  не по provider: одна версия задаёт одинаковые wording, semantic fields и meaning для
  Telegram и Max. Provider adapters меняют только escaping, markup, эквивалентную форму
  controls и transport limits. Если provider не поддерживает нужную кнопку, adapter
  включает тот же safe HTTPS URL в текст, не выбирая отдельный editorial template.
- Каждый security template обязан с максимальными allowlisted values для `ru` и `en`
  помещаться в одно сообщение обоих providers без splitting или truncation: одна
  outbox delivery равна одному atomic provider send. CI и startup validation проверяют
  worst-case rendering по более строгому лимиту Telegram/Max; invalid catalog не
  запускает dispatcher consumer и делает его readiness failed. Adapter не обрезает и
  не выбрасывает semantic fields; превышение исправляется новой короткой template
  version.
- Каждая пара notification type и `schema_version` имеет strict closed Malli payload
  schema, а `template_version` объявляет ровно одну совместимую `schema_version`.
  Projector валидирует required fields, types и отсутствие extras до outbox insert;
  failure откатывает PostgreSQL transaction, оставляет `session_claimed` в PEL и
  поднимает alert. Dispatcher повторяет defensive validation: stored mismatch не
  вызывает rendering/Bot API и идёт в обычный deterministic alert/DLQ. Coercion,
  defaults и игнорирование unknown fields запрещены.
- Новые notification schema/template versions активируются только двухрелизным
  expand/activate rollout. Expand release добавляет новую pair в catalog/dispatcher,
  пока producer пишет старую; только следующий successful release явно переключает
  producer constant, поэтому previous rollback image уже поддерживает обе versions.
  Старые versions остаются минимум на 90-дневный replay window. Runtime capability
  registry, dual-write и silent downgrade отсутствуют; contract test требует, чтобы
  current и rollback catalogs поддерживали emitted pair.
- Удаление старой schema/template pair требует одновременно минимум 90 дней с её
  последней emission и ноль ссылок из replayable undelivered/DLQ records. Deploy
  preflight группирует их по notification type, `schema_version` и `template_version`
  и fail-closed блокирует removal при ненулевом count. Такие records нельзя rewrite,
  drop или silently переводить на новую version ради gate. Delivered records, audit,
  semantic web-inbox и backup copies старого template не требуют и removal не блокируют.
- `new_session` outbox хранит только closed action code `manage_sessions`, не absolute
  URL, query, redirect target или action token. Catalog связывает code с named internal
  sessions-management route; dispatcher при send строит same-origin canonical HTTPS URL
  из текущего `PUBLIC_BASE_URL`. Startup validation проверяет HTTPS и allowlisted route.
  Смена public base URL намеренно обновляет queued delivery link, не меняя сохранённые
  locale, template version или semantic event fields.
- Generated `manage_sessions` link полностью identifier-free: в ней нет user, session,
  notification/outbox IDs либо provider, locale и tracking query parameters. Она ведёт
  только на canonical named route; отсутствующая browser session проходит обычный auth
  flow с server-side allowlisted return target. Click не создаёт notification-specific
  analytics или audit. Допустимы только aggregate HTTP metrics по route/status без
  user/outbox labels, чтобы identifiers не утекали в provider, history, logs/referrer.
- Если identifier-free link открыт в browser, уже authenticated под другим TrailBase
  account, sessions-management route показывает current account display name и только
  его sessions. Link является navigation, не identity/authorization proof, не bind-ит
  и не переключает account и не позволяет infer-ить ожидаемого получателя. Для смены
  account нужен explicit logout и обычный re-auth; automatic switch/merge,
  target-account lookup и сообщение о существовании другого account запрещены.
- Все sessions-management HTML/htmx responses и redirects имеют
  `Cache-Control: no-store` и `Referrer-Policy: no-referrer`: authenticated GET,
  partial, mutation, validation/error response и redirect. Caddy/CDN их не кэшируют;
  browser history может сохранить только identifier-free canonical URL, но не session
  list/response body. Cacheable summary endpoint и client-side persistent sessions
  cache в MVP запрещены.
- Domain и outbox хранят `occurred_at` только как UTC Instant. Messenger `new_session`
  всегда показывает его с явной меткой `UTC`; notification locale меняет format/wording,
  но не timezone. Account timezone setting не добавляется, зона не выводится из IP,
  provider profile или language. Web-inbox/sessions UI локализуют тот же Instant через
  browser `Intl`, показывают zone/offset и machine-readable UTC `datetime`; при
  недоступном client rendering fallback остаётся explicit UTC.
- Messenger показывает только absolute immutable `occurred_at`, форматированный до
  минут с explicit `UTC`; `recorded_at`, delivery/attempt time и relative «только что»/
  «N минут назад» отсутствуют. Initial send, retry и DLQ replay поэтому сохраняют один
  truthful timestamp. Web может вычислять relative age только как secondary presentation
  рядом с обязательным absolute local time; relative value не хранится и в outbox не
  входит.
- Каждый distinct `new_session` claim создаёт отдельные audit, web-inbox и messenger
  delivery без batching/coalescing по minute, provider или device summary. Dedupe
  применяется только к exact replay с теми же `event_id` и `session_digest`; два разных
  claims остаются разными security facts даже при одинаковых user-visible fields.
  Time-window/fingerprint batching, replacement и collapsed inbox row запрещены. UI не
  скрывает count/отдельные occurred times; strict delivery order не добавляется.
