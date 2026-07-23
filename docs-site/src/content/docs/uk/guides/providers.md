---
title: Провайдери
description: Усі способи, якими opencodex автентифікується й спілкується з LLM-провайдером — OAuth, API-ключ, наскрізний ChatGPT та локальний.
---

**Провайдер** — це один вихідний LLM-ендпоінт плюс те, як до нього дістатися: адаптер, base URL, режим
автентифікації та опційний список моделей. Провайдери живуть під `providers` у
`~/.opencodex/config.json`.

## Режими акаунтів OpenAI

| Ідентифікатор провайдера | Використання | Правило облікових даних/акаунта |
| --- | --- | --- |
| `openai` | Вхід у Codex | Pool (за замовчуванням) вибирає основний плюс додані акаунти; Direct використовує лише поточний вхід викликача/основний. |
| `openai-apikey` | OpenAI API | Лише налаштований API-ключ/пул ключів; ніколи не читає акаунти Codex. |

Використовуйте звичайний `gpt-5.6-sol` з опцією Pool/Direct на сторінці Providers або
`openai-apikey/gpt-5.6-sol` для API. Облікові маршрути ніколи не перетікають один в одного. Маршрут API
публікує метадані контексту 1 050 000 / макс. вводу 922 000. Його віртуальні ідентифікатори `sol-pro`,
`terra-pro` та `luna-pro` зберігають свою вибрану публічну ідентичність, тоді як дріт використовує
базову модель плюс `reasoning.mode: "pro"`.

Постачені конфіги v1 мігрують автоматично до маркера 2 і одного рядка з підтримкою опцій. Оригінальний
конфіг зберігається один раз у `~/.opencodex/config.json.pre-openai-tiers-v2.bak`; відновіть його
командою `cp ~/.opencodex/config.json.pre-openai-tiers-v2.bak ~/.opencodex/config.json`.

## Режими автентифікації

Конфіги провайдерів приймають три значення `authMode` (`key` за замовчуванням). Вбудований реєстр також
окремо позначає локальні пресети; вони зазвичай опускають і `authMode`, і `apiKey`.

| `authMode` | Як автентифікується | Ким використовується |
| --- | --- | --- |
| `key` | Надсилає ваш API-ключ (`Authorization: Bearer …` або `x-api-key` / `api-key` залежно від адаптера). Ключ може бути літералом чи посиланням `${ENV_VAR}`. | Більшість провайдерів. |
| `forward` | Ретранслює **ваші вхідні заголовки автентифікації Codex** дослівно до провайдера — ключ не зберігається. Це наскрізна передача входу ChatGPT. | OpenAI (адаптер `openai-responses`). |
| `oauth` | Розв'язує збережений токен доступу OAuth (автооновлюваний перед закінченням) і використовує його як bearer-ключ. | xAI, Anthropic, Kimi, Kiro, Google Antigravity, Cursor, GitHub Copilot. |

## 1. Вхід у ChatGPT (forward / наскрізний)

Провайдеру `openai` **не потрібен API-ключ**. Direct перенаправляє облікові дані з вашого наявного
`codex login`; Pool розв'язує основний або доданий акаунт Codex перед використанням того самого
бекенда:

```json
{
  "openai": {
    "adapter": "openai-responses",
    "baseUrl": "https://chatgpt.com/backend-api/codex",
    "authMode": "forward"
  }
}
```

Перенаправляється лише курований набір заголовків (`FORWARD_HEADERS`: authorization, ідентифікатор
акаунта ChatGPT, OpenAI beta/originator/session — див. [Адаптери](/opencodex/uk/reference/adapters/)).
Цей шлях також живить [sidecar-и вебпошуку та зору](/opencodex/uk/guides/sidecars/).

Наскрізний каталог ChatGPT також додає звичайні slug-и GPT-5.6 Sol/Terra/Luna (`gpt-5.6-sol`,
`gpt-5.6-terra`, `gpt-5.6-luna`) для акаунтів, які можуть їх використовувати.

## 2. Вхід в акаунт (OAuth)

Шість пресетів провайдерів використовують вхід OAuth — плюс GitHub Copilot через експериментальний
неофіційний міст device-flow. opencodex зберігає їхні облікові дані в `~/.opencodex/auth.json` і оновлює
їх автоматично. `chatgpt` також приймається CLI входу; він отримує облікові дані ChatGPT, створюючи
запис провайдера в режимі `forward`.

