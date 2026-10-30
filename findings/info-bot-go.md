# info-bot-go — верификация аудита 2026-10-29

- Дата верификации: 2026-10-30 (забег Z.ai).
- Локальный клон: `repos/info-bot-go`, ветка `main`, HEAD `41b9b27` (2026-10-19 13:22:38 +0300, merge PR #9 feat/smartzapyt-blue-design).
- Рабочее дерево чистое, соответствует HEAD.
- **Коммиты после аудита: нет.**

## П.1 — Бинарник и лог в git

**Статус: STILL_PRESENT**

- `git ls-files` содержит `dostup-probe`, `bot-test.log`, `tools/dostup-probe/main.go` — бинарник и лог отслеживаются, исходник рядом.
- `dostup-probe` — ELF 64-bit x86-64, не stripped, **7 956 479 байт (7.6 МБ)**; blob `404e875…`, режим 100755.
- `bot-test.log` (909 байт, blob `9307c18…`) — лог запуска от 2026/05/31: «module "start" registered … Info-Bot-Go starting…». Добавлены в `7458e56` (initial release) / `bc7d2e2`.

## П.2 — .env.example, дефолты config.go, битая ссылка README

**Статус: STILL_PRESENT**

- `.env.example` — ровно **14 переменных** (TELEGRAM_BOT_TOKEN, GEMINI_API_KEY, SMTP_HOST/PORT/USER/PASSWORD/FROM_ADDR, IMAP_HOST/PORT, GMAIL_USER, GMAIL_APP_PASSWORD, DOSTUP_EMAIL/PASSWORD/SESSION_FILE). Код читает **40** различных имён env в `internal/config/config.go` (+2 SMOKE_* в tools) — «~41» из отчёта подтверждается.
- `internal/config/config.go:129` — `AdminID: getEnvInt64("ADMIN_ID", 745130167)` — личный ID захардкожен как дефолт (строка точная).
- `internal/config/config.go:134` — `SharedMailbox: getEnvOrDefault("SMTP_FROM_ADDR", getEnvOrDefault("GMAIL_USER", "publicinquiry69@gmail.com"))` — fallback-ящик в коде (строка точная).
- `README.md:53` — «скрипт проверки SMTP/IMAP конвейера (`tools/test_mail`)»; `README.md:98` — `source .env && go run ./tools/test_mail/`. Каталог `tools/test_mail` **не существует** (в `tools/`: broadcast, dostup-probe, dostup-submit, dostup-test, smoke-*, status-test, test_cache, test_osint, verify_directory). Строки точные.

## П.3 — Рассинхрон версий Go

**Статус: STILL_PRESENT**

- `go.mod:3` — `go 1.24.1` (точно).
- `Dockerfile:1` — `FROM golang:1.21-alpine AS builder` (точно) → образ собирает бот версией ниже требуемой go.mod.
- `.github/workflows/release.yml:35` — `go-version: '1.21'` (точно).
- `.github/workflows/ci.yml:35` — `go-version: '1.24'`; `deploy.yml:22` — `'1.24'` (точно, как в отчёте).

## НОВОЕ-1 — Открытые PR и версии Actions на HEAD

**Статус: подтверждено, все 5 dependabot-бампов НЕ слиты**

Версии `uses:` на HEAD (все ниже предлагаемых dependabot):
- `actions/checkout@v4` — deploy.yml:19, release.yml:31, ci.yml:19,56, build-apk.yml:20 (PR хочет v7);
- `actions/setup-go@v5` — deploy.yml:20, release.yml:33, ci.yml:31 (PR хочет v7);
- `actions/upload-artifact@v4` / `download-artifact@v4` — deploy.yml:31,44, release.yml:47,59, ci.yml:66, build-apk.yml:61 (PR хочет v8);
- `webfactory/ssh-agent@v0.9.0` — deploy.yml:48 (PR хочет 0.10.0);
- `softprops/action-gh-release@v2` — release.yml:64, build-apk.yml:55 (PR хочет v3).

Все 5 веток dependabot датированы 2026-09-12 (подтверждено `git log origin/dependabot/...`), в main не слиты.

- **PR #10 fix/smartzapyt-deadline-response** (последний коммит `af29d13`, 2026-10-19 14:10): ремонт дедлайнов-обратных отсчётов и читаемого отображения ответов SmartZapyt + регрессионные тесты. Дифф от main: **23 файла, +1902/−43** — `mini-app/patches/response-preview.test.mjs` (+92), `mini-app/patches/routine-ui-repair.test.mjs` (+60), `mini-app/qa/deadline_dom_test.mjs` (+249) и т.д.
- **PR #6 docs/portfolio-overview** (последний коммит `1278add`, 2026-10-17): обновление README-обзора портфолио + скриншоты. Дифф от main: **3 файла, +56/−12** — README.md и два PNG (`smart-zapyt-analytics-2026-10-17.png` 201 954 Б, `smart-zapyt-overview-2026-10-17.png` 133 240 Б). Остальные коммиты ветки уже в main.

## НОВОЕ-2 — Прод-IP в истории и на HEAD

**Статус: подтверждено; 35.209.212.217 живёт и в ТЕКУЩИХ файлах, 34.46.98.209 — только история**

- `35.209.212.217` присутствует на HEAD в двух файлах:
  - `.github/workflows/deploy.yml:41` — `VM_HOST: 35.209.212.217` (плюс `VM_USER: u0_a566` на :42);
  - `.opencode/skills/info-bot/SKILL.md:3` — «сервером agrobot-prod (35.209.212.217)…»; `:59` — «пользователь `u0_a566`, хост 35.209.212.217 … откат … `/home/archi/backups-info-bot-20260912.tar.gz`».
  - Введено коммитами `ce9a91c`/`a390d94`/`050b58e` (2026-09-12) — так и не убрано.
- `34.46.98.209` в рабочем дереве **отсутствует** (`rg` по всему дереву — 0 совпадений); только в истории `d4daafa`/`e85714c` (2026-06-19, Mini App/Vercel-фиксы).

## Новые находки

1. **Сломан триггер авто-деплоя: `.github/workflows/deploy.yml:5` — `branches: ain]`.** Вместо `[main]` (или `main`) в фильтре стоит мусорная строка `ain]` — push-триггер не срабатывает ни для одной ветки, авто-деплой «zero-touch» (заявлен в SKILL.md:59 «пуш в `main` → … ssh-деплой») фактически мёртв; работает только ручной `workflow_dispatch`. Ошибка внесена самым первым коммитом workflow `ce9a91c` (2026-09-12) и не исправлена последующим `e50515d`. Провал деплоя незаметен: CI зелёный, релиз не едет.
2. **Продовый ops-ранбук в публичном репо: `.opencode/skills/info-bot/SKILL.md`** (отслеживается, `git ls-files` подтверждает). Строки :3, :59 раскрывают хост ВМ 35.209.212.217, юзера `u0_a566`, путь деплоя `/home/u0_a566/info-bot-new` → `sudo cp` → `/home/archi/info-bot/info-bot`, архив отката `/home/archi/backups-info-bot-20260912.tar.gz`, процедуру `systemctl restart`. Для репо с LICENSE/CI (публичного) это готовая карта для атаки на инфраструктуру.

## Не проверено

- Фактический статус (open/merge/close) PR на GitHub — не ходил на api.github.com; сведено по локальным origin-ссылкам.
- Прогоны CI/Actions по веткам PR, работоспособность деплоя на ВМ.
- Актуальность/принадлежность IP-адресов на сегодняшний день (не трогал сеть).
- Контент бинарника `dostup-probe` (не запускал, не дизассемблировал).
