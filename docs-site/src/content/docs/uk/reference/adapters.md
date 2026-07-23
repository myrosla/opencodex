---
title: Адаптери
description: Сім адаптерів провайдерів — на що кожен націлений, як будує запити та його особливості.
---

**Адаптер** перекладає між внутрішньою моделлю запиту/відповіді opencodex і одним мережевим форматом
провайдера. Кожен адаптер реалізує інтерфейс `ProviderAdapter` (`src/adapters/base.ts`):

```ts
interface ProviderAdapter {
  name: string;
  buildRequest(parsed, incoming?): AdapterRequest | Promise<AdapterRequest>;
  fetchResponse?(request, context): Promise<Response>;   // custom retry/transport
  parseStream(response): AsyncGenerator<AdapterEvent>;
  parseResponse?(response): Promise<AdapterEvent[]>;   // non-streaming
  runTurn?(parsed, incoming, emit): Promise<void>;      // bidirectional transport
}
```

`buildRequest` зводить `OcxParsedRequest` до вихідного HTTP-запиту; `parseStream` / `parseResponse`
піднімають відповідь провайдера назад у внутрішні `AdapterEvent`-и. `fetchResponse` дає адаптеру
володіти повторами/таймаутами, тоді як `runTurn` підтримує транспорти, які не можна представити як один
HTTP-fetch, за яким іде один потік відповіді.
[`bridge.ts`](/opencodex/uk/reference/architecture/#the-bridge) потім перетворює події на Responses SSE.

## `openai-chat`

**Ціль:** OpenAI **Chat Completions** (`POST {baseUrl}/chat/completions`) та кожен сумісний провайдер —
xAI, Kimi, DeepSeek, GLM, Groq, OpenRouter, Ollama (локальний і хмарний) та інші.
**Автентифікація:** `key` (Bearer).

- Конвертує внутрішні повідомлення в ролі OpenAI; відображає інструменти на `{type:"function",
  function:{…}}` та `tool_choice` (`auto`/`none`/`required` або іменована функція).
- **Переписує промпт ідентичності GPT-5 у Codex** на модельно-агностичний вступ, тож маршрутизовані
  моделі не заявляють, що вони від OpenAI.
- **Обмежує `reasoning_effort`** до оголошеної підмножини моделі, коли точний рівень недоступний;
  `xhigh` та `max` залишаються окремими мітками, доки провайдер явно не налаштує аліас. Адаптер **цілком
  опускає його** для ідентифікаторів у `provider.noReasoningModels`.
- Стрімить `delta.content` (текст), `delta.reasoning_content` (мислення) та `delta.tool_calls[]`;
  збирає `usage`.

## `openai-responses`

**Ціль:** OpenAI **Responses API**. **`passthrough: true`** — перенаправляє сире тіло запиту й стрімить
відповідь назад **без перекладу**.
**Автентифікація:** `forward` (ретрансляція заголовків викликача) або `key`.

- URL `forward` → `{baseUrl}/responses`. Провайдер `key` за замовчуванням використовує застарілу
  конструкцію `{baseUrl}/v1/responses`.
- Провайдер `key` може задати валідований відносний `responsesPath`; адаптер прибирає один завершальний
  слеш із `baseUrl` і надсилає `{trimmedBaseUrl}{responsesPath}`. Для Ark Agent Plan використовуйте
  `baseUrl: "https://ark.cn-beijing.volces.com/api/plan/v3"` з `responsesPath: "/responses"`.
- У режимі `forward` ретранслюється лише безпечний allowlist заголовків (`FORWARD_HEADERS`):
  authorization, ідентифікатор акаунта ChatGPT та заголовки OpenAI beta/originator/session. Це шлях
  входу ChatGPT, що також живить [sidecar-и](/opencodex/uk/guides/sidecars/).

## `anthropic`

**Ціль:** Anthropic **Messages** (`/v1/messages`).
**Автентифікація:** `key` (`x-api-key`) або `oauth` (Bearer + `anthropic-beta`, для Claude Pro/Max).

- Конвертує повідомлення в блоки вмісту Anthropic (текст, base64-зображення, `tool_use`, `thinking`).
- **Математика розширеного мислення:** Anthropic вимагає `max_tokens > thinking.budget_tokens`. Адаптер
  відображає зусилля міркувань на бюджет (minimal 1024 … max 32000), потім обчислює безпечний
  `max_tokens` із запасом виводу й **прибирає `temperature`/`top_p`**, коли мислення ввімкнено
  (Anthropic їх там забороняє).
- Завжди надсилає `anthropic-version: 2023-06-01`. Стрімить `content_block_delta` (`text_delta`,
  `thinking_delta`, сумісний `reasoning_delta`, `input_json_delta`). Декодер SSE зберігає стан подій між
  fetch-чанками й приймає термінальний `message_stop` без завершального переносу рядка.

## `google`

**Ціль:** Google **Gemini**, **Vertex AI** та Antigravity **Cloud Code Assist**. AI Studio використовує
`/v1beta/models/{model}:streamGenerateContent`; інші режими використовують свої нативні ендпоінти
Google.
**Автентифікація:** API-ключ, Vertex ADC або Google Antigravity OAuth, вибрані `googleMode`.

- Системний промпт → `systemInstruction`; повідомлення → `contents[]` (assistant → `model`); інструменти
  → `functionDeclarations`. Data-URL зображення → `inline_data`.
- Ідентифікатори викликів інструментів синтезуються, коли Gemini їх опускає. Antigravity зберігає й
  відтворює справжні значення `thoughtSignature`, тож неперервність міркувань переживає пізніші ходи.

## `kiro`

**Ціль:** сервіс Amazon CodeWhisperer Streaming `GenerateAssistantResponse`, який використовує Kiro
(`https://runtime.{region}.kiro.dev/`).
**Автентифікація:** токен доступу Kiro OAuth як Bearer, із метаданими region/profile з облікових даних
Kiro.

- Будує `conversationState` Kiro, відображає інструменти Codex та результати інструментів і надсилає
  блоки зображень, підтримувані дротом Kiro.
- Декодує `application/vnd.amazon.eventstream`, реконструює події тексту/мислення/інструментів, виявляє
  обрізаний JSON інструмента й оцінює використання, бо вихідний провайдер не повертає лічильники
  токенів.
- Використовує налаштований `baseUrl` дослівно, коли він власний. Канонічний URL
  `runtime.{region}.kiro.dev` слідує за API-регіоном імпортованих облікових даних; лише ця канонічна
  форма прийнятна для одного обмеженого відкату до `q.{region}.amazonaws.com` після збою ендпоінта,
  підпису, DNS чи з'єднання.
- Володіє відновленням після скидання з'єднання, безпечним для повтору, тим єдиним прийнятним відкатом
  ендпоінта та одним оновленням/повтором OAuth після HTTP 401. Клієнт володіє throttling, таймаутом та
  звичайними повторами сервісу; opencodex не примножує ці політики всередині адаптера.
- Його непотоковий парсер зливає той самий потік подій для циклу вебпошуку.

### Семантика завершення

Текстові події Kiro не несуть надійної фази завершення ходу. Тому, коли присутній звичайний клієнтський
інструмент, opencodex додає до вихідного запиту приватний інструмент `codex_kiro_final_answer`. Текст
прогресу стрімиться як коментар і не може завершити хід. Адаптер споживає приватний виклик, видає його
відповідь як остаточний текст і ніколи не показує приватний інструмент Codex чи Claude Code. Коли
sidecar вебпошуку активний, цей коментар усе одно стрімиться негайно; лише події, потрібні для рішення,
чи запросила модель синтетичний пошук, залишаються буферизованими.

Якщо Kiro видає прогрес без виклику інструмента завершення, адаптер робить одне продовження. Цей єдиний
повтор може завершитися валідованим приватним завершенням або звичайним остаточним текстом. Він не може
рекурсувати: порожній або лише-міркувальний повтор повертається як повторюваний неповний, тоді як
справжній виклик клієнтського інструмента тримає хід відкритим. Якщо повтор лише повторює попередній
коментар після нормалізації пробілів, дублікатний вивід придушується, тоді як хід усе одно завершується.
Запити без інструментів зберігають звичайну поведінку завершення тексту.

### Зусилля міркувань

`gpt-5.6-sol` має перевірену нативну підтримку зусилля. Його вибране значення `low`, `medium`, `high`,
`xhigh` чи `max` надсилається як `additionalModelRequestFields.reasoning.effort`. Інші моделі Kiro наразі
використовують емульовані міркування: opencodex перетворює вибраний рівень на обмежені інструкції
мислення у вмісті користувача, бо їхнє нативне поле зусилля не перевірено. Не інтерпретуйте оголошений
елемент керування зусиллям на цих моделях як доказ вихідно-нативної підтримки міркувань.

## `cursor`

**Ціль:** `agent.v1.AgentService/Run` у Cursor через потокову HTTP/2 Connect за `api2.cursor.sh`.
**Автентифікація:** OAuth/токен доступу Cursor із `provider.apiKey` або перенаправлений заголовок
authorization.

- Використовує `runTurn`, а не звичайний шлях fetch/parse. Запити, серверні події, аргументи
  інструментів, чекпоінти використання та відповіді клієнта кодуються схемами `@bufbuild/protobuf` у
  `cursor/gen/agent_pb.ts` і фреймуються як повідомлення Connect.
- Відтворює стан розмови через контент-адресовані blob-и, відображає серверні виклики інструментів назад
  до Codex, виявляє живі моделі Cursor через protobuf-RPC `GetUsableModels` і повторює лише до того, як
  запит запуску зафіксовано на дроті.
- Показує Cursor Router як `cursor/auto` плюс явні записи `cursor/auto-cost`, `cursor/auto-balance` та
  `cursor/auto-intelligence`. Явні рівні кодуються в `requested_model.parameters`, тоді як застарілий
  запис `cursor/auto` зберігає дефолт акаунта/команди.
- Cursor-нативне виконання локальної файлової системи/shell/мережі заборонено за замовчуванням. Явні
  інтеграції `mcpServers` та `desktopExecutor` мають окремі opt-in; `nativeLocalExec: "on"` вмикає
  ширший вбудований executor і обходить семантику затвердження/пісочниці Codex, а застаріле
  `unsafeAllowNativeLocalExec: true` залишається еквівалентним лише коли `nativeLocalExec` не задано.

## `azure-openai` (аліас: `azure`)

**Ціль:** **Azure OpenAI**. Обгортає `openai-responses` (тож також `passthrough: true`).
**Автентифікація:** `key` через заголовок `api-key` (не Bearer).

- Делегує побудову запиту наскрізному Responses, валідує, що `baseUrl` не містить нерозв'язаного
  шаблонного плейсхолдера, і замінює `Authorization` на `api-key`. Налаштований URL націлений
  безпосередньо на v1 Responses API від Azure, тож адаптер не додає `api-version`.

## Утиліти зображень (`image.ts`)

Спільні хелпери, використовувані адаптерами з підтримкою зору:

- `parseDataUrl(url)` — ділить URL `data:<type>;base64,<data>` на `{ mediaType, base64 }` для блоків
  зображень Anthropic/Google.
- `contentPartsToText(content)` — сплющує частини вмісту в текст для текстових повідомлень інструментів
  (неописане зображення стає коротким маркером `[image]`, ніколи не base64-blob-ом, що вибухає токенами).
