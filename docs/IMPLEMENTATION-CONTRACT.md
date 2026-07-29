# TrailBase — Implementation Contract

Статус: принятые решения
Дата фиксации: 2026-07-29

Этот документ уточняет ADR и roadmap до уровня проверяемых инвариантов. Он предназначен
для инженера, который реализует вертикальные срезы TrailBase. Если краткая формулировка
в старом ADR или roadmap противоречит этому документу, действует этот документ; саму
архитектурную мотивацию по-прежнему следует читать в ADR.

Точка продолжения незавершённого design interview хранится в
[Grill-me Checkpoint](GRILL-CHECKPOINT.md).

## 1. Runtime и границы сервисов

- Dev и первый production используют один Docker Compose contract.
- Production stack: Caddy, `web`, `bot-worker`, `parse-worker`, one-shot `migrate`,
  PostgreSQL/PostGIS, Valkey и MinIO.
- `web`, `bot-worker`, `parse-worker` и `migrate` собираются из одного application
  image, но запускаются разными командами.
- В dev доступен optional profile `webhook-dev` со stable-hostname Cloudflare Tunnel.
  Auth всегда тестируется через HTTPS; insecure session-cookie режима нет.
- Production reverse proxy — Caddy. Приложение и webhooks используют один public
  hostname. Точные webhook paths: `/webhook/telegram` и `/webhook/max`.
- Public URLs строятся только от настроенного `PUBLIC_BASE_URL`. `Host` проверяется
  по allowlist. Forwarded client IP принимается только от явно настроенных trusted
  proxies.
- Java runtime — Java 25 во всех окружениях. Runtime image — pinned Debian-based JRE,
  non-root, read-only root filesystem.
- Временные multipart/GPX-файлы пишутся в ограниченный `tmpfs` mount
  `/tmp/trailbase` с `noexec,nosuid`.
- Ring adapter — `http-kit`. M01/M02 handlers используют синхронный Ring API; долгие
  операции уходят в workers.

## 2. Конфигурация, secrets и lifecycle

- Секреты передаются через Docker secrets: bot tokens, webhook secrets, DB/MinIO
  credentials, backup keys и application crypto keys.
- Несекретная конфигурация передаётся через environment variables.
- Dev secrets находятся в gitignored `secrets/dev/*`; команда `bb init-dev-secrets`
  создаёт их по committed шаблону.
- Конфигурация валидируется Malli-схемой до старта сервера/worker. Ошибка завершает
  процесс с ненулевым кодом и сообщает только имя параметра, не значение.
- `web` при shutdown снимает readiness и завершает текущие запросы в течение 30 секунд.
- Workers перестают брать новые события, завершают текущие до 30 секунд и оставляют
  незавершённые сообщения pending. Другой consumer забирает их через `XAUTOCLAIM`
  после 60 секунд без heartbeat.

## 3. Идентификаторы и время

- Primary keys основных сущностей и справочников — UUIDv7, генерируемые приложением.
- Время хранится как PostgreSQL `timestamptz` в UTC; локальная зона применяется только
  при отображении.
- Clock, UUID generator и secure random являются заменяемыми зависимостями, чтобы тесты
  expiry, retries и audit не использовали `sleep`.
- Внешний ID пользователя хранится в `user_identities.provider_user_id` как `text`.
  Уникальность active identity: `(provider, provider_user_id)`.
- В M02 один account имеет не более одной active identity каждого provider:
  `(user_id, provider)` также уникальна. Поэтому account может одновременно иметь
  максимум одну Telegram и одну Max identity.
- Термин `provider_id` не используется из-за неоднозначности. ID самого бота находится
  в конфигурации; в M02 поддерживается один Telegram-бот и один Max-бот.

## 4. Bot-first authentication

### 4.1 Identity и создание аккаунта

- Telegram и Max — единственные identity и authentication providers. Email, телефон и
  отдельные recovery identities отсутствуют.
- Browser activation не требуется. Валидированная messenger identity достаточна для
  выполнения разрешённых chat operations и использования account без web session.
- Первый валидированный `/start` в private one-to-one chat от ранее неизвестной
  Telegram/Max identity атомарно создаёт active `users` row и первую
  `user_identities` row. Повторная доставка и конкурентные `/start` идемпотентно
  resolve в тот же account через unique provider identity; browser session для
  создания не нужна.
- Account creation, выпуск web-session/link tokens и identity linking разрешены только
  в private one-to-one chat с ботом. Group/channel events не создают account, не
  изменяют identity state и не раскрывают account state; bot отвечает статической
  ссылкой или инструкцией продолжить в private chat без auth token.
