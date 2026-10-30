# gophergpt — верификация аудита 2026-10-29

Проверка: 2026-10-30 (забег Z.ai). Работа только чтение, код не запускался.

## Коммиты после аудита: нет

HEAD = `a900c2d` (2026-09-03 02:59:00 +0300, «v0.1.0 release artifacts: 5 platform binaries + SHA256SUMS; release build tooling»), ветка `main` (единственная, `origin/HEAD -> origin/main`), совпадает с `origin/main`.

## Пункт 1. dist/ + SHA256SUMS без CI — STILL_PRESENT (полностью)

- dist/ — **6 файлов, 13 357 705 байт** (5 бинарников 13 357 210 + SHA256SUMS.txt 495), подтверждено stat'ом: darwin-amd64.gz 2 698 870; darwin-arm64.gz 2 653 743; linux-amd64.gz 2 684 306; linux-arm64.gz 2 583 085; windows-amd64.zip 2 737 206.
- Включены сознательно: .gitignore:24-26 — `# релизные артефакты dist/ — исключение из правил выше` / `!dist/` / `!dist/*`.
- SHA256SUMS сертифицируют локальную сборку неизвестной машины: tools/build_release.sh — `go build -trimpath -ldflags='-s -w' … -o "$BIN/gophergpt"` в /tmp и в конце `sha256sum gophergpt-$V-* > SHA256SUMS.txt`; никакого CI/attestation, единственный «контроль» — smoke-тест linux/arm64 на той же машине (build_release.sh:16-20).
- `.github/` отсутствует (ls -la корня; в git ls-files workflow-файлов нет). Тестов в репо 23 `func Test*` (main_test.go:8, data/data_test.go:15) + ~28 `func Example*` (pkg/*_test.go) — автозапуска нет.
- Релизов на GitHub нет — принято от главного агента (проверено им; api.github.com мной не ходил).

### Дубликат в Sereban-glitch.github.io — ПОДТВЕРЖДЁН байт-в-байт

sha256sum по обеим копиям (5 файлов) полностью совпали:

| Файл | SHA256 (одинаков в обоих репо) |
|---|---|
| gophergpt-v0.1.0-darwin-amd64.gz | e331a84d…bed4cad9 |
| gophergpt-v0.1.0-darwin-arm64.gz | 2e8e741e…128dd99 |
| gophergpt-v0.1.0-linux-amd64.gz | 60477796…acaa3f2ff |
| gophergpt-v0.1.0-linux-arm64.gz | d95dcb1e…24e7c238 |
| gophergpt-v0.1.0-windows-amd64.zip | b6da4e2d…fd74226 |

- Оба SHA256SUMS.txt идентичны по содержимому (по каждой строке — 2 совпадения при слиянии).
- data/jules_verne.txt и data/vocab в двух репо тоже идентичны (`cmp` → IDENTICAL).
- 13 357 705 байт бинарников продублированы в Git дважды (gophergpt + pages-репо).

## Пункт 2. data/fairy_tales.txt — STILL_PRESENT (все ссылки подтверждены)

- data/fairy_tales.txt — 12 296 396 байт (12,3 МБ) — в git отслеживается.
- Вшиты через go:embed только jules_verne.txt и vocab: data/data.go:15-18 — `//go:embed jules_verne.txt` → `dataset string`; `//go:embed vocab` → `vocab string`. fairy_tales в embed НЕТ.
- fairy_tales.txt используется только через флаг: main.go:37 — `corpus = flag.String("corpus", "", "путь к файлу корпуса .txt (пусто = вшитый корпус Жюль Верна)")`; main.go:70-71 — `if *corpus != "" { raw, err := os.ReadFile(*corpus)`.
- Имя `fairy_tales` не встречается ни в одном .go/.md (rg по коду и докам — 0 ссылок; в README.md:101 только общий пример `go run . -corpus my.txt`).
- data/fairy_vocab — 217 473 байта (217 КБ) — не упоминается нигде (rg "fairy_vocab" вне самого файла — 0 совпадений; не в embed, не во флагах, не в доках).

Итого в git мёртвым грузом: 12,3 МБ fairy_tales.txt + 217 КБ fairy_vocab ≈ 12,5 МБ данных, недостижимых штатным путём.

## Новые находки

1. **Форк не переименовал go-модуль: `module github.com/zakirullin/gpt-go` (go.mod:1).** Репозиторий публично называется Sereban-glitch/gophergpt, но путь модуля остался апстримовым; main.go:13-14 импортирует `github.com/zakirullin/gpt-go/data` и `…/pkg`. Следствие: `go install github.com/Sereban-glitch/gophergpt@latest` неработоспособен, любой внешний import потянет чужой (апстримовый) код, а не эти исходники. Уровень: 100% нужно исправить до любых «релизов».
2. **Релизные архивы распространяются без лицензии.** LICENSE — MIT © 2025 Artem Zakirullin (текст в LICENSE:1-3: «MIT License … Copyright (c) 2025 Artem Zakirullin»), но tools/build_release.sh пакует ТОЛЬКО бинарник (для zip: `z.write('/tmp/gg_bin/gophergpt.exe', 'gophergpt.exe')`; для .gz — голый `gzip -9 -c`), без копии LICENSE/README. MIT требует включать уведомление о правах во «all copies or substantial portions of the Software»; публикация 5 бинарников на GitHub Pages (см. дубликат в Sereban-glitch.github.io) без текста лицензии — нарушение условий MIT.

## Не проверено

- Наличие GitHub Releases (принято от главного агента; api.github.com не использовался, HTML-страницу не открывал).
- Восстановимость сборки: воспроизводимость бинарников из текущих исходников (код/сборка не запускались — ограничение ТЗ).
- Совпадение содержимого SHA256SUMS.txt с фактическими sha256 бинарников (совпало в обеих локальных копиях; происхождение самих файлов остаётся словесным).
