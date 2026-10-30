# AgroBot-external — верификация аудита 2026-10-29 (проверка 2026-10-30)

**Тип:** «Санированный» публичный снапшот коммерческого Telegram-бота (TS/Node + Python-коллекторы), 508 файлов в дереве main.
**Ветки:** `main` @ `729565b` (2026-09-16, «Public sanitized snapshot for external development», единственный коммит, сирота-история) + несмерженная `origin/agrobot-sales-refactor-2026-10-18` (2 коммита, общего предка с main нет — merge-base пуст).
**Коммиты после аудита: нет** (`git log --all --since=2026-10-28 --oneline` — пусто; последний коммит в репо — 5822c71 от 2026-10-21 на ветке handoff).

---

## Пункт 1 — Прод-IP и прод-пути при заявлении о санитизации
**Вердикт: STILL_PRESENT** (масштаб прод-путей БОЛЬШЕ заявленного аудитом: 115 файлов, а не 2)

- **IP 34.46.98.209 — ровно 8 файлов** на main (`git grep -l '34\.46\.98\.209' main` → 8):
  - `docs/tz_addendum_v1.md:21` — `| Server production | `34.46.98.209:/home/u0_a566/agrobot/webapp/data/reviews.json` | ...`; `:22` — `| Server mirror (Vercel src) | `34.46.98.209:/home/u0_a566/reviews-radar-vercel/data/reviews.json` | ...`; `:35` — `**SP (Server Production)** — `34.46.98.209:/home/u0_a566/...`. Source of truth...`. Точное совпадение с аудитом.
  - Дубли в 7 файлах ревью: `docs/agent_reviews/2026-07-17_GLM52_review.md`, `docs/agent_reviews/2026-07-27_GLM52_scraping_tools_benchmark.{json,md}`, `webapp/docs/agent_reviews/2026-07-17_GLM52_review.md`, `webapp/docs/agent_reviews/2026-07-17_GLM52_tz_addendum_v1.md`, `webapp/docs/agent_reviews/2026-07-27_GLM52_scraping_tools_benchmark.{json,md}`.
- **Прод-пути `/home/u0_a566/...` — 115 файлов** на main (аудит называл только 2): `webapp/api/export_api.py:22-24` (`REVIEWS_FILE/LOG/QUALITY_DECISIONS_FILE = "/home/u0_a566/..."`) и `:39-40` (чтение `.env` по `/home/u0_a566/agrobot/core/...`), `webapp/scripts/source_common.py:12-13` (`SHARED_DIR/WEBAPP_DIR` дефолты `/home/u0_a566/agrobot/...`) — подтверждены; дополнительно `README.md:23` («Исходники перенесены с `agrobot-prod:/home/u0_a566/agrobot`»), systemd-юнит `webapp/scripts/reviews-daily-collect.service`, ~60 скриптов webapp/scripts, 41 файл ревью и др.
- `EXTERNAL_SNAPSHOT.md:6` — «Telegram user IDs and deployment-only paths have been redacted.» — противоречит фактам выше. Подтверждено.

## Пункт 2 — README ссылается на отсутствующий core/.env.example
**Вердикт: STILL_PRESENT**

- `README.md:16` — «3. Настройте переменные окружения по `core/.env.example` и используемому модулю.» — файла нет: `git ls-tree -r main | rg -i 'env|example'` → 0 совпадений (в дереве нет ни одного .env* / *example*).
- `.gitignore:40` — `!**/.env.example` (белый лист для файла, которого нет; .gitignore:4 игнорирует `.env`, :38-39 — `*.pem`/`*.key`).
- Список переменных восстановим из кода (все 8 подтверждены):
  - `BOT_TOKEN` — напр. `core/artifacts/api-server/send-menu-cli.ts` (`new Bot(process.env.BOT_TOKEN!)`); `GEMINI_API_KEY` — `core/index-root.mjs` и др.; `DATABASE_URL` — `core/lib/db/src/index.ts`, `drizzle.config.ts`; `ADMIN_ID` — `core/artifacts/api-server/src/bot/index.ts:35-36` (`(process.env["ADMIN_ID"] ?? "").split(",")`), `sales_menu.ts:107`, `grill.ts:50`; `YOUTUBE_API_KEY` — `reviews_radar.sources.ts`, `test_debug.ts`; `SESSION_SECRET` — только в справке бота `core/artifacts/api-server/src/bot/handlers/backup.ts:56` (шаблон .env в /help);
  - `API_ID`/`API_HASH` — `agro_reviews_bridge.py:35-36` (`os.environ.get("API_ID"/"API_HASH")`, Telethon-клиент).