- Дополнительная identity связывается с account только явно из уже
  аутентифицированного account context. Нельзя автоматически объединять accounts по
  имени, username или другим profile fields.
- Если identity уже принадлежит другому аккаунту, link отклоняется. Account merge
  не входит в M02.
- Active account всегда сохраняет хотя бы одну active Telegram/Max identity; отвязать
  последнюю identity нельзя.
- При unlink active identity физически удаляется. Audit хранит внутренний identity UUID
  и provider, но не `provider_user_id`.
- Основное публичное имя — редактируемое `users.display_name`. Оно инициализируется
  именем первой identity и затем не обновляется автоматически.
- Provider display fields обновляются на `/start` и хранятся только как snapshot:
  `display_name`, `username`, `language_code`, ограниченный `profile_meta`.
- Provider avatar URL не сохраняется. Допустим только безопасный `provider_avatar_ref`;
  загрузка выполняется сервером.
- Публично отображаются только TrailBase display name и будущий TrailBase avatar.
  Provider usernames и identities не публикуются.
- Язык при первом взаимодействии выбирается в порядке: сохранённая настройка, bot profile,
  `Accept-Language`, русский. Ручной выбор больше не перезаписывается автоматически.

### 4.2 Опциональная browser session и identity linking

- Web-session и identity-link tokens живут в Valkey 10 минут и расходуются атомарным
  `GETDEL`.
- Для каждой provider identity действительна только последняя web-session ссылка.
- `web_session` и `link_identity` — разные token namespaces/purposes. Link token
  содержит `target_user_id` и ожидаемый candidate provider, но не candidate
  `provider_user_id`.
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
- `GET /auth?token=...` для web-session token не расходует token: preview/scanner не
  должен сжечь вход.
- GET создаёт короткую auth-flow запись в Valkey и выдаёт `HttpOnly`, `Secure`,
  `SameSite=Lax` flow-cookie плюс одноразовый form nonce.
- `POST /auth` проверяет cookie/nonce, расходует token, создаёт только browser session
  и делает redirect на чистый URL. Он не активирует account и не является обязательным
  для chat operations.
- Confirmation page показывает provider и экранированный display name, но не внешний
  provider ID.
- `return_to` допускает только внутренний путь, начинающийся с одного `/`; absolute URL
  и `//host` отклоняются. Нормализованный путь хранится server-side в auth-flow записи.
- Auth pages используют `Cache-Control: no-store`, `Referrer-Policy: no-referrer`, не
  подключают сторонние ресурсы. Значение query parameter `token` удаляется из access и
  application logs.
- Web-session/link tokens имеют 128 бит криптографической энтропии: 16 random bytes,
  Base64URL без padding.

### 4.3 Sessions в Valkey

- Sessions хранятся в отдельном Valkey service с persistent volume и AOF
  `appendfsync everysec`.
- Sliding TTL — один год и продлевается на каждом авторизованном запросе. Отдельного
  absolute lifetime нет.
- Cookie token имеет 128 бит CSPRNG-энтропии. Raw token находится только в cookie;
  Valkey key строится из `SHA-256(token)`.
- Cookie: `__Host-trailbase_session`, `Secure`, `HttpOnly`, `SameSite=Lax`, `Path=/`,
  без `Domain`.
- Session хранит `user_id`, `created_at`, `last_seen_at`, сокращённый `User-Agent` и
  CSRF state. IP в session/profile не сохраняется.
- Session не содержит roles/permissions. Они читаются из PostgreSQL на каждом
  авторизованном запросе, поэтому отзыв полномочий применяется сразу.
- Разрешены независимые сессии на нескольких устройствах. M02 включает страницу
  активных сессий, отзыв одной сессии, обычный logout текущей и logout-all.
- Новое browser authentication в уже авторизованном браузере выдаёт новый session ID
  и отзывает прежнюю
  сессию этого браузера. Сессии других устройств не затрагиваются.
- Потеря Valkey завершает все сессии. Valkey не восстанавливается из backup, чтобы не
  воскресить отозванные sessions/tokens.

### 4.4 Sensitive operations и account lifecycle

- Fresh bot authentication не старше 10 минут требуется перед unlink identity, сменой
  primary bot provider, logout-all, назначением ролей и деактивацией.
- Просмотр сессий и отзыв одной конкретной сессии доступны по обычной active session.
- При потере доступа ко всем linked messenger identities self-service recovery
  отсутствует. Существующие browser sessions продолжают обычную работу, но linking и
  sensitive operations без fresh bot auth недоступны; после утраты sessions account
  недоступен.
