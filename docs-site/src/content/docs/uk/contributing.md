---
title: Внесок у проєкт
description: Розробляйте opencodex — налаштування, структура, конвенції та як додати провайдера чи адаптер.
---

## Налаштування

```bash
git clone https://github.com/lidge-jun/opencodex.git
cd opencodex
bun install
bun run dev:proxy    # proxy API в режимі розробки
bun run dev:gui      # dev-сервер панелі (в іншому терміналі)
bun run typecheck    # bun x tsc --noEmit
bun run test         # bun test ./tests/
```

`bun run dev` залишається аліасом для `bun run dev:proxy`. Dev-сервер панелі — це `bun run dev:gui`;
запакована панель за `GET /` виробляється `bun run build:gui` (`gui/dist`).

## Команди збірки й тестування

Кореневий пакет — це Bun-нативний TypeScript; окремого кроку компіляції сервера немає. Використовуйте
закомічені скрипти, щоб локальні команди відповідали CI:

```bash
bun run typecheck                 # сувора перевірка TypeScript
bun run test                      # повна набірка tests/
bun test tests/router.test.ts     # сфокусований тестовий файл
bun run build:gui                 # збірка Vite GUI + підготовка пакета
bun run privacy:scan              # скан облікових даних/приватності, використовуваний CI
bun run prepare:package           # оновити лаунчери/ресурси пакета
```

Більшість тестів — це пласкі Bun-тести `tests/*.test.ts`. `tests/helpers/` містить спільні фікстури, а
`tests/e2e-style/` містить ширші сценарії паритету з нативним. Тримайте сфокусовану регресію поруч із
наявними тестами для підсистеми, яку ви змінюєте; запускайте повну набірку для спільної поведінки
маршрутизації, адаптерів, конфіга чи сервера.

Сайт документації, який ви читаєте, живе в `docs-site/` (Astro + Starlight):

```bash
cd docs-site && bun install && bun dev
```

## Публікація документації

Публічна документація публікується на GitHub Pages за <https://lidge-jun.github.io/opencodex/>.
Робочий процес `.github/workflows/deploy-docs.yml` запускається на push-ах до `main`, що торкаються
`docs-site/**` чи самого робочого процесу, збирає `docs-site` і розгортає згенерований сайт. Перед
push-ом змін документації виконайте:

```bash
cd docs-site
bun install --frozen-lockfile
bun run build
```

## CI та релізи

GitHub Actions навмисно залишаються невеликими:

- **Крос-платформний CI** (`.github/workflows/ci.yml`) запускається на pull request-ах та push-ах до
  `main`, що торкаються файлів runtime, тестів, пакета, скриптів, TypeScript чи робочого процесу. Його
  матриця Bun покриває Linux, Windows та macOS зі встановленням, typecheck, тестами, сканом приватності,
  smoke-збіркою release-хелпера, збіркою GUI та `ocx help`. Друга смуга з трьома ОС доводить, що
  глобальне встановлення npm працює без окремо встановленого Bun, використовуючи вбудоване середовище
  виконання пакета.
- **Реліз** (`.github/workflows/release.yml`) ручний. Він не діє як другий повний CI-конвеєр; перед
  dry-run чи публікацією він вимагає, щоб точний реліз-коміт (`GITHUB_SHA`) уже мав успішний запуск
  крос-платформного CI.

Використовуйте хелпер для релізів:

```bash
bun run release <version>           # комітить/пушить bump; workflow публікації за замовчуванням dry-run
bun run release <version> --publish # публікує після усвідомлення CI-гейтованого dry-run
bun run release:watch               # стежити за найновішим запуском Release workflow
```

## Супроводжувачі проєкту

Поточні супроводжувачі, їхні обов'язки та політика review й merge задокументовані в
[`MAINTAINERS.md`](https://github.com/lidge-jun/opencodex/blob/main/MAINTAINERS.md). Власність GitHub
review для репозиторію та чутливих до безпеки шляхів оголошено в `.github/CODEOWNERS`.

## Конвенції

- **Лише ES Modules** (`import`/`export`), TypeScript, режим `strict`. Тримайте `bun x tsc --noEmit`
  чистим.
- **Макс. ~500 рядків на файл** — розділяйте за відповідальністю (sidecar-и `web-search/` та `vision/` —
  хороші приклади малих, сфокусованих модулів за одним `index.ts`).
- **Обробляйте асинхронні помилки на межах** — sidecar-и ніколи не кидають у шлях запиту; вони деградують
  до плавного маркера.
- **Structure SOT** — поточні інваріанти супроводжувачів живуть у `structure/`. Тримайте публічні робочі
  процеси користувачів у `docs-site/`, а історичні нотатки досліджень — у `docs/`.
- **Зберігайте експорти** — інші модулі можуть від них залежати.

## Додавання провайдера до каталогу

Усі селектори й засіви провайдерів походять із канонічного реєстру (`src/providers/registry.ts`):

```ts
{
  id: "my-provider",
  label: "My Provider",
  baseUrl: "https://api.example.com/v1",
  adapter: "openai-chat",
  authKind: "key",
  dashboardUrl: "https://example.com/keys",
  models: ["model-a", "model-b"],
  defaultModel: "model-a",
  noVisionModels: ["model-a"],   // текстові моделі → sidecar зору описує зображення
},
```

`src/providers/derive.ts` подає цей запис у `ocx init`, `ocx provider`, пресети панелі, вхід за
API-ключем та засіви конфіга OAuth. `enrichProviderFromCatalog()` копіює метадані моделей та
класифікації можливостей у збережений конфіг провайдера. Реалізації протоколу OAuth усе одно живуть у
`src/oauth/`; лише метадані реєстру не є потоком OAuth.

## Додавання адаптера

Реалізуйте `ProviderAdapter` (див. [Адаптери](/opencodex/uk/reference/adapters/)) у `src/adapters/`,
зареєструйте його ім'я в `src/server/adapter-resolve.ts` і змостіть його вивід до внутрішніх
`AdapterEvent`-ів. Повторно використовуйте `image.ts` для обробки зображень і слідуйте `openai-chat.ts`
для звичайного стрімінгу/викликів інструментів; використовуйте `fetchResponse` лише коли адаптер володіє
повторами транспорту, або `runTurn` для справді двонаправленого транспорту, як-от Cursor. Додайте
сфокусовані тести під `tests/` і експортуйте фабрику з `src/index.ts`, коли вона належить до публічного
API пакета.

## Перевірте, перш ніж заявляти про завершення

Запустіть найвужчу команду, що доводить вашу зміну — `bun run typecheck` для типів, сфокусований
`bun test tests/<name>.test.ts` чи runtime-проба для поведінки, потім ширші гейти, доречні для зачепленого
інтерфейсу. opencodex віддає перевагу малим, перевірюваним комітам над великими партіями.
