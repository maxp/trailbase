# Контракт: classification и tags

Часть [Implementation Contract](../IMPLEMENTATION-CONTRACT.md). При противоречии с другими документами действует этот контракт.

## 15. Classification и tags

- Primary activity enum остаётся:
  `hike`, `bike`, `run`, `ski`, `water`, `horse`, `motor`, `other`.
- Classifier показывает до трёх ranked suggestions с confidence и объяснением
  признаков. Пользователь обязан явно выбрать activity перед moderation.
- Difficulty хранится как `(difficulty_scale, difficulty_code)`; rank получается из
  централизованного справочника внутри соответствующей шкалы. Значение `unknown`
  разрешено.
- Season использует только четыре bits: spring, summer, autumn, winter.
  `year-round` означает все четыре; mask `0` означает unknown.
- Несколько выбранных сезонов объединяются по `ANY`; разные фасеты — по `AND`.
- Duration может быть unknown.
- Tag set относится к `track_revision_id`, а не к stable track, и входит в
  модерируемый snapshot.

