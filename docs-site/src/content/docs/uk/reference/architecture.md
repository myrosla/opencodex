---
title: Архітектура
description: Внутрішня будова opencodex — мапа модулів, міст AdapterEvent, парсер запитів та кешування.
---

opencodex — це один процес Bun. Запит входить як OpenAI Responses, нормалізується до внутрішньої моделі,
маршрутизується, надсилається провайдеру через адаптер і мостом повертається в Responses SSE. Див.
[Як це працює](/opencodex/uk/getting-started/how-it-works/) щодо наскрізного потоку.

## Мапа модулів

```
src/
├── cli/                # диспетчеризація команд ocx, init, status, команди провайдерів
├── server/             # Bun.serve, проксі /v1/*, управлінський API /api/*, WS-міст
├── codex/              # інтеграція конфіга Codex, синхронізація каталогу, автентифікація/акаунти
├── providers/          # метадані провайдерів, пул API-ключів, квоти й мітки
├── adapters/           # сім мережевих адаптерів, спільні guard-и/утиліти, protobuf-транспорт Cursor
├── oauth/              # провайдери OAuth, каталог API-ключів, сховище/оновлення токенів
├── usage/              # вилучення використання запитів, JSONL-журнали, зведення, підсумки
├── lib/                # хелпери runtime, процесу, повторів, приватності, оцінки токенів
├── web-search/         # sidecar вебпошуку (синтетичний інструмент, цикл, executor, парсер)
├── vision/             # sidecar зору (опис + план)
├── config.ts           # ~/.opencodex/config.json, дефолти, PID, розв'язання env
├── router.ts           # ідентифікатор моделі → провайдер + адаптер
├── bridge.ts           # потік AdapterEvent → Responses SSE / JSON
├── reasoning-effort.ts # переклад зусилля міркувань, обмеження та рівні каталогу
├── responses/
│   ├── parser.ts       # запит Responses → OcxParsedRequest
│   ├── schema.ts       # валідація Zod
│   └── compaction.ts   # промпти віддаленої компакції, конверти, компактна історія
├── service.ts          # фонова служба launchd / systemd / Task Scheduler
├── types.ts            # основні інтерфейси + хелпери (modelInList, namespacedToolName)
└── index.ts            # публічний вхід
```

Три раніше великі вхідні файли тепер зберігають сумісність як фасади: `codex/catalog.ts` експортує сім
сфокусованих модулів `codex/catalog/*.ts`, `server/management-api.ts` диспетчеризує до дев'яти модулів
`server/management/*.ts`, а `server/responses.ts` експортує п'ять модулів `server/responses/*.ts`.

## Потік запиту

`server/index.ts` володіє межею HTTP і делегує площину даних Responses фасаду `server/responses.ts` та
його модулям `server/responses/*.ts`:

1. `server/index.ts` застосовує CORS та автентифікацію API, відхиляє нову роботу під час зливання й
   фіксує метадані життєвого циклу запиту. Він обслуговує `GET /v1/models`, `POST /v1/responses`,
   `POST /v1/responses/compact`, `POST /v1/images/generations` / `POST /v1/images/edits` (ретрансльовані
   до вихідного провайдера родини OpenAI через `server/images.ts` для вбудованого інструмента codex
   `image_gen`) та опційний WebSocket upgrade на `/v1/responses`.
2. `server/responses/core.ts` розпаковує й парсить JSON, розгортає локально запам'ятований ввід
   `previous_response_id`, коли доступний, потім викликає `responses/parser.ts`.
3. `router.ts` розв'язує звичайний ідентифікатор чи `provider/model`. Потім сервер розв'язує афінність
   акаунта Codex, оновлює OAuth провайдера за потреби й застосовує вибрані облікові дані до маршруту.
4. Перед основним викликом `vision/` описує зображення для моделей у `noVisionModels`; якщо безпечного
   шляху sidecar-а немає, зображення прибираються, а не надсилаються текстовому вихідному провайдеру.
5. `server/adapter-resolve.ts` застосовує будь-яке модельно-специфічне мережеве перевизначення й
   конструює один із семи адаптерів. Наскрізний Responses ретранслює нативне тіло, Cursor запускає свій
   двонаправлений транспорт `runTurn`, а перекладені адаптери будують/отримують/парсять вихідний запит.
