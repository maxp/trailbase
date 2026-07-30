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
- `TRAILBASE_SUPPORT_URL` обязателен в production и должен быть absolute HTTPS URL без
  userinfo или fragment. Он используется в нейтральной инструкции для disabled
  account; отсутствие или invalid value делает production configuration невалидной.
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
  `user_identities` row и минимальный `user_agreements` record. Повторная доставка и
  конкурентные `/start` идемпотентно resolve в тот же account через unique provider
  identity и не создают второе agreement; browser session для создания не нужна.
- Plain `/start` показывает, что такое TrailBase и бот, главное меню, обычную
  HTTPS-ссылку на актуальные правила и ссылку на CC BY 4.0, под которой размещаются
  пользовательские материалы. Сообщение объясняет: продолжая пользоваться ботом,
  пользователь автоматически соглашается с правилами, включая их будущие изменения.
  Отдельной кнопки «Согласен», pending agreement state и web-перехода нет;
  `/start <link-token>` остаётся отдельным linking flow.
- Главное меню Telegram/Max содержит пять основных действий в одном порядке:
  «Поиск», «Загрузить GPX», «Мои треки/черновики», «Настройки» и «Помощь».
  «Правила» и «Лицензия» доступны как вторичные ссылки и не входят в главное меню.
- «Настройки» содержит пять разделов: «Профиль» (`users.display_name` и язык),
  «Мессенджеры» (linked Telegram/Max identities и primary provider),
  «Уведомления» (category preferences), «Сессии» (active web sessions, revoke одной
  и logout-all) и «Аккаунт» (деактивация). Email, phone, password и recovery
  settings отсутствуют.
- Все пять разделов полностью работают в private Telegram/Max chat без web session.
  Profile/notification preferences изменяются прямо в chat; linking второго provider
  начинается там и завершается его `/start <link-token>`; web sessions можно
  просмотреть, revoke по одной или logout-all; account можно деактивировать. Уже
  принятые fresh-auth/confirmation requirements для sensitive operations сохраняются.
  Web settings UI является optional mirror тех же domain operations.
- «Мои треки/черновики» открывает в chat единый список всех принадлежащих
  пользователю tracks и upload/draft flows с явным пользовательским статусом:
  `draft`, `processing`, `pending_review`, `changes_requested`, `published` или
  `rejected`. Сначала идут элементы, требующие действия пользователя, затем остальные
  по `updated_at DESC`. Для просмотра списка web session не требуется.
- User-archived tracks исключены из обычного списка и доступны через вторичный фильтр
  «Архив» внутри «Мои треки/черновики». Карточка archived track показывает оставшийся
  срок до 30-дневной очистки и действие «Восстановить». Досрочный permanent delete
  archived track в MVP не предлагается.
- В MVP доступны только два list views: default «Все» без archived tracks и
  вторичный «Архив». Дополнительных filters по status нет; action-required entries
  выделяются уже принятым порядком и status label.
- User archive durable сохраняет непосредственно предшествующий lifecycle status и
  current revision. Restore до `purge_at` атомарно возвращает это состояние:
  опубликованный track снова становится `published` с той же approved revision без
  повторной moderation, непубликованный — прежним private status. Moderator removal
  этим flow не восстанавливается.
- «Восстановить» выполняется одним valid callback без дополнительного confirmation.
  Mutation атомарно проверяет owner, текущий `archived` status и `purge_at`.
  Повторная доставка операции идемпотентна и возвращает уже восстановленное состояние;
  foreign, stale или expired callback ничего не меняет.
- Действия карточки зависят от статуса: `draft` и `changes_requested` позволяют
  продолжить редактирование; `processing` — посмотреть прогресс или отменить;
  `pending_review` — только просмотреть; `published` — открыть, скачать sanitized
  GPX, начать новую revision или архивировать; `rejected` — посмотреть причину и
  создать исправленную revision. Удаление draft и архивация track всегда требуют
  отдельного подтверждения.
- Список содержит 10 элементов на страницу и inline controls «Назад»/«Далее».
  Pagination использует deterministic server-side keyset по принятому порядку с
  stable kind/UUID tie-breaker. Provider callback содержит только случайный opaque
  ID; Valkey record хранит cursor и binding к provider/chat/message/requester.
  Нажатие с несовпадающим binding не раскрывает account data и не меняет сообщение.
- List controls имеют абсолютный TTL 15 минут от первого открытия без sliding
  продления. После expiry или потери Valkey state callback не редактирует старое
  сообщение: bot сообщает, что список устарел, и предлагает заново открыть «Мои
  треки/черновики».
- Кнопки/команды «Правила» и «Лицензия» в Telegram/Max возвращают соответствующие
  HTTPS-ссылки. Полные документы и их копии в chat не хранятся.
- Правила доступны по одной ссылке на актуальный текст и могут изменяться на этой
  странице в любое время. Изменения правил не версионируются в account model, не
  требуют повторного agreement, не меняют account/session capabilities и не создают
  специальные notifications или другие domain events.
- Автоматическое agreement при первом plain `/start` является standing consent на
  размещение пользовательских contributions под CC BY 4.0. Submit revision
  показывает license reminder/link, но не требует отдельного acceptance action.
- PostgreSQL `user_agreements` хранит минимальное append-only evidence:
  `user_id`, `accepted_at`, `source = /start`, SHA-256 показанного notice,
  `rules_url` и `license_url`. Unique constraint `user_id` обеспечивает не более
  одного agreement на account.
- Повторный plain `/start` не обновляет `accepted_at`, notice hash или URLs в
  `user_agreements`: record навсегда фиксирует первый `/start`. Bot снова показывает
  notice/menu и обновляет только обычный provider profile snapshot.
- Agreement records не содержат IP, raw session/link tokens или provider callback
  payload; PostgreSQL является единственным durable source of truth. Изменение
  содержимого по `rules_url` ничего в agreement record или account state не меняет.
- Account creation, выпуск web-session/link tokens и identity linking разрешены только
  в private one-to-one chat с ботом. Group/channel events не создают account, не
  изменяют identity state и не раскрывают account state; bot отвечает статической
  ссылкой или инструкцией продолжить в private chat без auth token.
- Дополнительная identity связывается с account только явно из уже
  аутентифицированного account context. Нельзя автоматически объединять accounts по
  имени, username или другим profile fields.
- Agreement принадлежит `user_id`, а не отдельной messenger identity. Успешно
  привязанная Telegram/Max identity сразу использует capabilities target account без
  нового notice record; evidence первоначального `/start` не меняется.
- Если identity уже принадлежит другому аккаунту, link отклоняется. Account merge
  не входит в M02.
- Active account всегда сохраняет хотя бы одну active Telegram/Max identity; отвязать
  последнюю identity нельзя.
