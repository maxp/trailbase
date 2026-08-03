# Контракт: browser auth flow и delivery-health defaults

Часть [Implementation Contract](../../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

- Durable health хранится прямо в `user_identities`: `delivery_health text NOT NULL
  DEFAULT 'unknown'` с CHECK `unknown|reachable|unreachable` и nullable
  `delivery_health_observed_at timestamptz`. Compound CHECK требует NULL timestamp для
  `unknown` и non-NULL для observed states. PostgreSQL enum, TTL/manual-clear column,
  raw reason и отдельная health state/history table отсутствуют.
- Отдельной функции «Проверить доставку» нет: Web/bot action, delivery-probe routes,
  `delivery_kind=health_probe`, probe outbox/status/polling/cooldown, visible test
  message и его schema/template catalog отсутствуют.
- Timeout/network error после dispatch mutating atomic Valkey function считается
  ambiguous commit outcome, а не mutation-free `503`: function могла создать session и
  consume-ить token. Handler выполняет bounded retry/read-back по opaque commit
  identifier и возвращает success либо сохраняющий flow `503` только после
  подтверждённого committed/not-committed результата. Предыдущий mutation-free `503`
  применяется лишь до dispatch или при доказанном non-commit. Если outcome остаётся
  unknown, используется отдельная recoverable branch; terminal-invalid cleanup и
  слепое создание второй session запрещены.
- `POST /auth` проверяет cookie/nonce, расходует token, создаёт только browser session
  и делает redirect на чистый URL. Если browser уже authenticated, новая session
  rotate-ит и отзывает прежнюю session этого браузера. Session получает
  `fresh_authenticated_at` из token record, а не из времени GET/POST. Flow не
  активирует account и не является обязательным для chat operations.
- Confirmation page показывает provider и экранированный display name, но не внешний
  provider ID.
- `return_to` допускает только внутренний путь, начинающийся с одного `/`; absolute URL
  и `//host` отклоняются. Нормализованный путь хранится server-side в auth-flow записи.
- Все token-bearing `GET /auth` и `POST /auth` responses — confirmation, success,
  invalid/expired и `503` — используют `Cache-Control: no-store` и
  `Referrer-Policy: no-referrer`; auth pages подключают только same-origin static assets
  без third-party resources. Caddy/application access и security logs пишут route без
  query и никогда не пишут form body. Raw token и digest отсутствуют в error details,
  analytics и tracing attributes.
- Web-session/link tokens имеют 128 бит криптографической энтропии: 16 random bytes,
  Base64URL без padding.
