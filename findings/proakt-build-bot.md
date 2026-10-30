# proakt-build-bot — верификация пунктов аудита 2026-10-29

Проверка: 2026-10-30 (субагент Z.ai, режим «только чтение»).
Коммиты после аудита: нет. main = `fb2171b` (2026-10-18). Открытый PR #1 (ветка `origin/design/miniapp-command-center`): 13 коммитов с 2026-09-18, последний `bfe64ac` (2026-09-20) — тоже старше аудита.

Репозиторий: Go, MIT LICENSE присутствует (LICENSE:1-3, «MIT License, (c) 2026 Sereban-glitch»), обширные тесты (`*_test.go` по всем пакетам).

---

## П.1: публичные /s/*-эндпоинты без rate-limit и с вечным токеном — STILL_PRESENT

Фактические строки (сдвигов почти нет):

- **internal/httpapi/handlers.go:70-75** — :70 комментарий «публичный просмотр сметы заказчиком (без auth, доступ по токену)», :71-75 — пять маршрутов без обёртки `a(...)` (= `s.auth`, см. handlers.go:20): `GET /s/{token}`, `GET /s/{token}/photo/{id}`, `POST /s/{token}/approve`, `POST /s/{token}/comment`, `GET /s/{token}/print`.
- **Rate-limit: отсутствует в принципе** — grep по `internal/`/`cmd/` (ratelimit|limiter|throttle) — 0 совпадений вне тестов.
- **internal/store/store.go:94** — `share_token TEXT NOT NULL DEFAULT ''` (в таблице estimates, :86-96): колонки expiry/срока жизни нет (индекс :99 — уникальный, без TTL-логики).
- **internal/store/estimates.go:349-358** — `SetShareToken` (:352 — сам `UPDATE estimates SET share_token=$3 ...`): перевыпуск только по явному действию мастера — из бота (`internal/app/estimates.go:605`) или из Mini App (`internal/httpapi/estimates.go:378`, за auth+ownership). Автоматики/ротации нет.
- **internal/httpapi/share_v06.go:61** — `chatID, estID, title, err := s.svc.ApproveEstimateByToken(ctx, token)` — публичный `POST /s/{token}/approve` переводит смету в `approved` (store: `internal/store/estimates_v06.go:169-178` `UPDATE estimates SET status='approved' WHERE share_token=$1 AND status IN ('draft','sent')`) и шлёт мастеру уведомление «Заказчик принял смету» — фальшивое «согласована». **:99-104** — `AddComment(..., "client", in.Text)` + `NotifyClientComment` — фальшивые комментарии от имени заказчика.
- Сам токен криптостоек (16 байт hex; `internal/app/estimates.go:594-601`), но вечный, без rate-limit и без привязки согласования к личности.

## П.2: нет CI — STILL_PRESENT

- Каталог `.github/` отсутствует (в рабочем дереве и в git-дереве main: `git ls-tree -r main | grep -c '^\.github'` → 0).
- **README.md:408** — ровно эта строка: «Ниже перечислены сценарии тестов; это не отчёт о свежем прогоне:».

## PR #1 «Редизайн Mini App: понятный рабочий стол и навигация» (design/miniapp-command-center)

- `git log main..origin/design/miniapp-command-center`: 13 коммитов (2026-09-18 → 2026-09-20): редизайн Mini App «командный центр» (35d9382), демо-калибровка + клиентский портал + handoff для OpenCode (943546f), read-only кабинет заказчика с привязкой по Telegram ID, «чужое 404» (d0d53bf), живой API кабинета (7ccb85b), staging-режим NOPOLL + seed360 (9c62f99), seed с реальными строками (c32abc4), стартер-объекты и удаление объекта (75f9079, 12e71c8), fix теста (bfe64ac).
- `git diff --stat main...`: 51 файл, +4282/−386. Основное: `internal/httpapi/client.go` (+332, клиентские маршруты), `internal/store/client_access.go` (+98, доступ заказчика), `cmd/seed360` (+99), `internal/store/seed360*.go` (+~600, демо-данные), `internal/webapp/static/client.html|js|css` (кабинет заказчика), `deploy/proakt-360.service` (+31, systemd с харденингом, EnvironmentFile без секретов), `deploy/Caddyfile.360.example` (reverse_proxy 127.0.0.1:8444), `scripts/backup360.sh` (+11, DSN из env, секретов нет), `docs/HANDOFF_OPENCODE.md` (+120).
- **Безопасность новых маршрутов — нормально:** все `/api/client*` обёрнуты `s.auth` (Telegram initData HMAC; `internal/httpapi/client.go:16-27`), доступ проверяет `mustClient` → `CanClientSee`, чужое = 404 (client.go:45-56). `auth.go:54-57` — fail-closed при пустом botToken, replay-защита по auth_date (:90-97).
- **Критичных регрессий в коде не найдено**, НО: `/s/*` остаются публичными без изменений (handlers.go:70-74 на ветке) и в ветку зашиты реальные данные (см. новые находки).

---

## Новые находки

1. **(P2; PR-ветка) Реальные бизнес-данные заказчика в публичном репозитории:** `internal/store/seed360_real_gen.go:1-2` — «Code generated from Vitalik notes (Google Drive). DO NOT EDIT manually» — настоящие строки актов (позиции, объёмы, цены, суммы; напр. «шпаклёвка стен под стеклохолст … 400 м² × 120 = 48 000»), `docs/HANDOFF_OPENCODE.md:50` — «Основной объект: `Парковый 2 · квартира`» (реальный адрес), :52-56 — реальные итоги «12 актов: … Сумма 549 526 ₴ — реальный итог источника» и «Смета: 190 383,50 ₴ … реальный итог источника»; также `README.md:90` ветки. На main этого нет (main чист) — данные въедут в публичную историю при мерже PR.
2. **(P3; main) Бэкапы чатов пишутся с разделяемыми правами:** `internal/app/backups.go:39` — `os.MkdirAll(dir, 0o755)` (FILES_DIR/backups) и `os.Create` (:45, файл 0644 по umask): полные выгрузки бизнес-данных каждого чата (сметы/акты/прайс/шаблоны) читаются любым локальным пользователем сервера.

## Не проверено

- Статус/обсуждение PR #1 на GitHub (api.github.com не использовали).
- Прогон тестов (код не запускали); README заявляет, что интеграционные тесты требуют `PROAKT_TEST_DSN`.
- Поведение бота/Mini App на проде (не трогали).
