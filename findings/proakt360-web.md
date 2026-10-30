# proakt360-web — верификация пунктов аудита 2026-10-29

Проверка: 2026-10-30 (субагент Z.ai, режим «только чтение»).
Коммиты после аудита: нет (кроме указанного). main = `ef34e82` (2026-10-18); несмерженная ветка `origin/v2.2-server-sync` = `0841395` (2026-10-19, «feat: sync actual ProAkt 360 v2.2 source with production server», 148 файлов развёрнутых исходников вместо zip).

---

## П.1 (P0): fail-open аутентификация на main — STILL_PRESENT

main не изменился: дерево main состоит только из `README.md` + `proakt360-public-prepared.zip` (git ls-tree); все исходники по-прежнему только внутри архива.

- **src/db/auth.ts:38-40 (в zip)** — ровно эти строки: `export async function verifyPassword(...)` → `if (!isAuthRequired(s.passHash)) return true;` — без установленного пароля любой считается мастером.
- **src/db/auth.ts:14-16 (в zip)** — `sha256()` (одиночный SHA-256); **:30-33** — `setPassword` = `sha256(salt + newPassword)` — без итераций, быстрый хеш.
- **src/db/http.ts:57-63 (в zip)** — `requireMaster`: `if (required && !authed) { return jsonError(401, ...) } return null` — при `required=false` (нет пароля) guard всегда пропускает; все мастерские маршруты открыты.
- **src/app/api/backup/restore/route.ts:11-20 (в zip)** — разрушительный `restoreBackup` за этим fail-open guard (строка 20 — вызов `restoreBackup`).
- **README.md:41-42 (на main)** — сам называет это блокером №1 («Fail-open аутентификация…») и №2 (быстрый SHA-256).

Вердикт: **STILL_PRESENT**, main для посетителей остаётся уязвимой версией.

## П.1-б (ГЛАВНАЯ НОВОСТЬ): аутентификация переписана на ветке v2.2-server-sync — FIXED на ветке, НЕ влито в main

(а) fail-closed подтверждён по файлам ветки:
- `src/db/auth.ts:95-97` — `verifyPassword`: `if (!isAuthRequired(s.passHash)) return false;` — без пароля вход невозможен (открытым «демо без пароля» управляет только `PROAKT_DEMO_OPEN`).
- `src/db/auth.ts:34-38` — `SCRYPT_N=16384; SCRYPT_R=8; SCRYPT_P=1; SCRYPT_KEYLEN=64` (scrypt, формат `scrypt$N$r$p$salt$hash`, :67-76).
- `src/db/auth.ts:100-115` — прозрачный апгрейд старых `sha256(salt+пароль)`-хешей до scrypt при первом успешном входе (+ сброс сессий).
- `src/db/auth.ts:193-198` — `checkMaster`: пароль не задан и `PROAKT_DEMO_OPEN` не включён → `{ ok: false, status: 503, error: SETUP_ERROR }`; :44-48 — `demoOpenMode()`; :41-42 — SETUP_ERROR «задать пароль удалённо нельзя».
- `src/db/http.ts:84-93` — `requireMaster(req)`: 403 при чужом Origin (sameOrigin/CSRF, :61-74), затем 503/401 из checkMaster.
- Пароль задаётся только локально: `package.json:10` — `"setup:password": "tsx scripts/set-password.ts"`; `scripts/set-password.ts:8-10` — «CLI, а не веб-форма: … удалённо нельзя».
- Бонусы на ветке: rate-limit логина 5 попыток/15 мин (`src/app/api/auth/login/route.ts:18-19,32-39`), HttpOnly/SameSite/Secure cookie (:67-73), `src/db/rate-limit.ts` (новый файл), `next.config.ts` — `ignoreBuildErrors: false` (закрыт блокер №4), добавлены `scripts/test-restore.ts`/`verify-api.ts`/`e2e-check.ts` (блокер №5), `public/` в репо (блокер №3).

(б) **backup/restore на ветке защищён**: `src/app/api/backup/restore/route.ts:11` — `const denied = await requireMaster(req)` (теперь с передачей `req` → origin-проверка + fail-closed guard).

(в) **LICENSE: НЕТ ни на main, ни на ветке** — `git ls-tree -r` обеих версий не содержит файла лицензии (в zip его тоже нет).

(г) **z-ai-web-dev-sdk: ОСТАЛАСЬ мёртвой зависимостью** — `package.json:86` ветки: `"z-ai-web-dev-sdk": "^0.0.18"`; `git grep z-ai-web-dev-sdk` по `src/`+`scripts/` ветки — 0 импортов. Дополнительно не используется и `next-auth` (`package.json:67`, импортов в src нет).

(д) **.env.example ветки — чистый**: `DATABASE_URL="file:../db/custom.db"`, закомментированные `#PROAKT_DEMO_OPEN=1`, `#TELEGRAM_BOT_TOKEN="your_bot_token_here"`, `#WEBAPP_URL="https://your-domain.com/?v=2.2"` — реальных значений нет.

**Вывод по П.1:** улучшение «закрыть fail-open» ВЫПОЛНЕНО на ветке v2.2-server-sync, но НЕ влито в main — main остаётся уязвимым. Ветку нужно мержить после проверки новых находок ниже.

---

## Новые находки

1. **(P0; актуально И для main-zip, И для ветки) Обход авторизации на экспорте акта:** `src/app/api/export/act-text/[id]/route.ts` (строки 10-37) — GET-обработчик БЕЗ `requireMaster`/`checkMaster`; обращение к БД по числовому `id` без сессии/токена. Перечисление id отдаёт текст акта: номер, позиции (название/кол-во/цена), итог, период работ. На ветке все прочие мастерские маршруты под guard (30 route-файлов с `requireMaster`; соседние `export/act/[id]` и `export/estimate/[id]` — с guard), а этот пропущен. Единственные легитимно-публичные маршруты — `auth/*`, `client/[token]`, `api/route.ts`, `telegram/webhook`.
2. **(P1; только ветка) Неаутентифицированный Telegram-вебхук + зашитый прод-домен:** `src/app/api/telegram/webhook/route.ts:6` — POST без проверки секретного токена (нет `X-Telegram-Bot-Api-Secret-Token`, путь без секрета) → кто угодно может слать фейковые апдейты и заставить бота рассылать приветствие с кнопкой Mini App (`:30-48`, `sendMessage` на произвольный `chat_id`); `:4` — fallback `WEBAPP_URL` захардкожен как sslip.io-домен, кодирующий прод-IP (`https://proakt-360.<ip>.sslip.io/?v=2.2`); `:54` — наружу отдаётся `String(err)`.
3. **(P3; ветка) Публичные демо-токены клиента в seed:** `src/db/seed.ts:406,422,438` — захардкоженные `token: 'demo-act-zat-1'` и др. — после сида кабинеты заказчика открыты по известным токенам (наследие блокера №7 из README main).

## Не проверено

- Реальное поведение на проде (прод-IP не трогали); факт выката ветки на сервер (commit message заявляет «sync with production server» — не подтверждено).
- Прогон тестов/сборки ветки (код не запускали).
- Состояние PR/CI на GitHub (API не использовали).
