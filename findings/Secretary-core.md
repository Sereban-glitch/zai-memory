# Secretary-core — верификация аудита 2026-10-29

Проверка: 2026-10-30 (забег Z.ai). Работа только чтение, код не запускался.

## Коммиты после аудита: нет

HEAD = `edba7e8` (2026-10-18 06:15:36 +0000, «feat: full secretary core engine…»), ветка `main`, совпадает с `origin/main`. Последний коммит старше аудита на 11 дней.

## Пункт 1. Нет LICENSE — STILL_PRESENT

- Доказательство: `git ls-files | grep -i licen[cs]e|copying` → пусто (exit 1). В корне репо файла LICENSE нет (ls -la).
- README.md:119-120 — точное совпадение: «Лицензию на распространение отдельно определяет владелец; сейчас лицензия не назначена.»
- Противоречие с позиционированием подтверждено: README.md:3 — «Публичная чистая основа для внешнего агента.»

## Пункт 2. Импорт tests из продакшн-кода — STILL_PRESENT

- secretary/cli.py:45 — `from tests.fakes import FakeTransport` (внутри `_build_fake_engine`, без try/except — при отсутствии пакета tests будет ModuleNotFoundError).
- Документированная команда: README.md:84 — `python3 -m secretary.cli transcribe file.wav --fake-transport --markdown -o out.md`.
- examples/proxy_check.py:276 — `from tests.fakes import FakeTransport`.
- Дополнительно к аудиту: третья точка — examples/agrobot_orchestrator.py:100 — `from tests.fakes import FakeTransport`.
- Уточнение: `--fake-transport` прокидывается и в `serve` (secretary/cli.py:62-63: `if args.fake_transport: engine = _build_fake_engine(args)`), т.е. даже документированный HTTP-сервер зависит от тестового пакета.

## Пункт 3. Рассинхрон числа тестов — STILL_PRESENT (уточнён)

Статически в tests/: **280** методов `def test*` (rg -c по 19 файлам, сумма 280). Сверка с документами (5 документов, 5 разных чисел):

| Документ | Строка | Число |
|---|---|---|
| README.md | 82 | «246 тестов, офлайн» |
| OPERATOR-GUIDE-RU.md | 158 | «(246 тестов; медиатесты пропускаются без ffmpeg/ffprobe)» |
| docs/INTEGRATION.md | 96 | «(166 тестов; …» |
| docs/HISTORY.md | 39 | «Тестов стало 246 (было 239)» |
| HANDOFF.md | 10 | «266 тестов OK, 46 skipped» |
| HANDOFF.md | 30 | «После изменений 250 тестов прошли» |
| HANDOFF.md | 39 | «Проход 4 принят проверяющим (adcb62f, 198 тестов)» |
| HANDOFF.md | 277 | «198 тестов, все зелёные» |

Итого числа 166/198/239→246/250/266 при фактических 280 статических методах. Ни один документ не называет 280.

## Новые находки

1. **Нет CI — 280 тестов никогда не запускаются автоматически.** `.github/` отсутствует (ls; в `git ls-files` нет ни одного workflow-файла). На фоне рассинхрон чисел (П.3) это означает, что счётчики в доках дрейфуют без контроля. Уровень: 100% нужно (минимальный GitHub Actions `python3 -m unittest discover`).
2. **`--fake-transport` в `serve` открывает API без сети, но с тестовым кодом в рантайме.** secretary/cli.py:60-68 — `cmd_serve` принимает `args.fake_transport` и строит движок через `_build_fake_engine` (import из tests). Задокументированный прод-режим (`serve`, README.md:86) разделяет кодовую базу с тестовым транспортом: при pip-упаковке secretary без tests команда упадёт ModuleNotFoundError уже на старте сервера, а не только в CLI-транскрипции.

## Не проверено

- Фактический прогон тестов и реальное число собираемых unittest'ом кейсов (код не запускался — ограничение ТЗ); 280 — статический подсчёт `def test`.
- Количество skip'ов без ffmpeg (в HANDOFF — 46) — воспроизводится только прогоном.
- Состояние удалённого origin на GitHub (сеть не использовалась); свердка только по локальному клону.