- При unlink active identity физически удаляется. Audit хранит внутренний identity UUID
  и provider, но не `provider_user_id`.
- Основное публичное имя — редактируемое `users.display_name`. Оно инициализируется
  именем первой identity и затем не обновляется автоматически.
- Смена `users.display_name` применяется сразу без fresh auth и pre-moderation.
  Ввод нормализуется в Unicode NFC, внешние пробелы удаляются; пустое значение,
  control/newline characters и длина больше 200 Unicode code points отклоняются.
  Изменение audit-ится. Abuse обрабатывается общей moderation policy, а не блокирует
  каждую смену имени.
- Public track pages разрешают attribution через текущее `users.display_name`, поэтому
  после реактивации смена имени обновляет его на всех ранее published track pages.
  Отдельный snapshot author name в track/revision для web attribution не хранится.
- Provider display fields обновляются на `/start` и хранятся только как snapshot:
  `display_name`, `username`, `language_code`, ограниченный `profile_meta`.
- Provider avatar URL не сохраняется. Допустим только безопасный `provider_avatar_ref`;
  загрузка выполняется сервером.
- Публично отображаются только TrailBase display name и будущий TrailBase avatar.
  Provider usernames и identities не публикуются.
- Пользовательские UI locales в MVP — только `ru` и `en`. Язык при первом
  взаимодействии выбирается в порядке: сохранённая настройка, поддерживаемый
  `language_code` bot profile, поддерживаемый `Accept-Language`, затем `ru`. Ручной
  выбор применяется и к bot, и к web UI и больше не перезаписывается provider
  language. `simple` используется только как технический fallback для content
  labels/tags/POI и не является пользовательским locale.

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
- Карточка active web session в private chat показывает только безопасное сокращённое
  device/browser description, `created_at`, `last_seen_at`, текущее sliding
  `expires_at` и действие «Завершить». IP, geolocation, полный User-Agent, session ID
  и raw/hashed token не показываются и не попадают в provider callback; target
  разрешается server-side через bound opaque control state. Метки «текущая» в chat
  нет, поскольку chat не является browser session.
- Revoke одной выбранной session требует короткого confirmation с тем же
  device/browser summary, но не fresh auth. Confirmation mutation повторно и атомарно
  проверяет owner и существование target session; уже завершённая/expired session
  возвращает нейтральный idempotent result. `logout-all` остаётся отдельной
  sensitive operation с fresh bot authentication.
- `logout-all` после fresh auth требует explicit confirmation с текущим числом
  найденных sessions и атомарно отзывает все active web sessions account на момент
  mutation, возвращая фактическое число revoked. Messenger identities, chat access и
  account остаются active; unlink или deactivation не выполняются.
- Session не содержит roles/permissions. Они читаются из PostgreSQL на каждом
  авторизованном запросе, поэтому отзыв полномочий применяется сразу.
- После Valkey session lookup каждый authenticated request также проверяет active
  account в PostgreSQL. Disabled account немедленно теряет доступ независимо от того,
  удалён ли session key физически.
- Если PostgreSQL недоступен при session validation, account-specific HTTP request
  возвращает `503` с `Retry-After` и не выполняет mutation. Cached active status и
  Valkey session сами по себе не авторизуют запрос.
- Такой transient `503` не очищает browser cookie и не удаляет Valkey session, но
  также не обновляет `last_seen_at` и не продлевает sliding TTL, поскольку запрос не
  был авторизован. Ответ содержит `Cache-Control: no-store`; после восстановления
  PostgreSQL та же ещё не expired session снова может авторизовать запрос. Cookie
  очищается и session отзывается только при подтверждённом invalid, expired или
  disabled состоянии.
- Разрешены независимые сессии на нескольких устройствах. M02 включает страницу
  активных сессий, отзыв одной сессии, обычный logout текущей и logout-all.
- Новое browser authentication в уже авторизованном браузере выдаёт новый session ID
  и отзывает прежнюю
  сессию этого браузера. Сессии других устройств не затрагиваются.
- Account deactivation отзывает все web sessions и инвалидирует outstanding
  web-auth/link tokens. Reactivation не восстанавливает эти credentials; новый web
  login или identity-link flow пользователь инициирует отдельно.
- Потеря Valkey завершает все сессии. Valkey не восстанавливается из backup, чтобы не
  воскресить отозванные sessions/tokens.

### 4.4 Sensitive operations и account lifecycle

- Fresh bot authentication не старше 10 минут требуется перед unlink identity, сменой
  primary bot provider, logout-all, назначением ролей и деактивацией.
- Draft takeover не относится к sensitive operations и не требует fresh bot auth:
  достаточно authenticated owner и явного подтверждения. Web request требует active
  session и CSRF, bot request — validated identity в private chat.
- Просмотр сессий и отзыв одной конкретной сессии доступны по обычной active web
  session либо через validated identity в private chat; web mutation проверяет CSRF,
  chat callback — provider/chat/message/requester binding. Logout-all по-прежнему
  требует fresh bot authentication.
- При потере доступа ко всем linked messenger identities self-service recovery
  отсутствует. Существующие browser sessions продолжают обычную работу, но linking и
  sensitive operations без fresh bot auth недоступны; после утраты sessions account
  недоступен.
- Пользователь может сам деактивировать аккаунт после fresh auth и явного
  подтверждения. Reason code, free text и feedback при self-deactivation не
  запрашиваются. Все сессии отзываются, новые входы блокируются, содержимое
  сохраняется.
- Final self-deactivation confirmation не показывает динамические counts и явно
  перечисляет: отзыв всех web sessions и outstanding auth/link tokens; блокировку
  account-specific bot/web operations; сохранение public tracks под CC BY 4.0 и
  private content; завершение текущего parse только private draft и исключение
  pending review из moderator queue; admin-only возврат через
  `TRAILBASE_SUPPORT_URL`. Primary action — «Деактивировать аккаунт», secondary —
  «Отмена».
- Final confirmation не выдаёт новый fresh-auth interval: absolute `expires_at` равен
  `fresh_authenticated_at + 10 minutes` без sliding. Chat callback несёт только
  случайный 128-bit opaque ID; server state bound к
  user/provider/chat/message/requester/purpose. Web требует active session,
  session-bound CSRF и state, bound к тому же user/purpose. Confirm и Cancel
  одноразово и атомарно consume state. Foreign, stale, expired или replayed action
  ничего не меняет и требует заново начать operation с fresh auth.
- Confirmation state хранится в PostgreSQL
  `sensitive_operation_confirmations`, не в Valkey. Row содержит SHA-256 opaque ID,
  user/purpose/interface bindings, fresh-auth/expiry timestamps и `consumed_at`; raw
  ID существует только в chat control или browser form и не логируется. Confirm
  делает `SELECT ... FOR UPDATE`, повторно проверяет binding, expiry, account status
  и last-active-admin guard, затем в одной transaction ставит `consumed_at`,
  деактивирует account и пишет audit/outbox. Cancel в transaction только consume-ит
  row. Janitor удаляет consumed/expired confirmation rows через 24 часа.
