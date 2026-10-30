# Sereban-glitch.github.io — верификация аудита 2026-10-29

Проверка: 2026-10-30 (забег Z.ai). Работа только чтение, код не запускался, прод-IP не трогался.

## Коммиты после аудита: нет

HEAD = `3c25b71` (2026-10-17 03:36:26 +0300, «portfolio: show approved Smart Zapyt interface snapshots»), ветка `master`, совпадает с `origin/master`.

## Дефолтная ветка: `master` (не main)

- `git symbolic-ref HEAD` → `refs/heads/master`; `origin/HEAD -> origin/master`.
- Ветки в клоне: локальная только `master`; удалённые: `origin/master` и `origin/feat/live-archi-dashboard`. Ветки `main` нет вовсе.
- На `origin/feat/live-archi-dashboard` нет каталога `gophergpt/` (ветка старее его добавления в master 2026-09-02); archi-dashboard.* есть в обеих.

## Пункт 1. Хардкод прод-IP в клиентском JS — STILL_PRESENT

- archi-dashboard.js:3 — точное совпадение: `const STATUS_URL = 'https://35-209-212-217.sslip.io/api/status';`
- Дополнительно: прод-IP всплывает ещё в двух местах: archi-dashboard.test.js:127 (`assert.equal(seen.url, 'https://35-209-212-217.sslip.io/api/status')`) и deploy/Caddyfile:1.
- Файл отслеживается на master (git ls-files), т.е. публикуется на GitHub Pages; страница archi-dashboard.html:173 подключает этот скрипт (`<script src="./archi-dashboard.js"></script>`) — браузеры посетителей будут бить в прод-IP напрямую.

## Пункт 2. В git отслеживаются бинарники gophergpt — STILL_PRESENT (сумма уточнена)

`git ls-files -s gophergpt/dist/` — 6 файлов (5 бинарников + SHA256SUMS.txt), режим 100644:

| Файл | Байты |
|---|---|
| gophergpt-v0.1.0-darwin-amd64.gz | 2 698 870 |
| gophergpt-v0.1.0-darwin-arm64.gz | 2 653 743 |
| gophergpt-v0.1.0-linux-amd64.gz | 2 684 306 |
| gophergpt-v0.1.0-linux-arm64.gz | 2 583 085 |
| gophergpt-v0.1.0-windows-amd64.zip | 2 737 206 |
| **Итого 5 бинарников** | **13 357 210** |
| + SHA256SUMS.txt | 495 |
| **Весь dist/** | **13 357 705** |

- Число аудита «13 357 705» подтверждается в точности как сумма всех 6 файлов dist/ (5 бинарников дают 13 357 210).
- gophergpt/data/jules_verne.txt — 4 143 880 байт (~4,1 МБ, 3,95 МиБ), подтверждено.
- Бонус: gophergpt/data/vocab — ещё 224 133 байта тоже в git.
- Всё это публично скачивается с Pages: gophergpt/index.html:301-305 даёт прямые ссылки `<a href="dist/gophergpt-v0.1.0-….gz">` на каждый бинарник, :308 — на SHA256SUMS.txt; сама папка linked из главной index.html:548 (`<a href="gophergpt/">Лабораторный журнал →</a>`).

## Пункт 3. Смешение проектов — STILL_PRESENT

- dashboard-api/server.js:112 — `.listen(env.PORT || 18181, '127.0.0.1')` — Node-сервер (требует BOT_TOKEN, server.js:108) на Pages исполняться не может.
- gophergpt/*.go — компилируемый Go (block.go, head.go, layer.go, main.go, pkg/*) + go.mod/go.sum — на Pages не исполняется.
- Плюс deploy/: systemd-юнит и Caddyfile для внешней VM — тоже к Pages не относятся.

## Новые находки

1. **Публичный слив топологии прод-инфраструктуры.** deploy/Caddyfile:1 — `35-209-212-217.sslip.io { … reverse_proxy 127.0.0.1:18181 … }`; deploy/archi-dashboard-api.service:7-16 — `WorkingDirectory=/opt/archi-dashboard/dashboard-api`, `PGDATABASE=agrobot`, `PORT=18181`, `/opt/node-v24.16.0/bin/node`. Вместе с хардкодом STATUS_URL (П.1) любой посетитель «портфолио» получает полный маршрут: домен → Caddy → порт → БД. Уровень: 100% нужно вычистить из публичного репо.
2. **Тестовые файлы публикуются на прод-сайте, а CI нет.** В корне отслеживается archi-dashboard.test.js (8 test() из node:test, строки 2-3: `require('node:assert/strict')`, `require('node:test')`) и dashboard-api/*.test.js (4 файла) — они уедут на Pages как статика; при этом `.github/` в репо отсутствует — ни один из них автоматически не запускается. Также archi-dashboard.html не связан с index.html (ссылок нет — rg по index.html даёт только gophergpt/), но трекается на master и, значит, доступен по прямому URL.

## Не проверено

- Живое поведение GitHub Pages (не ходил в сеть; прод-IP 35.209.212.217 / 34.46.98.209 и api.telegram.org не трогались).
- Зеркало origin на GitHub (коммиты «за сутки» сверены только по локальному клону; api.github.com запрещён).
- Успешность сборки dashboard-api на Node >=24 (код не запускался).