## Пункт 3 — Внутренние и сторонние материалы в «публичном» снапшоте
**Вердикт: STILL_PRESENT** (число 62 уточнено: это все файлы .agents/skills; собственно Trail of Bits — 19 файлов)

- **Агентские ревью: ровно 115 файлов** (`git ls-tree -r main | rg 'agent_reviews' | wc -l` → 115): `docs/agent_reviews/` + полный дубль `webapp/docs/agent_reviews/`. **110 из 115 файлов — байт-в-байт дубли** (53 blob-хэша встречаются >1 раза; 2 хэша × 4 файла, 51 хэш × 2 файла).
- **Сторонние скиллы: 62 файла под `.agents/skills/`** (уточнение к аудиту): Trail of Bits `sharp-edges` — 19 файлов (`SKILL.md`, `agents/openai.yaml`, `assets/trail-of-bits-mark.svg`, 16 `references/*.md`), без поля license; `supabase-postgres-best-practices` — 36 файлов (frontmatter заявляет `license: MIT`, но LICENSE-файля нет); `evaluate-rag`, `gemini-api-dev`, `prompt-engineering` — 3 SKILL.md; 4 внутренних md (`reviews-radar-ops.md`, `reviews-radar-source-expansion.md`, `token_economy.md`, `workflow_and_research.md`). Ещё 2 файла скилла в `core/.gemini/skills/weather/`.
- **LICENSE-файла нет нигде** (`git ls-tree -r main | rg -i 'licen'` → 0), при этом `core/package.json:4` — `"license": "MIT"` и `README.md:3` — «Коммерческий Telegram-бот...». Противоречие подтверждено.
- **Черновик личной переписки:** `core/attached_assets/Pasted---1777416831163_1777416831194.txt` (109 строк) — обращение к клиентке («Наташ, не обязательно отвечать на всё текстом...»), внутренние детали продаж, Killer-фичи, продукция BERTHOUD RAPTOR 4240, показатели недобора урожая. Подтверждено.

---

## НОВОЕ (из аудита 2026-10-29): ветка origin/agrobot-sales-refactor-2026-10-18
**Вердикт: ПОДТВЕРЖДЕНО ЦЕЛИКОМ**

**(а) Содержимое ветки:** 2 коммита: `5b17a7e` (2026-10-18, «snapshot: current AgroBot candidate and scoped sales audit context») → `5822c71` (2026-10-21, «handoff: Natalia goal-driven autonomous stage with preserved pass9/pass10 checkpoints»). Сирота-ветка (merge-base с main отсутствует). Дерево — 6 файлов: `README.md`, `GOAL-TASK-20261021.md` (22 912 Б), `GOAL-STAGE-20261021.json` (451 Б), `PACKAGE.json` (2 702 Б), `agrobot-goal-stage-20261021.zip`, `agrobot-sales-refactor-source-20261018-v2.zip`.

**(б) IP/прод-пути/секреты на ветке:**
- `git grep '34\.46\.98\.209' origin/agrobot-sales-refactor-2026-10-18` → **0 совпадений** (по текстовым блобам; zip-блобы текстовым поиском не покрываются).
- Инспекция ZIP в памяти (без распаковки на диск; SHA256 обоих архивов сверены с манифестами — совпадают):
  - **`agrobot-goal-stage-20261021.zip` (6 035 650 байт ≈ 6 МБ): чист** — 0 совпадений по IP 34.46.98.209 / 35.209.212.217, путям `/home/u0_a566`, паттернам BOT_TOKEN=/GEMINI_API_KEY=/DATABASE_URL=/телеграм-токенам; включая вложенные `checkpoints/agrobot-zai-pass9-20261021.zip` (51 файл) и `pass10-20261021.zip` (67 файлов).
  - **`agrobot-sales-refactor-source-20261018-v2.zip` (2 527 285 байт, 553 файла): содержит прод-данные** — IP `34.46.98.209` в 4 файлах внутри (`docs/agent_reviews/2026-07-17_GLM52_review.md`, `2026-07-27_GLM52_scraping_tools_benchmark.{json,md}`, `docs/tz_addendum_v1.md` ×3 вхождения) и прод-пути `/home/u0_a566` в **36 файлах** внутри (в т.ч. `core/artifacts/api-server/src/lib/system-health.ts`, `reviews_radar.sources.ts`, `scripts/health_report.sh` и др.). BOT_TOKEN=/DATABASE_URL= внутри — только плейсхолдеры (проверено, замаскировано), реальных секретов не найдено.