- Пользователь может сам деактивировать аккаунт после fresh auth и явного
  подтверждения. Все сессии отзываются, новые входы блокируются, содержимое сохраняется.
- Реактивация в MVP доступна только администратору, требует fresh auth и audit.
- Вход disabled identity не создаёт новый аккаунт и показывает нейтральную инструкцию
  обратиться к администратору.

## 5. Roles, permissions и audit

- Глобальные роли: `user`, `moderator`, `admin`; связь many-to-many через `user_roles`.
- HTTP/domain operations проверяют permissions, а не имена ролей. Mapping
  `role -> permissions` централизован в коде.
- Первый admin назначается явной командой
  `bb grant-role <provider> <provider-user-id> admin`, а не переменной окружения и не
  правилом «первый вошедший».
- Нельзя транзакционно удалить роль или деактивировать последнего active admin.
- Admin audit — append-only: actor, action, target type/ID, metadata, request ID,
  timestamp. Для bootstrap actor равен `system`.
- Audit в MVP не удаляется автоматически.
- Деактивация отзывает все сессии. Изменение roles не отзывает сессии, поскольку
  permissions читаются из PostgreSQL на каждом запросе.

## 6. Messenger-only identity boundary

- Telegram и Max — полный и закрытый список identity providers.
- Email, phone, passwords, recovery contacts, OTP и отдельный account-recovery flow
  отсутствуют в schema, configuration, UI и roadmap.
- Доступ через chat не зависит от наличия browser session. Web остаётся first-class
  интерфейсом, но его активация не является условием существования или использования
  account.
- Потеря доступа ко всем linked messenger identities означает потерю self-service
  доступа к account; новый recovery mechanism требует отдельного будущего решения.

## 7. HTTP security

- Все state-changing browser endpoints с cookie требуют session-bound CSRF token.
  Hiccup forms используют hidden field, htmx — header.
- Webhooks и machine endpoints без cookie освобождаются от CSRF только при собственной
  аутентификации.
- Browser API same-origin; CORS не включается до появления отдельного клиента.
- CSP включена с M01: CSP-compatible Alpine build, local JS/CSS, без inline scripts и
  `unsafe-eval`; MapLibre получает точечные `worker-src blob:` и tile-provider origins.
- Production отправляет HSTS `max-age=31536000; includeSubDomains`, без `preload`.
  Dev HSTS не отправляет.
- Rate limits через Valkey:
  - `/auth`: 10 попыток/мин на IP;
  - bot events: 30/мин на provider identity;
  - admin mutations: 30/мин на user;
  - public search: 120/мин на IP, burst 30.
- При превышении возвращаются `429` и `Retry-After`. IP используется только как
  краткоживущий limiter key.

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
- Transient failure: пять попыток с exponential backoff, затем dead-letter stream.
  Replay DLQ выполняется отдельной admin/CLI operation.
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
- Bot moderation buttons ведут deep-link на web UI. Approve/reject не выполняются
  прямо из bot.
- Web inbox notification хранится один год. Пользователь может поставить `read_at`
  или `archived_at`, но не удалить запись до окончания retention.
- Moderation/catalog/informational categories настраиваются пользователем.
  Security notifications обязательны и всегда попадают в web inbox и primary bot.
- Security events: новая browser session, identity link/unlink, role и account changes.
- Смена primary bot provider всегда создаёт запись в web inbox и best-effort
  уведомляется через прежний и новый primary providers. Ошибка любой bot delivery не
  откатывает подтверждённую смену.
- Новая session создаёт уведомление со временем, provider и сокращённым описанием
  устройства, но без IP. Sliding TTL refresh существующей session уведомление не
  создаёт. Уведомление содержит deep-link только в authenticated UI управления
  active sessions; ссылка сама не выполняет revoke и не содержит action token.
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

## 9. Health, logs и metrics

- `/health/live` проверяет только процесс и публично отдаёт минимум данных.
- `/health/ready` проверяет PostgreSQL, Valkey и соответствующий worker consumer group;
  он доступен только внутри Compose network.
- `/metrics` Prometheus-compatible и внутренний. Минимальный набор: HTTP/DB/Valkey
  latency, statuses, stream lag/pending, retries, DLQ size, auth success/failure,
  rate-limit rejects.
- Все logs — structured JSON в stdout/stderr через Telemere. Контейнеры не ведут
  собственные log files.
- Сквозной correlation/request ID — UUIDv7. Он передаётся через webhook, stream,
  worker, outbound Bot API, logs и audit.
