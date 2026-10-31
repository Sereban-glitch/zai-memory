# Аудит montage-telegram-bot — субагент 2-a (новые находки)

Репозиторий: `/home/z/my-project/zai-audit/repos/montage-telegram-bot` (ветка main, 13 коммитов).
Метод: grep-паттерны (subprocess/shell=True/eval/queryRaw/drawtext/path join/env), затем полное чтение файлов и проверка каждой строки. Известные находки (токен бота, db/custom.db в git, Caddy :81, захардкоженный админ Go-API/CORS/500 МБ, .env, бинарник 10,6 МБ, IP/TG-ID, LICENSE) — НЕ повторяются.

Контур развёртывания, важный для понимания находок: Next.js — статический экспорт (`next.config.ts:4 output:"export"`, `src/app/api/**` не существует); API обслуживает Go-сервис (127.0.0.1:8445), бот создаёт задачи напрямую в SQLite, воркеры — `services/worker/worker.py` (compose/systemd) и `pull_worker.py` (ноутбук). Роуты Next.js API живут только в `services/api-routes-*-backup/` (v4/v5) и восстанавливаются копированием — это оговорено в самих директориях.

Итого: **15 новых находок** (P0: 1, P1: 4, P2: 8, P3: 2).

---

### [security] P0 — Path traversal при сохранении загруженного файла в Go-API
- Файл: services/api-go/main.go:293-296 (проверено чтением)
- `fName := fmt.Sprintf("%s_%s", genID("upload"), header.Filename)` → `fPath = filepath.Join(mediaDir, "uploads", fName)` → `os.Create(fPath)` + `io.Copy(outF, file)`
- `header.Filename` полностью контролируется клиентом (multipart). `filepath.Join` только «чистит» путь, `../` в имени файла остаются рабочими: имя `../../../../public/showcase/x.mp4` пишет файл за пределы uploads — в ./public (отдаётся Caddy/Next как статику), в `../db/custom.db` или дальше по ФС с правами процесса. Роут неавторизован (см. известные), т.е. любой, кто достучится до API, получает произвольную запись файла с собственным содержимым.
- Фикс: имя генерировать на сервере (`genID + безопасное расширение`), от исходного имени брать только `filepath.Base` и вырезать `../`, писать в фиксированный каталог.

### [security] P1 — SSRF/LFI: sourceUrl/filePath без валидации уходит в yt-dlp и shutil.copy2
- Файлы: services/worker/app/pipeline.py:92-99 (`cmd = ["yt-dlp", ..., url]`, url = `job.source_url` без проверки схемы); services/worker/worker.py:91-96 (то же) и worker.py:99-101 (`shutil.copy2(job["filePath"], input_file)` — путь из БД); приёмник URL — services/api-go/main.go:281,313-315 (sourceUrl принимается любым, JSON и multipart)
- yt-dlp принимает не только http(s): `file:///etc/passwd`, локальные пути и внутренние адреса (`http://127.0.0.1:8445/...`, `http://169.254.169.254/...`). Источник URL — любой пользователь бота (регекс `https?://` в чате, bot/main.py:372) или неавторизованный POST /api/jobs. Ответ внутреннего сервиса попадает в текст ошибки (`proc.stderr[-600:]`, pipeline.py:101 / worker.py:98) → поле Job.error → бот отправляет его в чат пользователю (bot/main.py:652-661), т.е. есть канал эксфильтрации. `filePath` также читается без ограничений.
- Фикс: белый список схем http(s), запрет приватных/loopback IP (resolver + проверка после редиректов), filePath — только внутри uploads c canonical-path проверкой.

