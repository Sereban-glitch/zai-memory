# bot_skaner_saitov — верификация аудита 2026-10-29

- Дата верификации: 2026-10-30 (забег Z.ai).
- Локальный клон: `repos/bot_skaner_saitov`, ветка `master`, HEAD `84288b2` (2026-10-19 04:05:51 +0300, «docs: link to Tubecook v4 video pipeline…»).
- Рабочее дерево чистое, соответствует HEAD. В дереве 19 файлов (README, bridge.py 70 КБ, скрипты отчётов, тесты, docs/superpowers).
- **Коммиты после аудита: нет.**

## П.1 — Нет LICENSE

**Статус: STILL_PRESENT**

- `git ls-files | rg -i 'license|copying'` — пусто, LICENSE не отслеживается и в дереве нет.
- `README.md:28-29` — «В дереве этой версии нет отдельного LICENSE. Права на использование и передачу требуют уточнения; английская инструкция ниже сохранена.» (строки точные, README сам признаёт проблему).

## П.2 — README ссылается на отсутствующие .env.example и start/stop-скрипты

**Статус: STILL_PRESENT**

- `README.md:56` — «1. Copy `.env.example` to `.env`.» — файла нет; более того, `.gitignore:11` прямо исключает `.env.example` из репозитория.
- `README.md:74-79` — блок `./start.sh`, `./restart.sh`, `./status.sh`, `./logs.sh`, `./healthcheck.sh`, `./stop.sh` — ни один файл не отслеживается; `.gitignore:33-38` перечисляет их под комментарием «Old scripts (utility, not tracked)».
- Итог: инструкция по установке из README невоспроизводима из клона — пользователь не может ни настроить env, ни запустить/остановить мост штатными командами.

## П.3 — Машинно-зависимые пути

**Статус: STILL_PRESENT (все строки точные)**

- `probe_telethon.py:21` — `config_dir = Path('/home/u0_a566/bot_skaner_saitov')` — абсолютный Termux-путь (плюс захардкоженный канал `@ludolov_zp` на `:29`).
- `drip_once.py:39` — `subprocess.run(["./start.sh"], cwd=DIR, capture_output=True)` — вызов отсутствующего в git скрипта.
- `drip_once.py:53` — `subprocess.run(["./.venv/bin/python", "./bridge.py", "--once"], …, timeout=280)` — путь к venv с точкой (у README-инструкции venv зовётся иначе: `python3 -m venv venv`, т.е. `./venv`, не `./.venv`).
- `drip_once.py:40` — `subprocess.run("crontab -l 2>/dev/null | grep -v drip_once | crontab -", shell=True)` — молотит пользовательский crontab.

## Новые находки

1. **Захардкоженный локальный ИИ-прокси с фиксированным ключом: `bridge.py:622-623`.** `:622` — `headers = {"Content-Type": "application/json", "x-api-key": "test"}`; `:623` — `session.post("http://127.0.0.1:18080/v1/messages", …)`. Эндпоинт и ключ не выносятся в env (в отличие от остального конфига: SOURCE_CHAT/TARGET_CHANNEL/RSS_* через env) — на любой другой машине генерация постов молча падает в «FALLBACK», а ключ «test» зашит в код. Модель по умолчанию тоже в коде: `bridge.py:616` — `os.environ.get("REPORT_AI_MODEL", "gemini-3.8-flash-tiered")`.
2. **Второй захардкоженный сервис: `bridge.py:1680`.** `await session.post("http://localhost:8083/telegram_job", json=payload)` — форвардинг постов из FREELANCE-каналов в локальный приёмник job'ов; URL не настраивается, ошибки глотаются (`except … print`) — тишина в логах при недоступности сервиса. В сумме с `drip_once.py` получается неотключаемая связка на localhost-сервисы, невидимая из README/env.

## Не проверено

- Наличие start.sh/.env/.venv на целевой (прод) машине — в клоне их нет, работу по месту не проверял.
- Рантайм-поведение bridge.py (код не запускался).
- Актуальность дефолтов FREELANCE_CHANNELS и списка переменных окружения на проде.