```bash
ocx login xai          # xAI Grok
ocx login anthropic    # Anthropic Claude (Pro/Max)
ocx login kimi         # Moonshot Kimi
ocx login kiro         # імпорт облікових даних kiro-cli (або резерв через токен)
ocx login google-antigravity
ocx login cursor       # автономний PKCE-вхід Cursor
ocx login github-copilot  # GitHub device flow → токен Copilot (Copilot Pro/Business)
ocx login chatgpt      # автономний OAuth-вхід ChatGPT
ocx logout <provider>
```

| Провайдер | Адаптер | Base URL | Примітки |
| --- | --- | --- | --- |
| `xai` | `openai-chat` | `https://api.x.ai/v1` | Каталог Grok із пріоритетом живого; `grok-4.5` — резерв за замовчуванням. |
| `anthropic` | `anthropic` | `https://api.anthropic.com` | Моделі Claude; живий список моделей отримується з `/v1/models`. |
| `kimi` | `openai-chat` | `https://api.kimi.com/coding/v1` | Кодинг-моделі Kimi K2.7/K2.6/K2.5. |
| `kiro` | `kiro` | `https://runtime.us-east-1.kiro.dev` | Вхід із пріоритетом імпорту повторно використовує встановлену сесію `kiro-cli`. |
| `google-antigravity` | `google` | `https://daily-cloudcode-pa.googleapis.com` | Google OAuth поверх дроту Cloud Code Assist. |
| `cursor` | `cursor` | `https://api2.cursor.sh` | Експериментальний PKCE-вхід, живий транспорт HTTP/2 та виявлення моделей із фільтром за акаунтом. |
| `github-copilot` | `openai-chat` | `https://api.githubcopilot.com` | Експериментально. GitHub device flow + обмін `copilot_internal` (OAuth-клієнт VS Code). Потребує активної підписки Copilot; не є офіційним стороннім API. |

Ви також можете розпочати OAuth із [веб-панелі](/opencodex/uk/guides/web-dashboard/).

### Кілька акаунтів OAuth

Провайдери OAuth, чиї облікові дані включають стабільний ідентифікатор акаунта або e-mail, можуть
зберігати більш ніж один вхід. Сторінка Providers показує ці акаунти у випадному списку, дає змогу
додати ще один і перемикає активний акаунт, не виводячи інші. Облікові дані Kimi й Kiro без ідентичності
замінюють свій активний слот, тоді як `chatgpt` завжди однослотовий, бо пулові акаунти Codex мають
окремий облік. Токени залишаються в `~/.opencodex/auth.json`; `/api/oauth/accounts` повертає лише
масковані метадані.

### Імпорт облікових даних Kiro

`ocx login kiro` шукає платформні сховища Kiro CLI й відкриває бази SQLite лише для читання. Дві змінні
середовища роблять вибір явним без копіювання облікових даних у opencodex:

- `KIROCLI_DB_PATH` вибирає нестандартну базу SQLite Kiro CLI. Шлях має вже існувати; opencodex не
  створює його й не змінює базу, файли WAL чи SHM.
- `KIROCLI_TOKEN_KEY` вибирає точний ключ токена `auth_kv`, коли база містить кілька інакше
  неоднозначних рядків токенів. Відсутній вибір провалює вхід замість вгадування.

Тримайте ці змінні й вибрану базу приватними. Не додавайте файли бази чи сиру діагностику входу до
баг-репортів.

## 3. Каталог API-ключів

opencodex постачає 53 вбудовані пресети: 42 на основі ключа, сім OAuth, три локальні та типовий пресет
наскрізного ChatGPT. Селектор **Add provider** у панелі відкриває панель провайдера ключа, валідує ключ
і зберігає його. Помітні записи:

| Провайдер | Base URL |
| --- | --- |
| **OpenAI (API-ключ)** | `https://api.openai.com/v1` |
| **Anthropic (API-ключ)** | `https://api.anthropic.com` |
| **OpenRouter** | `https://openrouter.ai/api/v1` |
| **Ollama Cloud** | `https://ollama.com/v1` |
| Google Gemini · Google Vertex AI | `https://generativelanguage.googleapis.com` · `https://aiplatform.googleapis.com` |
| Azure OpenAI | `https://{resource}.openai.azure.com/openai` |
| Umans AI · Neuralwatt | `https://api.code.umans.ai` · `https://api.neuralwatt.com/v1` |
| Mistral | `https://api.mistral.ai/v1` |
| MiniMax · MiniMax (CN) | `https://api.minimax.io/v1` · `https://api.minimaxi.com/v1` |
| DeepSeek | `https://api.deepseek.com` |
| Cerebras | `https://api.cerebras.ai/v1` |
| Together | `https://api.together.xyz/v1` |
| Fireworks | `https://api.fireworks.ai/inference/v1` |
| Moonshot (Kimi API) · Kimi (coding) | `https://api.moonshot.ai/v1` · `https://api.kimi.com/coding/v1` |
| Hugging Face | `https://router.huggingface.co/v1` |
| NVIDIA NIM | `https://integrate.api.nvidia.com/v1` |
| Z.AI (GLM Coding) | `https://api.z.ai/api/coding/paas/v4` |
| Qwen Cloud | Token plan (за замовчуванням): `https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1` · Pay as you go: `https://dashscope.aliyuncs.com/compatible-mode/v1` · або Custom |
| Tencent Cloud Coding Plan | `https://api.lkeap.cloud.tencent.com/coding/v3` |
| SiliconFlow | `https://api.siliconflow.cn/v1` |
| Xiaomi MiMo | `https://api.xiaomimimo.com/anthropic` |
| Kilo | `https://api.kilo.ai/api/gateway` |
| GitLab Duo | `https://cloud.gitlab.com/ai/v1/proxy/openai/v1` |
| Cloudflare AI Gateway | `https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway}/anthropic` |
| …та інші | opencode zen, Vercel AI Gateway, Venice, NanoGPT, Synthetic, Qianfan, Alibaba, Parallel, ZenMux, LiteLLM |

Більшість використовують адаптер `openai-chat` із bearer-ключем; кілька, що надають лише
Anthropic-сумісний ендпоінт (наприклад, **Xiaomi MiMo**), використовують адаптер `anthropic`
(`x-api-key`).

> **Обмеження використання Tencent Cloud Coding Plan:** Tencent документує цю підписку лише для
> інтерактивних кодинг-інструментів. Загальна API-автоматизація, бекенди власних застосунків та
> неінтерактивне пакетне використання заборонені й можуть спричинити призупинення ключа плану.

### Кілька API-ключів

Провайдери на основі ключа також можуть зберігати кілька ключів. Додавання ключа через сторінку
Providers зберігає його під `provider.apiKeyPool`, робить активним і віддзеркалює в `provider.apiKey`,
тож маршрутизація й адаптери продовжують читати те саме поле, що й раніше. Той самий випадний список
може перемкнути чи вилучити ключі; управлінський API — це `/api/providers/keys`, і він повертає лише
масковані ключі.

### Перемикання акаунтів із термінала