- OpenTelemetry на M01/M02 не добавляется; correlation IDs, metrics и logs достаточны.

## 10. GPX intake и validation

- Web и bot uploads используют один pipeline. Bot `/upload` доступен только в private
  one-to-one chat для identity, прошедшей provider webhook validation; browser session
  для upload не требуется. Group/channel upload не создаёт job или draft и предлагает
  продолжить в private chat.
- Полный upload flow может завершаться в Telegram/Max chat: приём файла, validation,
  выбор обязательных metadata, preview статуса parse и отправка revision на moderation.
  Web deep-link остаётся дополнительным, а не обязательным шагом.
- Лимит raw GPX — 10 MiB; лимит всего multipart request — 12 MiB. Provider-reported
  size/`Content-Length` проверяется, но download дополнительно имеет streaming hard cap.
- Принимается только несжатый GPX/XML 1.0 или 1.1. `.gz` и ZIP в MVP запрещены.
- Проверяется структура, а не строгая XSD. Неизвестные extensions безопасно
  игнорируются.
- DTD, XInclude, external entities и parser network access запрещены.
- Максимум 250 000 GPX points и XML depth 64.
- `NaN`, infinity и координаты вне диапазона отклоняют файл. Сегмент короче двух точек
  игнорируется с warning; отсутствие валидных сегментов завершает job ошибкой.
- Каждый `<trkseg>` становится отдельным сегментом 2D PostGIS `MultiLineString`.
  Разрывы не соединяются искусственной линией.
- Route-only GPX поддерживается: каждый `<rte>` становится сегментом. Если есть и
  `<trk>`, и `<rte>`, используются tracks, а routes показываются warning.
- Waypoints показываются в preview и могут стать POI suggestions, но не создают
  locations автоматически.
- GPX `<name>` и `<desc>` используются только как defaults формы. Name общий,
  description — plain-text map по языкам.
- Limits: name и provider display name — 200 Unicode code points, description —
  10 000 на язык, provider username — 100. Импортированное превышение обрезается с
  warning; пользовательский ввод сверх лимита отклоняется.

## 11. Geometry, elevation и duration

- Canonical spatial geometry всегда 2D `MultiLineString` SRID 4326.
- Canonical length — `ST_Length(geometry::geography)`. Parser length используется
  только для preview.
- Simplified geometries вычисляются в EPSG:3857 через
  `ST_SimplifyPreserveTopology` и возвращаются в 4326:
  - `geometry_simplified_z11`: 40 м;
  - `geometry_simplified_z13`: 10 м;
  - `geometry_simplified_z15`: 2 м.
- Elevation samples не помещаются полностью в PostgreSQL. Revision содержит
  LTTB-профиль максимум 2 000 samples: distance, elevation, lat/lon, segment index.
- При покрытии elevation менее 90% точек профиль и gain/loss считаются недоступными;
  отсутствующие значения не превращаются в нули.
- Gain/loss: ресемплирование по дистанции 10 м, median filter 5 samples, порог
  изменения 3 м. Revision хранит версию алгоритма.
- Сохраняются `elapsed_duration_s` и `moving_duration_s`. Search/display использует
  отдельно подтверждённый `duration_s`.
- Moving time включает интервалы со скоростью не ниже 0,5 км/ч. Интервалы с
  неположительным временем, gap больше 10 минут или физически невозможной скоростью
  исключаются с warning. Алгоритм версионируется.
- `duration_source`: `gpx_moving`, `gpx_elapsed`, `manual`, `estimated`, `unknown`.

## 12. Object storage и upload jobs

- Первый production использует MinIO в Compose, но приложение зависит только от
  configurable S3 endpoint/bucket/credentials.
- Все buckets private. Object keys — opaque UUID с техническим prefix (`raw/`,
  `exports/`, `photos/`), без user/track IDs и исходных filenames.
- Private raw GPX шифруется приложением до S3: случайный data key на object,
  AES-256-GCM ciphertext, data key зашифрован versioned master key.
- Cross-user raw dedup отсутствует. Каждый revision имеет собственный encrypted object.
  Integrity/retry использует private content HMAC, не публичный raw SHA-256.
- S3 и PostgreSQL согласуются через `upload_jobs`: pending DB row, object write,
  validation/parse, transactional revision creation, terminal status.
- Incomplete jobs и orphan objects старше 24 часов удаляет janitor.
- HTTP upload только принимает/шифрует object и ставит parse job. Web UI опрашивает
  status htmx-запросом каждые две секунды до terminal state. Для bot upload terminal
  status публикуется обратно в исходный chat, после чего bot продолжает metadata и
  moderation flow.
