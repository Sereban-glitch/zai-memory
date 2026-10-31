# Аудит proakt360-web — ветка origin/v2.2-server-sync (Task 2-c)

Дата: 2025. Аудитор: subagent 2-c. Метод: `git worktree add ../p360-v22 origin/v2.2-server-sync` (HEAD 0841395), чтение файлов, без запуска кода, без сети, без коммитов.
Известные находки №1–6 (act-text без requireMaster, telegram webhook без секретного токена, backup/restore защищён, LICENSE, z-ai-web-dev-sdk и т.д.) — НЕ повторяются, только сверены.

---

## НОВЫЕ НАХОДКИ

### [P0][auth] Обход мастер-аутентификации через Telegram initData при незаданном BOT_TOKEN
- Файл:строка (проверено): `src/db/auth.ts:205` (fallback на `""`), `src/db/auth.ts:171` (HMAC от botToken), `src/db/auth.ts:158-182` (verifyTelegramInitData — нет проверки пустого botToken), `src/db/auth.ts:203-209` (checkMaster принимает telegram-ветку)
- Цитата: `const botToken = process.env.TELEGRAM_BOT_TOKEN || process.env.BOT_TOKEN || "";` → `verifyTelegramInitData(tgInitData, botToken)`
- Почему чинить обязательно: `.env.example:25` документирует TELEGRAM_BOT_TOKEN как ОПЦИОНАЛЬНЫЙ (закомментирован) — т.е. дефолтное состояние продакшена «пароль задан, бота нет». При пустом botToken секрет подписи = `HMAC-SHA256("WebAppData", "")` — публично вычислимая константа. Атакующий сам генерирует `initData` (user JSON произвольный) с корректным hash, шлёт заголовок `x-telegram-init-data` → `checkMaster()` возвращает `{ok:true, via:"telegram"}` → requireMaster пропускает. Это полный обход fail-closed дизайна (auth.ts:15-20): ВСЕ мастерские роуты (чтение/запись/удаление, backup/restore, экспорт с ценами) доступны анонимно даже при установленном пароле.
- Фикс: в `verifyTelegramInitData` — ранний `return {valid:false}` при `!botToken` (fail-closed по аналогии с паролем); заодно проверять свежесть `auth_date` (см. P2-5).

### [P1][auth] Любой Telegram-пользователь = мастер (нет allowlist)
- Файл:строка: `src/db/auth.ts:207-208` — `if (tg.valid) return { ok: true, via: "telegram", user: tg.user }`
- Цитата: проверяется только подпись, id пользователя не сверяется ни с чем.
- Почему чинить: даже с корректным bot token любой, кто открыл Mini App (публичная /start-ссылка бота ProAktSmetaBot), получает права мастера на все данные. Бот по сути раздаёт мастер-доступ.
- Фикс: env `PROAKT_TELEGRAM_ALLOWED_IDS` (список id через запятую) или поле в appSettings; при отсутствии списка — telegram-ветку отключить.

### [P1][auth] В демо-режиме пароль мастера ставится удалённо через PUT /api/settings (захват инстанса)
- Файл:строка: `src/app/api/settings/route.ts:42-57` — старый пароль запрашивается только если `s.passHash` непуст ( :44)
- Цитата: `if (s.passHash) { ... verifyPassword(oldPassword) ... }` → иначе сразу `setPassword(newPassword)` + `session.deleteMany({})`
- Почему чинить: при `PROAKT_DEMO_OPEN=1` (пароль не задан) requireMaster пропускает всех (demo), и ЛЮБОЙ удалённый пользователь одним PUT ставит свой пароль → становится мастером, выкидывает остальных. Публичный демо-инстанс (например, на sslip.io-домене из известной находки №4) захватывается в один запрос.
- Фикс: в demo-режиме блокировать блок смены пароля в API (403 «только локально через setup:password»), как уже сделано для первичной настройки.

### [P1][idor+XSS] GET /api/photo/[id] — legacy-SVG отдаётся без авторизации, caption без экранирования
- Файл:строка: `src/app/api/photo/[id]/route.ts:14-33` (GET без какого-либо guard; guard только на PATCH :36 и DELETE :105); инъекция: `:28` — `${photo.caption || 'фото'}` в SVG без экранирования; `:31` — `Cache-Control: public`
- Аналогично: `src/app/api/photo/[id]/file/route.ts:24-35` — legacy-ветка возвращает SVG ДО проверки доступа (проверка только на `:37-53` для файловых фото; в той ветке экранирование частичное: `:30` срезает только `<>&`, кавычки остаются)
- Почему чинить: перечисление id → чтение всех legacy-фото (storage='svg', v1) и подписей без токена/сессии. Caption управляемое (PATCH, 300 симв.) и вставляется в `image/svg+xml` как есть → same-origin XSS при прямом переходе по ссылке (React от эмиссии не защищает, это серверный ответ). Кэширование public может закэшировать приватное фото.
- Фикс: проверку доступа (master | client-token+visible) поднять ВЫШЕ legacy-ветки в обоих роутах; caption экранировать полностью или отдавать legacy-SVG как `text/plain`; `Cache-Control: private`.

