# TrailBase — Grill-me Checkpoint

Статус: paused
Дата: 2026-07-31

Принятые решения сохранены в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). Этот файл хранит только точку
продолжения design interview и не заменяет контракт.

## Последнее подтверждённое решение

`track_issues.code` хранится как `text NOT NULL` с DB CHECK на
`raw_object_missing`, `raw_integrity_mismatch`, `sanitized_export_missing` и
`snapshot_integrity_unknown`, без PostgreSQL enum. Unknown code отклоняется; новый
code требует migration существующего CHECK вместе с checker, subject constraint и
capability mapping.

## Следующий вопрос

Должна ли БД отдельным CHECK контролировать допустимые пары
`track_issues.code`/`subject_type`?

Рекомендация: да. `raw_object_missing` и `raw_integrity_mismatch` допускают только
`raw_object`; `sanitized_export_missing` и `snapshot_integrity_unknown` — только
`revision`. Начальный набор не допускает row с `subject_type = track`; этот тип
зарезервирован для будущего track-wide code, который одной migration расширит оба
CHECK. Так ошибка application mapping не сможет записать issue к неверному subject.