- Transient S3/Valkey/PostgreSQL failures повторяются до трёх раз. Invalid GPX,
  отсутствующая geometry и превышение limits завершаются без retry.
- После успешного parse создаётся постоянный private track draft. Он не истекает и
  доступен только owner и moderators; unlisted/secret-link доступа нет.
- User quota: 100 private drafts и 1 GiB raw storage, с admin override.
- Максимум три active upload/parse jobs на пользователя.
- Новые uploads получают `503 Retry-After`, если parse queue больше 1 000 jobs или
  старейшая job ждёт больше 10 минут. Catalog и auth остаются ready.
- `parse-worker` использует отдельный stream и стартовую concurrency 2 на container.
  `bot-worker` использует другой stream, чтобы uploads не задерживали auth.

## 13. Track aggregate и revisions

- `tracks` хранит stable identity: ID, неизменяемый `owner_id`, lifecycle timestamps,
  status и `current_revision_id`.
- Geometry, S3 references, name/descriptions, activity, difficulty, season, duration
  и tags находятся в immutable full snapshot `track_revisions`.
- Private draft доступен только owner/moderators. Отправка создаёт `pending_review`.
- Первая публикация требует moderation. `changes_requested` обязательно содержит
  причину и допускает редактирование/повторную отправку без повторной загрузки GPX.
- Изменение опубликованного трека создаёт pending full revision. Публичная revision
  остаётся прежней до атомарного approval.
- Ownership transfer не входит в MVP. Будущий transfer требует audited operation и
  подтверждения обоих пользователей.
- Авторское «удаление» переводит track в `archived`; восстановление доступно 30 дней.
  Затем worker удаляет raw/export/photos, revisions и derived data, оставляя tombstone
  track UUID и audit без пользовательского содержимого.
- Moderator removal скрывает публикацию сразу, требует причину, не восстанавливается
  автором и хранит данные 90 дней для апелляции до такой же очистки.
- Автор может явно обрезать начало/конец маршрута с preview. Автоматической privacy
  обрезки нет. Опубликованная geometry соответствует одобренной trimmed revision.
- Повторный exact GPX того же пользователя вызывает warning, но может быть сохранён
  явно. Cross-user duplicate hints видит только moderator.
- Duplicate detection: hash canonicalized 2D geometry на grid `1e-6` градуса,
  direction-insensitive. Неидентичные кандидаты ищутся по bbox/length и
  `ST_HausdorffDistance`. Совпадение — только moderation hint.

## 14. Public export и лицензии

- Original raw GPX никогда не публикуется.
- Sanitized GPX генерируется для approved revision и хранится отдельно. Он содержит
  только lat/lon/elevation, одобренные name/description и license metadata.
- Timestamps, heart rate, cadence, device extensions и original waypoints удаляются.
- Export metadata содержит TrailBase author display name, canonical track URL,
  CC BY 4.0, ссылку на лицензию и ID опубликованной revision. Provider identity и
  internal user ID отсутствуют.
- Автор явно принимает CC BY 4.0 при отправке track на moderation.
- Каталог как база данных публикуется под ODbL 1.0. Перед первым bulk export требуется
  юридическая проверка текста attribution.
- Download endpoint проверяет publication status и отвечает `302` на presigned URL с
  TTL 5 минут.
- Filename — ASCII slug track name плюс короткий фрагмент track UUID; исходный upload
  filename не используется.

## 15. Classification и tags

- Primary activity enum остаётся:
  `hike`, `bike`, `run`, `ski`, `water`, `horse`, `motor`, `other`.
- Classifier показывает до трёх ranked suggestions с confidence и объяснением
  признаков. Пользователь обязан явно выбрать activity перед moderation.
- Difficulty хранится как `(difficulty_scale, difficulty_code)`; rank получается из
  централизованного справочника внутри соответствующей шкалы. Значение `unknown`
  разрешено.
- Season использует только четыре bits: spring, summer, autumn, winter.
  `year-round` означает все четыре; mask `0` означает unknown.
- Несколько выбранных сезонов объединяются по `ANY`; разные фасеты — по `AND`.
- Duration может быть unknown.
- Tag set относится к `track_revision_id`, а не к stable track, и входит в
  модерируемый snapshot.

## 16. POI/gazetteer

- `locations` хранит stable identity и `current_revision_id`.
- Geometry, names, category и OSM snapshot находятся в immutable
  `location_revisions`. OSM refresh создаёт pending revision и не перезаписывает
  ручные правки.
