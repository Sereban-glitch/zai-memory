# Аудит info-bot-go — субагент 2-b (NEW findings only)

Репозиторий: `/home/z/my-project/zai-audit/repos/info-bot-go`, ветка `main` (41b9b27).
Аудит читательский: код не запускался, изменения не коммитились. Известные находки (8 шт. из ТЗ) не повторяются.
Новых находок: **15** (1 × P0, 6 × P1, 8 × P2). Позитивные проверки (не-findings) — в конце.

---

### [P0][IMAP/Spoofing] Матчинг ответов по подстроке темы delivers произвольное письмо в чужой чат как «официальный ответ»
- Файл: internal/imap/watcher.go:264-277 (проверено), доставка — watcher.go:357-381
- Цитата: `originalSubject = strings.TrimPrefix(originalSubject, "Re: ")…` → `if strings.Contains(all[i].Subject, originalSubject) { return &all[i] }`
- Почему чинить обязательно: `BuildSubject` (internal/email/sender.go:199-201) ставит ВСЕМ запросам одинаковый префикс «Запит на публічну інформацію (ЗУ №2939-VI) — …». Любое письмо на общий ящик (адрес известен каждому органу-получателю и захардкожен как fallback в config.go:134) с темой «Re: Запит» матчится ЛЮБОЙ записью журнала; перебор идёт с конца (`ListAll`), т.е. письмо уходит самому «свежему» пользователю. Крайний случай: тема ровно «Re: » → `originalSubject=""` → `Contains(x, "")==true` для всех записей. Атакующий (в т.ч. «обиженный» орган) отправляет письмо → бот доставляет его текст (и вложения — watcher.go:378-380 пересылает файлы любого типа) в приватный чат жертвы под видом «📨 Отримана відповідь на ваш запит!». Текст и тема письма вставляются без экранирования в `tb.ModeMarkdown` (watcher.go:363-372) → можно вставить кликабельные фишинговые ссылки `[Оплата штрафу](https://evil)`. Это подделка核心-функции бота (доставка официальных ответов).
- Направление фикса: матчить только по `In-Reply-To`/`References` на Message-ID; subject-фолбэк — только точное равенство полной темы + сверка отправителя с `entry.RecipientEmail`; пустой `originalSubject` — не матчить вообще; тело/тему/from прогонять через markdownEscape перед `bot.Send`.

---

### [P1][Web/RateLimit] Слепое доверие X-Forwarded-For — обход всех per-IP лимитов API
- Файл: internal/web/middleware.go:41-56 (проверено)
- Цитата: `if xff := r.Header.Get("X-Forwarded-For"); xff != "" { … xff = xff[:i] … return ip }`
- Почему чинить обязательно: клиентский заголовок берётся первым элементом без проверки доверенного прокси. За Caddy атакующий шлёт `X-Forwarded-For: 1.2.3.4`, Caddy дописывает реальный IP в конец, а код берёт первый (подделанный) адрес. Ротация фейковых XFF = бесконечные новые корзины: обходятся лимиты public/auth/generate/analyze (ТЗ №4, D1), брутфорс `/api/app/auth` (см. следующую находку), флуд «дорогих» AI-эндпоинтов и бесплатных разборов (6/час на пользователя остаётся, но генерация/поиск/создание запросов — нет). При прямом доступе к :8081 (main.go:101) лимитов нет вовсе.
- Направление фикса: `RemoteAddr` как ключ по умолчанию; XFF учитывать только от адреса доверенного прокси (env `TRUSTED_PROXY`), и брать ПОСЛЕДНИЙ элемент цепочки, а не первый.

### [P1][Auth/BruteForce] Коды входа в приложение: 6 цифр, без счётчика попыток, fail-open фолбэк на время
- Файл: internal/applogin/store.go:37-57, 74-81 (проверено); эндпоинт internal/web/server.go:138
- Цитата: `code := random6()` … `// криптоджерело недоступне — не блокуємо вхід, код на основі часу` → `time.Now().UnixNano()%1000000`
- Почему чинить обязательно: пространство кода 10^6, TTL 30 минут, до 100 живых кодов (вероятность попадания на попытку ~1e-4). `/api/app/auth` защищён только per-IP лимитом, который обходится через XFF (пред. находка): ~10^6 запросов за 30 минут (~550 rps) дают вход в чужой аккаунт мини-аппа — профиль с ФИО/e-mail/адресом, история запросов, возможность подать запрос от имени жертвы. Фолбэк `UnixNano%1e6` предсказуем (nanosecond timestamp публично виден в Message-ID писем sender.go:32). `Exchange` не считает неудачные попытки — код не «сгорает» при переборе.
- Направление фикса: 8+ символов из большего алфавита; счётчик неудачных Exchange с инвалидацией кода (напр. 5 попыток); лимит на ЭНДПОИНТ по глобальному ключу, а не только IP; при недоступности crypto/rand — отказ, а не предсказуемый код.