- GOAL-TASK-20261021.md (текстовый блоб ветки): IP/телефоны/e-mail — 0; но см. NEW-2 ниже.

**(в) Размер zip-блоба:** `git ls-tree -l` → `agrobot-goal-stage-20261021.zip` = **6 035 650 байт (≈6 МБ)**, blob `bfb8415f...`; SHA256 `82feb0f1…b9b` совпадает с заявленным в `GOAL-STAGE-20261021.json` и README ветки. Второй архив — 2 527 285 байт, SHA256 `9e586496…e10` совпадает с `PACKAGE.json`.

## Новые находки

### NEW-1 (100%): Прод-IP и прод-пути доставлены в публичный репо ВНУТРИ ZIP на ветке handoff — невидимы для текстового поиска
- Ветка публичная — сама задача ветки это заявляет: `GOAL-TASK-20261021.md`: «Открытый, ранее согласованный очищенный снимок: https://github.com/Sereban-glitch/AgroBot-external/tree/agrobot-sales-refactor-2026-10-18».
- В `agrobot-sales-refactor-source-20261018-v2.zip` (blob `e2f6438d…`) внутри: IP `34.46.98.209` в 4 файлах, `/home/u0_a566` в 36 файлах (см. выше). `git grep` по ветке даёт 0 — находка видна только при вскрытии архива. README ветки перечисляет исключённое («Ключи/пароли/env/credentials/история git/node_modules/личная переписка/клиентские звонки не входят») — IP и прод-пути в список не попали и фактически включены.

### NEW-2 (100%): Раскрытие внутренней инфраструктуры и идентичности владельца в публичной ветке
- `GOAL-TASK-20261021.md` (ветка): содержит ссылку на **приватный** репозиторий с точным blob-путём: `https://github.com/Sereban-glitch/project-materials/blob/489080f114bf1d951a2b40426ff4cc82483256d1/AgroBot/docs/NATALIA_PRODUCT_BRIEF.md`; имена (Сергей — владелец, Наталья — клиент), внутренние SHA256/HEAD чекпоинтов pass8/9/10, детали QA-дефектов.
- `webapp/scripts/backup_to_github.sh:125` (main): `echo "[backup] pr: https://github.com/Sereban-glitch/AgroBot/pull/new/$BRANCH"` — раскрывает имя **приватного** прод-репозитория `Sereban-glitch/AgroBot` (бэкап-скрипт, зашитый в «санитизированный» снапшот; туда же — прод-пути `/home/u0_a566`, ROOT_DIR по умолчанию, строка 9).

### Уточнение к П.1 (не новая, а коррекция масштаба)
- Прод-пути `/home/u0_a566` встречаются в **115 файлах** main (аудит указал 2), включая README.md:23 и systemd-юнит `webapp/scripts/reviews-daily-collect.service`.

---

## Не проверено
- Реальная доступность ветки/файлов на GitHub (сеть не использовалась; публичность ветки подтверждается текстом самого GOAL-TASK-20261021.md).
- Содержимое приватных репозиториев Sereban-glitch/project-materials и Sereban-glitch/AgroBot (доступов нет, по ограничениям аудита).
- Прогон скриптов/зипов (код не запускался; ZIP инспектированы в памяти только на уровне имён файлов и паттернов, значения секретов не выносились).
- История старого (до-снапшотного) репозитория — в локальном клоне отсутствует (main — единственный коммит-сирота).
