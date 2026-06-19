# Приложение 4: Руководство по созданию хуков для Cline

> **Назначение:** Полное руководство по двум системам хуков в Cline — файловым хукам (`.cline/hooks/`) и плагинным runtime-хукам. Охватывает технику, форматы, примеры и диагностику.

> **Важно:** Файловые хуки работают в Cline CLI и VS Code. Плагинные runtime-хуки работают только в CLI и SDK.

---

## Оглавление

1. [Главный вопрос](#1-главный-вопрос)
2. [Две системы хуков: выбор правильной](#2-две-системы-хуков-выбор-правильной)
3. [Файловые хуки: как работают технически](#3-файловые-хуки-как-работают-технически)
4. [Доступные файловые хуки](#4-доступные-файловые-хуки)
5. [Формат ввода и вывода файловых хуков](#5-формат-ввода-и-вывода-файловых-хуков)
6. [Хранение и приоритеты файловых хуков](#6-хранение-и-приоритеты-файловых-хуков)
7. [Примеры файловых хуков](#7-примеры-файловых-хуков)
8. [Плагинные runtime-хуки: как работают технически](#8-плагинные-runtime-хуки-как-работают-технически)
9. [Семь плагинных хуков](#9-семь-плагинных-хуков)
10. [Управление потоком из плагинных хуков](#10-управление-потоком-из-плагинных-хуков)
11. [Примеры плагинных хуков](#11-примеры-плагинных-хуков)
12. [Диагностика проблем](#12-диагностика-проблем)
13. [Чеклист перед использованием хуков](#13-чеклист-перед-использованием-хуков)

---

## 1. Главный вопрос

Перед созданием хука задай себе один вопрос:

**«Нужно ли мне наблюдать за агентным циклом или управлять им в конкретной точке?»**

Хук нужен если:

| Ситуация | Пример |
|----------|--------|
| Нужно заблокировать опасное действие | Запрет `git push --force` на `main` |
| Нужно логировать каждый вызов инструмента | Аудит всех операций с файлами |
| Нужно добавить контекст перед следующим запросом к модели | Инъекция текущей ветки, статуса CI |
| Нужно уведомить о завершении задачи | Slack-сообщение после успешного деплоя |
| Нужно валидировать ввод пользователя | Проверка что задача содержит тикет-номер |

Хук **не нужен** если:
- Достаточно правила (`.cline/rules/`) — для статического контекста
- Достаточно воркфлоу (`.cline/workflows/`) — для повторяющейся процедуры
- Нужен новый инструмент для модели — используй плагин с `registerTool`

---

## 2. Две системы хуков: выбор правильной

В Cline существуют две независимые системы хуков:

| | Файловые хуки | Плагинные runtime-хуки |
|---|---|---|
| **Где хранятся** | `.cline/hooks/` | `.cline/plugins/` |
| **Язык** | Любой (bash, node, python) | TypeScript |
| **Работают в** | CLI + VS Code | CLI + SDK |
| **Интеграция** | Внешний процесс, JSON через stdin/stdout | In-process, типизированные коллбэки |
| **Когда использовать** | Простые скрипты, любой язык, VS Code | Сложная логика, типы, мутация данных |

**Правило выбора:**

> Нужно работать в VS Code или писать на bash/python → **файловые хуки**
>
> Нужна типизация, мутация ввода инструмента, замена результата → **плагинные хуки**

---

## 3. Файловые хуки: как работают технически

### Механизм выполнения

Когда наступает событие (например, перед вызовом инструмента):

1. Cline находит все файлы хуков для этого события в директориях хуков
2. Запускает их **параллельно** (`Promise.all`)
3. Каждый хук получает JSON через **stdin**
4. Каждый хук возвращает JSON через **stdout**
5. Результаты объединяются:
   - `cancel`: если ХОТЯ БЫ ОДИН вернул `true` → выполнение заблокировано
   - `contextModification`: все строки конкатенируются через `\n\n`
   - `errorMessage`: все сообщения конкатенируются через `\n`

### Важно: контекст влияет на СЛЕДУЮЩИЙ запрос

```
1. Модель решила: "вызову write_to_file с этими параметрами"
2. PreToolUse хук запускается → может заблокировать или добавить контекст
3. Если разрешено → инструмент выполняется с ОРИГИНАЛЬНЫМИ параметрами
4. Контекст добавляется в историю диалога
5. СЛЕДУЮЩИЙ запрос к модели включает этот контекст
6. Модель корректирует БУДУЩИЕ решения
```

Файловые хуки **не могут изменить параметры** текущего вызова инструмента. Для мутации параметров используй плагинные хуки.

### Требования к файлу хука

- **Без расширения**: `PreToolUse`, не `PreToolUse.sh`
- **Shebang в первой строке**: `#!/usr/bin/env bash`
- **Исполняемый**: `chmod +x PreToolUse`
- **Всегда возвращает валидный JSON** в stdout
- **Таймаут**: 30 секунд (настраивается через `HOOK_EXECUTION_TIMEOUT_MS`)
- **Лимит контекста**: 50KB (настраивается через `MAX_CONTEXT_MODIFICATION_SIZE`)
- **Windows**: не поддерживается

---

## 4. Доступные файловые хуки

| Хук | Когда срабатывает | Можно отменить? |
|-----|-------------------|-----------------|
| `TaskStart` | При старте НОВОЙ задачи | Да |
| `TaskResume` | При возобновлении существующей задачи | Да |
| `TaskCancel` | При отмене задачи пользователем | Нет |
| `TaskComplete` | При завершении задачи *(coming soon)* | Да |
| `UserPromptSubmit` | При отправке сообщения пользователем | Да |
| `PreToolUse` | Перед каждым вызовом инструмента | Да |
| `PostToolUse` | После каждого вызова инструмента | Нет |
| `PreCompact` | Перед компактизацией контекста *(coming soon)* | Нет |

---

## 5. Формат ввода и вывода файловых хуков

### Входной JSON (через stdin)

```json
{
  "clineVersion": "string",
  "hookName": "PreToolUse",
  "timestamp": "string",
  "taskId": "string",
  "workspaceRoots": ["string"],
  "userId": "string",

  "preToolUse": {
    "toolName": "write_to_file",
    "parameters": { "path": "src/index.ts", "content": "..." }
  }
}
```

Каждый хук получает только своё поле (`preToolUse`, `postToolUse`, `taskStart` и т.д.):

| Хук | Поле | Ключевые данные |
|-----|------|-----------------|
| `TaskStart` | `taskStart.taskMetadata` | `taskId`, `initialTask` |
| `TaskResume` | `taskResume.previousState` | `messageCount`, `lastMessageTs` |
| `TaskCancel` | `taskCancel.taskMetadata` | `completionStatus` |
| `UserPromptSubmit` | `userPromptSubmit` | `prompt`, `attachments` |
| `PreToolUse` | `preToolUse` | `toolName`, `parameters` |
| `PostToolUse` | `postToolUse` | `toolName`, `parameters`, `result`, `success`, `executionTimeMs` |

### Выходной JSON (через stdout)

```json
{
  "cancel": false,
  "contextModification": "Строка контекста для следующего запроса к модели",
  "errorMessage": "Сообщение об ошибке (показывается пользователю при cancel: true)"
}
```

| Поле | Тип | Описание |
|------|-----|----------|
| `cancel` | boolean | `true` → заблокировать выполнение |
| `contextModification` | string | Контекст для СЛЕДУЮЩЕГО запроса к модели |
| `errorMessage` | string | Сообщение при блокировке |

**Минимальный ответ** (разрешить выполнение):
```json
{"cancel": false}
```

---

## 6. Хранение и приоритеты файловых хуков

### Пути хранения

```
Проектные хуки:
  .cline/hooks/

Глобальные хуки:
  ~/Documents/Cline/Hooks/   (macOS / Windows)
  ~/.cline/hooks/            (Linux / через CLINE_DIR)
```

> **Примечание:** Cline также поддерживает legacy-путь `.clinerules/hooks/`. Если у тебя есть файлы в этой директории — они продолжат работать, но рекомендуется перенести их в `.cline/hooks/`.

### Включение хуков

**VS Code:** Настройки → Feature Settings → включить "Enable Hooks"

**CLI:** Хуки включены по умолчанию. Отключаются флагом `--yolo`.

### Параллельное выполнение

Все хуки для одного события (глобальные + проектные) запускаются **одновременно**. Порядок выполнения не гарантирован. Если нужна последовательность — объедини логику в один файл.

### Глобальные vs проектные

> **Это нужно во ВСЕХ проектах?**
> ├── Да → глобальный хук (`~/.cline/hooks/`)
> │         Примеры: запрет удаления `package.json`, логирование всех операций
> └── Нет → проектный хук (`.cline/hooks/`)
>           Примеры: правила конкретного репозитория, защита конкретных веток

Проектные хуки коммитятся в репозиторий и шарятся с командой.

---

## 7. Примеры файловых хуков

### Пример 1: Блокировка опасных git-команд

**Файл:** `.cline/hooks/PreToolUse`

```bash
#!/usr/bin/env bash
set -euo pipefail

input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName // ""')

if [[ "$tool_name" != "run_commands" ]]; then
  echo '{"cancel": false}'
  exit 0
fi

commands=$(echo "$input" | jq -r '.preToolUse.parameters.commands[]? // ""')

while IFS= read -r cmd; do
  if echo "$cmd" | grep -qE 'git push.*--force|git reset --hard|git clean -fd'; then
    echo "{\"cancel\": true, \"errorMessage\": \"Blocked dangerous command: $cmd\"}"
    exit 0
  fi
done <<< "$commands"

echo '{"cancel": false}'
```

### Пример 2: Добавление контекста о текущей ветке

**Файл:** `.cline/hooks/PreToolUse`

```bash
#!/usr/bin/env bash
input=$(cat)
tool_name=$(echo "$input" | jq -r '.preToolUse.toolName // ""')

# Добавляем контекст только перед записью файлов
if [[ "$tool_name" != "write_to_file" ]]; then
  echo '{"cancel": false}'
  exit 0
fi

branch=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "unknown")
context="Current git branch: $branch. Ensure changes are appropriate for this branch."

echo "{\"cancel\": false, \"contextModification\": \"$context\"}"
```

### Пример 3: Логирование всех вызовов инструментов

**Файл:** `~/.cline/hooks/PostToolUse`

```bash
#!/usr/bin/env bash
input=$(cat)

log_dir="$HOME/.cline/hook-logs"
mkdir -p "$log_dir"

echo "$input" >> "$log_dir/tool-usage.jsonl"

echo '{"cancel": false}'
```

### Пример 4: Валидация задачи при старте

**Файл:** `.cline/hooks/TaskStart`

```bash
#!/usr/bin/env bash
input=$(cat)
task=$(echo "$input" | jq -r '.taskStart.taskMetadata.initialTask // ""')

# Требуем наличие тикет-номера в задаче
if ! echo "$task" | grep -qE '[A-Z]+-[0-9]+'; then
  echo '{"cancel": true, "errorMessage": "Task must reference a ticket number (e.g. PROJ-123)"}'
  exit 0
fi

echo '{"cancel": false}'
```

### Пример 5: Node.js хук с типизацией

**Файл:** `.cline/hooks/PreToolUse`

```javascript
#!/usr/bin/env node
import { createInterface } from "readline";

const rl = createInterface({ input: process.stdin });
let raw = "";
rl.on("line", (line) => (raw += line));
rl.on("close", () => {
  const input = JSON.parse(raw);
  const { toolName, parameters } = input.preToolUse ?? {};

  if (toolName === "write_to_file" && parameters?.path?.endsWith(".env")) {
    process.stdout.write(
      JSON.stringify({
        cancel: true,
        errorMessage: "Writing to .env files is not allowed",
      })
    );
    return;
  }

  process.stdout.write(JSON.stringify({ cancel: false }));
});
```

---

## 8. Плагинные runtime-хуки: как работают технически

Плагинные хуки — это TypeScript-коллбэки внутри объекта `AgentPlugin`. Они выполняются **in-process**, без IPC и JSON-маршалинга.

### Ключевые отличия от файловых хуков

| | Файловые хуки | Плагинные хуки |
|---|---|---|
| Могут мутировать ввод инструмента | Нет | Да |
| Могут заменить результат инструмента | Нет | Да |
| Могут изменить запрос к модели | Нет | Да |
| Выполняются параллельно | Да | Нет (последовательно) |
| Доступ к типам TypeScript | Нет | Да |

### Регистрация хуков в плагине

```typescript
import type { AgentPlugin } from "@cline/core";

const plugin: AgentPlugin = {
  name: "my-hooks-plugin",
  manifest: { capabilities: ["hooks"] },  // обязательно

  hooks: {
    beforeRun(ctx) { /* ... */ },
    afterRun(ctx) { /* ... */ },
    beforeModel(ctx) { /* ... */ },
    afterModel(ctx) { /* ... */ },
    beforeTool(ctx) { /* ... */ },
    afterTool(ctx) { /* ... */ },
    onEvent(event) { /* ... */ },
  },
};

export default plugin;
```

### Порядок выполнения нескольких плагинов

Если несколько плагинов объявляют один и тот же хук — они выполняются **последовательно** в порядке регистрации. Если один возвращает `{ stop: true }` — следующие не вызываются.

---

## 9. Семь плагинных хуков

| Хук | Когда срабатывает | Может остановить цикл? | Типичное применение |
|-----|-------------------|------------------------|---------------------|
| `beforeRun` | Перед стартом цикла (один тёрн) | Да | Логирование, инициализация |
| `afterRun` | После завершения цикла (любой статус) | Нет | Уведомления, метрики |
| `beforeModel` | Перед каждым запросом к модели | Да (мутация запроса) | Инъекция контекста, редактирование промпта |
| `afterModel` | После ответа модели, до выполнения инструментов | Да | Блокировка по содержимому ответа |
| `beforeTool` | Перед каждым вызовом инструмента | Да (`stop` или `skip`) | Аудит, блокировка, мутация ввода |
| `afterTool` | После каждого вызова инструмента | Да (замена результата) | Постобработка, редактирование вывода |
| `onEvent` | На каждое событие `AgentRuntimeEvent` | Нет | Стриминг UI, телеметрия |

### Контексты и возвращаемые значения

**`beforeRun` / `afterRun`**
```typescript
beforeRun(ctx: { snapshot: AgentRuntimeStateSnapshot })
  → { stop?: boolean; reason?: string } | undefined

afterRun(ctx: { snapshot: ...; result: AgentRunResult })
  → void
```

**`beforeModel`**
```typescript
beforeModel(ctx: { snapshot: ...; request: AgentModelRequest })
  → {
      stop?: boolean;
      reason?: string;
      messages?: readonly AgentMessage[];   // заменить историю
      tools?: readonly AgentToolDefinition[]; // заменить список инструментов
      options?: Record<string, unknown>;    // temperature и др.
    } | undefined
```

**`afterModel`**
```typescript
afterModel(ctx: { snapshot: ...; response: AgentModelResponse })
  → { stop?: boolean; reason?: string } | undefined
```

**`beforeTool`**
```typescript
beforeTool(ctx: { snapshot: ...; tool: AgentTool; toolCall: ...; input: unknown })
  → {
      stop?: boolean;   // остановить ВЕСЬ цикл
      skip?: boolean;   // пропустить ЭТОТ вызов
      reason?: string;
      input?: unknown;  // заменить ввод инструмента
    } | undefined
```

**`afterTool`**
```typescript
afterTool(ctx: { snapshot: ...; tool: ...; input: ...; result: AgentToolResult; durationMs: number })
  → {
      stop?: boolean;
      reason?: string;
      result?: AgentToolResult;  // заменить результат
    } | undefined
```

**`onEvent`**
```typescript
onEvent(event: AgentRuntimeEvent) → void
```

---

## 10. Управление потоком из плагинных хуков

### `stop` vs `skip` в `beforeTool`

```typescript
beforeTool({ toolCall, input }) {
  // skip: пропустить ЭТОТ вызов инструмента, цикл продолжается
  if (toolCall.toolName === "fetch_url" && isBlacklisted(input)) {
    return { skip: true, reason: "URL в чёрном списке" };
  }

  // stop: остановить ВЕСЬ цикл агента
  if (toolCall.toolName === "run_commands" && isDangerous(input)) {
    return { stop: true, reason: "Опасная команда заблокирована" };
  }

  return undefined; // продолжить
}
```

### Мутация ввода инструмента

```typescript
beforeTool({ toolCall, input }) {
  if (toolCall.toolName === "write_to_file") {
    const typedInput = input as { path: string; content: string };
    return {
      input: {
        ...typedInput,
        content: `// Auto-generated\n${typedInput.content}`,
      },
    };
  }
  return undefined;
}
```

### Замена результата инструмента

```typescript
afterTool({ toolCall, result }) {
  if (toolCall.toolName === "read_file") {
    const sanitized = redactSecrets(result.content as string);
    return { result: { ...result, content: sanitized } };
  }
  return undefined;
}
```

### Инъекция контекста в запрос к модели

```typescript
beforeModel({ request, snapshot }) {
  const branch = sessionBranch ?? "unknown";
  const contextMessage = {
    role: "user" as const,
    content: `[Context] Current branch: ${branch}. Iteration: ${snapshot.iteration}.`,
  };
  return {
    messages: [contextMessage, ...request.messages],
  };
}
```

### `afterRun`: только успешные завершения

`afterRun` срабатывает для ВСЕХ терминальных статусов: `completed`, `aborted`, `failed`. Всегда добавляй guard:

```typescript
afterRun({ result }) {
  if (result.status !== "completed") return;
  sendNotification(`Done in ${result.iterations} iterations`);
}
```

---

## 11. Примеры плагинных хуков

### Пример 1: Аудит и метрики

**Файл:** `.cline/plugins/audit-plugin.ts`

```typescript
import type { AgentPlugin } from "@cline/core";

interface SessionMetrics {
  toolCalls: number;
  startedAt: number;
}

const metrics = new Map<string, SessionMetrics>();

const plugin: AgentPlugin = {
  name: "audit-metrics",
  manifest: { capabilities: ["hooks"] },

  setup(api, ctx) {
    const id = ctx.session?.sessionId ?? "default";
    metrics.set(id, { toolCalls: 0, startedAt: Date.now() });
  },

  hooks: {
    beforeTool({ toolCall }) {
      console.log(`[audit] Tool: ${toolCall.toolName}`);
      return undefined;
    },

    afterRun({ result }) {
      if (result.status !== "completed") return;
      console.log(
        `[audit] Done: ${result.iterations} iterations, ` +
        `${result.usage.inputTokens + result.usage.outputTokens} tokens`
      );
    },
  },
};

export default plugin;
```

### Пример 2: Защита защищённых веток

**Файл:** `.cline/plugins/branch-guard.ts`

```typescript
import type { AgentPlugin } from "@cline/core";

const PROTECTED_BRANCHES = ["main", "master", "production"];
const DANGEROUS_PATTERNS = [
  /^git push.*--force/,
  /^git push origin (main|master|production)/,
  /^git reset --hard/,
];

let sessionBranch: string | undefined;

const plugin: AgentPlugin = {
  name: "branch-guard",
  manifest: { capabilities: ["hooks"] },

  setup(api, ctx) {
    sessionBranch = ctx.workspaceInfo?.latestGitBranchName;
  },

  hooks: {
    beforeTool({ toolCall, input }) {
      if (toolCall.toolName !== "run_commands") return undefined;

      const { commands = [] } = input as { commands?: string[] };
      const isProtected = PROTECTED_BRANCHES.includes(sessionBranch ?? "");

      if (isProtected) {
        const dangerous = commands.find(cmd =>
          DANGEROUS_PATTERNS.some(p => p.test(cmd))
        );
        if (dangerous) {
          return {
            stop: true,
            reason: `Blocked on protected branch '${sessionBranch}': ${dangerous}`,
          };
        }
      }

      return undefined;
    },
  },
};

export default plugin;
```

### Пример 3: Редактирование секретов из вывода инструментов

**Файл:** `.cline/plugins/secret-redactor.ts`

```typescript
import type { AgentPlugin } from "@cline/core";

const SECRET_PATTERNS = [
  /sk-[a-zA-Z0-9]{48}/g,
  /ghp_[a-zA-Z0-9]{36}/g,
  /AKIA[0-9A-Z]{16}/g,
  /password\s*[:=]\s*\S+/gi,
];

function redact(text: string): string {
  let result = text;
  for (const pattern of SECRET_PATTERNS) {
    result = result.replace(pattern, "[REDACTED]");
  }
  return result;
}

const plugin: AgentPlugin = {
  name: "secret-redactor",
  manifest: { capabilities: ["hooks"] },

  hooks: {
    afterTool({ result }) {
      const content = typeof result.content === "string"
        ? result.content
        : JSON.stringify(result.content);

      const redacted = redact(content);
      if (redacted === content) return undefined;

      return { result: { ...result, content: redacted } };
    },
  },
};

export default plugin;
```

### Пример 4: Стриминг событий в UI

**Файл:** `.cline/plugins/event-stream.ts`

```typescript
import type { AgentPlugin } from "@cline/core";

const plugin: AgentPlugin = {
  name: "event-stream",
  manifest: { capabilities: ["hooks"] },

  hooks: {
    onEvent(event) {
      switch (event.type) {
        case "assistant-text-delta":
          process.stdout.write(event.text ?? "");
          break;
        case "content_start":
          if (event.contentType === "tool") {
            console.log(`\n[tool] ${event.toolName}`);
          }
          break;
        case "finish":
          console.log(`\n[finish] ${event.reason}`);
          break;
      }
    },
  },
};

export default plugin;
```

---

## 12. Диагностика проблем

### Файловые хуки

#### Хук не запускается

- Убедись что "Enable Hooks" включён в настройках VS Code (для VS Code)
- Проверь что файл исполняемый: `chmod +x .cline/hooks/PreToolUse`
- Убедись что shebang в первой строке: `#!/usr/bin/env bash`
- Проверь синтаксис скрипта: `bash -n .cline/hooks/PreToolUse`
- Убедись что имя файла точно совпадает: `PreToolUse`, не `pre-tool-use` или `PreToolUse.sh`

#### Хук завершается с ошибкой

- Хук должен всегда возвращать валидный JSON в stdout
- Добавь обработку ошибок:
  ```bash
  trap 'echo "{\"cancel\": false}"' EXIT
  ```
- Проверь Output panel в VS Code (канал Cline) на наличие ошибок

#### Хук таймаутится

- Таймаут по умолчанию: 30 секунд
- Избегай тяжёлых операций (сетевые запросы, сложные вычисления)
- Увеличь таймаут: `HOOK_EXECUTION_TIMEOUT_MS=60000`

#### Контекст не влияет на поведение модели

- Помни: `contextModification` влияет на СЛЕДУЮЩИЙ запрос, не текущий
- Убедись что контекст конкретный и actionable
- Проверь что контекст не превышает 50KB

#### Windows не поддерживается

Файловые хуки не работают на Windows. Используй плагинные runtime-хуки вместо них.

### Плагинные хуки

#### "capabilities must be a non-empty array"

```typescript
manifest: { capabilities: ["hooks"] }  // минимум одна capability
```

#### Хуки не срабатывают

- Убедись что `"hooks"` есть в `manifest.capabilities`
- Убедись что объект `hooks` присутствует на плагине
- Проверь что плагин загружен: `cline config`

#### `afterRun` срабатывает при отмене

```typescript
afterRun({ result }) {
  if (result.status !== "completed") return;
  // ...
}
```

#### Состояние утекает между сессиями

```typescript
const state = new Map<string, MyState>();

setup(api, ctx) {
  const id = ctx.session?.sessionId;
  if (id) state.set(id, { /* ... */ });
}
```

#### `beforeTool` не мутирует ввод

```typescript
// Неправильно — мутация объекта не работает
beforeTool({ input }) {
  (input as any).extra = "value";
  return undefined;
}

// Правильно — возвращай новый объект
beforeTool({ input }) {
  return { input: { ...(input as object), extra: "value" } };
}
```

---

## 13. Чеклист перед использованием хуков

### Файловые хуки

- [ ] Файл без расширения (`PreToolUse`, не `PreToolUse.sh`)
- [ ] Shebang в первой строке (`#!/usr/bin/env bash` или `#!/usr/bin/env node`)
- [ ] Файл исполняемый (`chmod +x`)
- [ ] Скрипт всегда возвращает валидный JSON в stdout
- [ ] Добавлена обработка ошибок (`trap` или try/catch)
- [ ] Хук завершается быстро (цель: <100ms, максимум: 30 сек)
- [ ] `contextModification` конкретный и actionable
- [ ] Проектный хук — в `.cline/hooks/`, глобальный — в `~/.cline/hooks/`
- [ ] Протестирован вручную:
  ```bash
  echo '{"hookName":"PreToolUse","preToolUse":{"toolName":"test","parameters":{}}}' \
    | .cline/hooks/PreToolUse
  ```

### Плагинные хуки

- [ ] `"hooks"` есть в `manifest.capabilities`
- [ ] Объект `hooks` присутствует на плагине
- [ ] `afterRun` проверяет `result.status === "completed"` если нужны только успехи
- [ ] `beforeTool` возвращает новый объект с `input`, не мутирует напрямую
- [ ] Состояние, которое не должно утекать между сессиями, ключуется по `ctx.session?.sessionId`
- [ ] Все поля `ctx` проверяются на наличие перед использованием
- [ ] Плагин протестирован: `cline -i "..."` с включённым плагином
