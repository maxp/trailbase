# TrailBase — Grill-me Checkpoint

Статус: paused
Дата: 2026-08-02

Принятые решения сохранены в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). Этот файл хранит только точку
продолжения design interview и не заменяет контракт.

## Последнее подтверждённое решение

Открытая Web identity card не получает automatic inbound-recovery refresh: polling, SSE
и WebSocket только ради health warning отсутствуют. Current page может оставаться stale
до следующего обычного navigation/htmx card refresh; следующий server render читает
authoritative identity row и убирает warning. Bot card обновляется при следующем user
request. `Cache-Control: no-store` сохраняется.

## Следующий вопрос

Нужно ли единичное событие `target_unreachable` создавать operator alert?

Рекомендация: нет. Считать его обычным per-user state transition. Оставить aggregate
metrics по provider и normalized outcome без identity/target labels; alert создавать
только по provider-wide threshold/circuit condition, а не при блокировке бота одним
пользователем.
