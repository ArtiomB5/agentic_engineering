# Приложение 3: Руководство по созданию плагинов для Cline CLI

> **Назначение:** Полное руководство по проектированию, написанию, тестированию и публикации плагинов для Cline. Охватывает оба формата — одиночный файл и пакет с зависимостями.

> **Важно:** Плагины работают только в Cline CLI, SDK и Kanban. В расширениях VS Code и JetBrains эта функция пока недоступна.

---

## Оглавление

1. [Главный вопрос](#1-главный-вопрос)
2. [Как плагины работают технически](#2-как-плагины-работают-технически)
3. [Два формата плагина](#3-два-формата-плагина)
4. [Анатомия AgentPlugin](#4-анатомия-agentplugin)
5. [Capabilities — таблица возможностей](#5-capabilities--таблица-возможностей)
6. [Инструменты: registerTool](#6-инструменты-registertool)
7. [Хуки: hooks](#7-хуки-hooks)
8. [Другие точки расширения](#8-другие-точки-расширения)
9. [Контекст сессии: ctx](#9-контекст-сессии-ctx)
10. [Хранение и загрузка плагинов](#10-хранение-и-загрузка-плагинов)
11. [Установка через CLI](#11-установка-через-cli)
12. [Плагин-пакет: полная структура](#12-плагин-пакет-полная-структура)
13. [Тестирование](#13-тестирование)
14. [Диагностика проблем](#14-диагностика-проблем)
15. [Чеклист перед публикацией](#15-чеклист-перед-публикацией)

---

## 1. Главный вопрос

Перед созданием плагина задай себе один вопрос:

**«Нужно ли мне дать агенту новую возможность, которой нет в стандартном наборе инструментов?»**

Плагин нужен если:

| Ситуация | Пример |
|----------|--------|
| Нужен новый инструмент для модели | Запрос к внешнему API, работа с БД, вызов CLI-утилиты |
| Нужно наблюдать за агентным циклом | Логирование, метрики, аудит вызовов инструментов |
| Нужно блокировать опасные действия | Запрет `git push` на защищённую ветку |
| Нужно изменить системный промпт программно | Инъекция динамического контекста |
| Нужна кастомная компактизация истории | Замена середины диалога суммаризацией |

Плагин **не нужен** если:
- Достаточно правила (`.cline/rules/`) — для статического контекста
- Достаточно воркфлоу (`.cline/workflows/`) — для повторяющейся процедуры
- Достаточно скилла (`.cline/skills/`) — для специализированных инструкций

---

## 2. Как плагины работают технически

### Жизненный цикл плагина

При старте сессии хост запускает четыре фазы:

```
1. resolve   — собрать объекты плагинов из файлов и конфига
2. validate  — проверить manifest каждого плагина
3. setup     — вызвать setup(api, ctx) один раз для каждого плагина
4. activate  — заморозить реестр, запустить агентный цикл
```

После `activate` реестр заморожен — динамическая регистрация/отмена регистрации невозможна.

### Два инварианта реестра

- **Каждый вызов `api.register*` требует соответствующей capability.** Вызов `api.registerRule(...)` без `"rules"` в `manifest.capabilities` бросает ошибку.
- **Capabilities и handlers должны совпадать.** Объявить `"hooks"` без объекта `hooks` на плагине — ошибка валидации.

### Один плагин — все хосты

Один и тот же плагин работает в Cline CLI, Kanban и любом приложении на `@cline/core`. Пиши один раз — работает везде.

---

## 3. Два формата плагина

### Формат 1: Одиночный файл

Один `.ts` файл с `export default plugin`. Самый простой способ начать.

```
.cline/plugins/
└── my-plugin.ts
```

**Когда использовать:** нет внешних зависимостей, простая логика, быстрый прототип.

### Формат 2: Пакет с package.json

Директория с `package.json`, npm-зависимостями и опциональными ассетами.

```
my-cline-plugin/
├── package.json
├── index.ts
├── tools/
│   └── my-tool.ts
├── hooks/
│   └── audit.ts
└── assets/
    └── templates/
```

**Когда использовать:** нужны npm-зависимости (`zod`, `yaml`, SDK сторонних сервисов), несколько точек входа, бандлинг ассетов, публикация через npm или git.

Оба формата используют одинаковый API плагина. Пакет добавляет только управление зависимостями и ассетами.

---

## 4. Анатомия AgentPlugin

### Минимальный рабочий плагин

```typescript
import type { AgentPlugin } from "@cline/core";
import { createTool } from "@cline/core";

const plugin: AgentPlugin = {
  name: "hello-plugin",          // обязательно, уникально в рамках сессии
  manifest: {
    capabilities: ["tools"],     // объявляет что setup() зарегистрирует
  },
  setup(api, ctx) {
    api.registerTool(
      createTool({
        name: "say_hello",
        description: "Greet a person by name.",
        inputSchema: {
          type: "object",
          properties: { name: { type: "string" } },
          required: ["name"],
        },
        async execute({ name }: { name: string }) {
          return { greeting: `Hello, ${name}!` };
        },
      }),
    );
  },
};

export default plugin;
```

### Поля объекта AgentPlugin

| Поле | Обязательно | Описание |
|------|-------------|----------|
| `name` | Да | Уникальный идентификатор плагина в сессии. Используй namespace: `my-org-plugin`, не `plugin`. |
| `manifest.capabilities` | Да | Непустой массив. Объявляет что плагин регистрирует. |
| `setup(api, ctx)` | Нет | Вызывается один раз перед стартом агентного цикла. Здесь регистрируются все инструменты, команды, правила. |
| `hooks` | Нет | Объект с lifecycle-коллбэками. Требует `"hooks"` в capabilities. |

### Правила именования

- `name` должен быть уникальным в рамках сессии
- Используй namespace: `my-org-redactor`, не `redactor`
- Конфликт имён → ошибка валидации, плагин не загружается

---

## 5. Capabilities — таблица возможностей

| Capability | Что открывает в `api` | Когда нужна |
|---|---|---|
| `tools` | `api.registerTool()` | Дать модели новый инструмент |
| `commands` | `api.registerCommand()` | Добавить слэш-команду в чат |
| `rules` | `api.registerRule()` | Инжектировать текст в системный промпт |
| `messageBuilders` | `api.registerMessageBuilder()` | Переписать сообщения перед отправкой модели |
| `providers` | `api.registerProvider()` | Добавить кастомного провайдера модели |
| `automationEvents` | `api.registerAutomationEventType()` | Эмитировать нормализованные события автоматизации |
| `hooks` | Объект `hooks` на плагине | Наблюдать за агентным циклом или управлять им |

Большинство реальных плагинов используют 1–3 capability.

---

## 6. Инструменты: registerTool

Инструменты — основной способ дать агенту новые возможности. Используй хелпер `createTool()` из `@cline/core`:

```typescript
import { createTool } from "@cline/core";

api.registerTool(
  createTool({
    name: "get_weather",
    description: "Get current weather for a city. Returns temperature and conditions.",
    inputSchema: {
      type: "object",
      properties: {
        city: { type: "string", description: "The city name" },
        units: {
          type: "string",
          enum: ["celsius", "fahrenheit"],
          description: "Temperature units. Default: celsius.",
        },
      },
      required: ["city"],
    },
    async execute(input, context) {
      const { city, units = "celsius" } = input as { city: string; units?: string };
      // context.sessionId, context.conversationId, context.cwd доступны
      const data = await fetchWeather(city, units);
      return { city, temperature: data.temp, condition: data.condition };
    },
  }),
);
```

### Правила написания инструментов

| Правило | Плохо | Хорошо |
|---------|-------|--------|
| Имя — snake_case глагол | `WeatherTool`, `weather` | `get_weather`, `create_github_issue` |
| Описание — для модели, не для человека | "Gets weather" | "Get current weather for a city. Returns temperature and conditions. Use when the user asks about weather." |
| JSON Schema с `required` | `required` не указан | `required: ["city"]` |
| Возвращать JSON-сериализуемые значения | Объект с методами | Простые объекты, строки, числа, массивы |
| Бросать ошибку при невалидном вводе | Возвращать `null` | `throw new Error("city is required")` |
| Маленькие фокусированные инструменты | Один инструмент с `mode: "create"\|"read"\|"delete"` | Три отдельных инструмента |

---

## 7. Хуки: hooks

Хуки — типизированные in-process коллбэки, которые запускаются внутри агентного цикла. Требуют `"hooks"` в `manifest.capabilities`.

```typescript
const plugin: AgentPlugin = {
  name: "audit-plugin",
  manifest: { capabilities: ["hooks"] },
  hooks: {
    beforeRun(ctx) { /* ... */ },
    afterRun({ result }) { /* ... */ },
    beforeModel(ctx) { /* ... */ },
    afterModel(ctx) { /* ... */ },
    beforeTool({ toolCall, input }) { /* ... */ },
    afterTool({ toolCall, result }) { /* ... */ },
    onEvent(event) { /* ... */ },
  },
};
```

### Таблица хуков

| Хук | Когда срабатывает | Может остановить цикл? | Типичное использование |
|-----|-------------------|------------------------|------------------------|
| `beforeRun` | Перед стартом цикла (один тёрн пользователя) | Да | Логирование, инициализация |
| `afterRun` | После завершения цикла (успех, отмена, ошибка) | Нет | Уведомления, метрики |
| `beforeModel` | Перед каждым запросом к модели | Да (мутация запроса) | Инъекция контекста, правки промпта |
| `afterModel` | После ответа модели, до выполнения инструментов | Да | Блокировка по выводу модели |
| `beforeTool` | Перед каждым вызовом инструмента | Да (`{ stop: true }`) | Аудит, блокировка опасных вызовов |
| `afterTool` | После каждого вызова инструмента | Может заменить результат | Постобработка, редактирование вывода |
| `onEvent` | На каждое событие `AgentRuntimeEvent` | Нет | Стриминг в UI, телеметрия |

### Блокировка опасного инструмента

```typescript
beforeTool({ toolCall, input }) {
  if (toolCall.toolName === "run_commands") {
    const { commands } = input as { commands?: string[] };
    if (sessionBranch === "main" && commands?.some(c => c.startsWith("git push"))) {
      return { stop: true, reason: "Blocked git push on protected branch" };
    }
  }
  return undefined; // явное "продолжить"
}
```

### afterRun: только успешные завершения

`afterRun` срабатывает для всех терминальных статусов: `completed`, `aborted`, `failed`. Если нужно только успешное завершение:

```typescript
afterRun({ result }) {
  if (result.status !== "completed") return;
  console.log(`Done in ${result.iterations} iteration(s)`);
}
```

---

## 8. Другие точки расширения

### registerCommand — слэш-команды в чате

```typescript
manifest: { capabilities: ["commands"] },

setup(api) {
  api.registerCommand({
    name: "echo",
    description: "Echo input back",
    handler: async (input) => `echo: ${input}`,
  });
}
```

Команда становится доступна как `/echo` в чате.

### registerRule — инъекция в системный промпт

```typescript
manifest: { capabilities: ["rules"] },

setup(api, ctx) {
  const branch = ctx.workspaceInfo?.latestGitBranchName;
  api.registerRule({
    id: "branch-context",
    content: `Current git branch: ${branch ?? "unknown"}`,
    source: "my-plugin",
  });
}
```

Используй для динамического контекста, который нельзя захардкодить в файл правила.

### registerMessageBuilder — переписать историю перед моделью

```typescript
manifest: { capabilities: ["messageBuilders"] },

setup(api) {
  api.registerMessageBuilder({
    name: "summarize-middle-history",
    build(messages) {
      if (estimateTokens(messages) < THRESHOLD) return messages;
      return [...prefix, summary, ...recent];
    },
  });
}
```

Несколько builders запускаются в порядке регистрации: вывод одного — вход следующего.

**Когда использовать `beforeModel` вместо builder:** только если нужен доступ к runtime-снапшоту или мутация объекта запроса. Чистые переписывания сообщений — в builder.

---

## 9. Контекст сессии: ctx

Второй аргумент `setup(api, ctx)` — контекст хоста. **Все поля опциональны** — feature-detect перед использованием, один плагин должен работать в разных хостах.

```typescript
ctx.session?.sessionId          // стабильный ID сессии
ctx.client?.name                // "cline-cli", "cline-vscode", и т.д.
ctx.workspaceInfo?.rootPath     // корень рабочего пространства
ctx.workspaceInfo?.latestGitBranchName
ctx.workspaceInfo?.latestGitCommitHash
ctx.workspaceInfo?.associatedRemoteUrls
ctx.automation?.ingestEvent     // эмитировать события автоматизации
ctx.logger?.log                 // структурированный логгер плагина
ctx.telemetry                   // только in-process, не в sandboxed плагинах
```

### Два главных правила ctx

1. **Всегда используй `ctx.workspaceInfo?.rootPath` вместо `process.cwd()`** — CLI может быть запущен с `--cwd` без `chdir`, VS Code не имеет единого CWD.
2. **Не используй `import.meta.url` для поиска рабочего пространства** — это даёт путь к самому плагину, не к проекту пользователя.

### Состояние между setup и hooks

```typescript
let sessionRoot: string | undefined;
let sessionBranch: string | undefined;

const plugin: AgentPlugin = {
  name: "my-plugin",
  manifest: { capabilities: ["tools", "hooks"] },
  setup(api, ctx) {
    sessionRoot = ctx.workspaceInfo?.rootPath;
    sessionBranch = ctx.workspaceInfo?.latestGitBranchName;
  },
  hooks: {
    beforeTool({ toolCall }) {
      if (sessionBranch === "main") { /* ... */ }
    },
  },
};
```

### Многосессионный хост

Один Node-процесс может хостить несколько сессий одновременно. Если плагин будет работать в таком хосте — ключи состояния по `sessionId`:

```typescript
const stateBySession = new Map<string, MyState>();

setup(api, ctx) {
  const id = ctx.session?.sessionId;
  if (id) stateBySession.set(id, { root: ctx.workspaceInfo?.rootPath });
}
```

---

## 10. Хранение и загрузка плагинов

### Пути хранения

```
Проектные плагины:
  .cline/plugins/

Глобальные плагины:
  ~/.cline/plugins/

Установленные через cline plugin install:
  ~/.cline/plugins/_installed/
    npm/       # npm-пакеты
    git/       # git-репозитории
    remote/    # файлы по URL
    local/     # локальные копии
```

### Три способа загрузки

**1. Авто-обнаружение (CLI)**

CLI сканирует `.cline/plugins/` и `~/.cline/plugins/` при старте. Просто положи файл:

```bash
mkdir -p .cline/plugins
cp my-plugin.ts .cline/plugins/
cline -i "do the thing my plugin enables"
```

**2. `extensions: [...]` в SDK-конфиге**

```typescript
import plugin from "./my-plugin";
import { ClineCore } from "@cline/core";

const host = await ClineCore.create({ backendMode: "local" });
await host.start({
  config: {
    extensions: [plugin],
    extensionContext: {
      workspace: { rootPath: process.cwd(), cwd: process.cwd() },
    },
    // ...остальной конфиг
  },
  prompt: "...",
  interactive: false,
});
```

**3. `pluginPaths: [...]` для пакетов**

```typescript
config: {
  pluginPaths: ["./path/to/my-plugin-package"],
}
```

---

## 11. Установка через CLI

```bash
cline plugin install <source> [options]
# сокращение:
cline plugin i <source>
```

### Источники установки

```bash
# Файл по URL (GitHub blob или raw)
cline plugin install https://github.com/owner/repo/blob/main/plugins/my-plugin.ts

# Git-репозиторий
cline plugin install --git https://github.com/owner/repo.git
cline plugin install --git github.com/owner/repo

# Конкретная ветка или тег
cline plugin install https://github.com/owner/repo.git@v1.2.0

# npm-пакет
cline plugin install --npm @scope/my-plugin

# Локальный путь (файл или директория)
cline plugin install ./my-plugin.ts
cline plugin install ./my-plugin-package
```

### Флаги

| Флаг | Описание |
|------|----------|
| `--npm` | Источник — npm-пакет |
| `--git` | Источник — git-репозиторий |
| `--force` | Перезаписать существующий плагин |
| `--json` | Вывод в JSON (для скриптов) |
| `--cwd <path>` | Установить в `<path>/.cline/plugins` (проектный плагин) |

### Проверить установку

```bash
cline config
```

Плагин должен появиться в секции plugins.

---

## 12. Плагин-пакет: полная структура

### Структура директории

```
my-cline-plugin/
├── package.json          # обязательно: discovery contract
├── tsconfig.json         # опционально: для локальной проверки типов
├── index.ts              # точка входа плагина
├── README.md             # документация для пользователей
├── tools/                # опционально: разбивка по файлам
│   ├── create-issue.ts
│   └── list-issues.ts
├── hooks/
│   └── audit.ts
└── assets/               # опционально: бандлинг ассетов
    └── templates/
        └── report.md
```

### package.json

```json
{
  "name": "my-cline-plugin",
  "version": "0.1.0",
  "description": "What this plugin does, in one sentence.",
  "type": "module",
  "exports": {
    ".": "./index.ts"
  },
  "cline": {
    "plugins": [
      {
        "paths": ["./index.ts"],
        "capabilities": ["tools", "hooks"]
      }
    ]
  },
  "peerDependencies": {
    "@cline/core": "*"
  },
  "peerDependenciesMeta": {
    "@cline/core": { "optional": true }
  },
  "dependencies": {
    "zod": "^4.1.5"
  }
}
```

**Ключевые поля:**

- **`type: "module"`** — обязательно. Плагины Cline — ES-модули.
- **`cline.plugins`** — discovery contract. Массив объектов с `paths` и `capabilities`.
- **`@cline/core` как optional peer dep** — хост уже предоставляет `@cline/core`, `@cline/sdk`, `@cline/agents`, `@cline/llms`, `@cline/shared`. Не включай их в `dependencies`.
- **`dependencies`** — только твои зависимости (парсеры, SDK сторонних сервисов).

### Если нет поля `cline.plugins`

Установщик переходит к авто-обнаружению: ищет стандартные точки входа, затем рекурсивно сканирует `.ts` и `.js` файлы (пропуская `node_modules` и `.git`).

### Разрешение ассетов внутри пакета

```typescript
import { dirname, join } from "node:path";
import { fileURLToPath } from "node:url";
import { readFileSync, existsSync } from "node:fs";

const MODULE_DIR = dirname(fileURLToPath(import.meta.url));
const TEMPLATES_DIR = join(MODULE_DIR, "assets", "templates");

function loadTemplate(name: string): string | undefined {
  const path = join(TEMPLATES_DIR, `${name}.md`);
  return existsSync(path) ? readFileSync(path, "utf8") : undefined;
}
```

`import.meta.url` — единственное место где он уместен в плагине: для файлов **внутри пакета**. Для путей рабочего пространства — только `ctx.workspaceInfo?.rootPath`.

### Несколько плагинов в одном пакете

```json
"cline": {
  "plugins": [
    { "paths": ["./tools-plugin.ts"], "capabilities": ["tools"] },
    { "paths": ["./hooks-plugin.ts"], "capabilities": ["hooks"] }
  ]
}
```

Каждый файл должен `export default` свой объект плагина.

---

## 13. Тестирование

### Юнит-тесты: мок api

Объект плагина — простые данные. Можно вызвать `setup()` с минимальным контекстом:

```typescript
import plugin from "../my-plugin";

const tools: unknown[] = [];
const api = {
  registerTool: (t: unknown) => tools.push(t),
  registerCommand: () => {},
  registerRule: () => {},
  registerMessageBuilder: () => {},
  registerProvider: () => {},
  registerAutomationEventType: () => {},
};

await plugin.setup?.(api as never, {
  workspaceInfo: { rootPath: "/tmp/fake-workspace" },
});

// tools содержит зарегистрированные инструменты
// вызови tool.execute(input, ctx) напрямую
```

### E2E-тест: runDemo()

Добавь `runDemo()` в файл плагина для быстрой проверки end-to-end:

```typescript
async function runDemo(): Promise<void> {
  const host = await ClineCore.create({ backendMode: "local" });
  try {
    const result = await host.start({
      config: {
        providerId: "anthropic",
        modelId: "claude-sonnet-4-6",
        apiKey: process.env.ANTHROPIC_API_KEY ?? "",
        cwd: process.cwd(),
        enableTools: true,
        systemPrompt: "You are a helpful assistant. Use tools when needed.",
        extensions: [plugin],
        extensionContext: {
          workspace: { rootPath: process.cwd(), cwd: process.cwd() },
        },
      },
      prompt: "Use do_thing on the target 'world'.",
      interactive: false,
    });
    console.log(result.result?.text ?? "");
  } finally {
    await host.dispose();
  }
}

if (import.meta.main) {
  await runDemo();
}
```

Запуск:

```bash
ANTHROPIC_API_KEY=sk-... bun run my-plugin.ts
```

### CLI smoke test

```bash
mkdir -p .cline/plugins
cp my-plugin.ts .cline/plugins/
cline -i "trigger something that exercises the plugin"
```

Для пакетов:

```bash
cline plugin install ./my-cline-plugin
cline -i "..."
```

Если плагин не прошёл валидацию или `setup()` — CLI выводит ошибку и продолжает без него.

---

## 14. Диагностика проблем

### "capabilities must be a non-empty array"

Забыл `manifest.capabilities` или передал `[]`. Исправь:

```typescript
manifest: { capabilities: ["tools"] }  // минимум одна capability
```

### "registerRule requires the 'rules' capability"

Вызываешь `api.registerRule(...)` без `"rules"` в capabilities. Добавь:

```typescript
manifest: { capabilities: ["tools", "rules"] }
```

### Инструмент не виден модели

- Проверь `enableTools: true` в конфиге сессии
- Убедись что `"tools"` есть в `manifest.capabilities`
- Убедись что `api.registerTool()` вызывается в `setup()`, не в хуке

### `ctx.workspaceInfo` undefined в тестах

Хост не передал `extensionContext.workspace`. В SDK-коде задай явно:

```typescript
extensionContext: {
  workspace: { rootPath: process.cwd(), cwd: process.cwd() },
}
```

### Состояние утекает между сессиями

Модульные переменные разделяются между сессиями в одном процессе. Ключи по `ctx.session?.sessionId`:

```typescript
const stateBySession = new Map<string, MyState>();
```

### `afterRun` срабатывает на отменённых сессиях

Добавь guard:

```typescript
afterRun({ result }) {
  if (result.status !== "completed") return;
  // ...
}
```

### Конфликт имён плагинов

Два плагина с одинаковым `name` → ошибка валидации. Используй namespace:

```typescript
name: "my-org-github-plugin"  // не "github"
```

### Плагин не загружается из пакета

- Проверь что `name` в `package.json` совпадает с именем директории
- Проверь что `type: "module"` есть в `package.json`
- Проверь что `cline.plugins[].paths` указывают на существующие файлы

---

## 15. Чеклист перед публикацией

### Общее

- [ ] `manifest.capabilities` — непустой массив
- [ ] Каждый вызов `api.register*` имеет соответствующую capability
- [ ] Если `hooks` присутствует — `"hooks"` есть в capabilities
- [ ] `ctx.workspaceInfo?.rootPath` используется для путей рабочего пространства (не `process.cwd()`)
- [ ] Все поля `ctx` проверяются на наличие перед использованием (`ctx.session?.sessionId`)
- [ ] `afterRun` проверяет `result.status === "completed"` если нужны только успешные завершения
- [ ] Состояние, которое не должно утекать между сессиями, ключуется по `ctx.session?.sessionId`

### Инструменты

- [ ] Имена инструментов — snake_case глаголы
- [ ] Описания написаны для модели (когда использовать, что принимает, что возвращает)
- [ ] JSON Schema с явным `required`
- [ ] `execute()` бросает ошибку при невалидном вводе
- [ ] Возвращаемые значения JSON-сериализуемы

### Пакет

- [ ] `package.json` содержит `type: "module"`, `cline.plugins`, `@cline/core` как optional peer dep
- [ ] `@cline/core`, `@cline/sdk` и другие `@cline/*` пакеты — в `peerDependencies`, не в `dependencies`
- [ ] Ассеты внутри пакета разрешаются через `import.meta.url`, не `process.cwd()`
- [ ] Smoke test: `cline plugin install ./my-plugin`, `cline -i "..."` — работает

### Финальная проверка

- [ ] Юнит-тест: `setup()` вызван с мок-api, инструменты зарегистрированы
- [ ] E2E-тест: `runDemo()` или CLI smoke test прошёл
- [ ] README.md описывает что делает плагин и как его установить
