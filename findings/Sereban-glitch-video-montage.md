# Sereban-glitch-video-montage — верификация аудита 2026-10-29

- Дата проверки: 2026-10-30. Проверка ТОЛЬКО ЧТЕНИЕ (код не запускался, деревья не менялись).
- HEAD: `1490f17` (2026-10-24, ветка `main`), рабочее дерево чистое.
- **Коммиты после аудита: нет** (единственный коммит репо — 2026-10-24; аудит 2026-10-29).

E-mail в цитатах редактирован (первые 4 символа + длина).

---

## П.1: Нет requirements.txt — зависимости без манифеста и пинов

**Статус: STILL_PRESENT** (строки совпали точно, ±0).

Доказательства:
- Манифест отсутствует: `git ls-files | grep -iE 'requirements|pyproject|setup\.(py|cfg)|Pipfile|poetry'` — пусто.
- Импорты сторонних пакетов без фиксации версий:
  - `scripts/transcribe.py:10` — `from faster_whisper import WhisperModel` (дубль: `scripts/long/transcribe.py:10`);
  - `scripts/make_thumbnail.py:6` — `from PIL import Image, ImageDraw, ImageEnhance, ImageFont` (дубль: `scripts/long/make_thumbnail.py:6`);
  - `scripts/upload_youtube.py:9` — `from googleapiclient.discovery import build` (дубли: `scripts/publish/upload_youtube.py:9`, см. также `:7-11` google.auth/oauth2);
  - плюс `numpy` (`scripts/transcribe.py:9`).
- `scripts/setup_venv.sh:9` — `.venv/bin/pip install faster-whisper google-api-python-client google-auth-oauthlib Pillow numpy` — установка «по памяти», без версий.
- `RESTORE.md:37` — та же команда установки без манифеста (инструкция восстановления среды).

## П.2: Реальный e-mail в login_hint

**Статус: STILL_PRESENT** (все строки совпали точно, ±0).

Доказательства (e-mail редактирован: `daer…@gmail.com`, 18 символов):
- `profiles/humanrightszp.json:10` — `"login_hint": "daer…@gmail.com"`.
- `templates/config.example.json:50` — `"login_hint": "daer…@gmail.com"` — в «примере», т.е. расходится с назначением шаблона.
- `templates/long.config.example.json:50` — `"login_hint": "daer…@gmail.com"`.
- `RESTORE.md:55` — «выбери канал `daer…@gmail.com`» — e-mail и в инструкции.
- `README.md:39` — «В репозитории **нет и не будет** токенов авторизации, паролей, секретных ключей или персональных видеозаписей» — заявление ложно по духу: персональный e-mail владельца канала присутствует в 4 местах (выше). Ср. `SKILL.md:84` — «профили … **не содержат секретов**» (формально так, но login_hint — персональные данные).
- Контекст: `profiles/humanrightszp.json:2-5` — channel_id/handle/telegram канала «Свободные Люди Запорожья» (публичные идентификаторы, не редактировал).

## П.3: Нет LICENSE

**Статус: STILL_PRESENT.**

Доказательства: `git ls-files | grep -iE '^(licen[cs]e|copying|notice)'` — пусто.
При этом `RESTORE.md:22` предписывает `git clone https://github.com/Sereban-glitch/Sereban-glitch-video-montage.git …` — пуллику без лицензии использовать нельзя (all rights reserved по умолчанию).

---

## Новые находки (в аудите 2026-10-29 отсутствовали)

### Н.1: Избыточный OAuth-скоп youtube.force-ssl для задачи «только загрузка»

- `scripts/auth_youtube.py:9-12` и идентичная копия `scripts/publish/auth_youtube.py:9-12`:
  ```python
  SCOPES = [
      "https://www.googleapis.com/auth/youtube.upload",
      "https://www.googleapis.com/auth/youtube.force-ssl",
  ]
  ```
  `youtube.force-ssl` — это полный read/write-менеджмент канала (удаление видео, изменение метаданных/плейлистов, модерация). Пайплайну нужны: upload (загрузка) и channels.list(mine=true) для сверки канала (`scripts/publish/upload_youtube.py:51-55`) — для последнего достаточно read-only-скопа (`youtube.readonly`). Выданный по согласию токен с force-ssl (лежит в `token.json`, живёт до отзыва) даёт компрометируемому токену право удалить весь канал. Нужно сузить скопы — иначе «минимальные привилегии» OAuth нарушены.

### Н.2 (сопутствующая, инженерная): весь пайплайн продублирован в 3 экземплярах без общей точки

- `scripts/*.py` == `scripts/long/*.py` == `scripts/publish/*.py` — побайтово идентичны (проверено `diff`: auth_youtube.py, upload_youtube.py, transcribe.py, make_thumbnail.py — все IDENTICAL); плюс специализированные `scripts/short/*`.
- Следствие уже видно в Н.1: исправление (например, сужение SCOPES) придётся делать минимум в двух файлах (`scripts/auth_youtube.py` и `scripts/publish/auth_youtube.py`), иначе копии разъедутся. Устранить дублирование (общий модуль/пакет) нужно до первых же правок логики.

---

## Не проверено

- Фактическое поведение `pip install` без пинов на свежем окружении (исполнение кода запрещено инструкцией).
- Содержимое локального `~/.config/youtube/client_secret.json` / `token.json` на машинах владельца (вне репо, в git отсутствуют — `.gitignore:2-3` `token*.json`, `client_secret*.json`, в tracked-файлах их нет).
- GitHub: issues/PR/форки (api.github.com не использовался).