- Geometry kind (`point`, `line`, `area`) отделён от semantic `category_id`.
  Categories образуют модерируемый иерархический справочник.
- Основные названия хранятся в `name_i18n`; aliases — отдельно с language, source и
  normalized form. OSM names/alt names импортируются с provenance, moderator выбирает
  main display name.
- OSM import — async job с configurable Overpass endpoint, rate limit, retries и
  response cache. Moderator видит preview/diff и отдельно применяет revision.
- Provenance: `osm_type`, `osm_id`, `osm_version`, `osm_timestamp`, `fetched_at`,
  source URL и hash нормализованного ответа. `(osm_type, osm_id)` уникальна среди
  active locations.
- POI links — отдельно модерируемые annotations конкретной geometry revision, а не
  причина создавать новую track revision.
- Link хранит source (`autodetect`, `author_suggestion`, `moderator`), status
  (`suggested`, `approved`, `rejected`), confidence, reviewer и timestamps.
- Одна logical location link может иметь несколько occurrences. Каждая occurrence
  хранит `seq`, cumulative `distance_along_m`, `segment_index`,
  `segment_distance_m`. Разрыв между segments не добавляет расстояние.
- High-confidence связь с уже approved location публикуется автоматически:
  - point: расстояние до track не больше 25 м;
  - line: track находится в пределах 50 м не менее 100 м;
  - area: длина track внутри polygon не меньше 50 м.
- Остальные candidates в общих радиусах остаются `suggested`.
- Окончательно rejected autodetect link создаёт suppression для
  `(track_revision_id, location_id)`. Повторный запуск или новая версия алгоритма не
  добавляет связь снова; пересмотр выполняет moderator. Новая geometry revision трека
  оценивается независимо.
- Первый author dispute переводит auto-approved link в `disputed` и немедленно
  скрывает его до moderator decision. Если moderator проверил и восстановил link,
  следующий author flag создаёт обращение, но не скрывает verified link автоматически.
- Dispute требует reason code: `too_far`, `route_does_not_pass`, `wrong_location`,
  `duplicate`, `other`. Комментарий optional, кроме `other`; moderator decision всегда
  содержит причину.
- Новая location geometry revision запускает async revalidation всех связанных
  annotations:
  - autodetected links пересчитываются;
  - author/moderator links получают `needs_review`, если новая geometry им
    противоречит;
  - до результата публичной остаётся последняя approved связь со внутренним
    `revalidating`;
  - результат переключается атомарно.
- Revalidation использует пять retries с backoff, затем DLQ. Прежняя approved связь
  остаётся публичной с внутренним `revalidation_failed`; состояние дольше 24 часов
  вызывает operator/moderator alert.
- Duplicate locations объединяются moderator merge в выбранную canonical location:
  aliases/provenance переносятся, annotations перепривязываются с дедупликацией
  occurrences, старый UUID становится tombstone с permanent redirect, операция
  audit-ится.
- Location merge обратим 90 дней по immutable merge snapshot и mapping перенесённых
  annotations. После срока redirect постоянный, snapshot может быть удалён.
- Ошибочная связанная location переводится в `archived`: скрывается из карты, поиска
  и public links, URL остаётся tombstone, annotations идут на review. Несвязанная
  archived location физически очищается через 90 дней, сохраняя UUID tombstone,
  redirect/merge history и audit.
- Используемая semantic category не удаляется: она получает `deprecated` и
  `replacement_category_id`. Category key неизменяем.
- Category hierarchy — строгое дерево с одним `parent_id`. Родительский search filter
  включает descendants; JSON API поддерживает `category_exact`.
- Обычный пользователь предлагает seed point и optional OSM URL. Line/area geometry
  импортирует или создаёт moderator.
- Suggestion limits: 10 новых заявок в сутки и 20 одновременно pending на user,
  с admin override.
- Право `submit_location_suggestions` можно ограничить отдельно от аккаунта. Restriction
  audit-ится, содержит причину и optional expiry.
- Low-zoom clustering использует глобальную deterministic hex-grid в EPSG:3857.
  Client-side clustering отключён. Для line/area representative point —
  `ST_PointOnSurface`.

## 17. Map delivery и browser state

- Production tile provider выбирается отдельно; URL и attribution задаются
  конфигурацией. `tile.openstreetmap.org` допустим только для dev/test малой нагрузки.
- MapLibre attribution control всегда видим и содержит OSM contributors и выбранного
  provider. Auth page не загружает карту или tiles.
