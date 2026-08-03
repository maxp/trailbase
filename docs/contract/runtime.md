# Контракт: runtime и platform

Часть [Implementation Contract](../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 1. Runtime и границы сервисов

- Dev и первый production используют один Docker Compose contract.
- Production stack: Caddy, `web`, `bot-worker`, `parse-worker`, one-shot `migrate`,
  PostgreSQL/PostGIS, Valkey 9.x или новее и MinIO. Valkey 9.0 — минимальная версия:
  auth delivery state использует hash-field expiration commands.
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
  account и для обращения по moderator removal; отсутствие или invalid value делает
  production configuration невалидной.
- `web` при shutdown снимает readiness и завершает текущие запросы в течение 30 секунд.
- Workers перестают брать новые события, завершают текущие до 30 секунд и оставляют
  незавершённые сообщения pending. Другой consumer забирает их через `XAUTOCLAIM`
  после 60 секунд без heartbeat.

## 3. Идентификаторы и время

- Primary keys основных сущностей и справочников — UUIDv7, генерируемые приложением.
- Short track ID — вычисляемый display/reference из последних 12 lowercase hex
  `tracks.id`, canonical form `trk-xxxx-xxxx-xxxx`. Это не primary key, secret или
  гарантированно unique identifier; отдельные column, sequence и generator не
  создаются. Input lookup после ASCII trim/case normalization принимает полный
  canonical UUID либо форму с ровно 12 hex, с optional exact prefix `trk-` и group
  hyphens. Short lookup использует non-unique expression index; несколько совпадений
  требуют полный UUID. Первые 8 hex UUIDv7 в GPX filename являются отдельным
  display-only suffix и как short track ID/lookup key не используются.
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

