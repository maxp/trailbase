# TrailBase — Grill-me Checkpoint

Статус: paused
Дата: 2026-07-29

Принятые решения сохранены в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). Этот файл хранит только точку
продолжения design interview и не заменяет контракт.

## Последнее подтверждённое решение

Terminal recovery request хранится 90 дней, затем физически удаляются request,
candidate linkage, display snapshots, free-text notes и risk flags. Бессрочный audit
оставляет request UUID, terminal status/timestamps, actor UUID, reason code и affected
account UUID — без contact values/HMAC и provider user IDs.

## Следующий вопрос

Нужен ли отдельный appeal workflow для rejected recovery request в первом recovery
slice?

Рекомендация: нет. После 24-часового cooldown пользователь начинает новый flow и заново
доказывает оба фактора. Moderator может использовать 90-дневную запись для incident
review, но не reopen-ит terminal request и не обходит contact/candidate proofs. Appeal
UI и новый status не входят в первый slice.