- Zoom matrix:
  - z0–12: только server-side POI clusters;
  - z13: `geometry_simplified_z11`;
  - z14: `geometry_simplified_z13`;
  - z15: `geometry_simplified_z15`;
  - z16+: z15 context layer плюс full geometry overlay выбранного track.
- Cluster feature содержит count и bbox. Click делает `fitBounds` с `maxZoom: 14`;
  одиночный POI открывает detail.
- `/api/tracks.geojson` возвращает максимум 1 000 features, `truncated: true` при
  переполнении и предлагает приблизить карту.
- Selection при переполнении: completeness, `published_at DESC`, UUID. Completeness —
  по одному баллу за description, difficulty, season, duration и approved POI; это
  не публичный quality rating.
- GeoJSON coordinates имеют шесть знаков после запятой. Properties минимальны:
  track ID, name, activity, difficulty code, length, published revision ID/time.
- GeoJSON использует `ETag` и
  `Cache-Control: public, max-age=30, stale-while-revalidate=60`.
- Viewport bbox расширяется наружу до фиксированной zoom-grid для повторного
  использования cache.
- Client делает debounce 150 мс, отменяет старый fetch через `AbortController` и
  отбрасывает stale responses по sequence number.
- URL state: `map=z/lat/lon`, отдельные filter params, канонический detail path
  `/tracks/:id`. Filter/map updates используют `replaceState`, явный detail —
  history entry.
- Каждая map entity имеет keyboard-accessible эквивалент в semantic sidebar.
- Без WebGL карта скрывается, но sidebar, search, detail и upload metadata остаются
  работоспособными.
- Public track detail полностью SSR: name, description, author, metrics, POI, license
  и download доступны без JavaScript.

## 18. Search contract

- Search остаётся в PostgreSQL: tsvector/GIN, `pg_trgm`, PostGIS/GIST, btree facets и
  joins по approved revision annotations.
- `/search` возвращает Hiccup/HTML для htmx; `/api/v1/search` — стабильный JSON. Оба
  используют одну domain search service.
- Telegram/Max `/search` использует ту же domain search service и выполняется без
  browser session. Bot адаптирует filters и pagination к chat controls; search semantics
  и permission checks не расходятся с web/API.
- В private chat search выполняется с permissions связанного account. В group/channel
  chat `/search` разрешён только как account-stateless read-only поиск по public
  published catalog: он не создаёт account, не возвращает private/unlisted tracks или
  персональные данные и не сохраняет user history/settings.
- Filter/pagination controls у group-search результата принимает только requester,
  инициировавший конкретный `/search`. Callback cryptographically связывается с
  provider, chat, message, requester identity и search cursor/query state. Нажатие
  другим участником не меняет общий результат и предлагает запустить собственный
  `/search`.
- Если channel context не предоставляет стабильную requester identity, `/search`
  публикует только статическую первую страницу public results без filter/pagination
  controls. Для продолжения bot добавляет обычную ссылку в private chat без auth token
  и account state.
- Interactive search controls в private/group chats имеют абсолютный TTL 15 минут от
  создания результата без sliding refresh. После expiry callback не изменяет старое
  сообщение и предлагает повторить `/search`.
- Chat-search callback содержит только случайный 128-bit opaque ID. Valkey record хранит
  provider/chat/message/requester binding, query, filters, cursor и исходный absolute
  expiry; query и identity не попадают в provider callback payload. Потеря Valkey или
  записи инвалидирует controls без восстановления и предлагает повторить `/search`.
- Успешный filter/page callback атомарно проверяет binding/expiry, расходует текущий
  Valkey record и создаёт новый opaque ID с обновлённым search state, сохраняя исходный
  absolute `expires_at`. Только callback, выигравший ротацию, может редактировать
  result message; повторный или конкурентный callback со старым ID считается stale и
  message не меняет.
- Если rotation завершилась, а provider edit вернул transient error или ambiguous
  outcome, retry использует тот же новый ID и state. После исчерпания bounded retry
  новый Valkey record удаляется, старый не восстанавливается, а result message
  считается неинтерактивным и предлагает повторить `/search`.
- Search-result edit делает максимум пять total attempts с exponential backoff и
  jitter. Retry разрешён только для timeout/network errors, `429` и `5xx`; `Retry-After`
  соблюдается, только если следующая попытка укладывается в исходный `expires_at`.
  Остальные `4xx` terminal сразу. После terminal failure ephemeral edit не попадает в
  DLQ или поздний replay.