### [security] P1 — ffmpeg filter injection через пользовательский `title` в воркере (экзофилт в публичную обложку)
- Файл: services/worker/worker.py:84 (`title = params.get("title", ...)`), worker.py:152-158 (`-vf ...drawtext=font='DejaVu Sans':text='{title}'...`)
- `title` контролируется клиентом (Go-API мержит произвольные ключи params — main.go:252-272, мини-апп шлёт свои поля) и вставляется в фильтр без экранирования — в отличие от pull_worker.py:218 (`replace("'", "")...`) и tts_narrator.py:311-312 (`_drawtext_escape`). Одинарная кавычка/двоеточие в title позволяют выйти из аргумента drawtext и добавить свои фильтры, в т.ч. `drawtext=textfile=/app/.env` — содержимое файла отрисуется в thumbnail, который worker.py:165-169 кладёт в `public/showcase/{job_id}.png`, публично отдаваемый вебом.
- Фикс: экранировать как в tts_narrator._drawtext_escape (или передавать текст через временный файл/`textfile` с контролируемым путём).

### [security] P1 — Фальсифицируемые сессии: HMAC-секрет с захардкоженным fallback
- Файл: src/lib/telegram.ts:47-58 — `const SECRET = process.env.AUTH_SECRET || 'montage-kitchen-dev-secret'`; `signSession`/`verifySession` подписывают cookie `userId.mac` (обрезок HMAC-256 до 128 бит), используется src/lib/session.ts:7-27
- AUTH_SECRET нигде не задан (.env — только DATABASE_URL; docker-compose webapp — тоже). Любой, знающий публичный строковый секрет из репо, куёт cookie любого userId, включая админского (`u_admin`), и получает полный доступ к admin-роутам v4/v5-бэкапа (`services/api-routes-v5-backup/admin/**` проверяют только роль из сессии). Сейчас роуты выведены из сборки (static export), но лежат в репо как «backup» для восстановления — включение их одной копипастой активирует обход авторизации.
- Фикс: убрать fallback, падать при отсутствии AUTH_SECRET, сверять MAC timing-safe, ротация секрета.

### [security] P1 — Чтение произвольного локального файла ботом через resultUrl из БД
- Файл: services/bot/main.py:633-636 — `if res_url and res_url.startswith("/demo/"): local_file = PUBLIC_DIR / ... elif res_url and Path(res_url).exists(): local_file = Path(res_url)` → `bot.send_video(..., FSInputFile(str(local_file)))` (639-644)
- `resultUrl` записывается в БД неавторизованным `POST /api/worker/complete` (main.go:444-467; отсутствие auth — известная находка, но здесь новое следствие): атакующий создаёт свою задачу, затем complete с `resultUrl=/app/db/custom.db` (или `/etc/passwd`, `/app/.env`) — нотификатор бота отправит этот файл в Telegram-чат владельца задачи, т.е. атакующему. Проверок «файл принадлежит задаче» нет.
- Фикс: принимать только `/demo/<job_id>.mp4` внутри PUBLIC_DIR, остальное игнорировать (плюс авторизация worker-роутов).

### [security] P2 — GET /api/jobs отдаёт метаданные ВСЕХ задач без авторизации; DELETE отменяет чужие задачи
- Файл: services/api-go/main.go:231-246 (`SELECT ... FROM Job ORDER BY createdAt DESC LIMIT 30` — без фильтра по пользователю; в выборку входят sourceUrl, filePath, params (с youtube_meta), error) и main.go:370-374 (`UPDATE Job SET status='cancelled' WHERE id=?` — без проверки владельца)
- Любой клиент Mini App (а роут не требует даже initData) читает ссылки/файлы/ошибки всех пользователей и может отменять их задачи. Известная находка про отсутствие middleware покрывает admin/worker-роуты; здесь — кросс-пользовательская утечка данных и DoS очереди именно на jobs-роутах.
- Фикс: фильтр `WHERE userId=?` от сессии, ownership-проверка в DELETE.

