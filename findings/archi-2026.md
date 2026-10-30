# archi-2026 — верификация аудита 2026-10-29

- Дата проверки: 2026-10-30. Проверка ТОЛЬКО ЧТЕНИЕ (код не запускался, деревья не менялись, checkout не выполнялся).
- HEAD: `3c560b7` (2026-10-27, ветка `main`), рабочее дерево чистое.
- **Коммиты после аудита: нет** (последний коммит 2026-10-27, аудит 2026-10-29).

Прод-IP в цитатах замаскирован (35.209.*.*).

---

## П.1: Прод-IP в deploy/config.yaml

**Статус: STILL_PRESENT** (строки совпали точно, ±0).

Доказательства:
- `deploy/config.yaml:14` — `base_url: http://127.0.0.1:18080   # туннель до 35.209.*.*:18080 (см. archi-tunnel.service)` — IP в комментарии.
- `deploy/config.yaml:45` — `host: agrobot-prod         # alias из ~/.ssh/config (или IP 35.209.*.*)` — тот же IP в комментарии блока `bridge.remote`.

В остальных файлах репо IP в открытом виде не встречается (grep по `*.go/*.py/*.md/*.service/*.sh`: только config.yaml; `deploy/archi-tunnel.service:26-27` использует alias `agrobot-prod` без IP).

## П.2: setup-piper.sh — latest-релиз без контроля целостности, исполнение, sudo

**Статус: STILL_PRESENT** (строки совпали точно, ±0).

Доказательства:
- `deploy/setup-piper.sh:36` — `BIN_URL="$GH/latest/download/piper_linux_x86_64.tar.gz"` — плавающий «latest» без фиксации версии.
- `deploy/setup-piper.sh:47-50` — `curl -fSL --retry 3 -o /tmp/piper.tar.gz "$BIN_URL"` → `tar -xzf` → `install -m 755 /tmp/piper/piper "$DEST/piper"` — ни sha256, ни подписи, ни проверки размера.
- `deploy/setup-piper.sh:58` — `echo "Проверка голоса" | "$DEST/piper" …` — скачанный бинарь исполняется сразу.
- `README.md:128` — `sudo bash deploy/setup-piper.sh` — предписано от root (то же в шапке самого скрипта, `setup-piper.sh:10-13`).
- Модели голосов качаются так же без контроля: `setup-piper.sh:54-55`.

## П.3: Нет LICENSE

**Статус: STILL_PRESENT.**

Доказательства: `git ls-files | grep -iE '^(licen[cs]e|copying|notice)'` — пусто. Файла LICENSE в HEAD нет.

---

## Новые находки (в аудите 2026-10-29 отсутствовали)

### Н.1: В истории git лежат два бинарника bin/archi по ~8,98 МБ (подтверждено)

Задание требовало подтвердить новую находку аудита. Подтверждено полностью:

- Добавлен: коммит `9f42f62` (2026-10-27, «feat(archi): initial import of 2026 dual-node Go DevOps agent from Z.ai GLM-5.3») — blob `abcc240b7d43ae7b5207fe48566914fcfca357c6`, **8 982 688 байт (≈8,98 МБ)**.
- Заменён (пересобран): коммит `8d2e5ed` (2026-10-27, «feat(agent): integrate mentor persona…») — blob `bcdaa1e88a40aabb8b565bc0fe7f5da787155365`, тот же размер 8 982 688 байт.
- Удалён из дерева: коммит `174ddf0` (2026-10-27, «feat(voice): implement universal STT/TTS…»); сейчас `bin/` игнорируется (`.gitignore:7` — `bin/`).
- Команды: `git log --all --diff-filter=A --oneline -- bin/` → `9f42f62`; `--diff-filter=D` → `174ddf0`; размеры — `git cat-file -s <sha>`.
- Проверка блоба на секреты (strings): токенов/ключей внутри не обнаружено. Это «просто» ~18 МБ мёртвого веса истории (два блоба) + сам факт бинарников в истории; для очистки нужен filter-repo/rewrite, простое удаление в HEAD уже сделано.
- Единственные блобы >1 МБ во всей истории — эти два `bin/archi` (проверено `git cat-file --batch-check` по всем объектам).

### Н.2 (средний риск): «Первый написавший = владелец» при пустом allowlist

- `internal/telegram/bot.go:147-160` — `IsAllowed`: «Если список allowed пуст, первый написавший пользователь автоматически становится авторизованным владельцем и администратором бота» (код: `b.allowed[userID] = true; b.admins[userID] = true`).
- `internal/config/config.go:734-738` — валидация сознательно не считает пустой allowlist ошибкой.
- При этом у агента есть `local_bash`/`remote_bash` на ноутбук и прод-VM GCP (`internal/tools/bash.go:14-44`) и SSH-мост (`internal/bridge/ssh.go`). Если бот задеплоен с незаполненным `telegram.allowed`, любой случайный/злоумышленный Telegram-пользователь, написавший ботом первым, получает полный контроль DevOps-агента (deny/approval-паттерны смягчают, но не исключают: approval спрашивает у этого же «владельца»).
- Штатный `deploy/config.yaml:126-129` содержит заглушку `allowed: [000000000]` — не пусто, т.е. режим не активен, пока владелец не очистит список вручную; риск именно race при первом запуске.

---

## Не проверено

- Прод-состояние `/etc/archi/config.yaml` на ноутбуке/VM (клонируется из репо или правился вручную — локальной копии нет).
- Установка piper на реальной машине (исполнение скриптов запрещено инструкцией).
- Работоспособность `sudo make deploy` / systemd-юнитов (не запускалось).
- GitHub-метаданные (issues/PR) — api.github.com не использовался.
