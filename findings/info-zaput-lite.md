# info-zaput-lite — верификация аудита 2026-10-29 (проверка 2026-10-30)

**Тип:** Python, Telegram-бот информационных запросов (Уkrainian FOI-запросы), 4 файла.
**Ветка/коммиты:** `main` @ `51f4178` («Fix: Downgrade dependencies for Python 3.9 compatibility»).
**Коммиты после аудита: нет** (`git log --all --since=2026-10-28 --oneline` — пусто).

---

## Пункт 1 — Нет .gitignore, утечка ПДн при `git add .`
**Вердикт: STILL_PRESENT**

- .gitignore отсутствует: в рабочей директории и в `git ls-files` только 4 файла (`ai.py`, `db.py`, `main.py`, `requirements.txt`).
- `db.py:5` — `DB_PATH = "info_zaput.db"` (относительный путь → файл создаётся в корне репо при `init_db()`, `main.py:127`).
- `db.py:31-34` — `INSERT INTO requests (message_id, chat_id, user_id, recipient_name, recipient_email, subject) VALUES (?, ?, ?, ?, ?, ?)` — факт записи chat_id/user_id/e-mail. Таблица (db.py:12-24) также хранит `recipient_email`, `subject`, статусы.
- `.env` в репо сейчас нет, но при его создании и первом `git add .` — утечка БД с ПДн и креденциалов (защиты нет).
- Примечание: ПДн-поля фактически пишет и вызов `log_request` из `main.py:119` (chat.id, from_user.id, subject).

## Пункт 2 — Нет README, недокументированные переменные, ADMIN_ID=0, python-dotenv не используется
**Вердикт: STILL_PRESENT** (уточнение по числу переменных: фактических 8, а не 6)

- README отсутствует (в репо 4 файла, см. выше) — переменные нигде не задокументированы.
- Фактическое число `os.getenv` в коде — **8 вызовов / 8 уникальных переменных**:
  - `main.py:11` `TELEGRAM_TOKEN`; `main.py:12` `ADMIN_ID` (дефолт `"0"`); `main.py:13` `SMTP_SERVER`; `main.py:14` `SMTP_PORT`; `main.py:15` `EMAIL_USER`; `main.py:16` `EMAIL_PASS`; `main.py:17` `IMAP_SERVER`;
  - `ai.py:37` `GEMINI_API_KEY`.
  - Из них без дефолта (обязательные): `TELEGRAM_TOKEN`, `EMAIL_USER`, `EMAIL_PASS`, `GEMINI_API_KEY` (4); с дефолтами: `ADMIN_ID`, `SMTP_SERVER`, `SMTP_PORT`, `IMAP_SERVER`.
- `main.py:12` — `ADMIN_ID = int(os.getenv("ADMIN_ID", "0"))`; проверки в каждом хендлере `if message.from_user.id != ADMIN_ID: return` (`main.py:62, 67, 77, 87`) без логов/ответа → при незаданном ADMIN_ID бот молча игнорирует все сообщения. Подтверждено.
- `requirements.txt:3` — `python-dotenv` объявлен, но `load_dotenv`/`dotenv` в коде не вызывается ни разу (grep по `load_dotenv|dotenv` даёт единственное совпадение в requirements.txt:3). Подтверждено.

## Пункт 3 — Захардкоженный e-mail, recipientHint игнорируется
**Вердикт: STILL_PRESENT**

- `main.py:117` — `target_email = "publicinquiry69@gmail.com"` (точное совпадение; комментарий на 116: «В реальности email должен быть в data или справочнике»).
- `main.py:118` — `send_mail(target_email, data['subject'], data['body'])`: отправка всегда на зашитый адрес.
- `ai.py:16` — промпт требует `"recipientHint": "конкретний орган или 'Офіс Генерального прокурора'"`; фактически `recipientHint` используется только для отображения (`main.py:98`) и логирования (`main.py:119`), на адрес доставки не влияет → запрос «в прокуратуру» уходит на личный ящик publicinquiry69@gmail.com. Подтверждено.

---

## Новые находки

### NEW-1 (100%, функциональный баг): `message_id` всегда `'unknown'` → UNIQUE-нарушение со второй отправки
- `main.py:23-26` — `MIMEText` создаётся с заголовками Subject/From/To, `Message-ID` не задаётся; `smtplib.SMTP.send_message` не возвращает сгенерированный сервером Message-ID в локальный объект.
- `main.py:32` — `return msg.get('Message-ID', 'unknown')` → фактически всегда `'unknown'`.
- `db.py:14` — `message_id TEXT UNIQUE` в таблице `requests`.
- Следствие: первая отправка пишет строку с `message_id='unknown'`, каждая следующая `log_request` (`main.py:119`, `db.py:31-34`) падает с `sqlite3.IntegrityError: UNIQUE constraint failed` → пользователь получает «❌ Помилка пошти», письмо при этом уже отправлено (лог не пишется, трекинг статусов/ответов сломан).

### NEW-2 (100%, отсутствие авторизации): callback-хендлер не проверяет отправителя
- `main.py:108-109` — `@bot.callback_query_handler(func=lambda call: True)` / `def callback_query(call):` — единственный обработчик без проверки `call.from_user.id == ADMIN_ID` (в остальных хендлерах проверка есть: main.py:62, 67, 77, 87).
- `main.py:110-118` — при `call.data == "send_mail"` сразу отправляется письмо `bot.last_draft` — черновик, хранящийся глобально на объекте бота (`main.py:105` — `bot.last_draft = data`). Любой источник callback (например, бот добавлен в группу, где есть сообщение с кнопкой) инициирует реальную SMTP-отправку без авторизации, плюс глобальное состояние перезаписывается каждым новым черновиком (race/подмена адресата в multi-user сценарии).

---

## Не проверено
- Фактическая отправка писем/работа SMTP/IMAP (код не запускался — режим read-only).
- Поведение `gemini-2.5-flash` (ai.py:7) — API не вызывался.
- История Telegram-чата/ящика publicinquiry69@gmail.com — вне репозитория.