### [security] P2 — Демо-вход администратором в backup-роутах авторизации (+ нет проверки auth_date)
- Файлы: services/api-routes-v5-backup/auth/route.ts:51-55 (ветка «демо-режим» логинит первым `role:'admin'` пользователем, если нет initData ИЛИ не задан TELEGRAM_BOT_TOKEN — условие `if (initData && botToken)`); docker-compose.yml:13-15 (webapp не получает TELEGRAM_BOT_TOKEN → в compose-контуре демо-ветка срабатывает всегда); src/lib/telegram.ts:15-41 (`auth_date` парсится, но свежесть не проверяется ни одним вызовом — реплей initData бессрочный)
- Восстановление v4/v5-роутов (их назначение по имени каталога) даёт полный админ-доступ любому посетителю без Telegram. Это тот же класс, что известная hardcoded-админка Go-API, но другой код и другой вектор (Next.js-роуты), поэтому фиксить нужно отдельно.
- Фикс: убрать демо-ветку из прод-сборки (env-флаг), проверять `auth_date` (напр. ±24 ч), требовать botToken.

### [security] P2 — Path traversal при сохранении аплоада ботом (имя файла от Telegram)
- Файл: services/bot/main.py:262-263 — `save_path = MEDIA_DIR / "uploads" / f"{uuid.uuid4().hex}_{file_name}"`; `await bot.download_file(file_info.file_path, destination=save_path)`
- `file_name` — клиентская строка (`getattr(video, "file_name", ...)`, bot/main.py:256), aiogram её не sanitize: имя `../../../../public/showcase/evil.mp4` (лишние `../` компенсируют uuid-префикс) пишет файл за пределы `/app/media/uploads` — в т.ч. в примонтированный `./public` (статика сайта) или `./db`. Квоты/блокировки на этапе загрузки тоже нет (проверка dailyUsed только в callback setup: bot/main.py:303) — любой пользователь бота может забивать диск файлами до 2 ГБ.
- Фикс: `Path(file_name).name` + генерируемое имя, лимит размера/частоты на входе хендлера.

### [security] P2 — RSS-источник domdara по HTTP (MITM → автоконтент и HTML-инъекция в Telegram)
- Файлы: services/worker/app/config.py:36 (`domdara_feed_url: str = "http://domdara.org/feed/"`); src/lib/seed.ts:137; scripts/news/domdara_parser.py:12
- Фид и статьи забираются по открытому HTTP с `follow_redirects=True` (domdara_rss.py:165-168): активный MITM подменяет тексты/заголовки, из которых автоматически генерируется news-Short и (после одобрения) выгружается на YouTube-канал владельца; заголовки статьи без экранирования попадают в сообщения с `parse_mode="HTML"` (notify.py:107-113, approval_text:62-73) — HTML/ссылочная инъекция в чат модерации.
- Фикс: https:// для фида (+ pin/проверка сертификата), экранировать HTML в notify.