- PostgreSQL account status — немедленный source of truth для deactivation. Та же
  Confirm transaction пишет idempotent cleanup outbox command только с
  user/operation IDs, без raw tokens. Worker по account indexes удаляет все Valkey
  sessions и outstanding web-auth/link tokens, используя обычные retry и DLQ.
  Cleanup failure не возвращает доступ и не меняет committed account state;
  cleanup backlog/DLQ вызывает operator alert.
- Дополнительный credential epoch/version не вводится. Deactivation flow очищает
  account sessions и outstanding auth/link tokens непосредственно в Valkey через
  описанный cleanup command; отдельного слоя invalidation state нет.
- Опубликованные tracks деактивированного account остаются в public catalog с
  сохранённым TrailBase author display name и CC BY 4.0 attribution. Деактивация
  блокирует новые mutations, но не скрывает и не удаляет published content; private
  drafts/content остаются private. Скрытие или удаление track выполняется только
  отдельной track lifecycle operation.
- Public catalog не раскрывает факт или причину деактивации автора: published track
  и attribution отображаются без badge/account-status marker. Disabled status
  доступен самому пользователю только через нейтральный auth response, а
  администраторам — через management UI и audit.
- Уже начатый parse при деактивации может завершить technical cleanup/validation и
  создать только private draft; автоматического submit нет. `pending_review`
  сохраняет durable status без отдельного `suspended`, но исключается из moderator
  queue. Любая approve/publish mutation повторно проверяет active owner и fail-closed
  для deactivated account. После реактивации pending item снова виден в очереди, а
  private draft отправляет сам пользователь.
- Deactivation и approve/publish сериализуются row lock на owner `users` row и
  соблюдают lock order `user -> track -> revision`. Первая успешно committed
  transaction определяет результат: committed approval публикует track, который
  последующая деактивация оставляет public; committed deactivation заставляет
  ожидавший approval повторно увидеть disabled owner и завершиться без публикации.
- Встроенного bot workflow «Запросить реактивацию» в MVP нет. `/start` disabled
  account показывает `TRAILBASE_SUPPORT_URL`; bot не создаёт ticket, не пересылает
  free-form message администраторам и не меняет account state.
- Реактивация в MVP доступна только администратору через management UI после
  out-of-band проверки, требует fresh auth, обязательной причины и audit. Успешная
  реактивация создаёт locked-on account-lifecycle notification пользователю.
- Reactivation reason code обязателен и выбирается только из
  `support_request_verified`, `administrative_correction`, `other`. Audit note
  optional, кроме `other`, где после trim она должна быть непустой; control/newline
  characters и длина больше 1 000 Unicode code points отклоняются. Code и note
  хранятся только в append-only admin audit и не попадают в user notification или
  application logs.
- Reactivation только переводит тот же account обратно в active. Сохранённые linked
  identities, roles, settings, private content и pending moderation state снова
  доступны; следующий validated private-chat event получает обычное меню. Revoked
  sessions, invalidated auth/link tokens и suppressed notifications не
  восстанавливаются, новый agreement не создаётся. Lifecycle notification не
  содержит auth token; новый web login пользователь инициирует отдельно через bot.
- Деактивация и последующая реактивация не удаляют и не изменяют исходный
  `user_agreements` record и не требуют нового agreement. Полное физическое удаление
  account регулируется отдельной data-retention/privacy policy и не входит в MVP.
- Любой private-chat event от linked disabled identity resolve-ится в прежний account
  и никогда не создаёт новый. Доступны только `/start` с нейтральной инструкцией,
  «Помощь», «Правила», «Лицензия» и read-only поиск public catalog под stateless
  public principal. Upload, «Мои треки», settings, web-auth/link tokens,
  account-specific данные и любые mutations недоступны. Group/channel stateless
  public search не меняется.

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
- Reactivation audit metadata содержит validated reason code и optional note по
  account-lifecycle contract; произвольного reason code нет.
- Self-deactivation audit хранит `actor = user`, action, interface/provider, request
  ID и timestamp без reason code или free text.
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
  - public search: 120/мин на IP, burst 30;
  - anonymous canonical GPX download: 30/мин на normalized client IP, burst 10.
- При превышении возвращаются `429` и `Retry-After`. IP используется только как
  краткоживущий limiter key.
- GPX download limiter применяется до publication lookup и S3 signing и считает все
  попытки, включая будущий `404`. Browser session лимит не повышает. Последующий S3
  GET по presigned URL этим limiter не учитывается.

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
- Bot moderation buttons ведут deep-link на web UI. Approve/reject не выполняются
  прямо из bot.
- Web inbox notification хранится один год. Пользователь может поставить `read_at`
  или `archived_at`, но не удалить запись до окончания retention.
- Security notifications и moderation events, требующие действия owner
  (`changes_requested`, `rejected`), обязательны и всегда попадают в web inbox и
  primary bot. В settings они показываются locked-on и не могут быть отключены.
  Catalog, informational и остальные moderation results настраиваются пользователем.
- Для deactivated account действует явное исключение: материализуются и доставляются
  только locked-on security/account-lifecycle notifications, включая confirmation
  деактивации и административную реактивацию, в web inbox и primary linked messenger.
  Все moderation notifications, включая обычно locked-on `changes_requested` и
  `rejected`, а также catalog/informational categories подавляются без backlog/replay
  после реактивации. Domain events и audit продолжают записываться.
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
- Reactivation lifecycle notification содержит только факт и timestamp реактивации,
  `TRAILBASE_SUPPORT_URL` и инструкцию снова открыть bot. Admin identity, internal
  reason, audit ID и web-auth/link tokens пользователю не раскрываются; обязательная
  admin reason остаётся только в audit.
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

- Web и bot uploads используют один pipeline. Любой GPX attachment в private
  one-to-one chat от identity, прошедшей provider webhook validation, автоматически
  начинает новый upload flow после проверки account quota и лимита трёх active
  flows; browser session и предварительная команда не требуются. `/upload` только
  предлагает прислать файл и не создаёт обязательный pending mode. Attachment другого
  типа не создаёт job и получает короткую инструкцию о допустимом GPX-формате.
  Group/channel upload не создаёт job или draft и предлагает продолжить в private
  chat.
- Полный upload flow может завершаться в Telegram/Max chat: приём файла, validation,
  выбор обязательных metadata, preview статуса parse и отправка revision на moderation.
  Web deep-link остаётся дополнительным, а не обязательным шагом.
