# TrailBase — Grill-me Checkpoint

Статус: paused
Дата: 2026-07-30

Принятые решения сохранены в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). Этот файл хранит только точку
продолжения design interview и не заменяет контракт.

## Последнее подтверждённое решение

Raw GPX не шифруется и не преобразуется приложением. Private S3 object хранит exact
original upload bytes; data/master keys, crypto envelope, keyring, rewrap и decrypt
path для raw отсутствуют. `raw_objects` сохраняет только storage/lifecycle metadata,
quota/reference/cleanup model остаётся без изменений.

## Следующий вопрос

Допустимо ли transparent provider-side SSE или disk encryption для raw storage?

Рекомендация: да, как deployment control. TrailBase всё равно записывает и читает
exact original bytes и не знает о шифровании диска/S3 provider; это не возвращает
application envelope, keys или decrypt path. Если требование «без шифрации» означает
запрет любого at-rest encryption, нужно явно запретить SSE/disk encryption и в
infrastructure configuration.
