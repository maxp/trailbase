# TrailBase — Grill-me Checkpoint

Статус: paused
Дата: 2026-07-29

Принятые решения сохранены в
[Implementation Contract](IMPLEMENTATION-CONTRACT.md). Этот файл хранит только точку
продолжения design interview и не заменяет контракт.

## Последнее подтверждённое решение

Если после ACK/rotation search query завершается timeout/transient error, новый
callback state удаляется, а старый ID не восстанавливается. Прежний result content
остаётся видимым, но terminal edit убирает controls и предлагает повторить `/search`,
используя принятый provider retry policy.

## Следующий вопрос

Разрешать ли одному пользователю несколько одновременно незавершённых upload flows в
private chat?

Рекомендация: да, до уже принятого лимита трёх active upload/parse jobs, но без
неявного «текущего upload». Каждый status/control/metadata prompt связывается с
конкретным `upload_job_id`; свободный текст принимается только как reply на prompt
этого job, иначе bot просит выбрать upload.