- Один пользователь может вести в private chat несколько upload flows одновременно в
  пределах лимита трёх active upload flows. Неявного «текущего upload» нет:
  status, controls и metadata prompts всегда связаны с конкретным `upload_job_id`.
  Свободный текст принимается как metadata только в reply на prompt этого job; иначе
  bot просит явно выбрать upload.
- После успешного parse bot использует редактируемую draft summary card, а не
  обязательный последовательный wizard. Name и Activity явно отмечены как required;
  actions открывают bound prompts для этих и optional metadata, а пользователь
  заполняет поля в любом порядке. «Отправить на модерацию» при незаполненных required
  полях не выполняет mutation и перечисляет недостающие поля. Готовый draft сначала
  показывает final summary с CC BY 4.0 reminder/confirmation.
- После принятия attachment bot сразу отправляет одну status card, привязанную к
  `upload_job_id`, и best-effort редактирует её при переходах
  download/validation/parse вместо отдельных сообщений на каждый этап. Terminal
  success превращает card в draft summary с дальнейшими actions, terminal failure —
  в безопасное объяснение с retry action. Если provider edit недоступен или
  завершается ошибкой, bot отправляет одну replacement card; durable domain job не
  запускается повторно и не зависит от состояния сообщения.
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
- Один принятый attachment создаёт ровно один TrailBase draft, даже если GPX содержит
  несколько `<trk>`; автоматического split на несколько drafts нет. Валидные
  `<trkseg>` всех `<trk>` становятся компонентами canonical `MultiLineString` в
  порядке документа.
- Отдельный segment manifest в MVP не хранится: границы и deterministic order
  сегментов уже задаются компонентами `MultiLineString`, а исходная иерархия
  `<trk>/<trkseg>` остаётся в original raw GPX и доступна только через re-parse.
- Parse result сохраняет только scalar diagnostics `source_track_count`,
  `source_route_count` и `valid_segment_count`, без source indexes или metadata
  отдельных `<trk>`/`<rte>`. При нескольких source elements выбранного geometry kind
  draft summary и final summary показывают неблокирующий warning с source/valid
  counts рядом с обычным map preview. Отдельного confirmation gate нет: final submit
  подтверждает весь snapshot; пользователь может отменить draft и загрузить
  отдельные файлы, если нужны отдельные catalog tracks.
- Route-only GPX поддерживается: каждый `<rte>` становится сегментом. Если есть и
  `<trk>`, и `<rte>`, используются tracks, а routes показываются warning. Один
  route-only attachment с несколькими `<rte>` также создаёт один draft без automatic
  split.
- Waypoints показываются в preview и могут стать POI suggestions, но не создают
  locations автоматически.
- GPX `<name>` и `<desc>` используются только как неподтверждённые defaults формы.
  Initial Name выбирается в порядке: непустой file-level `<metadata><name>`; затем
  единственное distinct непустое `<trk><name>` среди всех tracks (одинаковые значения
  считаются одним); затем безопасно нормализованный stem provider filename без path,
  control characters и расширения. При нескольких разных `<trk><name>` parser не
  выбирает первый и не объединяет строки, а показывает warning и переходит к
  filename fallback. После копирования это обычное редактируемое draft name, а raw
  filename отдельно не хранится и публично не показывается. Если все источники
  отсутствуют, Name остаётся пустым. Name общий, description — plain-text map по
  языкам. Для route-only input применяется тот же порядок, но element-level
  candidate берётся из единственного distinct непустого `<rte><name>`; несколько
  разных route names дают warning и filename fallback.
- Initial Description выбирается из непустого file-level `<metadata><desc>`, затем из
  единственного distinct непустого `<trk><desc>` среди всех tracks; одинаковые
  значения считаются одним. При нескольких разных `<trk><desc>` parser ничего не
  выбирает и не объединяет, показывает warning и оставляет optional Description
  пустым для ручного ввода. Выбранный default остаётся неподтверждённым и записывается
  только в owner UI locale на момент upload. Для route-only input применяется тот же
  порядок с единственным distinct непустым `<rte><desc>`; несколько разных route
  descriptions дают warning и пустой optional field.
- Caption исходного GPX attachment не копируется в draft Description или иные domain
  metadata: он может содержать контекст чата или инструкцию. Description получает
  только выбранный по этому precedence GPX `<desc>` default либо текст, явно
  введённый через bound Description prompt. Caption остаётся лишь внутри принятого
  24-hour retention raw webhook payload и не становится domain content.
- Если GPX `<desc>` не содержит locale, он записывается как неподтверждённый default
  в ветвь текущего сохранённого UI locale owner на момент upload (`ru` или `en`).
  Language detection, translation и копирование в обе ветви не выполняются. Owner
  может изменить language binding и добавить другую версию через metadata actions;
  последующая смена UI locale не переносит уже сохранённый Description.
- При отображении Description presentation layer сначала использует requested UI
  locale, затем другую из `ru`/`en` с явной меткой фактического языка. Автоматического
  перевода нет; если обе ветви пусты, секция отсутствует. JSON API возвращает полный
  description map с обеими language branches и не применяет presentation fallback.
- Перед submit на moderation обязательны непустой final name и явно выбранный primary
  activity. Автоматическое agreement уже записано при `/start` и не является
  отдельным submit gate. Неизменённый GPX name считается подтверждённым только вместе
  с final submit summary; classifier suggestion не считается выбором activity.
  Summary напоминает о CC BY 4.0 и содержит license link.
- Description optional. Difficulty, season, tags и auto-derived metrics не блокируют
  submit. Metrics показываются в final summary и подтверждаются вместе со всем
  snapshot без отдельного подтверждения каждого поля.
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
- Сохраняются `elapsed_duration_s` и `moving_duration_s`. Default canonical
  `duration_s` выбирается из `moving_duration_s`, затем из `elapsed_duration_s`, иначе
  остаётся unknown. Оба рассчитанных значения, canonical value и source показываются
  в final submit summary без отдельного обязательного подтверждения duration.
- Пользователь может заменить canonical duration вручную или выбрать unknown; это
  меняет `duration_s`/`duration_source`, но не удаляет auto-derived moving/elapsed
  values из revision snapshot.
- Manual duration хранится как целое число секунд в диапазоне
  `1..31_536_000` (365 дней). Значение вне диапазона отклоняется. Unknown хранится как
  `duration_s = NULL`, `duration_source = unknown`; ноль не является duration.
- Для outlier comparison используется тот же default auto candidate:
  `moving_duration_s`, если доступен, иначе `elapsed_duration_s`. Warning требует
  отдельного подтверждения, когда
  `max(manual / auto, auto / manual) > 10`, но не запрещает сохранить значение.
  Ровно 10× warning не вызывает; без auto candidate comparison не выполняется.