Використовуйте `ocx account list`, `ocx account current` та `ocx account use`, щоб переглядати чи
перемикати ті самі пули Codex, OAuth та API-ключів, не відкриваючи панель. Див.
[Довідник CLI](/opencodex/uk/reference/cli/#ocx-account-subcommand) щодо команд, JSON-виводу та
поведінки нових сесій.

### Шляхи прев'ю GPT-5.6

GPT-5.6 Sol/Terra/Luna додано до резервних списків провайдерів, щоб `ocx sync` міг тримати моделі
видимими навіть поки живі каталоги відстають:

| Маршрут Codex | Додані ідентифікатори моделей | Видимий у Codex контекст |
| --- | --- | --- |
| Вхід у Codex (Pool або Direct) | `gpt-5.6-*` | 372 000 |
| OpenAI (API-ключ) | `openai-apikey/gpt-5.6-*` плюс `*-pro` | 1 050 000 (макс. ввід 922 000) |
| OpenRouter | `openrouter/openai/gpt-5.6-sol`, `openrouter/openai/gpt-5.6-terra`, `openrouter/openai/gpt-5.6-luna` | 1 050 000 |
| Cursor | `cursor/gpt-5.6-sol`, `cursor/gpt-5.6-terra`, `cursor/gpt-5.6-luna` | 1 000 000 |

Нативні записи GPT-5.6 зберігають запіновані вихідні драбини міркувань (наприклад, Luna має `max`, але
не `ultra`). Маршрутизовані записи використовують метадані свого провайдера й відображення міркувань.
Усі чотири шляхи залишаються обмеженими з боку вихідного провайдера; живе виявлення Cursor додатково
фільтрує свій статичний засів до моделей, які може використовувати акаунт, що ввійшов.

:::note[Шлюзи та проксі підписок]
Провайдера включено, коли opencodex має відповідний мережевий адаптер, **а не** на основі того, чи це
«агентний» продукт. Поточні ідентифікатори адаптерів: `openai-chat`, `openai-responses`, `anthropic`,
`google` (режими AI Studio, Vertex та Antigravity/Cloud Code Assist), `azure` / `azure-openai`, `kiro`
та `cursor`. Пропрієтарний API без однієї з цих реалізацій, як-от нативний Amazon Bedrock, безпосередньо
не підтримується.
**GitHub Copilot** — це OAuth-провайдер (`ocx login github-copilot`), що обмінює вхід GitHub device-flow
на короткоживучий API-токен Copilot — а не вставлений API-ключ. **GitLab Duo** залишається шлюзом
ключа/токена підписки на своєму OpenAI-сумісному ендпоінті. **Cloudflare AI Gateway** потребує вписаних
у URL ідентифікаторів вашого акаунта + шлюзу.

Cursor відстежується окремо як експериментальний адаптер. `adapter: "cursor"` з'являється в `ocx init`
та селекторі Add Provider панелі як експериментальний запис локального конфіга зі статичними резервними
метаданими каталогу моделей Cursor. Коли налаштовано токен доступу Cursor, opencodex використовує живий
транспорт HTTP/2 Cursor. Його резервний засів v2.7.1 включає `gpt-5.6-sol` / `terra` / `luna` (контекст
1M) плюс `grok-4.5` / `grok-4.5-fast` (500K); живе виявлення вирішує, які залишаться видимими для
акаунта. Керована сервером нативна виконавча функціональність Cursor read/write/delete/ls/grep/shell/fetch
вимкнена за замовчуванням, бо вона обходить шлях затвердження й пісочниці Codex; встановлюйте
`unsafeAllowNativeLocalExec: true` на об'єкті `providers.cursor` у `~/.opencodex/config.json` лише для
довірених локальних експериментів (або через **Providers → Cursor → Edit JSON** у панелі). Див.
[Довідник конфігурації](/opencodex/uk/reference/configuration/#cursor-provider-adapter-cursor) щодо
повного прикладу. MCP, запис екрана та computer-use доступні як executor-хуки; без налаштованого
локального executor-а opencodex повертає типізовані результати no-executor замість блокування запиту
політикою. OAuth Cursor та живе виявлення моделей увімкнені для цього експериментального адаптера;
Cursor усе одно не показується в списках входу за ключем.
:::

### Ollama Cloud

Ollama Cloud — це хостований (не локальний) Ollama, OpenAI-сумісний за `https://ollama.com/v1` із ключем
із [ollama.com/settings/keys](https://ollama.com/settings/keys). opencodex класифікує його хмарну лінійку
за здатністю до зору, тож [sidecar зору](/opencodex/uk/guides/sidecars/) вмикається лише для текстових
моделей. Текстові моделі (наприклад, `glm-5.2`, `deepseek-v4-pro`, `gpt-oss`, `qwen3-coder`,
`minimax-m2.x`, `nemotron-3-*`) перелічено в `noVisionModels`; зорово-нативні моделі (наприклад,
`kimi-k2.6`, `minimax-m3`, `gemma4`, `qwen3.5`, `gemini-3-flash-preview`) — ні. Зіставлення толерантне до
тегів `:size` в Ollama, тож `gpt-oss` покриває `gpt-oss:120b` та `gpt-oss:20b`.

## 4. Локальні провайдери

Спрямуйте opencodex на локальний OpenAI-сумісний сервер — зазвичай із порожнім ключем:

| Провайдер | Base URL |
| --- | --- |
| Ollama (локально) | `http://localhost:11434/v1` |
| vLLM | `http://localhost:8000/v1` |
| LM Studio | `http://localhost:1234/v1` |

## Будь-який OpenAI-сумісний ендпоінт

Якщо провайдер розмовляє Chat Completions, його обробляє адаптер `openai-chat` — виберіть **Custom** у
панелі або `custom` у `ocx init` і введіть base URL. Див.
[Довідник конфігурації](/opencodex/uk/reference/configuration/) щодо кожного поля провайдера
(`headers`, `noReasoningModels`, `noVisionModels`, `models`, …).
