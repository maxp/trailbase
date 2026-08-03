# Контракт: permissions и HTTP security

Часть [Implementation Contract](../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 5. Roles, permissions и audit

- Глобальные роли: `user`, `moderator`, `admin`; связь many-to-many через `user_roles`.
- HTTP/domain operations проверяют permissions, а не имена ролей. Mapping
  `role -> permissions` централизован в коде.
- Permission `track_appeal_decide` входит только в mapping фиксированной роли `admin`;
  роль `moderator` её не получает. Оба appeal outcome используют одну permission и
  existing fresh-auth check через management UI.
- Первый admin назначается явной командой
  `bb grant-role <provider> <provider-user-id> admin`, а не переменной окружения и не
  правилом «первый вошедший».
- Нельзя транзакционно удалить роль или деактивировать последнего active admin.
- Admin audit — append-only: actor, action, target type/ID, metadata, request ID,
  timestamp. Для bootstrap actor равен `system`.
- Appeal audit сохраняет actual admin actor; permission и fresh auth проверяются до
  загрузки sensitive audit context.
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