- Outlier acknowledgement хранится durable в PostgreSQL и привязано к exact
  `manual_duration_s`, comparison source/value и версии duration algorithm.
  Confirmation mutation проверяет текущую lease generation и атомарно сохраняет
  acknowledgement. Оно переживает resume/takeover, но инвалидируется при изменении
  manual value, reparse или версии алгоритма. Valkey хранит только одноразовый
  prompt/control token и не является источником acknowledgement state.
- Moderation summary для acknowledged outlier показывает canonical manual duration,
  comparison auto value/source, ratio и факт acknowledgement. Это informational flag:
  он не вызывает automatic reject и сам по себе не меняет priority moderation queue.
- Moderator не изменяет duration или другие поля pending revision напрямую: он
  approve-ит snapshot как есть либо возвращает `changes_requested` с обязательной
  причиной. Исправление делает автор в новом immutable full revision без обязательной
  повторной загрузки GPX; validation и outlier acknowledgement вычисляются заново.
  Отдельная audited administrative repair operation не служит shortcut moderation.
- Moving time включает интервалы со скоростью не ниже 0,5 км/ч. Интервалы с
  неположительным временем, gap больше 10 минут или физически невозможной скоростью
  исключаются с warning. Алгоритм версионируется.
- `duration_source`: `gpx_moving`, `gpx_elapsed`, `manual`, `estimated`, `unknown`.

## 12. Object storage и upload jobs

- Первый production использует MinIO в Compose, но приложение зависит только от
  configurable S3 endpoint/bucket/credentials.
- Все buckets private. Object keys — opaque UUID с техническим prefix (`raw/`,
  `exports/`, `photos/`), без user/track IDs и исходных filenames.
- Private raw GPX не шифруется и не преобразуется приложением: S3 object содержит
  exact original upload bytes. Application crypto envelope, data/master keys,
  keyring, rewrap и decrypt path для raw отсутствуют.
- Raw GPX служит только внутренним source для parse/re-parse. Public или owner-only
  download route, presigned URL и HTTP streaming endpoint отсутствуют; raw читают
  только backend workers. Пользователь может скачать только опубликованный sanitized
  GPX.
- После успешного parse raw не имеет отдельного TTL и хранится, пока
  связанный draft/track не очищен физически. Он сохраняется во всех lifecycle states,
  включая published и archive/appeal retention, и всё время учитывается в user
  raw-storage quota. Raw удаляется при подтверждённом физическом удалении draft либо
  purge track; 24-hour janitor применяется только к incomplete jobs и orphan objects.
- Sanitized GPX не шифруется приложением: S3 хранит exact export bytes в private
  bucket, а клиент получает их напрямую по presigned HTTPS URL. Transparent
  provider-side SSE или disk encryption допустимы как deployment control, но не
  меняют object bytes или TrailBase API; backend decrypt proxy отсутствует.
- Каждый новый upload всегда создаёт новый immutable exact raw object;
  content-based dedup отсутствует даже внутри одного user, а также cross-track и
  cross-user. Re-parse или metadata-only revision внутри того же track lineage без
  нового upload ссылается на существующий raw object и не копирует source bytes.
  Raw-storage quota считает физический object один раз; cleanup удаляет его только
  после исчезновения последней durable reference. Integrity/retry использует private
  content HMAC, не публичный raw SHA-256.
- PostgreSQL table `raw_objects` является единственной durable ownership/storage
  boundary для raw: row хранит `owner_id`, opaque S3 key, exact `byte_size`, private
  content HMAC, lifecycle state и timestamps. `upload_jobs.raw_object_id` и
  `track_revisions.raw_object_id` ссылаются на эту row; revisions не копируют object
  key или storage metadata. Quota считает distinct `raw_objects`, а cleanup
  определяет последнюю durable reference по БД.
- Retained `track_revisions.raw_object_id` всегда pin-ит raw; FK обязательный для
  GPX-derived revision и использует `ON DELETE RESTRICT`. Nullable
  `upload_jobs.raw_object_id` использует `ON DELETE SET NULL` и является pin только
  пока job может продолжить upload/parse либо explicit transient retry до 24-hour
  incomplete deadline. После successful revision commit job больше не pin-ит object,
  потому что его удерживает revision; cancel/permanent failure снимает job pin и
  сразу разрешает cleanup. Историческая job row может сохранить correlation до
  physical raw delete, после чего FK становится `NULL`.
- Добавление новой revision reference и last-reference cleanup сериализуются одним
  порядком: owner `users` row, затем `raw_objects` row `FOR UPDATE`. Reuse после lock
  проверяет совпадение owner, принадлежность existing raw тому же track lineage и
  `state = ready`, затем вставляет revision reference. Cleanup под теми же locks
  заново проверяет pin predicate и только затем атомарно ставит `delete_pending` с
  outbox command. `delete_pending` необратим: если cleanup победил, поздний reuse
  отклоняется и требует нового upload. `ON DELETE RESTRICT` остаётся DB-защитой от
  physical row delete при retained revision.
- `raw_objects.state` имеет только `pending`, `ready`, `delete_pending`. Row
  `pending` создаётся до S3 PUT; только checksum-validated `ready` можно привязать к
  revision. Потеря последней durable reference атомарно меняет `ready` на
  `delete_pending` и пишет idempotent cleanup outbox command. Незавершённый
  `pending` по 24-hour janitor также переходит в `delete_pending`. После успешного
  idempotent S3 delete worker физически удаляет row. Upload failure хранит
  `upload_jobs`, delete retries/DLQ — cleanup command; отдельных `error` и `deleted`
  states у `raw_objects` нет.
- Raw cleanup outbox command содержит только `raw_object_id`, без S3 key или HMAC.
  Worker по ID читает current `delete_pending` row и opaque key,
  выполняет idempotent S3 DELETE (`404` равен success), затем физически удаляет row.
  Если row уже отсутствует, replay считается success. State, отличный от
  `delete_pending`, либо retained revision FK обрабатываются fail-closed через
  retry/DLQ и operator alert. Crash после S3 success до DB delete безопасен:
  повторный DELETE получает success/`404` и завершает row cleanup.
- Для user raw-storage quota каждый новый `pending` атомарно резервирует полный
  per-file limit 10 MiB, не доверяя provider size или `Content-Length`. После
  checksum-validated upload переход в `ready` заменяет reservation фактическим
  byte size. `delete_pending` не входит в quota с момента commit:
  terminal upload failure переводит row туда немедленно, а janitor остаётся
  fallback. Задержка/DLQ физического S3 delete считается operational storage leak и
  не блокирует новые uploads owner-а.
