# TrailBase — Grill-me Checkpoint

Статус: paused
Дата: 2026-08-01

Принятые решения сохранены в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). Этот файл хранит только точку
продолжения design interview и не заменяет контракт.

## Последнее подтверждённое решение

Browser re-auth переиспользует обычный bot-issued `web_session` token и существующий
`/auth` GET/POST flow без отдельного re-auth credential/table/cookie/consume endpoint.
Token выдаётся после fresh bot authentication и хранит исходный
`fresh_authenticated_at`; consume rotate-ит current browser session, переносит этот
timestamp и возвращает по bound `return_to`. Freshness истекает через 10 минут и не
продлевается ordinary activity или sliding session TTL.

## Следующий вопрос

Считаем ли explicit private-chat action «Подтвердить вход», непосредственно выпускающий
`web_session` link, достаточной fresh bot authentication без дополнительного PIN,
пароля или второй confirmation-кнопки?

Рекомендация: да. Freshness создаёт только явная user-initiated command/callback в
private one-to-one chat, bound к provider/user/chat/message/requester, которая сразу
выпускает link; timestamp равен времени validated webhook event. Обычное недавнее
сообщение, notification click, background event или existing browser session freshness
не создают. Дополнительная кнопка после уже explicit action не доказывает новый factor
и только усложняет flow; PIN/password/TOTP в принятой bot-first модели отсутствуют.