### [P1][Auth/TokenStorage] initData (bearer-креденшл на 24 ч) хранится в localStorage; Android allowBackup=true
- Файл: internal/web/static/assets/index-BzgxKBHD.js:9 (проверено, минифицирован; то же в index-status-67d6c83201ba.js и index-status-9a447b71d167.js); mobile/android/app/src/main/AndroidManifest.xml:5
- Цитата: `localStorage.setItem(`smartzapyt_init_data`,e.initData)` … `return e&&e.initData?e.initData:localStorage.getItem(`smartzapyt_init_data`)||```
- Почему чинить обязательно: сервер принимает подписанный initData 24 часа (internal/web/auth.go:57) — это полноценный токен сессии. Он кладётся в localStorage без TTL/очистки: (а) любой XSS в WebView крадёт токен на сутки; (б) `allowBackup="true"` — токен уезжает в Google Backup / извлекается `adb backup` на старых устройствах; (в) токен не отзывается сервером (нет logout-стейт). Плюс протухший бандл `index-status-9a447b71d167.js` с той же логикой навсегда в бинарнике.
- Направление фикса: sessionStorage + удаление по «застреванию» (age > auth_date окно); серверные короткоживущие сессии с logout; `android:allowBackup="false"` (или backup rules, исключающие WebView-данные); выпилить устаревший бандл.

### [P1][Secrecy/Logging] Токен бота и ключи Gemini утекают в логи через *url.Error при сбоях скачивания
- Файл: internal/bot/handlers/voice.go:406-413 + лог на :139; internal/bot/handlers/analyze.go:292-295 + логи :231/:253; internal/ai/gemini.go:133, 150-153, 232 (проверено)
- Цитата: `url := fmt.Sprintf("https://api.telegram.org/file/bot%s/%s", m.deps.Cfg.BotToken, file.FilePath)` → `log.Printf("[VOICE] download error: %v", err)`; `…generateContent?key=%s` → `fmt.Errorf("gemini API request failed: %w", err)`
- Почему чинить обязательно: при любой сетевой ошибке `http.Get`/`client.Post` возвращает `*url.Error`, чей `.Error()` содержит ПОЛНЫЙ URL — с `bot<TELEGRAM_BOT_TOKEN>` или `?key=<GEMINI_API_KEY>`. Эти ошибки логируются (`[VOICE]`, `[ANALYZE] photo/document download`, `[ANALYZE] AI error` analyze.go:678, `[WEB] analyze error` web/api.go:791). Логи бота уже однажды попали в git (bot-test.log — известно), т.е. поверхность утечки реальная: токен бота = полный контроль над ботом и всеми его правами.
- Направление фикса: выдавать токен/ключ в заголовок (`Authorization: Bearer`) вместо query; при логировании ошибок оборачивать/редактировать URL (strip query); в downloadVoice заодно добавить проверку StatusCode и LimitReader (сейчас io.ReadAll без лимита и без проверки 200 — voice.go:407-413).