- Quota не кэшируется в `users`. Каждая quota-changing transaction сначала блокирует
  owner row `FOR UPDATE`, затем вычисляет по `raw_objects` indexed SQL sum:
  `pending` даёт 10 MiB, `ready` — `byte_size`, `delete_pending` — zero.
  Новый `pending` вставляется только после проверки суммы вместе с reservation в той
  же transaction. Partial covering index по `owner_id`/`state` для `pending|ready`
  включает `byte_size`; денормализованный counter и reconciliation job
  отсутствуют.
- S3 и PostgreSQL согласуются через `upload_jobs`: pending DB row, object write,
  validation/parse, transactional revision creation, terminal status.
- Incomplete jobs и orphan objects старше 24 часов удаляет janitor.
- HTTP upload сохраняет exact original bytes в private S3 object и ставит parse job.
  Web UI опрашивает status htmx-запросом каждые две секунды до terminal state. Для
  bot upload worker
  best-effort обновляет одну job status card; terminal success показывает draft
  summary и продолжает metadata/moderation flow, terminal failure — безопасную
  причину и retry action. Provider edit failure создаёт replacement card, но не
  повторяет domain job.
- Transient S3/Valkey/PostgreSQL failures повторяются до трёх раз. Invalid GPX,
  отсутствующая geometry и превышение limits завершаются без retry.
- После исчерпания автоматических попыток transient infrastructure failure показывает
  explicit retry. Он повторно проверяет active account/quota, атомарно занимает
  свободный upload slot и ставит тот же `upload_job_id` в очередь без второго job;
  сохранённый raw object используется повторно. Если slot недоступен, job не
  меняется и bot показывает active flows. Повторный callback идемпотентен и не
  запускает две попытки одновременно.
- Invalid GPX, отсутствие geometry и превышение limits не повторяют те же bytes:
  status card предлагает отправить исправленный GPX как новый attachment. Если
  raw object transient-failed job не был сохранён или уже очищен, retry также
  просит прислать файл заново вместо создания попытки без source.
- После успешного parse создаётся постоянный private track draft. Он не истекает и
  доступен только owner и moderators; unlisted/secret-link доступа нет.
- Если owner деактивирован до revision commit, уже начатый worker может завершить job
  только созданием private draft и terminal technical status. Он не отправляет draft
  на moderation и не создаёт publication side effects; orphan cleanup остаётся
  обычной ответственностью janitor.
- Явный cancel после успешного parse закрывает active chat upload flow и освобождает
  slot, но не удаляет private draft или raw. Draft остаётся под quota и может
  быть продолжен через `/drafts` или web. Физическое удаление draft/raw — отдельное
  явное действие с подтверждением.
- User quota: 100 private drafts и 1 GiB raw storage, с admin override. Проверка raw
  quota учитывает fixed 10 MiB reservation для `pending`, actual byte size для
  `ready` и zero для `delete_pending`.
- Максимум три active upload flows на пользователя. Slot занят на стадиях
  intake/download/parse и после успешного parse при ожидании обязательных metadata или
  submit на moderation. Slot освобождается только после submit, явной отмены или
  terminal intake/parse failure.
- Resume сохранённого draft через `/drafts` атомарно занимает active slot. Если все три
  slots заняты, draft остаётся закрытым, а bot показывает active flows и предлагает
  завершить или отменить один.
- Один draft имеет максимум один global active edit/upload lease во всех interfaces:
  Telegram, Max и web. Повторный вход показывает существующий flow и предлагает явный
  takeover. Takeover атомарно переносит lease без нового slot, меняет lease generation
  и инвалидирует старые prompts/controls; каждая metadata mutation проверяет текущую
  generation, чтобы предотвратить lost updates.
- Takeover требует обычной owner authentication и explicit confirmation, но не fresh
  bot authentication: web использует session-bound CSRF, chat — validated private
  webhook identity.
- После commit takeover прежний bot prompt best-effort редактируется: controls
  удаляются, показывается нейтральное сообщение о продолжении в другом interface.
  Ошибка provider edit не rollback-ит lease и только логируется/метрится.
- Для web отдельный realtime transport не добавляется. Следующая mutation со stale
  generation получает `409` с сообщением о takeover и reload/resume action.
- Post-parse lease закрывается после 24 часов без успешного user metadata/control
  action. Успешная mutation продлевает idle deadline в той же транзакции; status reads,
  polling и worker events его не продлевают. Expiry освобождает slot, сохраняя
  draft/raw. Intake/download/parse используют отдельный 24-часовой incomplete-job
  timeout.
- Global draft lease, generation, active interface, last user action и three-slot
  accounting хранятся в PostgreSQL рядом с durable upload/draft state. Acquire,
  takeover, generation validation, metadata mutation и slot transition выполняются
  транзакционно. Valkey хранит только ephemeral prompt/control tokens; его потеря не
  меняет ownership, lease или slot count.
- Durable flow row содержит `slot_no` с `CHECK (slot_no BETWEEN 1 AND 3)` и
  `closed_at`. Partial unique indexes для `closed_at IS NULL` гарантируют уникальность
  `(user_id, slot_no)` и, при ненулевом draft, `draft_id`. Acquire транзакционно
  выбирает свободный slot и обрабатывает unique conflict; correctness не зависит от
  race-prone проверки `COUNT(*) < 3`.
- Перед каждым upload/resume acquire та же транзакция синхронно закрывает eligible
  expired post-parse leases пользователя, увеличивает их generation и освобождает
  slots, затем выделяет `slot_no`. Janitor вызывает ту же domain operation как
  фоновый fallback; корректность acquire не зависит от расписания janitor.
- Slot lifecycle operations `upload/resume/cancel/submit/expiry` сначала выполняют
  `SELECT ... FOR UPDATE` durable `users` row и соблюдают единый lock order
  `user -> flow -> draft`. Операции разных users не блокируют друг друга; partial
  unique indexes остаются DB backstop. PostgreSQL advisory locks не используются.
- Cancel до создания draft транзакционно ставит `cancel_requested_at` и увеличивает
  flow generation, но slot остаётся занят до terminal `cancelled`. Worker проверяет
  cancel перед S3 write, parse и revision commit.
- Если cancel выигрывает race, draft/revision не создаётся, temporary raw object
  удаляется best-effort и остаётся под janitor fallback. Если revision commit уже
  победил, применяется post-parse cancel: draft/raw сохраняются, flow закрывается и
  slot освобождается.
- Новые uploads получают `503 Retry-After`, если parse queue больше 1 000 jobs или
  старейшая job ждёт больше 10 минут. Catalog и auth остаются ready.
- `parse-worker` использует отдельный stream и стартовую concurrency 2 на container.
  `bot-worker` использует другой stream, чтобы uploads не задерживали auth.

## 13. Track aggregate и revisions

- `tracks` хранит stable identity: ID, неизменяемый `owner_id`, lifecycle timestamps,
  status и `current_revision_id`.