### [P2][auth] initData без проверки свежести auth_date — пожизненный replay
- Файл:строка: `src/db/auth.ts:158-182` — `auth_date` из параметров не анализируется (упоминается только в клиентском `src/lib/router.ts:135`)
- Почему чинить: перехваченный/утёкший initData (логи, referer) даёт вечный мастер-доступ, тогда как cookie-сессии имеют TTL 30 дней. Рекомендация Telegram — проверять свежесть.
- Фикс: reject при `Date.now()/1000 - auth_date > N` (напр. 24ч), N в env.

### [P2][deps] xlsx@0.18.5 (npm-канал заморожен, известные CVE)
- Файл:строка: `package.json:85`
- Цитата: `"xlsx": "^0.18.5"`
- Почему чинить: npm-версия SheetJS не обновляется; по памяти CVE-2023-30533 (prototype pollution при парсинге XLSX) и CVE-2024-22363 (ReDoS). В коде xlsx используется ТОЛЬКО на запись (`src/db/export.ts:38,63,87` — aoa_to_sheet), парсинга пользовательских XLSX нет, поэтому прямой эксплойт сейчас недостижим — риск регрессии при будущем «импорте из Excel».
- Фикс: перейти на официальный канал SheetJS ≥0.20.2 или exceljs; пока минимум — зафиксировать и завести задачу.
- Сюда же: `next-auth ^4.24.11` (`package.json:67`) — импортов в src/ НЕТ (проверено rg) — мёртвая зависимость с историей CVE в линейке 4.x, убрать.

### [P2][abuse] Клиентские endpoинты без rate-limit
- Файл:строка: `src/app/api/estimates/[id]/comments/route.ts:50-83` (POST от клиента по токену), `src/app/api/estimates/[id]/approve/route.ts:27-39`
- Почему чинить: login защищён (5/15мин), а держатель клиентского токена может спамить комментарии (2000 симв., без лимита — раздувание БД/лента мастеру) и дёргать approve без ограничений.
- Фикс: `rateLimit("comment:"+token, 20, 3600_000)` по образцу login.

