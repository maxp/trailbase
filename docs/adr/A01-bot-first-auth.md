# ADR A01 — Bot-first Auth Model

**Status**: Accepted
**Date**: 2026-07-25

**Уточнение 2026-07-28:** tokens и sessions находятся в Valkey; browser login —
двухшаговый GET/POST flow, provider linking только явный. Полный контракт:
[Implementation Contract](../IMPLEMENTATION-CONTRACT.md#4-bot-first-authentication).

## Контекст

TrailBase — публичный каталог GPX-треков. Требуется auth-модель, поддерживающая_web-сайт как first-class приложение и Telegram/Max ботов как основной канал идентификации. По пользователю: «экаунты авторизуются через ботов или по емайлу» — с последующим уточнением: email/телефон оставлены **только как вариант восстановления доступа**, не как login.

## Решение

1. **Боты Telegram/Max — единственный канал login.** Email/телефон существуют исключительно как recovery-канал (потеря Telegram-аккаунта, смена номера).
2. **Bot = IdP, web = SP** через deep-link bridge:
   - Юзер `/start` в боте → бот выписывает одноразовый short-lived token;
   - Бот отправляет URL вида `https://catalog/auth?token=...` (deep-link);
   - **Клик → web-backend верифицирует token → создаёт server-side session cookie.**
3. **Единый аккаунт с провайдерным списком.** Один `user_id`, к нему привязано 1..n
   login identities (`telegram:*`, `max:*`) в `user_identities` и 0..n
   recovery-каналов в `user_recovery`. Первый подтверждённый browser login создаёт
   аккаунт; последующие identities добавляются только explicit link-flow.
4. **Боты не дублируют CRUD-UI.** Бот = login bridge + асинхронные уведомления / модерация-инбокс. Все CRUD-операции с треками идут в веб-приложении.

## Альтернативы рассмотренные

- **Email как равноправный login-канал (magic-link или пароль).** Отвергнуто пользователем: email/телефон — recovery-only.
- **Bot-only application, веб read-only browsing.** Отвергнуто: лишило бы веб-каталог полноценного управления треками, заставило дублировать UI в чате, ограничило просмотр карты в чате.
- **Telegram Login Widget / Max OAuth Widget (OAuth-like на сайте).** Отвергнуто: два разных виджета вместо одного паттерна; Telegram-Login-Widget требует публичного домена, плохо для self-hosted без DNS; не покрывает Max единообразно.
- **Two parallel identity systems (bot-first + email-first в разных таблицах).** Отвергнуто: множит двойные аккаунты и рассинхрон.

## Последствия

- Положительные: один auth-паттерн для двух ботов; server-side session — CSRF-protected; веб = first-class приложение без платформенных виджетов; low фишинг-risk (нет email-формы login).
- Отрицательные: два round-trip на вход (bot `/start` → клик по ссылке); спам-risk на token (нужен rate-limit).
- Recovery-flow (email/phone activation) — отдельный moderated flow с верификацией; вне основного login.