### [P1][Authz/IDOR] POST /api/followups: слаг из клиента без проверки владения — запись в чужие публичные гілки
- Файл: internal/web/requests.go:189-216 (проверено); подпись — internal/dostup/followup.go:15-16
- Цитата: `req.Slug = strings.TrimSpace(req.Slug)` … `link, err := s.dostup.SubmitFollowUp(req.Slug, req.Text)`
- Почему чинить обязательно: все запросы идут через ОДИН общий аккаунт портала, поэтому портал пускает допис в любую «свою» гілку. Бот ограничивает пользователя его гілками (followup.go:74, `FollowUps.List(c.Sender().ID, 8)`), а веб-эндпоинт — нет: любой пользователь мини-аппа дописывает произвольный текст в публичную переписку другого пользователя (slug'ы публичны — видны в /api/search-requests и на портале). Порча чужого официального запроса + возможные юридические последствия для жертвы. Заодно здесь НЕТ moderation.Check (в отличие от handleCreateRequest:97-103) — «острые» тексты летят в портал в обход ТЗ №10.
- Направление фикса: перед SubmitFollowUp проверять `s.followUps` на принадлежность slug пользователю (по аналогии с ботом); прогонять текст через moderation.Check; жёсткий per-user лимит дописей.

### [P1][Abuse/Bypass] Голосовой «🚀 Відправити зараз» обходит модерацию, лимит 3/час и дневной кап 280
- Файл: internal/bot/handlers/voice.go:230-251 (проверено); сравнить internal/bot/handlers/newrequest.go:298-326
- Цитата: `func (m *VoiceModule) handleFastSend…` — внутри НЕТ `m.deps.RateLimit.Allow`, НЕТ `Stats.DailyLimitReached(280)`, НЕТ `moderation.Check` (в отличие от handleSendConfirm)
- Почему чинить обязательно: пользователь с голосовой черновиком (создаётся одной диктовкой, voice.go:162-168) шлёт письма органам с общего ящика без антиспровокационного скрининга ТЗ №10, без 3/час и без 280/день: спам органам от имени общего ящика → блэклистинг ящика/домена, репутационный и юридический риск для проекта. Черновик после отправки не чистится от RateLimit, SentLog пишется — т.е. канал полностью рабочий.
- Направление фикса: вынести общий pre-send guard (rate limit + daily cap + moderation) в одну функцию и вызывать из handleFastSend, handleSendConfirm и moderation-флоу.

---

### [P2][Auth/Logging] В логах — фрагменты HMAC-подписи и user-JSON; auth_date из будущего принимается
- Файл: internal/web/auth.go:33, 87 (проверено), :57
- Цитата: `log.Printf("[AUTH-VALIDATE] user field length: %d, starts with: %s"…)`; `log.Printf("…computed=%s received=%s match=%v", computedHex[:16]+"…", hash[:…]+"…"…)`
- Почему чинить обязательно: в прод-логи пишется префикс пользовательских данных и по 16 hex-символов вычисленной и принятой подписи — материал для офлайн-анализа/корреляции при утечке логов (которые уже однажды оказались в git). Проверка `time.Now().Unix()-authDateInt > 86400` не отсекает auth_date в будущем (отрицательная разность проходит) — окно валидности расширяется подделкой на стороне клиента неограниченно... (клиент не может подделать подпись, но скомпрометированный когда-то initData с будущей датой жил бы дольше 24 ч).
- Направление фикса: убрать эти debug-логи (оставить fact-of-failure); добавить `authDateInt > now+300 → reject`.

### [P2][Auth/Leak-in-URL] initData принимается из query-параметра — креденшл в URL
- Файл: internal/web/api.go:143-144 (проверено); потребитель SSE — internal/web/events.go:67-69
- Цитата: `} else if queryData := r.URL.Query().Get("init_data"); queryData != "" {`
- Почему чинить обязательно: EventSource не умеет заголовки, поэтому подписанный initData (24-часовой токен) кладут в URL `/api/events?init_data=…`. URL с токеном оседает в access-логах обратного прокси (Caddy), в истории/закладках, реферерах. Утечка лога прокси = сутки доступа к аккаунту.
- Направление фикса: короткоживущий одноразовый ticket для SSE (обмен initData → ticket на 30 сек), либо секционные cookies; как минимум — не логировать query на прокси.

### [P2][Portability/InfoDisclosure] Захардкоженные пути прод-VM + отсутствующий в репо PDF-скрипт + сырой вывод в ошибке
- Файл: internal/bot/handlers/newrequest.go:398-417 (проверено)
- Цитата: `exec.Command("/home/u0_a566/pdf_gen_env/bin/python3", "/home/u0_a566/tools/generate_foi_pdf.py", …)` … `output, err := cmd.CombinedOutput(); return nil, fmt.Errorf("PDF generation failed: %s: %w", string(output), err)`
- Почему чинить обязательно: (а) скрипт `tools/generate_foi_pdf.py` в репозитории НЕТ — канал email+PDF гарантированно падает на любом окружении, кроме прод-VM (маскирует реальную работоспособность и вынуждает fallback); (б) `generate_foi_pdf.py` получает ФИО/адрес/e-mail пользователя аргументами командной строки — они видны в `ps`/`/proc` всем пользователям VM; (в) `string(output)` (стектрейс питона с путями) уходит пользователю через `❌ Помилка надсилання: %s` (newrequest.go:339) — раскрытие внутренностей.
- Направление фикса: путь к скрипту/интерпретатору через env (`PDF_GEN_CMD`); включить скрипт в репо; передавать данные через stdin/временный файл 0600; в ошибке — фиксированный текст, вывод — только в лог.

### [P2][InfoDisclosure] Сырые внутренние ошибки показываются пользователям бота
- Файл: internal/bot/handlers/newrequest.go:339; voice.go:250; analyze.go:839; followup.go:306 (все проверено)
- Цитата: `return c.Send(fmt.Sprintf("❌ Помилка надсилання: %s\n\nЧернетка збережена…", err))` (аналогично в остальных)
- Почему чинить обязательно: в err попадают SMTP-адреса/порты (sender.go:84 `SMTP send failed [host:port]: …`), URL с токенами (см. P1 про url.Error), пути файловой системы, HTML портала. Пользователь видит внутреннюю топологию (SMTP-релей, VM-пути) — разведка инфраструктуры без всякого доступа.
- Направление фикса: пользователю — код/человекочитаемое сообщение, `err` — только в log.Printf.

### [P2][Legal/Hygiene] Скрап сайта Grok (xAI) зашит в бинарник и раздаётся с домена бота
- Файл: internal/web/static/__grok/ (icon-180.png, install/styles.css, install/assets/homescreen/* — 30+ файлов, проверено); internal/web/server.go:22-23, 145-156
- Цитата: `//go:embed static/*` + `fileServer := http.FileServer(http.FS(staticFS))`
- Почему чинить обязательно: директория — несвязанный с проектом скрап страницы установки Grok (лого, сплэши). Через `go:embed static/*` она компилируется в бинарник и публично раздаётся по `https://<домен-бота>/__grok/…`: чужой брендинг на своём домене (имперсонация/товарный знак), лишние ~1.1 МБ в бинарнике, мёртвый код. Ссылок на `__grok` в кодовой базе нет (проверено grep'ом).
- Направление фикса: удалить каталог из static (и из истории при следующем перепаковывании); добавить тест, что embed содержит только ожидаемые файлы.

### [P2][Repo/PII] ~38 МБ тяжёлых/чужих артефактов в git, включая скрап отзывов с персональными данными
- Файл: история git (проверено `git rev-list --objects --all` + `cat-file`)
- Цитата (размеры): `info-bot` 9 746 926; `info-bot.bak` 9 586 544; `info-bot.bak2` 9 586 711; `mobile/.../public/reviews.json` 9 030 168; `dostup-probe` (в корне, tracked) 7 956 479; `xlsx.full.min.js` 881 727; `market_insights.json` 383 949; ~15 устаревших `index-*.js` по ~360-380 КБ
- Почему чинить обязательно: (а) три скомпилированных бинарника самого бота в истории — в скомпилированных Go-бинарниках остаются строковые константы, риск утечки секретов при сборках тех коммитов; (б) `reviews.json` — 2 797 скрапнутых отзывов агротехники (Case IH, John Deere…) с полями `comment_author`, `source_url`, `full_text` — персональные данные третьих лиц из чужого проекта, ранее упаковывались в APK (`app/src/main/assets/public/`); (в) `xlsx.full.min.js` — vendored SheetJS с CDN, поставляется офлайн-копией без контроля версий (supply-chain); (г) дубли бандлов раздувают клон. Известная находка про `tools/dostup-probe` — это отдельный файл; здесь новые: `info-bot{,.bak,.bak2}`, корневой `dostup-probe`, reviews/market_insights/xlsx.
- Направление фикса: `git filter-repo` (или BFG) для бинарников и reviews.json; пересборка APK-ассетов только из актуального mini-app; `.gitignore` на `info-bot*`, `*.apk`, `dostup-probe`.

### [P2][Ops/InfoLeak] Дев-артефакт my_requests.html: CSRF-токен и идентичность общего аккаунта портала
- Файл: my_requests.html:6, 34-36 (проверено); то же slug в internal/dostup/testdata/request_moz_attachments.html
- Цитата: `<meta name="csrf-token" content="CtrgkbUy…"/>` … `href="https://dostup.org.ua/feed/user/hromadskyi_monitorynh"`
- Почему чинить обязательно: файл — скрап страницы запросов общего аккаунта «Громадський моніторинг» (`hromadskyi_monitorynh`): раскрывает, какой аккаунт портала делает ВСЕ запросы пользователей бота (обезличивание канала теряется — можно скармливать порталу целевые «запросы», мониторить очередь и лимиты аккаунта, соц-инжинирить саппорт портала). CSRF-токен в файле связан с той же сессией. Аналогичный слаг в тестовой фикстуре.
- Направление фикса: удалить файл из HEAD и истории; тестовые фикстуры — обезличивать (dummy-слаг, пустой csrf-token).

### [P2][Config] Ещё небезопасные/мёртвые дефолты в config.go (сверх известных ADMIN_ID и fallback-почты)
- Файл: internal/config/config.go:130, 132 (проверено)
- Цитата: `ChannelID: getEnvOrDefault("CHANNEL_ID", "@svobodnye_ludi_zp")`; `MiniAppURL: getEnvOrDefault("MINI_APP_URL", "https://mini-app-deployment.vercel.app/")`
- Почему чинить обязательно: (а) дефолт канала — реальный живой Telegram-канал: связывает опенсорс-репозиторий с прод-каналом (модерация/copilot пишут туда), при деплое без env посты пойдут в чужой/прод канал; (б) дефолт MiniAppURL — чужой vercel-деплой: кнопка WebApp у бота открывает левый хост, CORSAllowlist() его осознанно игнорирует (config.go:255 «минуя стандартный мёртвый дефолт») — мёртвый, но опасный при частичном конфиге дефолт.
- Направление фикса: пустые дефолты + валидация при старте («MINI_APP_URL обязателен, если включён WebApp»); канал — только явный env.

### [P2][Mail/Logging+DoS] PII корреспондентов в логах IMAP; чтение тела письма без лимита
- Файл: internal/imap/watcher.go:197, 208 (проверено), :311 (`io.ReadAll(p.Body)` без лимита), :340 (лог имени вложения)
- Цитата: `log.Printf("[IMAP] bounce detected from %s, subject: %s"…)`; `log.Printf("[IMAP] no match for message from %s, subject: %s"…)`
- Почему чинить обязательно: каждый входящее письмо (тема+отправитель — персональные данные корреспондентов и пользователей) пишется в лог целиком; вложения читаются `io.Copy` без ограничения (Gmail ограничивает ~25 МБ, но self-hosted SMTP — нет: исчерпание диска). Отдельно: `matchReply` и логи — единственное место, где видно, что unmatched письма обрабатываются и помечаются прочитанными — ок.
- Направление фикса: логировать только хэш/домен отправителя и длину темы; `io.LimitReader` на inline и attachment (напр. 25 МБ).

---

## Позитивные проверки (проверено, проблемы НЕТ)
- XSS в мини-аппе: React-бандл без `dangerouslySetInnerHTML` (есть тест blue-design.test.mjs:65); legacy mini-app/index.html экранирует `escHtml()` (index.html:1698-1702, 1472-1475).
- Заголовочные инъекции SMTP: `To/Cc` с CRLF блокируются `net/smtp` `validateLine` на RCPT-стадии; `Subject` — B-encoded; `Reply-To` = env/валидированный profile email (dostup.go:520-532 `validEmail` режет `\r\n`).
- Path traversal вложений IMAP: `sanitizeFilename` (watcher.go:453-473) — Base + whitelist.
- SSRF в DownloadAttachment: только относительные пути, внешние редиректы отклоняются, лимит байт (attachments.go:137-177).
- Stars-платежи: payload валидируется, uid сверяется с отправителем, дедуп по charge_id (stars.go:208-216).
- Session/sentlog/moderation/stars/dostup-файлы пишутся с 0600 (session.go:429, sentlog.go:87, store.go:97).
- Файлы сессий: ключ `user-%d` — инъекций пути нет.

## Сводка
| # | Приоритет | Кратко |
|---|---|---|
| 1 | P0 | IMAP матчинг по подстроке темы → подделка «ответа органа» в чужом чате (+MD-инъекция, +вложения) |
| 2 | P1 | X-Forwarded-For из запроса → обход всех rate-limit API |
| 3 | P1 | 6-значные коды /api/app/auth без антибрутфорса, fail-open фолбэк |
| 4 | P1 | initData в localStorage (24 ч токен) + allowBackup=true |
| 5 | P1 | Токен бота / ключи Gemini в логах через *url.Error |
| 6 | P1 | /api/followups без проверки владельца слага (+без модерации) |
| 7 | P1 | Голосовая быстрая отправка обходит модерацию/лимиты/кап |
| 8-15 | P2 | auth-логи подписей; init_data в query; хардкод /home/u0_a566; сырые ошибки юзерам; __grok в бинарнике; ~38 МБ блобов + PII reviews.json; my_requests.html (CSRF+аккаунт портала); дефолты ChannelID/MiniAppURL; PII в IMAP-логах |
