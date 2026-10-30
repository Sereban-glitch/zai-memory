# tubecook — верификация аудита 2026-10-29

- Дата верификации: 2026-10-30 (забег Z.ai).
- Локальный клон: `repos/tubecook`, ветка `main`, HEAD `2aab325` (2026-10-20 19:47:37 +0300, «fix(tts): robust synthesize with retry…»).
- Рабочее дерево чистое, соответствует HEAD.
- **Коммиты после аудита: нет.**

## П.1 — Цифры бенчмарка в README против bench_report

**Статус: STILL_PRESENT (строки точные; в JSON номера строк фактические 8/29/50 вместо 8/28/48 — в допуске ±3)**

README (все строки точные):
- `README.md:48` — «tests/  128 тестов (pytest) — вся логика без сети»;
- `README.md:123` — «81 из 81 теста пройдены (100% green)»;
- `README.md:124` — «Замер рендера VA-API (`/dev/dri/renderD128`): **204 сек (~3.4 мин)**, пиковое ОЗУ **926 МБ** (с запасом внутри лимита 3 ГБ)»;
- `README.md:125` — «Замер рендера CPU (`libx264`): 217 сек, пиковое ОЗУ 1.21 ГБ»;
- `README.md:127` — «Полный отчет замера: bench_report_20261019_0159.json».

Факты по `bench_report_20261019_0159.json`:
- `:8` — `"wall_s": 199.2` (bench_base_cpu); `:29` — `"wall_s": 185.1` (bench_base_vaapi); `:50` — `"wall_s": 187.3` (bench_extended_cpu). Значения 199.2/185.1/187.3 совпадают с отчётом аудита.
- Полей памяти **нет вообще**: единственные размерные поля — `master_mb` (размер мастер-файла): 4.03 (:17), 16.79 (:38), 4.27 (:59). «926 МБ» и «1.21 ГБ» из README артефактом не подтверждаются.
- Ни одно из чисел README не совпадает с JSON: VA-API 204 с против 185.1 (Δ≈19 c), CPU 217 с против 199.2 (Δ≈18 c). При этом `scripts/hp_bench.sh:5` заявляет «замеряет время и пиковую RAM, пишет bench_report», а `:30` извлекает `Maximum resident set size` — но в JSON эти данные не попадают (печать только в консоль).
- Противоречие «81 vs 128» подтверждено; фактически в `tests/` определено **123 функции** `def test_*` — т.е. обе цифры README не соответствуют текущему дереву.

## П.2 — env.example без 11 переменных config.py

**Статус: STILL_PRESENT**

- `config.py:250-256` (7 шт.): `TUBECOOK_DOMDARA_ENABLED` (:250), `TUBECOOK_DOMDARA_FEED_URL` (:251), `TUBECOOK_DOMDARA_POLL_INTERVAL` (:252), `TUBECOOK_DOMDARA_LOOKBACK_DAYS` (:253), `TUBECOOK_DOMDARA_SEED_MODE` (:254), `TUBECOOK_DOMDARA_TTL_DAYS` (:255), `TUBECOOK_DOMDARA_USER_AGENT` (:256); `config.py:259-261` (4 шт.): `AI_PROXY_URL` (:259), `AI_PROXY_KEY` (:260), `REPORT_AI_MODEL` (:261), `AI_MODEL` (:261) — итого **11 отсутствующих в env.example** (в файле 32 ключа). Бонус вне диапазона отчёта: fallback `TELEGRAM_BOT_TOKEN` на `config.py:264` тоже не задокументирован.
- `config.py:181` — `domdara_enabled: bool = True` (строка точная) — сбор Домдары включён по умолчанию; активируется в демоне `service.py:97` (`if cfg.domdara_enabled:`).

## Новые находки

1. **Нет CI вообще.** В git отслеживается единственный dot-файл `.gitignore`; каталога `.github/` (workflows) нет — ни тесты (123 функции, включая сторожевой `tests/test_no_secrets.py`), ни линт не запускаются автоматически ни на push, ни на PR. Для «образцового» репо с заявленным «100% green» в README это процессная дыра: регресс фиксируется только локальным `scripts/run_tests.sh`.
2. **Сбор Домдары по открытому HTTP.** `tubecook/config.py:182` и `:251` — дефолт `domdara_feed_url = "http://domdara.org/feed/"`; `tubecook/ingest/domdara.py:93` (`urllib.request.urlopen`) тянет и ленту, и полный HTML статей без шифрования. MITM может подменить контент, который уходит в ИИ-разбор и далее в публикуемые видео (связка подтверждается и README bot_skaner_saitov:96 — «cert certificate is broken on https», т.е. осознанный компромисс, не задокументированный в env.example).

## Не проверено

- Реальное число собранных pytest-тестов (коллекция с параметризацией) — код не запускался (read-only).
- Происхождение цифр «204/217/926 МБ» — возможно, другой прогон hp_bench.sh, артефакта в репо нет; подтвердить невозможно.
- Состояние прод-инсталляции по deploy/*.service (Workdir `%h/projects/tubecook_v4` — на целевой машине).