- Chat-представление «Мои треки/черновики» объединяет принадлежащие пользователю
  upload/draft flows и tracks в один список с нормализованными статусами. Элементы,
  требующие действия owner, сортируются первыми; остальные — по последнему изменению.
- User-archived tracks не входят в default list и доступны через filter «Архив».
  До `purge_at` карточка показывает оставшийся срок и позволяет восстановление;
  отдельной операции досрочного permanent delete в MVP нет.
- Status filters кроме default «Все» и «Архив» отсутствуют в MVP.
- User archive сохраняет pre-archive status/current revision. Restore изменяет тот же
  `track_id` и до `purge_at` возвращает сохранённое состояние без создания revision:
  прежний approved snapshot снова публикуется, private track остаётся private.
  Moderator removal является отдельным lifecycle state и не eligible для user restore.
- Restore не создаёт отдельный confirmation state. Один valid owner callback
  атомарно проверяет `archived`/`purge_at` и меняет lifecycle state; повторная
  доставка той же операции идемпотентна.
- Chat actions проверяют текущий durable status перед mutation. `pending_review`
  доступен owner только для просмотра; исправление `changes_requested`/`rejected`
  создаёт новую immutable revision. Удаление draft и архивация track выполняются
  только после отдельного подтверждения.
- Chat list pagination возвращает по 10 entries и использует server-side keyset с
  deterministic tie-breaker; callback state не раскрывает entry IDs или cursor
  provider-у и привязан к requester и конкретному сообщению.
- Исходный `expires_at` chat list state равен 15 минутам от открытия и не меняется при
  переходах между страницами. Expired или потерянный ephemeral state не
  восстанавливается и не позволяет изменить старое сообщение.
- Geometry, S3 references, name/descriptions, activity, difficulty, season, duration
  и tags находятся в immutable full snapshot `track_revisions`.
- Private draft доступен только owner/moderators. Отправка создаёт `pending_review`.
- `pending_review` deactivated owner не меняет status, но не попадает в moderator
  queue. Approve/publish всегда повторно проверяет active owner; после реактивации item
  автоматически снова eligible для очереди без resubmit.
- Immutable pending revision до approval получает private sanitized export и durable
  `export_state` (`pending`, `ready`, `error`, `discarded`). Approve/publish разрешён
  только при `export_state = ready`; та же PostgreSQL transaction под обычным lock
  order переключает revision/track в `published` и обновляет `current_revision_id`.
  Generation failure оставляет revision в `pending_review`, старый current остаётся
  public, а export job проходит retry/DLQ. Отдельного публичного `publishing` status
  нет.
- До approval private sanitized object доступен только backend workers. Owner и
  moderator UI не получают presigned URL и не имеют отдельного authenticated
  download route; moderator видит только `export_state` и retry action. Download
  появляется после publication через canonical current-revision route.
- `pending_review` item с `export_state = pending|error` остаётся видимым moderator-у
  с internal badge «Подготовка экспорта»/«Ошибка экспорта». Snapshot можно изучить и
  перевести в `changes_requested` или `rejected`, но Approve disabled до `ready`.
  После исчерпания automatic retries moderator может вызвать идемпотентное
  «Повторить экспорт», operator получает alert. Owner во всех этих случаях видит
  только обычный `pending_review` без infrastructure details.
- `changes_requested`/reject transaction одновременно ставит export lifecycle в
  `discarded` и пишет high-priority idempotent delete command private sanitized
  object. Revision и moderation audit сохраняются, object не переиспользуется:
  исправленная immutable revision получает новый export, а старый при необходимости
  генерируется заново. Worker меняет только `pending -> ready`; late completion после
  discard не оживляет state и ставит созданный object на delete. Cleanup использует
  retry/DLQ и operator alert.
- Approve/publish и account deactivation берут owner row lock до track/revision locks
  (`user -> track -> revision`). Если approval commit-нулся первым, опубликованный
  snapshot остаётся public после деактивации; если первой commit-нулась deactivation,
  approval fail-closed не меняет revision/publication state.
- Первая публикация требует moderation. `changes_requested` обязательно содержит
  причину. Авторская правка создаёт новый immutable full revision и повторно
  отправляет его без обязательной загрузки GPX; changes-requested revision остаётся
  неизменным audit snapshot.
- Изменение опубликованного трека создаёт pending full revision. Публичная revision
  остаётся прежней до атомарного approval.
- Ownership transfer не входит в MVP. Будущий transfer требует audited operation и
  подтверждения обоих пользователей.
- Авторское «удаление» переводит track в `archived`; восстановление доступно 30 дней
  через filter «Архив». Досрочная физическая очистка пользователю не предлагается.
  Restore возвращает durable pre-archive status и current revision; повторная
  moderation неизменённого approved snapshot не требуется.
  После `purge_at` worker удаляет raw/export/photos, revisions и derived data,
  оставляя tombstone track UUID и audit без пользовательского содержимого.
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

- Original raw GPX никогда не публикуется и не выдаётся owner-у отдельным private
  download. Для raw отсутствуют HTTP streaming endpoint и presigned delivery; он
  доступен только backend workers как внутренний parse/re-parse source.
- Sanitized GPX pre-generate-ится приватно для immutable pending revision и хранится
  отдельно. Он содержит только lat/lon/elevation, submitted name/description и
  license metadata; public доступ появляется лишь после approve transaction при
  `export_state = ready`.
- На application layer sanitized object является незашифрованным exact XML в private
  bucket. TrailBase выдаёт его напрямую по five-minute presigned HTTPS URL без
  decrypt proxy; прозрачное provider-side SSE/disk encryption не меняет этот
  контракт.
- До approval отсутствуют owner/moderator download endpoint и presigned delivery
  private sanitized object; доступ к object имеет только backend. Public download
  возникает только после approve через canonical current-revision route.
- Timestamps, heart rate, cadence, device extensions и original waypoints удаляются.
- Provenance и license записываются только в стандартные поля GPX 1.1, без
  application extensions: root `creator="TrailBase revision:<uuid>"`;
  `<metadata><author><name>` содержит TrailBase author display name;
  `<metadata><copyright author="..."><license>` содержит canonical
  `https://creativecommons.org/licenses/by/4.0/`; `<metadata><link href="...">`
  содержит canonical track URL. Пользовательские Name и Description остаются в
  `<trk><name>` и `<trk><desc>`. Provider identity, internal user ID, raw filename и
  application extensions отсутствуют.
- Каждый sanitized GPX содержит ровно один `<trk>` на TrailBase revision. В нём
  находятся один submitted `<name>` и один сформированный `<desc>`, а каждый
  component canonical `MultiLineString` сериализуется отдельным `<trkseg>` в
  deterministic component order. Исходная grouping нескольких `<trk>` не
  восстанавливается; route-only input также экспортируется как `<trk>`, не `<rte>`,
  поскольку публичная единица каталога — один TrailBase track.
