# Контракт: Mini Apps Telegram и MAX

Часть [Implementation Contract](../IMPLEMENTATION-CONTRACT.md). При противоречии с
другими документами действует этот контракт.

## Обязательные поверхности

- TrailBase реализует Mini App внутри Telegram и MAX. Обычный browser web UI остаётся
  first-class интерфейсом; Mini Apps не заменяют chat flows и не делают browser
  activation обязательной для операций, доступных в private chat.
- Telegram Mini App и MAX Mini App используют одну информационную архитектуру, порядок
  основных разделов, терминологию, domain operations и server-rendered UI. Для одной
  возможности не создаются независимые provider-specific экраны или бизнес-правила.
- Возможность, добавляемая в authenticated web UI, считается реализованной только после
  проверки её основного пользовательского сценария в обычном браузере, Telegram Mini
  App и MAX Mini App, если milestone явно не ограничивает поверхность.

## Provider adapters и интерфейс

- Различия Telegram и MAX изолированы в двух тонких client adapters над официальными
  Mini App bridge API. Domain services, HTTP handlers, routes, schemas и permissions
  общие для обеих Mini Apps и обычного web UI.
- Общий UI mobile-first и визуально максимально одинаков в обеих Mini Apps. Adapter
  учитывает provider theme/color scheme, viewport и safe-area изменения, системные
  Back/Main controls, confirmation закрытия, haptic feedback и правила открытия
  внутренних, messenger и внешних ссылок только там, где соответствующая возможность
  существует в текущем provider API.
- Provider-native control используется, когда он передаёт ту же семантику действия.
  При отсутствии возможности adapter показывает эквивалентный web control внутри Mini
  App. Основной пользовательский сценарий не исчезает только из-за различий SDK;
  необязательное косметическое поведение может деградировать без ошибки.
- Неизвестная или неподдерживаемая версия bridge API получает совместимый базовый UI без
  provider-native enhancements. Отсутствие bridge object означает обычный browser mode,
  а не попытку доверять параметрам URL.
- Provider SDK assets загружаются только на Mini App entry routes и только с официальных
  Telegram/MAX origins из закрытого CSP allowlist. Остальные страницы сохраняют
  local-assets-only policy; произвольные third-party scripts запрещены.

## Запуск, identity и session

- Mini App принимает raw Telegram `initData` или MAX `WebAppData` и передаёт его backend
  без интерпретации как доверенного identity state. Backend использует отдельный
  provider validator, проверяет подпись по официальному алгоритму, обязательные поля,
  provider/bot binding и возраст `auth_date`: не старше 10 минут и не более чем на 30
  секунд в будущем по injected application UTC clock. Parsed/unsafe client object не
  используется для authentication или authorization.
- Успешная валидация сопоставляет provider user ID только с active
  `user_identities(provider, provider_user_id)`. Mini App не создаёт account или
  identity: неизвестному, unlinked либо inactive пользователю session не выдаётся и
  показывается provider-native переход в private chat для обычного `/start` flow.
- Валидированный запуск создаёт или заменяет session этого Mini App webview по общему
  browser-session lifecycle и выдаёт тот же защищённый cookie и CSRF state, что обычный
  web UI. Telegram и MAX launch payload не хранятся в session и не попадают в logs,
  metrics, traces, analytics или DLQ.
- Mini App launch не является fresh authentication: созданная session имеет
  `fresh_authenticated_at = null`. Sensitive operations используют существующий
  explicit fresh-auth flow через private one-to-one chat; launch payload, недавний
  запуск Mini App и provider-native biometric/device state его не заменяют.
- Повторная отправка одного launch payload и параллельные старты идемпотентны в пределах
  окна допустимого `auth_date`: они не создают несколько активных sessions и
  notifications. Payload с неверной подписью, binding или возрастом завершается
  fail-closed без session и domain changes.

## Проверки

- Contract tests используют отдельные Telegram и MAX fixtures для valid, expired,
  malformed, duplicate и wrong-bot signed launch data; raw payload и bot token не
  выводятся в failure messages.
- End-to-end проверка каждой Mini App включает launch, session/CSRF, theme и viewport
  adaptation, back/navigation behavior, основной state-changing сценарий и fallback
  при недоступной provider capability.
