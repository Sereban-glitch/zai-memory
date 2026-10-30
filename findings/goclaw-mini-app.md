# goclaw-mini-app — верификация аудита 2026-10-29 (сверка 2026-10-30)

Репозиторий: Sereban-glitch/goclaw-mini-app (локальный полный клон).
Стек по факту: Next.js **16.1.1** (package.json:16), а не Next 15, как фигурировало в контексте аудита.
Коммиты после аудита: нет (HEAD `bbd817b` от 2026-06-28, рабочих изменений в дереве нет).

---

## П.1 API-прокси не верифицирует initData на сервере — STILL_PRESENT

- Фактический путь роута: `src/app/api/[...path]/route.ts` (catch-all для `/api/*`).
  - `route.ts:68-76` — заголовки пробрасываются как есть, без проверки подписи:
    `const initDataProvider = request.headers.get("X-Init-Data");` … `headers["X-Init-Data"] = initDataProvider;` (68-73), `headers["X-User-ID"] = userIdHeader;` (75).
  - `route.ts:22-52` — GET/POST/PUT/DELETE без какого-либо middleware авторизации.
- `src/components/dashboard/OpencodeSection.tsx:123-127` — `fetch('/api/opencode/exec?XTransformPort=4096', { method: 'POST', … body: JSON.stringify({ command: cmd }) })` — произвольная команда через прокси.
- «Аутентификация» только на клиенте: `src/lib/telegram.ts:70` — `const isAuthenticated = !!(initData || userId);`; к тому же фолбэк из localStorage (telegram.ts:43-67) делает и её подделываемой.
- Уточнение (усиливает пункт): ни один клиентский fetch вообще не отправляет заголовки X-Init-Data/X-User-ID (rg по src/ — вхождения только в route.ts). Т.е. пересылка заголовков — мёртвый код, а API-эндпоинты фактически полностью открыты.

## П.2 README противоречит архитектуре + mock под бейджем «Live» — STILL_PRESENT

- `README.md:26` — «**Vercel** — только статический хостинг / CDN», при этом в репо есть serverless-роут `src/app/api/[...path]/route.ts` и `vercel.json:2` — `"framework": "nextjs"` (не «статика»).
- `README.md:46` — `NEXT_PUBLIC_SERVER_URL=http://your-server-ip`; rg `NEXT_PUBLIC_SERVER_URL` по src/ — **0 вхождений** (единственное вхождение в репо — сам README). Переменная кодом не читается.
- `src/components/dashboard/MonitoringSection.tsx:79-96` — при ошибке/502: комментарий (строка 80) «Use mock data with slight randomization for realism», далее 81-95 — случайный дрейф всех метрик (cpu/ram/disk/network/processes/temperature/loadAvg).
- `src/app/page.tsx:75-78` — статический бейдж «Live» в хедере, никак не связан с реальным состоянием соединения. (В самой секции мониторинга есть честный бейдж «Мок», MonitoringSection.tsx:129-136, но хедер «Live» остаётся зелёным всегда.)
- Аналогично OpencodeSection: на любую ошибку API — молчаливый фолбэк на MOCK_RESPONSES (OpencodeSection.tsx:136-158).

## П.3 next.config.ts — STILL_PRESENT

- `next.config.ts:5` — `ignoreBuildErrors: true` (внутри `typescript: { … }`). Бонусом рядом: `next.config.ts:7` — `reactStrictMode: false`.

## История node_modules (прошлый пункт «не проверено») — ЗАКРЫТО: НЕ БЫЛ ЗАКОММИЧЕН

Полный клон, всего 2 коммита, оба 2026-06-28:
- `3c5f22e` «Initial backup of goclaw mini app» (11:46:38) и `bbd817b` «chore: ignore node_modules to fix push limit» (11:48:49).
- `git log --all --diff-filter=A --name-only -- '*node_modules*'` → пусто.
- `git rev-list --objects --all | grep node_modules` → 0 строк.
- `git ls-tree -r 3c5f22e | grep -c node_modules` → 0; в HEAD тоже 0.
- Объектов в паке 37, размер 163 KiB; крупнейший blob 248 726 байт = package-lock.json. Дangling-объектов нет (git fsck чист).
- Diff коммита bbd817b — ровно одна строка: `.gitignore +node_modules/` (в 3c5f22e .gitignore содержал только `.vercel`).
Вывод: в сохранённой (успешно запушенной) истории node_modules отсутствует — автор упёрся в push limit локально, добавил .gitignore до первого успешного пуша. Утечки зависимостей через историю нет.

## Новые находки

1. **Задокументированный деплой на Vercel неработоспособен по построению.** Прокси обращается к `http://localhost:8090…4096` (`src/app/api/[...path]/route.ts:5-11`) — на serverless-инстансе Vercel это localhost самого инстанса, бэкенды туда никогда не досягаемы → гарантированный 502 (`route.ts:112-117`) → весь UI вечно живёт на моках (MonitoringSection, OpencodeSection). README.md:5-22 предписывает именно такой деплой. Нужна реальная переменная окружения с адресом сервера (обещанная, но не реализованная NEXT_PUBLIC_SERVER_URL) и честная архитектурная секция в README.
2. **Ложные состояния успеха в UI.** `src/components/dashboard/SettingsSection.tsx:92-98` — при падении POST конфигурации (catch → «Offline save simulation») всё равно ставится `saveSuccess=true` и пользователю показывается «✓ Сохранено» — настройки не сохранены. `OpencodeSection.tsx:56-58,148-157` — мок-ответ на команду `restart` рапортует «go-agent-v6 перезапущен успешно (pid: 28451)» при полном отсутствии реального действия; бейдж «Подключено» (`OpencodeSection.tsx:189-192`) статичен и не отражает фолбэк на моки.
3. Мелочи: `package.json:21` — зависимость `recharts` нигде не импортируется (мёртвый вес); фактическая версия фреймворка Next 16.1.1 против «Next.js 15» в контексте прошлого аудита; package-lock.json в репо присутствует (сборка воспроизводима — в отличие от agy-claw-bot).

## Не проверено

- Runtime-поведение (запуск кода запрещён методикой): фактический ответ Vercel-функции, реальная недоступность localhost-бэкендов — вывод по коду, не по выполнению.
- Внешнее состояние (issues/PR/releases, статистика GitHub) — не проверялось (api.github.com не использовался).
