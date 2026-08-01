# TrailBase — Grill-me Checkpoint

Статус: paused
Дата: 2026-08-01

Принятые решения сохранены в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). Этот файл хранит только точку
продолжения design interview и не заменяет контракт.

## Последнее подтверждённое решение

После successful PostgreSQL active identity/account check одна atomic Valkey function
валидирует flow/nonce, source token и active pointer, создаёт либо rotate-ит browser
session, consume-ит token/pointer и удаляет flow. Все credential mutations имеют одну
linearization point; distributed PostgreSQL transaction нет. Два concurrent POST одной
form создают ровно одну session, а проигравший terminal invalid и не revoke-ит успешную
session.

## Следующий вопрос

Должен ли общий `/auth` rate-limit reject быть retryable `429 Too Many Requests` с
`Retry-After`, сохранять flow/token/cookie/nonce и никогда не redirect-ить на
`result=invalid`?

Рекомендация: да. Один budget 10/min на normalized client IP охватывает initial GET,
clean confirmation GET и POST; IP берётся только из trusted Caddy forwarding contract.
Limiter срабатывает до credential lookup и не выполняет flow/token/session cleanup.
Response использует `no-store`, `no-referrer`, generic body и `Retry-After`; отдельного
per-flow/per-nonce budget, automatic retry и terminal marker нет.
