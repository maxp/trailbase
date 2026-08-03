# TrailBase

## Источники решений

Перед изменениями прочитай относящиеся к задаче разделы:

1. `docs/IMPLEMENTATION-CONTRACT.md` — принятые инварианты; имеет приоритет.
2. `docs/ROADMAP.md` — границы, зависимости и acceptance criteria срезов.
3. `docs/adr/` — мотивация архитектурных решений.

Не разрешай противоречия и пробелы в продуктовых решениях молча. Укажи точное место
неопределённости и запроси решение, если от него существенно зависит реализация.

## Границы работы

- Реализуй только запрошенный вертикальный срез и соблюдай зависимости roadmap.
- Не добавляй будущие возможности и не меняй принятый стек попутно.
- Не дублируй детали реализации здесь: они принадлежат Implementation Contract.
- Сохраняй несвязанные пользовательские изменения в рабочем дереве.
- При изменении принятого решения обнови Implementation Contract и связанный ADR или
  roadmap в той же задаче.

## Безопасность

- Не коммить и не логируй secrets, raw auth/session tokens и чувствительные payloads.
- Для изменения поведения добавляй или обновляй проверку, воспроизводящую это поведение.

## Завершение задачи

- Проверь acceptance criteria активного среза и выполни релевантные тесты.
- После появления project tasks используй `bb ci` как итоговую локальную проверку.
- Для миграций проверяй направления up и down на пустой базе.
- Не заявляй о завершении без свежего результата проверки; перечисли непройденные
  проверки и причину.
- Коммить и пушь только по явному запросу. Добавляй в коммит только файлы задачи и
  после push проверяй upstream и чистоту рабочего дерева.

## Engineering Principles

- Do not preserve backward compatibility unless explicitly required.
- Choose the simplest implementation that fully meets the current requirements.
- Prefer established, well-maintained libraries over custom implementations.
- Fix root causes, not symptoms.
- Recommend best practices, even when they require refactoring.
