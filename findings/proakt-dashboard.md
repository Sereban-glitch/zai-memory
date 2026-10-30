# proakt-dashboard — верификация пунктов аудита 2026-10-29

Проверка: 2026-10-30 (субагент Z.ai, режим «только чтение»).
Коммиты после аудита: нет. main = `4670daf` (2026-10-18). Состав: 10 файлов (main.go, static/, deploy.sh, README.md, go.mod `go 1.19`, .env.example и пр.).

---

## П.1: ноль авторизации + данные всех чатов + демо-фолбэк — STILL_PRESENT

- **main.go:33-37** — ровно эти строки, без какого-либо механизма доступа: `http.Handle("/", FileServer)`, `/api/stats`, `/api/objects`, `/api/acts` — ни auth, ни токена, ни ограничения адреса.
- **main.go:61** — ровно эта строка: `SELECT id, name, customer, status, created_at FROM objects ORDER BY created_at DESC LIMIT 50` — объекты ВСЕХ чатов, без `WHERE chat_id`. Аналогично `handleStats` (:52-54: COUNT/SUM по всей БД) и `handleActs` (:86-93: JOIN по всем объектам).
- **static/app.js:88-90** — ровно эти строки (fallback при недоступности API): `animateValue("totalObjects", 0, 12, ...)`, `animateValue("totalActs", 0, 84, ...)`, `animateValueCurrency("totalRevenue", 0, 162400, ...)` — демо-значения маскируют аварию БД; :100-108 — renderActs с фейковыми объектами («ЖК Новопечерские Липки, кв. 12» и т.п.). Комментарий :76: «with Mock Fallback in case backend is unreachable during dev».

## П.2: deploy.sh — пароль из прод-.env и DSN в systemd-юните — STILL_PRESENT

- **deploy.sh:13** — ровно эта строка: `DB_DSN=$(sudo cat /home/master-build/proakt/.env | grep "^DB_DSN=" | cut -d "=" -f 2-)` — читает DSN (логин/пароль) из прод-.env билд-бота.
- **deploy.sh:21-37** — ровно эти строки: `sudo bash -c "cat > /etc/systemd/system/proakt-dashboard.service <<EOF … Environment=\"DB_DSN=\$DB_DSN\" … EOF"` — DSN вшит в юнит-файл через `cat >`; файл создаётся с дефолтными правами 0644 — читается всеми локальными пользователями. Плюс сам DSN передаётся через аргумент `bash -c` (виден в списке процессов на время записи).

## П.3: нет LICENSE — STILL_PRESENT

- `git ls-files` (10 файлов) и рабочее дерево не содержат LICENSE — несмотря на то, что соседние проекты (proakt-build-bot MIT) лицензию имеют.

---

## Новые находки

1. **(P1) Сырые ошибки БД наружу:** main.go:63 и main.go:95 — `http.Error(w, err.Error(), 500)` в `handleObjects`/`handleActs`: любому посетителю отдаётся текст ошибки lib/pq (может содержать хост/порт/имя БД из DSN и детали схемы) — утечка инфраструктуры при нулевой авторизации.
2. **(P1) Слушает все интерфейсы, а не localhost:** main.go:45 — `http.ListenAndServe(":"+port, nil)` → 0.0.0.0:8080 (PORT по умолчанию 8080, :39-42). README.md:33 при этом обещает «Веб-интерфейс будет доступен по адресу http://localhost:8080» — вводит в заблуждение; в связке с П.1 (ноль авторизации + все данные всех чатов) панель открыта на каждый сетевой интерфейс хоста, включая публичный, если его не закрывает файрвол.

## Не проверено

- Реальная экспозиция на проде (прод-IP не трогали; файрвол/привязку адреса на сервере проверить нельзя из репозитория).
- Наличие/права фактического systemd-юнита `proakt-dashboard.service` на сервере (утверждается только deploy.sh).
- Прогон приложения (код не запускали).
