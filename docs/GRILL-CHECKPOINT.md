# TrailBase — Grill-me Checkpoint

Статус: paused
Дата: 2026-07-28

Принятые решения сохранены в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). Этот файл хранит только точку
продолжения design interview и не заменяет контракт.

## Последнее подтверждённое решение

Каждая новая session создаёт обязательное security notification в web inbox и primary
bot provider. Сообщение содержит время, provider и сокращённое описание устройства,
но не IP. Sliding TTL refresh существующей session уведомление не создаёт.

При unlink identity notification уходит всем оставшимся providers и best-effort
отвязываемому provider до удаления связи.

## Следующий вопрос

Должно ли security notification о новой session содержать deep-link на страницу
active sessions, где пользователь может немедленно отозвать неизвестную session?

Рекомендация: да. Ссылка открывает только authenticated session-management UI; сама
ссылка не выполняет revoke и не содержит action token.
