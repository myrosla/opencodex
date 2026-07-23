---
title: Встановлення
description: Встановіть проксі opencodex (ocx), його передумови та переконайтеся, що він працює.
---

opencodex встановлює дві еквівалентні назви команд — `ocx` і `opencodex`. Обидві запускають той самий
невеликий локальний HTTP-сервер (побудований на Bun). Запити до моделей ідуть до провайдера, вибраного
маршрутизацією; опційні sidecar-и зору й вебпошуку також можуть використовувати ваш вхід у ChatGPT,
коли маршрутизованій моделі це потрібно.

## Передумови

| Вимога | Навіщо |
| --- | --- |
| **[Node](https://nodejs.org) ≥ 18** | `ocx` працює на середовищі виконання Bun, але воно встановлюється автоматично під час `npm install` — вам **не** потрібно ставити Bun самостійно. |
| **[OpenAI Codex](https://openai.com/codex)** (CLI, App або SDK) | Клієнт, перед яким стоїть opencodex. opencodex записує у `$CODEX_HOME/config.toml` (за замовчуванням `~/.codex/config.toml`). |
| Акаунт провайдера або API-ключ | Anthropic, xAI, Kimi, Ollama Cloud, OpenRouter, OpenAI-сумісний ендпоінт або ваш вхід у ChatGPT. |

## Встановлення

```bash
npm install -g @bitkyc08/opencodex
```

:::note[npm заблокував postinstall від bun?]
Останні версії npm можуть блокувати postinstall-скрипт bun (`npm warn
install-scripts ... blocked because they are not covered by allowScripts`),
через що вбудоване середовище виконання Bun залишається непідготовленим.
Перевстановіть, дозволивши скрипт bun — і завжди вказуйте ім'я пакета
(скорочена пропозиція npm його опускає, що перевстановило б поточну директорію):

```bash
npm install -g --allow-scripts=bun @bitkyc08/opencodex

# якщо початкове встановлення використовувало sudo, продовжуйте з sudo:
sudo npm install -g --allow-scripts=bun @bitkyc08/opencodex
```
:::

Переконайтеся, що обидва аліаси команд є у вашому `PATH`:

```bash
ocx --version
opencodex --version
```

### Канали релізів

Стабільний канал `latest` уже включає підтримку каталогу GPT-5.6 Sol/Terra/Luna для ChatGPT,
OpenAI API-ключа, OpenRouter та експериментальних маршрутів Cursor. Доступ з боку вихідного
провайдера все ще обмежений акаунтом; записи каталогу самі по собі не надають доступу. Використовуйте
канал preview лише для тестування невипущених збірок opencodex:

```bash
npm install -g @bitkyc08/opencodex@preview
ocx update --tag preview
```

## Запуск із вихідного коду

Щоб працювати над самим opencodex:

```bash
git clone https://github.com/lidge-jun/opencodex.git
cd opencodex
bun install
bun run dev:proxy   # запускає proxy API в режимі розробки (src/cli/index.ts start)
bun run dev:gui     # запускає dev-сервер панелі (в іншому терміналі)
```

`bun run dev` залишається аліасом для `bun run dev:proxy`. Proxy API надає `/healthz`,
`/v1/responses` та `/api/*`; `GET /` обслуговує запаковану панель лише після того, як `bun run build:gui`
створить `gui/dist`. Під час роботи над панеллю запускайте фронтенд окремо через `bun run dev:gui`.

## Що створюється

Стан opencodex зберігається під `$OPENCODEX_HOME` (за замовчуванням `~/.opencodex`). Файли інтеграції
з Codex — під `$CODEX_HOME` (за замовчуванням `~/.codex`).

| Шлях | Призначення |
| --- | --- |
| `$OPENCODEX_HOME/config.json` | Ваші провайдери, провайдер за замовчуванням, порт та опції. |
| `$OPENCODEX_HOME/ocx.pid` | PID запущеного проксі (захист від запуску кількох екземплярів). |
| `$OPENCODEX_HOME/runtime-port.json` | Живий PID, hostname та порт, зокрема автоматично вибраний резервний порт. |
| `$OPENCODEX_HOME/auth.json` | Збережені облікові дані OAuth (коли ви виконуєте `ocx login`). |
| `$OPENCODEX_HOME/catalog-backup*.json` | Резервні копії каталогу моделей Codex, зроблені перед тим, як opencodex його редагує. |
| `$CODEX_HOME/config.toml` | На loopback opencodex додає кореневий `openai_base_url`, що належить маркеру; не-loopback прив'язки використовують `model_provider = "opencodex"` плюс `[model_providers.opencodex]`, щоб Codex міг надсилати заголовок API-автентифікації. |
| `$CODEX_HOME/opencodex.config.toml` | Резервний/еталонний профіль, записаний поряд з основним конфігом Codex. |
| `$CODEX_HOME/opencodex-catalog.json` | Синхронізований каталог нативних і маршрутизованих моделей, який використовує Codex. |

:::note
opencodex ніколи не видаляє ваш конфіг Codex. Кожна інтеграція оборотна — `ocx stop`, `ocx restore`
або `ocx eject` прибирають рівно ті рядки, які додав opencodex, і відновлюють нативний Codex.
:::

## Далі

Переходьте до [Швидкого старту](/opencodex/uk/getting-started/quickstart/), щоб налаштувати свого
першого провайдера, або прочитайте [Як це працює](/opencodex/uk/getting-started/how-it-works/) для
ознайомлення з архітектурою.