6. Для маршрутизованих моделей із хостованим інструментом `web_search` `web-search/` показує синтетичну
   функцію, виконує справжній пошук через sidecar ChatGPT, повертає результати маршрутизованій моделі й
   повторює в межах налаштованого ліміту циклу.
7. `bridge.ts` виробляє Responses SSE або JSON. `server/request-log.ts` та `usage/` збирають
   термінальний статус, латентність, мітки провайдера/моделі й best-effort використання токенів, не
   змінюючи відповідь.

## Парсер

`responses/parser.ts` валідує вхідний запит через `responses/schema.ts` (Zod), потім будує
`OcxParsedRequest`:

- **Повідомлення** — елементи `input` стають нормалізованим `OcxMessage[]`: user / developer /
  assistant / toolResult. Елементи `reasoning` стають блоками мислення; елементи `function_call`,
  `custom_tool_call` та `tool_search_call` стають викликами інструментів; їхні відповідники `*_output`
  стають результатами інструментів.
- **Інструменти** — функціональні інструменти проходять наскрізь; **інструменти з простором імен (MCP)
  сплющуються** до `namespace__name` (і відновлюються на зворотному шляху); **вільні** інструменти
  (наприклад, `apply_patch`) та інструменти виявлення **tool_search** позначаються; **хостовані
  інструменти** (`web_search`, генерація зображень, …) прибираються й повторно впроваджуються sidecar-ом
  лише якщо він їх оброблятиме.
- **Зображення** — зберігаються як справжні частини вмісту (data URL або віддалений https), ніколи не
  вбудовуються як текст.
- **Прапорці функцій** — `_webSearch` (запитано хостований вебпошук), `_structuredOutput`
  (`text.format` — це json_schema / json_object) та `_compactionRequest` (віддалена компакція v2).

## Міст

`bridge.ts` перетворює внутрішній потік `AdapterEvent` адаптера назад у Responses SSE, який Codex
розуміє:

| AdapterEvent | Видані Responses SSE |
| --- | --- |
| `text_delta` | `response.output_text.delta` → `…done`, `response.content_part.done`, `response.output_item.done` |
| `thinking_delta` | `response.reasoning_summary_text.delta` → `…done`, закриття елемента |
| `reasoning_raw_delta` | Сирий елемент `reasoning_text` (або прихований конверт round-trip) |
| `thinking_signature` / `redacted_thinking` | Збережено в конверті міркувань `encrypted_content` |
| `tool_call_start` | `response.output_item.added` (тип: `function_call` / `custom_tool_call` / `tool_search_call`) |
| `tool_call_delta` | `response.function_call_arguments.delta` (пропущено для freeform / tool_search) |
| `tool_call_end` | `response.function_call_arguments.done` → `response.output_item.done` |
| `web_search_call_begin` / `web_search_call_end` | Один живий елемент `web_search_call` плюс цитати URL |
| `heartbeat` | Позначає вихідну активність; без видимого користувачу елемента виводу |
| `done` | `response.completed` (з використанням) |
| `error` | `response.failed` (з `last_error`) |

Міст також виконує **heartbeat keep-alive** (RC3): під час вихідної тиші він видає ігнорований парсером
SSE-подію `response.heartbeat` кожні 2 секунди, щоб перезвести таймер простою Codex. Типовий **дедлайн
застою** — 300 секунд (`stallTimeoutSec`); його досягнення перериває вихідний провайдер і видає
`response.incomplete` з причиною `upstream_stall_timeout`, запобігаючи тому, щоб зависле з'єднання
блокувало Codex безмежно.

Виклики інструментів роз'яснюються на три типи елементів Responses за допомогою мапи простору імен,
набору freeform та набору tool-search, захоплених парсером — тож простори імен MCP, freeform-інструменти
в стилі `apply_patch` та клієнтсько-виконуваний `tool_search` проходять в обидва боки. Варіант
`buildResponseJSON()` виробляє один непотоковий об'єкт відповіді з тих самих подій.

## Управлінський API, OAuth та використання

