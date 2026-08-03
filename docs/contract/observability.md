# Контракт: health, logs и metrics

Часть [Implementation Contract](../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 9. Health, logs и metrics

- `/health/live` проверяет только процесс и публично отдаёт минимум данных.
- `/health/ready` проверяет PostgreSQL, Valkey и соответствующий worker consumer group;
  он доступен только внутри Compose network.
- `/metrics` Prometheus-compatible и внутренний. Минимальный набор: HTTP/DB/Valkey
  latency, statuses, stream lag/pending, retries, DLQ size, auth success/failure,
  rate-limit rejects.
- Fresh bot confirmation имеет ровно один дополнительный low-cardinality counter
  `fresh_auth_confirmation_total{provider,result}` с `provider=telegram|max` и
  `result=accepted|duplicate|invalid_event|invalid_binding|account_unavailable|internal_error`.
  Identity/request/event IDs, timestamp, route и error-detail labels запрещены.
- Его `duplicate` increment допустим только после committed fresh-auth acceptance;
  exact replay получает `2xx` без token/link/message/session side effects. До commit
  `internal_error` остаётся retryable через общий worker retry/DLQ, не duplicate.
- Все logs — structured JSON в stdout/stderr через Telemere. Контейнеры не ведут
  собственные log files.
- Сквозной correlation/request ID — UUIDv7. Он передаётся через webhook, stream,
  worker, outbound Bot API, logs и audit.
- OpenTelemetry на M01/M02 не добавляется; correlation IDs, metrics и logs достаточны.

