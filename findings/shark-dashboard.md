# shark-dashboard — верификация аудита 2026-10-29 (сверка 2026-10-30)

Репозиторий: Sereban-glitch/shark-dashboard (локальный полный клон, HEAD `df81bf8` от 2026-06-18).
Коммиты после аудита: нет.

---

## П.1 Неавторизованный доступ к метрикам (0.0.0.0 + CORS `*`) — STILL_PRESENT

- `main.go:31` — `addr := flag.String("addr", "0.0.0.0", "bind address")` — дефолтный бинд на все интерфейсы.
- `main.go:84-87` — три маршрута без авторизации: `mux.Handle("/", pageHandler)`, `mux.Handle("/api/events", sseHandler)`, `mux.Handle("/api/metrics", metricsHandler)`.
- Фактический путь sse.go: **`internal/handler/sse.go`** (не в корне), `internal/handler/sse.go:40` — `w.Header().Set("Access-Control-Allow-Origin", "*")`.
- `README.md:88-91` (Security Note; заявлено 90-91 — в допуске) — сам предписывает «Bind it to `127.0.0.1` and access it securely via an SSH tunnel: `ssh -L 8081:127.0.0.1:8081 …`», тогда как дефолт кода — 0.0.0.0.
- Чувствительные данные: `internal/model/metrics.go:11` — `Processes []ProcessInfo`; `internal/model/metrics.go:65-76` — структура `ProcessInfo` (Name, Group, PID, State, RAM, StartTime, ExitStatus, Description) — имена сервисов, PID, RAM.

## П.2 Docker-быстрый старт сломан (8081 vs 8080) — STILL_PRESENT

- `main.go:30` — `port := flag.Int("port", 8081, "HTTP server port")`.
- `Dockerfile:14` — `EXPOSE 8080`; `Dockerfile:15` — `ENTRYPOINT ["./shark-dashboard"]` без флага `-port`.
- `README.md:24` — `docker run -d -p 8080:8080 --name shark ghcr.io/sereban-glitch/shark-dashboard:latest`.
- Итог: приложение слушает 8081, контейнер публикует 8080 → connection refused у каждого, кто идёт по README.

## П.3 Версии Go разъехались — STILL_PRESENT

- `go.mod:3` — `go 1.25.0` (+ `go.mod:5` — `require golang.org/x/sys v0.46.0`).
- `Dockerfile:1` — `FROM golang:1.21-alpine AS builder`; `.github/workflows/release.yml:35` — `go-version: '1.21'`.
- Дополнительно: бейджи README сами противоречивы — `README.md:5` «Go-1.22+» и `README.md:12` «Go-1.21+» при go.mod 1.25.0. Сборка тулчейном 1.21 против директивы `go 1.25.0` потребует неявной загрузки нового тулчейна (GOTOOLCHAIN) либо упадёт — релизный пайплайн ненадёжен.

## Новые находки

1. **`.env.example` — мёртвая конфигурация, включая несуществующую «авторизацию».** Файл декларирует `SHARK_PORT=8080` (:5), `SHARK_REFRESH=5` (:8), `SHARK_TOKEN=` (:12, «Authentication token (optional, for protecting the dashboard)»), `SHARK_PM=auto` (:15) — rg по репо: **ни одного** `os.Getenv`/`SHARK_` в Go-коде; конфигурация только через flags (main.go:30-33). Опасность: пользователь выставляет `SHARK_TOKEN`, полагая, что закрыл дашборд, — а тот по-прежнему открыт без всякой авторизации (README.md:89 честно говорит, что auth нет). Вдобавок `SHARK_PORT=8080` противоречит реальному дефолту 8081 (main.go:30) и подпитывает портовую путаницу из П.2. deploy.sh.example эти переменные тоже не читает.
2. **Quick start «из исходников» тоже падает: цели `make build` не существует.** `README.md:27` — `make build && ./shark-dashboard`; Makefile:14 — `.PHONY: all build-arm64 build-local clean run` — цели `build` нет, а реальные цели кладут бинарь в `build/` (Makefile:21, 28), а не в `./shark-dashboard`. Обе половины команды из README невыполнимы («No rule to make target 'build'»).
3. Дополнительно: `web/templates/index.html:7-10` — четыре внешних CDN (cdn.tailwindcss.com, unpkg.com ×2, cdn.jsdelivr.net) при заявлении «zero-dependency Web UI» (README:3) и позиционировании «Edge-Native… survive restricted environments» (README:67): на изолированной ноде/без интернета UI теряет стили и функциональность, хотя шаблон встроен в бинарник (`main.go:21` go:embed) и CORS `*` (sse.go:40) делает его ещё и удобным для встраивания третьими лицами.

## Не проверено

- Runtime-поведение (запуск запрещён методикой): подозрение на обрыв SSE каждые ~30 с — `main.go:105` `WriteTimeout: 30 * time.Second` против бесконечного стрима в `internal/handler/sse.go:70-83` (типовая Go-ловушка; EventSource переподключается сам). Требует живой проверки.
- Реальная собираемость релизным тулчейном Go 1.21 при `go 1.25.0` (может сработать через автозагрузку тулчейна — зависит от окружения runner'а).
- Внешнее состояние (issues/PR, наличие GHCR-образа) — не проверялось (api.github.com не использовался).