- Provider-specific callback acknowledgement отправляется сразу после проверки
  requester binding/expiry и успешной atomic rotation, до search query и result edit.
  Он подтверждает только приём нажатия; дальнейшую работу выполняет `bot-worker`.
  Webhook ingress `2xx` остаётся отдельным подтверждением доставки event.
- Stale, foreign или expired callback не ротирует state и не меняет result message;
  provider acknowledgement содержит короткое нейтральное объяснение.
- Если после ACK/rotation search query завершается timeout или transient error, новый
  callback state удаляется, а старый ID не восстанавливается. Bot сохраняет прежний
  result content, но terminal edit убирает controls и добавляет нейтральную инструкцию
  повторить `/search`; edit использует тот же provider retry policy.
- Instant search запускается с трёх символов. Явный submit длиной 1–2 символа выполняет
  только exact/prefix search без trigram.
- Exact/prefix tsvector results ранжируются выше fuzzy. `pg_trgm` — fallback, если
  точных результатов мало или score низкий.
- Все facet counts считаются server-side по полной выборке. Используется disjunctive
  faceting: каждый facet применяет все filters, кроме собственного.
- Pagination — HMAC-защищённый opaque keyset cursor, связанный с query hash.
  Default page size 20, maximum 100.
- При `q` default sort: relevance, completeness, publication time, UUID.
  Без `q`: newest, UUID. Дополнительные sorts: newest, length asc/desc,
  duration asc/desc. Popularity в MVP отсутствует.
- Поиск может быть привязан к map viewport. Режим «Искать в этой области» включён по
  умолчанию, отключаем пользователем и сохраняется в URL.
- Search SQL получает `statement_timeout = 800ms`, целевой p95 ниже 200 мс.
  Timeout возвращает `503` и retry affordance.
- Запрос медленнее 200 мс логирует fingerprint, facets, bbox size, row counts и
  duration без query text/user ID. Sampled `EXPLAIN (ANALYZE, BUFFERS)` выполняется
  только в test environment.

## 19. Static assets и frontend build

- htmx, CSP-compatible Alpine, MapLibre bridge и styles поставляются локально.
- Frontend assets собираются `esbuild` через npm; committed `package-lock.json`,
  только `npm ci`.
- Node используется только в multi-stage build и отсутствует в runtime image.
- Content-hashed assets получают
  `Cache-Control: public, max-age=31536000, immutable`; HTML — `no-cache`.
- Assets отдаёт приложение из classpath. Caddy не требует shared static volume.

## 20. Миграции, CI, deploy и backup

- Migrations применяет one-shot `migrate` service до `web` и workers.
- Автоматического schema rollback нет. Down migrations тестируются, но запускаются
  вручную.
- Schema evolution следует expand/contract: additive migration, code rollout,
  отдельный backfill, удаление legacy в следующем release.
- PostgreSQL/PostGIS, Valkey, Caddy, MinIO и build/runtime images не используют
  `latest`. Production pin-ит version и digest.
- Test commands:
  - `bb test`: unit/contract без containers;
  - `bb test-integration`: real PostgreSQL/Valkey и local Telegram/Max HTTP stubs;
  - `bb test-all`: обязательный pre-merge gate;
  - `bb ci`: локальный эквивалент CI pipeline.
- Provider fixtures обезличены и включают positive, malformed и unknown event cases.
- GitHub Actions запускает migrations up/down, tests и production image build.
- Image публикуется в GHCR из protected `main` под immutable commit SHA.
- Production deploy запускается вручную через protected GitHub Environment с approval
  и явным SHA.
- Deploy использует SSH под ограниченным `trailbase-deploy`; production не содержит
  self-hosted GitHub runner и не собирает source.
- Хранятся current и previous successful image SHA. Application rollback переключает
  image назад без schema rollback.
- Post-deploy gate: service readiness, live/ready endpoints, migration version,
  worker heartbeat и public-page smoke. Провал автоматически возвращает previous
  application image и сохраняет diagnostics.
- Valkey не резервируется. PostgreSQL: daily full backup и continuous WAL archive.
  MinIO: versioning плюс off-host backup/replication.
- Initial targets: RPO до 15 минут, RTO до 4 часов.
- Ежемесячно выполняется automated restore drill в изолированном окружении с проверкой
  migrations, ключевых row counts и выборочных S3 objects. Backup шифруется отдельным
  ключом.

## 21. Нормативные внешние ссылки

- [Creative Commons Attribution 4.0](https://creativecommons.org/licenses/by/4.0/)
- [Open Data Commons ODbL 1.0](https://opendatacommons.org/licenses/odbl/1-0/)
- [OpenStreetMap Foundation Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/)