`server/management-api.ts` живить панель і диспетчеризує сфокусовані групи маршрутів до
`server/management/*.ts`. Його маршрути `/api/*` покривають безпечні конфіг/налаштування, CRUD
провайдерів та пули ключів, вибір моделей/обмеження контексту/елементи керування v2, синхронізацію
каталогу, діагностику й debug-журнали, використання й квоти, налаштування sidecar-ів, оновлення,
згенеровані клієнтські API-ключі, вхід/статус/вихід OAuth та вибір акаунта, керування акаунтами Codex і
плавну зупинку. `server/auth-cors.ts` вимагає `OPENCODEX_API_AUTH_TOKEN` і для `/api/*`, і для `/v1/*`,
коли проксі прив'язується поза loopback; налаштовані записи `corsAllowOrigins` розширюють allowlist
локального походження.

Реалізації OAuth живуть у `oauth/`; токени доступу завантажуються чи оновлюються безпосередньо перед
маршрутизованим викликом, тоді як `oauth/token-guardian.ts` може проактивно оновлювати лише
провайдерів, чия політика це дозволяє. Облікові дані пулу Codex/ChatGPT та афінність тредів живуть під
`codex/` і тримаються поза управлінськими відповідями. Використання запитів нормалізується до `OcxUsage`,
показується в термінальних подіях Responses й агрегується `usage/` для панелі та опційної JSONL-діагностики.

## Транспорт та компакція

`server/index.ts` за замовчуванням обслуговує HTTP/SSE на `/v1/responses`. Якщо Codex намагається
Responses WebSocket upgrade, поки `websockets` дорівнює `false`, opencodex повертає
`426 upgrade_required`; Codex тоді відкочується на HTTP для цієї сесії. Коли встановлено
`"websockets": true`, той самий ендпоінт приймає upgrade й використовує WebSocket-міст.

Компакція контексту Codex працює для маршрутизованих моделей. `server/responses/compact.ts` обробляє
`POST /v1/responses/compact`, запускаючи внутрішній маршрутизований хід підсумовування й повертаючи
компактну історію, тоді як `responses/parser.ts` та `bridge.ts` обробляють ходи віддаленої компакції v2
`compaction_trigger`, видаючи рівно один синтетичний елемент виводу `compaction`.

## Кешування та каталог

- `codex/model-cache.ts` тримає TTL-кеш живих результатів `/models` у пам'яті для кожного провайдера (за
  замовчуванням 5 хв, що відповідає власному кешу Codex), із застарілим відкатом, коли fetch зазнає
  невдачі.
- `codex/catalog/sync.ts`, експортований через фасад `codex/catalog.ts`, зливає маршрутизовані моделі в
  каталог Codex як записи з простором імен, ранжує виведені
  [моделі субагентів](/opencodex/uk/guides/codex-integration/#the-subagent-picker) першими, фільтрує
  `disabledModels` і може повністю відновити незайманий каталог з одноразової резервної копії.

## Зусилля міркувань

`reasoning-effort.ts` перекладає мітки міркувань Codex у мережеві значення кожного провайдера. Каталог
Codex оголошує мітки, які Codex приймає (`low` / `medium` / `high` / `xhigh` / `max`), але вихідні
провайдери можуть підтримувати лише меншу підмножину чи вимагати справжнього аліаса. Модуль:

- Визначає канонічні `CODEX_REASONING_LEVELS` та їхній порядок сортування.
- Обмежує запитане зусилля до найближчого підтримуваного рівня, коли точний рівень недоступний.
- Розв'язує перевизначення `reasoningEffortMap` для кожної моделі та провайдера для власних мережевих
  відображень.
- Цілком прибирає зусилля для моделей, перелічених у `noReasoningModels`.

## Основні типи

Внутрішня модель живе в `types.ts`: `OcxParsedRequest`, `OcxContext`, об'єднання `OcxMessage`,
`OcxContentPart` (текст / зображення), `OcxToolCall`, `OcxTool`, `AdapterEvent` та типи конфіга
(`OcxConfig`, `OcxProviderConfig`). Широко використовуються два хелпери: `namespacedToolName()` та
`modelInList()` (толерантне до тегів `:size` зіставлення для `noVisionModels` / `noReasoningModels`).
