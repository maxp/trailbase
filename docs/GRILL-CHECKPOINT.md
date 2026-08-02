# TrailBase — Grill-me Checkpoint

Статус: paused
Дата: 2026-08-02

Принятые решения сохранены в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). Этот файл хранит только точку
продолжения design interview и не заменяет контракт.

## Последнее подтверждённое решение

Удаление старой schema/template pair требует одновременно минимум 90 дней с последней
emission и ноль ссылок из replayable undelivered/DLQ records. Deploy preflight
группирует их по notification type, `schema_version` и `template_version` и блокирует
removal при ненулевом count. Records нельзя rewrite, drop или silently переводить на
новую version ради gate. Delivered records, audit, semantic web-inbox и backup copies
removal не блокируют.

## Следующий вопрос

Должен ли `new_session` outbox хранить только закрытый action code `manage_sessions`, а
dispatcher строить актуальный canonical HTTPS URL из `PUBLIC_BASE_URL` при send?

Рекомендация: да. Absolute URL, query и redirect target в payload не сохраняются.
Catalog связывает `manage_sessions` с named internal route управления сессиями, а
adapter при send строит same-origin URL из текущего `PUBLIC_BASE_URL`; startup validation
проверяет HTTPS, allowlisted route и отсутствие action token. Смена public base URL
намеренно влияет на queued delivery, чтобы ссылка оставалась рабочей, но не меняет
зафиксированные locale, template version или semantic event fields.
