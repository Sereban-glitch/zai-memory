# agy-claw-bot — верификация аудита 2026-10-29 (сверка 2026-10-30)

Репозиторий: Sereban-glitch/agy-claw-bot (локальный полный клон, HEAD `682a9b5` от 2026-06-18).
Коммиты после аудита: нет.

---

## П.1 Быстрый старт падает: имя токена, dotenv, ADMIN_CHAT_ID — STILL_PRESENT

- `README.md:25` — `cp .env.example .env  # заполнить TELEGRAM_BOT_TOKEN, GEMINI_API_KEY`; `.env.example:5` — `TELEGRAM_BOT_TOKEN=`.
- `bot.js:14-15` — читается другое имя: `const BOT_TOKEN = process.env.BOT_TOKEN;` / `if (!BOT_TOKEN) { console.error('❌ BOT_TOKEN environment variable not set'); process.exit(1); }` → копирование по README даёт немедленный exit(1).
- dotenv не подключён: bot.js:1-9 (telegramify-markdown, child_process, http, https, fs, path, os, node-cron) и package.json:12-18 — пакета dotenv нет → файл `.env` вообще не загружается.
- `ADMIN_CHAT_ID` из `.env.example:11` кодом не читается: `bot.js:54` — `const ALLOWED_ADMIN_ID = 745130167;` (зашит), `bot.js:702` — `const ADMIN_CHAT_ID = ALLOWED_ADMIN_ID;`.
- Дополнительно (усиливает): `.env.example:14 GEMINI_MODEL` и `:17 MAX_ITERATIONS` тоже мёртвые — bot.js:18 `DEFAULT_MODEL = 'gemini-2.5-flash'` и bot.js:102 `MAX_AGENT_ITERATIONS = 10` зашиты, env не читаются.

## П.2 Зависимости с ^, нет lock-файла, npm install в Docker — STILL_PRESENT

- `package.json:12-18` — пять зависимостей с `^`: node-cron `^4.2.1`, node-telegram-bot-api `^0.67.0`, pdf-parse `^2.4.5`, pdfkit `^0.19.0`, telegramify-markdown `^1.3.3`.
- `package-lock.json` в репо отсутствует (проверено ls/git).
- `Dockerfile:12` — `RUN npm install --production` (без lock `npm ci` упал бы, но и `npm install` невоспроизводим).

## П.3 Авторизация только в одном хендлере; команды без проверки; слабый блэк-лист — STILL_PRESENT

- Единственная проверка — `bot.js:554-556` внутри `bot.on('message')` (объявление хендлера :549): `if (userId !== ALLOWED_ADMIN_ID) { bot.sendMessage(… '⛔ Доступ запрещен…'); return; }`.
- `/del` — `bot.js:474-494`, без авторизации: путь извлекается из текста reply-сообщения (`/сохранен:\s*([^\n]+)/`, :478) и `fs.unlinkSync(filePath)` (:483) — любой пользователь бота может удалять файлы.
- `/clear` — `bot.js:508-512` (очистка memory.json любого юзера), `/model` — `bot.js:514-532`, `callback_query` — `bot.js:535-546` — все без проверки ID.
- Блэк-лист execBash — `bot.js:106-111`: `BLOCKED_COMMANDS = [/^rm\s+-rf\s+\//, /^sudo\b/, /^kill\b/, …]` — все паттерны привязаны к началу строки; байпас тривиален (`cd / && sudo …`, `echo x; rm -rf /`, `bash -c '…'`), исполняется через `spawn('bash', ['-c', command])` (:192).
- `README.md:77` — «This bot acts as a remote shell.»
- `soul.md:3` — публично раскрывает имя/юзернейм бота: «общаешься с создателем через Telegram-бота @Console_bot».

## Новые находки

1. **КРИТИЧНО: удалённый Gemini API-ключ остался в git-истории.** Коммит `01ad494` «security: remove hardcoded Gemini API key…» убрал ключ только из рабочего дерева `knowledge_indexer.py`, но ключ полностью извлекаем из истории (`git show 67b670c:knowledge_indexer.py` — строка с ключом формата `AIzaSy…` на месте; ключ присутствует в `b0a551e` и `67b670c`, внесён уже в initial commit). Значение ключа в отчёт не выносится. Обычное «удаление в новом коммите» не чистит историю публичного репо → нужен revoke/rotate ключа и чистка истории (git filter-repo / BFG) + проверка билдов GHCR, где слой мог попасть в образ.
2. **Хардкод Termux-путей `/home/u0_a566` ломает заявленный Docker-деплой.** `bot.js:193-194` — bash-инструмент исполняется с `cwd: '/home/u0_a566/console-bot'` и `HOME: '/home/u0_a566'`; `bot.js:591` — фото пишутся в `/home/u0_a566/tg_images`; `bot.js:632` — knowledge_base в `/home/u0_a566`. В образе (Dockerfile: node:20-alpine, WORKDIR /app) каталога `/home/u0_a566/console-bot` нет → `spawn` с несуществующим cwd падает ENOENT — главный инструмент («Claw») неработоспособен в Docker/GHCR-деплое. Пути нужно вынести в env/конфиг. Плюс soul.md:8-9,27-28 публично раскрывает внутреннюю топологию файловой системы.
3. Мелочи: `soul.md:14` раскрывает характеристики сервера («Debian Linux, Node.js v24.16.0, 8GB RAM»); `bot.js:48` — `fs.writeFileSync(memory.json)` синхронно на каждое сообщение (блокировка event loop); `.env.example` обещает GEMINI_MODEL/MAX_ITERATIONS, которые код не читает (см. П.1).

## Не проверено

- Runtime-поведение (запуск запрещён методикой): фактический эффект ENOENT для spawn в alpine-контейнере — вывод по коду/докам Node.
- Не проверялось, скомпрометирован ли ключ на момент сверки (обращения к Google/Gemini API не выполнялись; проверка только по git-истории).
- Внешнее состояние репо (issues/PR, загрузки GHCR-образа) — не проверялось (api.github.com не использовался).
