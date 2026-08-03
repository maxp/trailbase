# Контракт: Valkey sessions и account lifecycle

Часть [Implementation Contract](../../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

### 4.3 Sessions в Valkey

- Sessions хранятся в отдельном Valkey service с persistent volume и AOF
  `appendfsync everysec`.
- Claimed session имеет sliding TTL один год, который продлевается на каждом
  авторизованном запросе, без отдельного absolute lifetime. Provisional session до
  первого validated request использует exact auth-flow deadline без sliding.
- Cookie token имеет 128 бит CSPRNG-энтропии. Raw token находится только в cookie;
  Valkey key строится из `SHA-256(token)`.
- Cookie: `__Host-trailbase_session`, `Secure`, `HttpOnly`, `SameSite=Lax`, `Path=/`,
  без `Domain`.
- Session хранит `user_id`, `created_at`, `last_seen_at`, nullable
  `fresh_authenticated_at`, сокращённый `User-Agent` и CSRF state. NULL означает, что
  session ещё не проходила fresh authentication. `fresh_authenticated_at` не меняется от
  обычных requests, `last_seen_at` или sliding TTL; sensitive web operation допускает
  его только до absolute `fresh_authenticated_at + 10 minutes`. IP в session/profile
  не сохраняется. `reauthenticated_at` отсутствует: при credential rotation original
  `created_at` сохраняется, а факт/время re-auth уже задаёт `fresh_authenticated_at`.
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
- Там, где fresh bot authentication выпускает web-session link, она использует
  описанное explicit private-chat действие «Подтвердить вход»; сам validated event
  является единственным confirmation и не дополняется вторым prompt или credential.
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
  для deactivated account. После реактивации pending item снова виден в очереди при
  `export_state = ready`, а private draft отправляет сам пользователь.
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