### [bug/repro] P2 — worker/app/* импортирует несуществующий пакет `backend` — «новый» воркер не запускается
- Файлы: services/worker/app/main.py:36-37 (`sys.path.insert(0, .../services/"backend")` → `from backend.app.models import ...`), app/db.py:10-12, app/pipeline.py:80; каталог services/backend в репо отсутствует (проверено LS)
- `python -m app.main` (заявленный в docstring) падает ImportError; кроме того Go-схема Job (main.go:109-131) не содержит колонок `priority`/`approval_status`, которыми оперируют claim_next (app/main.py:65-78) и цикл модерации (app/main.py:203-292). Код создаёт ложное впечатление работающего конвейера Zero-Conflict/Human-in-the-Loop, тогда как реально исполняется только worker.py.
- Фикс: вернуть/вынести модуль backend и миграции схемы либо явно пометить app/* как несобираемый и удалить из доков.

### [bug] P2 — Кнопки модерации и URL-задач бота не имеют обработчиков
- Файлы: services/bot/main.py:393,399 (кнопки `callback_data="urljob:short:{...}"` — обработчика `urljob:` нет в списке хендлеров, проверено grep'ом по `@dp.callback_query`); notify.py:83-88 (клавиатуры `approve:`/`regen:`/`reject:` — ботом не обрабатываются, значит approval_status из app/main.py:203-292 никогда не выставится); bot/main.py:497-506 (`ai_hooks:` только печатает сообщение, задачу не создаёт); bot/main.py:438 (approve_news читает `/home/qwerty/projects/court-montage/token.json` — хост-путь, в контейнере отсутствует) + services/bot/requirements.txt (нет google-api-python-client — импорт в bot/main.py:434 упадёт)
- Пользовательские сценарии из UX (смонтировать по ссылке из чата, одобрить news-Short) молча не работают; «старый» approve_news гарантированно падает в compose-контуре.
- Фикс: добавить хендлеры `urljob:`/`approve:`/`regen:`/`reject:` (или убрать кнопки), перенести YouTube-OAuth в env воркера.

### [bug/security] P2 — Прод-контейнер фронтенда запускает dev-сервер; деплой-юниты противоречат сборке; всё от root
- Файлы: Dockerfile:23 (`CMD ["npm", "run", "dev"]` при `NODE_ENV=production`); deploy/montage-webapp.service:12 (ExecStart на `.next/standalone/server.js`, но next.config.ts:4 — `output: "export"`, standalone не генерируется); все Dockerfile'ы без `USER` (root); Dockerfile:17 `COPY . .` запекает в образ `db/custom.db` (ПДн), `.env` и бэкапы роутов — при пуше образа утекут вместе со слоями
- Dev-сервер в проде отдаёт стек-трейсы и не оптимизирован; systemd-юнит несовместим с конфигом — деплой по DEPLOY.md не воспроизводится; root-контейнеры увеличивают радиус поражения при любом из найденных RCE-классов (drawtext/yt-dlp/traversal).
- Фикс: `next build && next start` (или standalone), USER node, .dockerignore для db/.env/services.

### [security] P3 — Prisma-клиент логирует все SQL-запросы с параметрами
- Файл: src/lib/db.ts:9-11 — `new PrismaClient({ log: ['query'] })` без условия на окружение
- В логи контейнера попадают запросы с параметрами (telegramId, username и пр.) — утечка ПДн в stdout/логи. Сейчас слой БД фактически мёртв (static export), но включается вместе с восстановлением роутов.
- Фикс: `log: process.env.NODE_ENV === 'development' ? ['query'] : ['error']`.

### [security] P3 — Markdown-инъекция в сообщениях бота через имя пользователя
- Файл: services/bot/main.py:186-199 (и аналогичные 405-413, 358-366) — `f"👋 **Привет, {message.from_user.first_name}!**"` c `parse_mode="Markdown"`
- `first_name` в Telegram контролируется клиентом: символы `*`, `_`, `[` ломают разметку и позволяют вставить свои ссылки/форматирование в ответы бота (кликджек-обман пользователя).
- Фикс: экранировать Markdown или уходить на plain text для пользовательских строк.

---
## Проверено и НЕ найдено (для полноты)
- Инъекций через Prisma `$queryRaw/$executeRaw` нет (grep по src — 0 совпадений; работа через типизированные вызовы).
- `eval/exec/pickle/yaml.load/marshal` в Python — нет (только безопасные `subprocess`-вызовы списками; единственный `shell=True` — engine.py:220-227, но интерполируется только серверный путь конфига внутри media_dir, job_id генерируется кодом — вектора через пользователя нет).
- XSS в React-рендеринге — `dangerouslySetInnerHTML` отсутствует.
- История git: кроме уже известного токена бота в pull_worker/domdara_to_short, других секретов в коммитах нет (проверены все 13 коммитов, .env всегда contained только DATABASE_URL).
- examples/websocket — учебный socket.io с CORS `*` и без auth (порт 3003), в прод-контур не подключён; отмечаю как потенциальный риск при деплое.

## Топ приоритетов
1. P0 path traversal (main.go:293) — произвольная запись файла неавторизованным клиентом.
2. P1 SSRF/LFI через sourceUrl→yt-dlp + эксфильтрация через Job.error→чат бота.
3. P1 ffmpeg filter injection через title (worker.py:156) с выходом в публичную /showcase-обложку.