- `<trkpt><ele>` создаётся только для сохранившейся после trimming point с валидным
  исходным elevation. При отсутствии значения `<ele>` у этой point отсутствует:
  export не интерполирует elevation, не подставляет zero и не использует
  smoothed/LTTB profile. Правило одинаково для track/route input и не зависит от 90%
  coverage threshold, который управляет только derived profile и gain/loss.
- Numeric text export-а детерминирован: lat/lon округляются `HALF_EVEN` максимум до
  7 знаков после decimal point, `<ele>` — максимум до 2. Используются decimal dot и
  plain notation без exponent; незначащие trailing zeros удаляются, negative zero
  нормализуется в `0`. Одинаковые revision inputs дают byte-identical numeric text.
- Весь serialized GPX для одной revision byte-identical: UTF-8 без BOM, фиксированная
  XML declaration `<?xml version="1.0" encoding="UTF-8"?>`, фиксированные GPX 1.1
  namespace declarations и schema-conforming element/attribute order. Serializer
  пишет compact XML без insignificant whitespace и ровно один LF в конце. Generation
  timestamps, random values и иные runtime-dependent bytes отсутствуют.
- Durable SHA-256 и byte-size поля export-а в PostgreSQL отсутствуют. S3 PUT
  подписывает actual payload SHA-256, не `UNSIGNED-PAYLOAD`, и `export_state` может
  стать `ready` только после успешной checksum-validated записи. Byte determinism
  проверяется serializer golden tests; hash/size не публикуются как HTTP `ETag` и не
  меняют `no-store` canonical download route.
- `<metadata><bounds>` в sanitized GPX отсутствует. Viewers вычисляют bounds из
  `<trkpt>`; derived element дублировал бы geometry, а обычный min/max longitude
  неоднозначен для antimeridian tracks.
- TrailBase activity, difficulty и tags не сериализуются в `<trk><type>` или
  `<metadata><keywords>`. Эти GPX-поля не имеют interoperable vocabulary для
  TrailBase taxonomy; classification остаётся доступна по canonical track URL и в
  JSON API без отдельного versioned GPX mapping.
- Standard GPX `<desc>` export-а является deterministic plain text: каждая непустая
  description branch включается с явной меткой `[ru]`/`[en]` в фиксированном порядке
  `ru`, затем `en`, блоки разделяются пустой строкой. Если обе ветви пусты, `<desc>`
  отсутствует. Viewer-specific variants, machine translation и custom XML extensions
  не используются; значения проходят обычный XML escaping.
- Author display name в уже сгенерированном sanitized GPX является immutable частью
  export object и не переписывается массово при смене `users.display_name`. Новый
  export object, включая export новой approved revision, использует актуальное имя на
  момент генерации.
- Автоматическое agreement первого `/start` действует как standing consent на
  публикацию contributions под CC BY 4.0 и допускает последующие изменения правил.
  Каждая отправка на moderation показывает reminder и canonical license link, но не
  требует отдельного acceptance.
- Каталог как база данных публикуется под ODbL 1.0. Перед первым bulk export требуется
  юридическая проверка текста attribution.
- Anonymous public download sanitized GPX не требует account/browser session и не
  создаёт identity или session. Canonical download endpoint повторно проверяет
  publication status, применяет до lookup/signing лимит 30 запросов/мин на normalized
  client IP с burst 10 и отвечает `302` на presigned URL с TTL 5 минут. Все попытки,
  включая `404`, расходуют лимит; session его не повышает, S3 GET после redirect не
  учитывается. Private, archived и moderator-removed revision получают одинаковый
  `404`; bot публикует canonical track/download URL, а не presigned URL.
- Все ответы canonical download endpoint (`302`, `404`, `429`, `5xx`) содержат
  `Cache-Control: no-store`; route не использует CDN cache или `ETag`. Каждый request
  заново проходит limiter и publication check, поэтому cached redirect не обходит
  revision switch/removal. Presigned S3 URL остаётся отдельной delivery-ссылкой с
  собственным five-minute expiry.
- `/tracks/:track-id/download` разрешает только current published revision.
  Revision-specific public download endpoint в MVP отсутствует; sanitized exports
  старых revisions не адресуются публично и могут очищаться по storage retention.
  Commit нового approved `current_revision_id` автоматически переключает тот же
  canonical download URL. Старой ссылкой нельзя запросить superseded moderation
  snapshot, geometry до privacy trim или removed track.
- Любой переход export из current/public в superseded или non-public пишет
  high-priority idempotent delete command старого sanitized object. S3 delete
  использует retry/DLQ и operator alert; proxy-download для мгновенного revoke не
  добавляется. После успешного delete ранее выданный presigned URL перестаёт
  работать; при временной задержке остаточный доступ ограничен его исходным TTL не
  более пяти минут.
- Sanitized S3 object хранит response metadata
  `Content-Type: application/gpx+xml; charset=utf-8` и
  `Content-Disposition: attachment; filename="<ascii-name-slug>-<uuid-fragment>.gpx"`.
  Поэтому headers применяются к final S3 response после presigned redirect; inline
  rendering и отдельный UTF-8 filename fallback не используются.
- S3 хранит и отдаёт exact uncompressed XML bytes. `Content-Encoding` отсутствует;
  gzip, ZIP и второе compressed representation в MVP не создаются. Canonical
  attachment всегда имеет расширение `.gpx`.
- Filename строится из final submitted revision Name без transliteration dependency:
  Unicode NFKD, lowercase ASCII letters/digits сохраняются, каждый run остальных
  codepoints становится одним `-`, края trim-ятся. Slug обрезается до 64 символов с
  повторным trailing-`-` trim; пустой slug становится `track`. Затем добавляются `-`,
  первые 8 lowercase hex символов stable track UUID и `.gpx`. Исходный upload
  filename не используется; полный Unicode Name остаётся внутри GPX.

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
- Web/bot search и detail выбирают Description по UI locale с fallback на вторую
  доступную ветвь и явной меткой языка. JSON API возвращает обе `ru`/`en` ветви без
  скрытого перевода или server-side подмены map.
- Telegram/Max `/search` использует ту же domain search service и выполняется без
  browser session. Bot адаптирует filters и pagination к chat controls; search semantics
  и permission checks не расходятся с web/API.
- В private chat search выполняется с permissions связанного account. В group/channel
  chat `/search` разрешён только как account-stateless read-only поиск по public
  published catalog: он не создаёт account, не возвращает private/unlisted tracks или
  персональные данные и не сохраняет user history/settings.
- Private-chat `/search` от linked deactivated identity использует тот же stateless
  public principal, а не permissions disabled account: только public published
  catalog, без history/settings, private results или account-specific facets.
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
