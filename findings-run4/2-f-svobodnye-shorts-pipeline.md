# Аудит svobodnye-lyudi-shorts-pipeline — главный агент (Task 2-f)

**Репозиторий:** https://github.com/Sereban-glitch/svobodnye-lyudi-shorts-pipeline · создан 2026-10-31 07:52:38 +0200 (коммит b5faf9f, единственный) · 23 файла, 3 080 строк · ветка main · PR: 0 · теги: 0.
**Контекст:** обнаружен этим забегом (4) сверкой профиля — 21-й публичный репозиторий аккаунта; забегам 1–2 неизвестен. Конвейер YouTube Shorts «Свободные Люди Запорожья»: Google Flow (Omni 1.1 Flash) через Chrome CDP, Edge-TTS (ru-RU-DmitryNeural), Faster-Whisper, FFmpeg, Playwright, YouTube Data API.

## Проверено и чисто
- Секреты: grep по паттернам (Telegram-токены, AIza…, IP-адреса, e-mail) — 0 попаданий; config.example.json содержит только плейсхолдер `channel_owner@gmail.com` и публичный channel_id.
- ПДн: манифесты ссылаются на публичные судебные решения (номера дел, названия судов, legislation.govt.nz) — персональных данных частных лиц нет.
- requirements.txt закрывает все сторонние импорты (edge-tts, playwright, faster-whisper, Pillow, bs4, requests, numpy, google-api-*, проверено grep'ом импортов).
- Коммит от noreply-адреса GitHub (sereply-noreply@users.noreply.github.com) — e-mail не утёк.
- .gitignore закрывает token.json, client_secret*.json, *.pickle, *.pem, *.key, .env*, config.json, media.
- Положительный паттерн: проверка канала в upload_youtube.py:73-74 (`expected_channel_id` — «чужой канал» → SystemExit) — его нет в старом Sereban-glitch-video-montage.

## Новые улучшения

### P1-1 [security] Весь WordPress REST API domdara.org по голому HTTP
- `scripts/case_queue.py:38` — `SITE = "http://domdara.org"`; `:39` — `API = f"{SITE}/wp-json/wp/v2"`; fetch (`:134-137`, urlopen) используется sync/list/next/inspect/scaffold.
- Тот же MITM-класс, что известный пункт tubecook №7 (http://domdara.org/feed/), но здесь больше поверхности: синк всего каталога постов рубрики «Право», полные тексты решений, номер дела/суд/дата (CASE_RE/COURT_RE/DATE_RE :70-127) → заголовки и тексты идут в сценарии публично выкладываемых Shorts. Подмена ленты = подмена контента канала.
- Фикс: `https://domdara.org`.

### P1-2 [bug] scaffold гарантированно падает: ссылается на файлы, которых нет в репо
- `scripts/case_queue.py:407` — `shutil.copy(skill / "templates" / "manifest.example.json", manifest_path)` при `skill = Path(__file__).resolve().parent.parent` (= корень репо). Каталога `templates/` **в git ls-files нет** (23 файла — все перечислены) → FileNotFoundError.
- `:418-420` — `assets/bgm_legal.mp3` — каталога assets/ нет (фича тихо мертва, есть проверка exists()).
- `:430` — финальное сообщение «запустить make_short.sh» — файла make_short.sh в репо нет (README quickstart использует отдельные python3-команды).
- Фикс: закоммитить templates/manifest.example.json (или генерировать из manifests/), assets/ с лицензионной музыкой, поправить сообщение.

### P2-3 [legal] README заявляет MIT, файла LICENSE нет
- `README.md:145` — «📜 Лицензия — MIT License»; `git ls-files | grep -i lic` — пусто. Тот же паттерн, что известный пункт AgroBot-external (package.json «MIT» без файла). Публичный код без файла лицензии юридически «все права сохранены» — заявление README не имеет силы.
- Фикс: добавить файл LICENSE (MIT, год 2026, имя владельца).

### P2-4 [bug] upload_youtube.py требует metadata.json, который ничто не создаёт
- `scripts/upload_youtube.py:55` — `meta = json.loads((proj / "metadata.json").read_text(...))`; поля: title, description, tags, categoryId, defaultLanguage, defaultAudioLanguage, privacyStatus, selfDeclaredMadeForKids, embeddable (`:76-90`).
- Ни один скрипт репо не пишет metadata.json (grep — единственные упоминания в самом upload_youtube.py); манифесты manifests/*.json не содержат ни одного из этих полей (проверено структурой: project_id, channel, topic, source_url, case_number, court_name, decision_date, target_resolution, banner_text, card_badge, scenes).
- Шаг 7 quickstart («Загрузка на YouTube (unlisted)») из README падает FileNotFoundError для проектов, собранных по документированному пути.
- Фикс: metadata.example.json + генератор из манифеста (title из topic, description из source_url и т.д.).

### P2-5 [security] OAuth-скоп шире задачи: youtube.force-ssl при задаче «только загрузка»
- `scripts/upload_youtube.py:28-31` — `SCOPES = ["https://www.googleapis.com/auth/youtube.upload", "https://www.googleapis.com/auth/youtube.force-ssl"]`.
- Тот же класс, что известный пункт №20 забега 2 (scripts/auth_youtube.py:9-12 в Sereban-glitch-video-montage): force-ssl даёт полное управление каналом (удаление видео, правка плейлистов, комментарии); для загрузки+thumbnail достаточно youtube.upload.
- Фикс: оставить только youtube.upload; токен, выданный под широким скопом, перевыпустить.

### P2-6 [hygiene] Состояние очереди пишется в репо-каталог, которого нет в .gitignore; докстринг обещает другое место
- `scripts/case_queue.py:41` — `STATE_DIR = Path(__file__).resolve().parent.parent / "state"` → `<репо>/state/` (posts_cache.json, processed.json создаются ensure_state :140-144).
- `.gitignore` (43 строки) не содержит `state/` (grep «state» — 0) → первый `git add .` закоммитит локальное состояние.
- Докстринг `:12-13` обещает `~/.config/opencode/skills/domdara-shorts/state/processed.json` — код пишет в другое место: документация и код противоречат друг другу (след переноса из навыка opencode).
- processed.json содержит project-пути пользователя (`:366` — expanduser().resolve()).
- Фикс: `state/` в .gitignore (или вынести в ~/.config, как обещает докстринг).

### P3-7 [docs] Докстринг generate_audio.py описывает удалённый gTTS-фолбэк
- `scripts/generate_audio.py:7` — «3. Google TTS (gTTS) as reliable automatic fallback»; код `:119-120` — при провале edge-tts падает RuntimeError («Падение для исключения подмены голоса!») — README:90 и SKILL.md (раздел 1.1) описывают поведение правильно (ПОЛНЫЙ ЗАПРЕТ gTTS).
- Фикс: убрать строку из докстринга (мелочь, но это первый файл, который читает контрибьютор).

## Не проверено
- Поддерживает ли domdara.org https на практике — не проверял обращением (сетевой запрос не выполнялся).
- Поведение Chrome CDP на порту 9222 в чужих сетях (README предписывает `google-chrome --remote-debugging-port=9222` — на локальной машине риск ограничен, но при запуске на сервере/порт-форвардинге сессия Google отдаётся любому локальному процессу; класс-риск CDP, не ошибка кода).
- Работоспособность селекторов Google Flow (flow_drive.py) против живого UI — статический анализ.
