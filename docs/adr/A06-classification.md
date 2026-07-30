# ADR A06 — Модель классификации треков

**Status**: Accepted
**Date**: 2026-07-25

**Уточнение 2026-07-29:** tags относятся к immutable track revision; season использует
четыре bits без отдельного year-round bit; difficulty и duration допускают unknown.
Canonical duration выбирается moving → elapsed → unknown; manual override ограничен
целыми секундами `1..31_536_000`, а unknown хранится как `NULL`. Полный контракт:
[Implementation Contract](../IMPLEMENTATION-CONTRACT.md#15-classification-и-tags).

**Уточнение 2026-07-30:** activity, difficulty и tags остаются в TrailBase
track page/JSON API и не сериализуются в sanitized GPX `<trk><type>` или
`<metadata><keywords>`, поскольку interoperable vocabulary для taxonomy отсутствует.

## Контекст

TrailBase — каталог треков. Требуется модель классификации, которая: (а) поддерживает преgaps-навигацию по каталогу (пеш/вел/лыжи/вода/...); (б) экспонирует «окраску» трека (loop, scenic, overnight, family-friendly...); (в) даёт range-фильтр (продолжительность) и enum/bitmask-фильтр (сезон, трудность); (г) не скатывается в free-form-помойку через год; (д) автовыводит тип активности из GPX-профиля как подсказку. Пользователь добавил четыре фасета: **известные локации** (вынесены в A04 как отдельная сущность), **трудность, продолжительность, сезон**.

## Решение

1. **Иерархическая таксономия с модерацией.**
   - **Основной тип активности** (1 на трек): enum `activity_type ∈ {hike, bike, run, ski, water, horse, motor, other}`.
   - **Вторичные теги** из контролируемого модератором словаря: `loop`, `one-way`, `point-to-point`, `scenic`, `alpine`, `river`, `overnight`, `family-friendly`. Трек имеет 1 основной тип + N вторичных.
   - Теги модератор может пополнять. **Заявка тега от юзера** → модератор одобряет/отвергает → тег появляется в словаре.
2. **Структурные фасеты поверх таксономии** (таксономия остаётся «окрашивающей»):
   - **Difficulty** — наследует стандарт по виду активности:
     - hike: SAC Scale (T1–T6);
     - bike: MTB-Trail-Difficulty (S0–S5 / singletrack grade);
     - другие: 3-level enum `easy/moderate/hard` fallback.
   - **Season** — bitmask `{spring, summer, autumn, winter}`; year-round = все четыре,
     mask 0 = unknown.
   - **Duration** — integer seconds; source ∈
     `{gpx_moving, gpx_elapsed, manual, estimated, unknown}`. Default выбирается
     moving → elapsed → unknown; manual value находится в диапазоне
     `1..31_536_000`, unknown хранится как `NULL`, zero запрещён. Outlier comparison
     использует moving, иначе elapsed; `max(manual / auto, auto / manual) > 10`
     требует warning confirmation, но не блокируется. Без auto candidate warning нет.
     Durable acknowledgement привязано к manual value, comparison source/value и
     algorithm version; оно переживает resume/takeover и инвалидируется при изменении
     этих inputs или reparse. Moderator видит comparison и acknowledgement как
     informational flag без automatic reject или queue reprioritization и не меняет
     duration напрямую: correction после `changes_requested` делает автор в новом
     immutable revision.
3. **Auto-suggestion при загрузке** (предсказание рудиментарным классификатором на скорость+размах высот) — подсказка загрузчику; финальный выбор — человеком. Делается на профиле GPX, не server-side тяжёлым ML.
4. **GPX export** не вводит отдельное versioned mapping activity/difficulty/tags в
   свободные `<trk><type>` или `<metadata><keywords>`; canonical classification
   публикуется через track page и JSON API.

## Альтернативы рассмотренные

- **Жёсткий enum + scalar (без тегов).** Отвергнуто: заминает гибриды (одна trip — пешком + половина на великах по равнине), теряет стилистическую «окраску».
- **Free-form tags.** Отвергнуто: через год «пеш»/«пеший»/«пешком»/«walking»/«hike» в 5 вариантах; GIN/триграммный поиск не помогает без модерации.
- **Таксономия без enum** (визуальный тип — тег). Отвергнуто: теряет primary навигационный фасет, ломает UI лендинга (`activity=hike` сверху).

## Последствия

- Положительные: enum даёт первичный навигационный фасет (лента activity сверху лендинга); теги — расширяемость без free-form; модератор-управляемый словарь — единственный реалистичный способ держать словарь в открытом каталоге; duration = range-фильтр native.
- Отрицательные: один трек = один activity_type (гибриды приходится сводить к доминанте или through secondary tag), модераторское бремя на словаре (очередь заявок), миграция словаря тегов — schema с ростом.
- Difficulty per-activity-standard означает таблица `activity_type × scale` lookup; UI показывает шкалу по выбору активности.