### [P3] Мелочи (перечень)
- `src/app/api/settings/route.ts:11-19` и `src/app/api/auth/me/route.ts:10-21` — публично отдают masterName/companyName/hasPassword (fingerprinting мастера). Приемлемо, но можно отдавать только после входа.
- `src/app/api/telegram/webhook/route.ts:58-64` — GET публично раскрывает URL WebApp и имя бота (дополнение к известной #4).
- `src/app/api/estimates/[id]/approve/route.ts:32-38` — клиент по токену может перевести `draft`-смету сразу в `approved`, минуя `sent` (логика статусов).
- `src/db/seed.ts:152-174` — угадываемые demo-токены (`demo-rivija-42` и т.п.); сами данные вымышленные (проверено, PII нет).
- `src/app/api/route.ts` — «Hello, world!» в корне API: шум/фингерпринт стенки, удалить.
- `src/db/queries.ts:593` (makeBackup) — бэкап JSON включает appSettings с passHash; допустимо (scrypt, роут за requireMaster), но чище исключать хэш из экспорта.
- Нет CSP/X-Frame-заголовков (next.config.ts — только кэш); для Mini App во фрейме Telegram — добавить хотя бы `frame-ancestors telegram...` и CSP для остального.

## ПРОВЕРЕНО — ЧИСТО
- SQL-инъекции: `queryRaw`/`executeRaw` в ветке отсутствуют (rg по src/ — 0 попаданий), всё через Prisma-клиент. ✓
- eval / new Function — нет. `dangerouslySetInnerHTML` — единственный в `src/components/ui/chart.tsx:83` (шаблонный CSS shadcn из темы, не пользовательский ввод). ✓
- Path traversal: `src/db/uploads.ts:145,151` — `path.basename()`; имена файлов генерируются сервером (`safeName`, :39-41), пользовательское имя не используется. ✓
- Загрузка фото: лимит 15 МБ/файл, ≤10 файлов (`photos/route.ts:29,45`), mime-allowlist (:18-24), содержимое полностью перекодируется sharp→JPEG (перезапись EXIF/полей). ✓
- Ветки/привязки при загрузке валидируются принадлежностью объекту (`photos/route.ts:59-75`). ✓
- Токены: сессия `randomBytes(32)` hex + TTL 30д + аннулирование при смене пароля (auth.ts:82,137-147); share `randomBytes(16)` hex (share/route.ts:20); act token UUID-24hex (queries.ts:412) — энтропия достаточная. ✓
- Секретов в репо нет (rg по шаблонам токенов/ключей — только ложные срабатывания lockfile); .env.example без секретов; postinstall-скриптов нет. ✓
- seed.ts: данные синтетические, без реальных клиентов/телефонов (проверено rg). ✓
- middleware.ts отсутствует (обходов через middleware нет); next.config.ts — только no-store заголовки. ✓
- rate-limit login: in-memory, sweep-очистка (rate-limit.ts:11-19) — ок для одноинстансового SQLite. ✓

## ТАБЛИЦА API-РОУТОВ v2.2 (40 route.ts)

| Роут | Методы | Защищён? |
|---|---|---|
| /api | GET | НЕТ — «Hello, world!» (безвредно, P3) |
| /api/acts | GET | ✓ requireMaster |
| /api/acts/[id] | GET, DELETE | ✓ requireMaster |
| /api/acts/[id]/payments | POST | ✓ requireMaster |
| /api/auth/login | POST | by design публичный; rate-limit 5/15м + sameOrigin ✓ |
| /api/auth/logout | POST | by design (чистит cookie) ✓ |
| /api/auth/me | GET | by design публичный; лёгкое инфо-раскрытие (P3) |
| /api/backup | GET | ✓ requireMaster (известная №5 — сверено :11) |
| /api/backup/restore | POST | ✓ requireMaster |
| /api/catalog/import | POST | ✓ requireMaster |
| /api/client/[token] | GET | токен-гейт (clientToken/shareToken/actToken) ✓ |
| /api/dashboard | GET | ✓ requireMaster |
| /api/estimates | GET, POST | ✓ requireMaster |
| /api/estimates/[id] | GET, PATCH, DELETE | ✓ requireMaster |
| /api/estimates/[id]/act | POST | ✓ requireMaster |
| /api/estimates/[id]/approve | POST | ✓ checkMaster \| client-token (P2: без rate-limit, draft→approved) |
| /api/estimates/[id]/comments | GET, POST | ✓ checkMaster \| client-token (P2: без rate-limit) |
| /api/estimates/[id]/lines | POST | ✓ requireMaster |
| /api/estimates/[id]/lines/[lid] | PATCH, DELETE | ✓ requireMaster |
| /api/estimates/[id]/lines/[lid]/move | POST | ✓ requireMaster |
| /api/estimates/[id]/room | POST | ✓ requireMaster |
| /api/estimates/[id]/share | POST | ✓ requireMaster |
| /api/export/act-text/[id] | GET | НЕТ — известная находка №3 (сверено :11-19) |
| /api/export/act/[id] | GET | ✓ requireMaster |
| /api/export/estimate/[id] | GET | ✓ requireMaster |
| /api/objects | GET, POST | ✓ requireMaster |
| /api/objects/[id] | GET, DELETE | ✓ requireMaster |
| /api/parse-line | POST | ✓ requireMaster |
| /api/payments/[id] | DELETE | ✓ requireMaster |
| /api/photo/[id] | GET | НЕТ для legacy-SVG ветки — НОВАЯ P1 |
| /api/photo/[id] | PATCH, DELETE | ✓ requireMaster |
| /api/photo/[id]/file | GET | ✓ master/client-token для файлов; НО legacy-ветка до проверки — НОВАЯ P1 |
| /api/photos | POST | ✓ requireMaster |
| /api/price | GET, POST | ✓ requireMaster |
| /api/price/[name] | DELETE | ✓ requireMaster |
| /api/price/suggest | GET | ✓ requireMaster |
| /api/room-presets | GET | ✓ requireMaster |
| /api/settings | GET | публичный; лёгкое инфо-раскрытие (P3) |
| /api/settings | PUT | ✓ requireMaster, НО demo-захват паролем — НОВАЯ P1 |
| /api/telegram/webhook | POST, GET | НЕТ — известная №4; GET дополнительно раскрывает webapp URL (P3) |
| /api/templates | GET, POST | ✓ requireMaster |
| /api/templates/[id] | DELETE | ✓ requireMaster |

Итог: 36/40 handler-группы защищены корректно; незащищённые — известные №3/№4, безвредные публичные (auth, me, /api, settings GET, client/[token]) и НОВАЯ дыра photo/[id] legacy-ветка. Главная проблема ветки — не роуты, а telegram-ветка checkMaster (P0-1) и demo-захват (P1-3).

## РЕКОМЕНДОВАННЫЙ ПОРЯДОК ФИКСОВ (до мержа в main)
1. P0-1: пустой botToken → reject в verifyTelegramInitData (+ auth_date freshness — сразу P2-5).
2. P1-2: allowlist telegram user ids.
3. P1-3: запрет установки пароля через API в demo-режиме.
4. P1-4: авторизация выше legacy-SVG ветки в photo-роутах + экранирование caption.
5. P2-7/P2-6 и P3 — следующим коммитом.
