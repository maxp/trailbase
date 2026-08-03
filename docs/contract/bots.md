# Контракт: webhooks, bot workers и notifications

Часть [Implementation Contract](../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 8. Webhooks и bot workers

- Используются только webhooks; polling не реализуется.
- Регистрация Telegram и Max webhooks выполняется отдельной идемпотентной командой
  `bb webhook-register`, а не при старте приложения.
- У каждого бота отдельный webhook secret. Основная защита — native provider
  signature/secret и строгая payload validation; IP allowlist optional.
- Неверный/отсутствующий secret получает одинаковый `404`. Security log не содержит
  secret или body.
- Webhook body ограничен 1 MiB. Secret проверяется до полного JSON parsing, насколько
  позволяет adapter.
- Валидный неизвестный event получает `200`, игнорируется и логируется как факт:
  provider, event type/id, schema version, request ID. Лог rate-limited; полный payload
  не пишется.
- Невалидный payload получает `400`, не ставится в очередь и увеличивает метрику.
- После validation и dedup событие помещается в Valkey Stream, ingress сразу отвечает
  `2xx`.
- `bot-worker` обрабатывает события одной `(provider, provider_user_id)`
  последовательно, разных identities — параллельно.
- Dedup provider event ID хранится семь дней. Business operations дополнительно
  идемпотентны в PostgreSQL.
- Для explicit fresh-auth confirmation эта же dedupe record содержит
  `processing|accepted`: claim `processing` и Stream enqueue атомарны. Processing
  replay только acknowledge-ится; accepted replay increment-ит специальный
  `duplicate`. Новый namespace/table и отдельный TTL отсутствуют.
- Worker atomic Lua/Valkey operation одновременно создаёт единственный 10-minute
  `web_session` token record и меняет processing на accepted до Bot API send. Delivery
  retry использует тот же link; внешний network call не является частью commit.
- При issuance нового token для той же provider identity эта operation также заменяет
  active-token pointer, удаляет previous token record и его ещё существующий raw
  delivery field. Previous accepted marker сохраняется; отдельные bot message, revoke
  audit и notification не создаются.
- Гонку previous-link consume/new issuance разрешает порядок atomic Valkey operations:
  consume-first удаляет matching token record/pointer и может завершить auth;
  issuance-first заменяет pointer и terminally отклоняет old link. Поздняя issuance не
  отзывает создаваемую/созданную session; distributed Valkey/PostgreSQL transaction не
  вводится.
- Active-token pointer — отдельный non-secret Valkey key по internal identity UUID,
  содержащий только SHA-256 token digest. Он получает `PEXPIREAT` exact token deadline
  при issuance и удаляется/заменяется atomic consume/replacement. Missing pointer или
  target terminal invalid и idempotently очищается; sliding TTL/janitor отсутствуют.
- Token record адресуется тем же `SHA-256(raw_token)`, что pointer, и не хранит raw
  value в record/key. 128-bit CSPRNG token использует unsalted SHA-256 без custom
  HMAC/salt. Raw token допустим только в link/browser request и short-lived delivery
  field; raw и digest отсутствуют в logs/metrics/traces/DLQ.
- Exact raw link для retry находится только в short-lived delivery field accepted
  dedupe record до successful send или token expiry; затем field удаляется, а
  non-secret accepted marker остаётся до общего seven-day TTL. Secret не попадает в
  logs/metrics/traces/DLQ; отдельного delivery namespace нет.
- Valkey 9.x является minimum. Atomic issuance задаёт field-level absolute expiry
  `HPEXPIREAT` на token deadline, successful delivery делает idempotent `HDEL`; marker
  продолжает key-level seven-day TTL без janitor или отдельного expiry key.
- Не доставленный до token/delivery-field expiry fresh-auth link завершается terminally
  без stale send, re-issuance, изменения accepted marker или отдельного user
  notification. Existing event replay остаётся duplicate; новый credential возможен
  только после нового explicit «Подтвердить вход» provider event. Failure виден только
  в общих Bot API delivery metrics/alert без raw token.
- Для post-issuance delivery действует максимум пять total attempts с exponential
  backoff/jitter. Retry допускается только для timeout/network, `429` и `5xx`, причём
  `Retry-After` и следующая attempt должны помещаться в исходный token deadline.
  Остальные `4xx`, exhausted budget или невозможность уложить retry terminal сразу с
  idempotent `HDEL`, сохранением accepted marker и без DLQ/late replay. Это исключение
  из следующего общего правила; pre-issuance `internal_error` использует общий flow.
- Transient failure: пять попыток с exponential backoff, затем dead-letter stream.
  Replay DLQ выполняется отдельной admin/CLI operation.
- Недоступность PostgreSQL во время account-specific bot operation считается
  transient failure: event проходит тот же retry/DLQ и не выполняет domain changes;
  Valkey credential или cached active status сами по себе не авторизуют operation.
- Успешно обработанные raw webhook payloads сохраняются 24 часа; DLQ — 30 дней.
- Identity commands и web-session/link tokens обрабатываются только в private chat.
  В группе или канале bot предлагает открыть private chat без token и без раскрытия
  account state.

### 8.1 Исходящие уведомления и web inbox

- Web inbox — источник истины для moderation и security notifications. Пользователь
  выбирает один primary bot provider; обычное событие не дублируется во все providers.
- Первая identity становится primary при создании account. Linking второго provider
  не меняет primary автоматически; после linking UI предлагает отдельное явное
  действие для смены primary.
- Если пользователь отвязывает текущую primary identity, оставшаяся identity другого
  provider атомарно становится primary в той же transaction. Отдельный выбор и
  дополнительный fresh auth не требуются: unlink уже является sensitive operation.
- Domain operation сначала транзакционно сохраняет решение и PostgreSQL outbox event.
  Dispatcher публикует событие в отдельный Valkey Stream; ошибка bot delivery не
  откатывает domain transaction.
- Delivered outbox events хранятся 30 дней; undelivered/DLQ — 90 дней. Domain audit
  живёт независимо и бессрочно.
- Outbox UUID используется как idempotency key, если provider это поддерживает.
  Provider message ID сохраняется. При API без idempotency допустим редкий повтор:
  сообщения informational и не выполняют mutation без отдельного one-time token.
- Bot moderation buttons ведут deep-link на web UI. Approve, `changes_requested` и
  moderator removal не выполняются прямо из bot.
- Web inbox notification хранится один год. Пользователь может поставить `read_at`
  или `archived_at`, но не удалить запись до окончания retention.
- Security notifications и moderation events, требующие действия owner
  (`changes_requested`), обязательны и всегда попадают в web inbox и primary bot. В
  settings они показываются locked-on и не могут быть отключены.
  Catalog, informational и остальные moderation results настраиваются пользователем.
- `changes_requested` notification содержит локализованный actionable label
  correction `reason_code` и optional owner-visible `reason_note`. Code/note берутся
  из сохранённого moderation decision, не из free-form delivery payload; note не
  пишется в application logs.
- Moderator removal/hide является locked-on moderation event для active account,
  включая containment во время full track lock: removal transaction атомарно
  сохраняет обычный moderation audit, web-inbox record и primary-bot outbox intent.
  Notification показывает только факт moderator removal, public-safe moderation
  reason label из закрытого removal reason catalog, short track ID,
  `TRAILBASE_SUPPORT_URL` и 90-дневный purge/appeal deadline; full lock,
  `track_issues`, snapshot и internal data не упоминаются.
  `technical_containment` локализуется нейтрально как «Техническая недоступность».
  Audit-only `reason_note` в notification не попадает. Detection/recheck issues
  по-прежнему notification не создают. Для deactivated account действует общая
  suppression policy moderation notifications без backlog/replay.
- Каждый первый committed `uphold_removal` или `restore_after_appeal` также является
  locked-on moderation notification для active owner: outcome transaction атомарно
  сохраняет web-inbox record и primary-bot outbox intent вместе с outcome/audit.
  Uphold показывает только факт подтверждения removal, short track ID и purge deadline
  без reopen/second-appeal action. Restore published branch показывает canonical track
  link; private/editable branch — action edit/resubmit. Admin reason/note, actor, audit
  ID и internal context не раскрываются. Blocked/failed request и idempotent retry не
  создают notification. Для deactivated account действует общая suppression policy
  moderation notifications без backlog/replay.
- Для deactivated account действует явное исключение: материализуются и доставляются
  только locked-on security/account-lifecycle notifications, включая confirmation
  деактивации и административную реактивацию, в web inbox и primary linked messenger.
  Все moderation notifications, включая обычно locked-on `changes_requested`, а
  также catalog/informational categories подавляются без backlog/replay после
  реактивации. Domain events и audit продолжают записываться.
- Deactivation transaction отменяет все ещё `pending` non-security notification
  outbox delivery intents и помечает связанные unread web-inbox records как
  suppressed. Они сохраняются по обычному retention, но не показываются после
  реактивации. Delivery, атомарно claimed в `sending` до deactivation, может
  завершиться и не отзывается. Security/account-lifecycle records и deliveries не
  отменяются.
- При создании account optional moderation results, включая `published`, включены по
  умолчанию; catalog и informational выключены. Locked-on categories не зависят от
  этих defaults.
- Optional preference является account-level и одновременно управляет web inbox и
  primary bot: выключенная category не создаёт web inbox notification и не
  планирует bot delivery. Отдельных per-channel toggles в MVP нет. Domain state,
  domain event и audit сохраняются независимо от notification preference; locked-on
  categories всегда материализуются и доставляются для active account, кроме явно
  описанной выше deactivated-account policy.
- Preference читается один раз внутри domain transaction, создающей notification и
  outbox delivery intent. Последующее изменение settings не отменяет и не добавляет
  уже принятое уведомление; новое значение действует только на будущие события.
  Dispatcher не перечитывает текущую preference, поэтому delivery deterministic и
  не зависит от race со settings. Lifecycle suppression при deactivation является
  отдельным явным исключением, а не изменением preference.
- Security events: новая browser session, identity link/unlink, role и account changes.
- Detection, repeat detection, resolution и recurrence `track_issues` не создают
  user notification, web-inbox record или messenger delivery. Из issue-related
  данных owner видит только текущее derived `has_problems` и generic warning
  непосредственно в track projection web/private Telegram/Max; issue codes, scopes
  и admin detail не раскрываются.
- Reactivation lifecycle notification содержит только факт и timestamp реактивации,
  `TRAILBASE_SUPPORT_URL` и инструкцию снова открыть bot. Admin identity, internal
  reason, audit ID и web-auth/link tokens пользователю не раскрываются; обязательная
  admin reason остаётся только в audit.
- Смена primary bot provider всегда создаёт запись в web inbox и best-effort
  уведомляется через прежний и новый primary providers. Ошибка любой bot delivery не
  откатывает подтверждённую смену.
- Claim новой ordinary-login session создаёт уведомление со временем, provider и
  сокращённым описанием устройства, но без IP; provisional commit его не создаёт.
  Sliding TTL refresh существующей session и same-browser re-auth claim при valid
  current session того же user уведомление не создают. Уведомление содержит deep-link
  только в authenticated UI управления active sessions; ссылка сама не выполняет
  revoke и не содержит action token.
- При unlink identity событие отправляется всем оставшимся providers и best-effort
  отвязываемому provider; web inbox сохраняется всегда. Transaction записывает в
  outbox зашифрованный snapshot delivery target для detached identity до её физического
  удаления. Snapshot не попадает в audit, web inbox, логи или долговременный DLQ;
  dispatcher делает максимум пять попыток с exponential backoff и jitter, но не
  дольше 15 минут от commit unlink. Достижение любого предела — terminal failure:
  ciphertext немедленно удаляется, detached delivery больше не replay-ится, а метрика
  не содержит идентификатор адресата.
- Если unlink одновременно меняет primary, создаётся одно `identity_unlinked` event с
  `detached_provider` и `new_primary_provider`, а не отдельное
  `primary_provider_changed`: одна web inbox запись и одно сообщение каждому
  требуемому provider.

