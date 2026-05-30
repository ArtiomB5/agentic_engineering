# Глава 7. Автоматизация и масштабирование

## Содержание

* [7.1. Headless Mode — Headless режим и CI/CD интеграция](#71-headless-mode--headless-режим-и-cicd-интеграция)
* [7.2. Scheduled Agents — Запуск по расписанию](#72-scheduled-agents--запуск-по-расписанию)
* [7.3. Multi-Agent Teams — Координация агентов](#73-multi-agent-teams--координация-агентов)
* [7.4. Kanban — Параллельный запуск с изолированными worktrees](#74-kanban--параллельный-запуск-с-изолированными-worktrees)
* [7.5. Chat Connectors — Агент в мессенджерах](#75-chat-connectors--агент-в-мессенджерах)
* [Общий принцип главы](#общий-принцип-главы)
* [Практические выводы](#практические-выводы)

---

До этой главы мы работали с Cline в интерактивном режиме: один разработчик, одна сессия, один агент. Теперь пора поднять ставки. Cline CLI открывает возможности, которых нет в VS Code и JetBrains: запуск агентов без участия человека, задачи по расписанию, целые команды агентов и управление ими прямо из Telegram.

Всё, о чём пойдёт речь дальше, относится к CLI, Kanban и SDK. Разделы 7.2, 7.3 и 7.5 в VS Code Extension и JetBrains Plugin недоступны.

---

## 7.1. Headless Mode — Headless режим и CI/CD интеграция

Интерактивный интерфейс — правильный инструмент для разработки. Но многие задачи не требуют вашего присутствия: ревью PR, анализ diff перед мержем, проверка зависимостей, генерация release notes. Для этого нужен агент, который запускается, делает дело и возвращает результат — без диалогов и ожидания.

**Headless режим** превращает Cline в Unix-утилиту: принимает ввод через stdin, возвращает вывод через stdout, интегрируется в любой пайплайн.

Активируется он автоматически в трёх случаях:

| Вызов | Что происходит |
|-------|----------------|
| `cline --json "task"` | Включается JSON output mode |
| `cat file \| cline "task"` | stdin перенаправлен — TUI не нужен |
| `cline "task" > output.txt` | stdout перенаправлен — некому показывать |

В headless режиме Cline не рисует TUI и не ждёт вашего ввода. Задача выполняется, процесс завершается.

**Базовые паттерны, которые пригодятся каждый день:**

```
# Ревью git diff одной командой
git diff | cline "review these changes for potential regressions"

# JSON-вывод для парсинга
cline --json "list all TODO comments" | jq -r 'select(.type == "agent_event" and .event.text) | .event.text'

# Цепочка: объясни изменения → напиши commit message
git diff | cline "explain these changes" | cline "write a commit message"

# Полностью автономное выполнение — без единого подтверждения
cline --auto-approve true "run tests and fix any failures"

# Plan Mode в headless — спроектировать, не трогая код
cline -p "design migration plan for adding user roles"
```

**Ограничение команд в CI:** В CI-окружении агент не должен иметь доступ к `rm -rf /` или `sudo`. Используйте переменную `CLINE_COMMAND_PERMISSIONS`:

```
export CLINE_COMMAND_PERMISSIONS='{"allow": ["npm *", "git *"], "deny": ["rm -rf *", "sudo *"]}'
cline --auto-approve true "run tests and fix failures"
```

Для долгих задач ставьте таймаут:

```
cline --timeout 600 "run full test suite and fix all failures"
```

**Схема JSON-вывода:** При использовании `--json` каждая строка — отдельный JSON-объект:

```
{"type":"say","text":"I'll create the file now.","ts":1760501486669,"say":"text"}
```

| Поле | Тип | Описание |
|------|-----|----------|
| type | "ask" или "say" | Категория сообщения |
| text | string | Содержимое |
| ts | number | Unix timestamp в миллисекундах |
| say | string | Подтип для "say" |
| reasoning | string | Опциональное рассуждение модели |

**Интеграция с GitHub Actions:** Cline CLI — обычный npm-пакет, который работает в любом CI. Вот как сделать бота, который отвечает на упоминания @cline в issues:

```yaml
name: Cline Issue Assistant
on:
  issue_comment:
    types: [created, edited]

jobs:
  respond:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
      - run: npm install -g cline
      - run: cline auth --provider openrouter --apikey "${{ secrets.OPENROUTER_API_KEY }}"
      - run: |
          RESULT=$(cline --auto-approve true --json "Analyze this issue: $ISSUE_URL" | \
            jq -r 'select(.type == "agent_event" and .event.type == "done") | .event.text')
          # post $RESULT as issue comment
```

Полный пример с обнаружением @cline и публикацией ответа — в документации Cline.

**Коротко о главном для CI:**

- **Всегда** `--auto-approve true` — иначе агент зависнет в ожидании человека, которого нет.
- **Всегда** `CLINE_COMMAND_PERMISSIONS` — ограничьте, что агенту можно делать.
- **Всегда** `--json` + `jq` для парсинга. Текстовый вывод нестабилен.
- **Всегда** `--timeout` для задач с непредсказуемым временем.
- API-ключи — только в secrets CI, передавайте через `cline auth --provider ... --apikey ...`.

**OpenSpec в CI/CD:** Для CI используйте флаг `--force` (пропускает подтверждения): `openspec init --tools cline --profile core --force`. Команда `openspec update` пересоздаёт артефакты без интерактива. Для программного доступа — `--json`: `openspec list --json`, `openspec status --change <name> --json`.

---

## 7.2. Scheduled Agents — Запуск по расписанию

Многие задачи разработки повторяются с пугающей предсказуемостью: ежедневный дайджест PR, еженедельная проверка зависимостей, ежемесячный отчёт о здоровье кодовой базы. Без автоматизации они либо делаются вручную (тратя ваше время), либо не делаются вообще (накапливая технический долг).

**Scheduled Agents** решают это раз и навсегда: настраиваете расписание один раз — агент запускается сам, результат можно отправить прямо в Telegram или Slack.

Создать расписание можно двумя способами. Полная форма с флагами:

```
cline schedule create "PR summary" \
  --cron "0 9 * * MON-FRI" \
  --prompt "List all open PRs and their review status" \
  --workspace /path/to/repo \
  --model anthropic/claude-sonnet-4-6
```

Или интерактивный wizard:

```
cline schedule
```

Wizard проведёт вас через всё: создание, просмотр, историю выполнений, статистику.

**Управление расписаниями:**

| Команда | Назначение |
|---------|------------|
| `cline schedule list` | Показать все расписания |
| `cline schedule trigger <schedule-id>` | Запустить прямо сейчас |
| `cline schedule pause <schedule-id>` | Приостановить |
| `cline schedule resume <schedule-id>` | Возобновить |
| `cline schedule delete <schedule-id>` | Удалить навсегда |
| `cline schedule executions <schedule-id>` | История запусков |

**Шпаргалка по cron:**

| Выражение | Когда сработает |
|-----------|-----------------|
| `*/5 * * * *` | Каждые 5 минут |
| `0 * * * *` | Каждый час |
| `0 9 * * *` | Ежедневно в 9:00 |
| `0 9 * * 1-5` | По будням в 9:00 |
| `0 9 * * 1` | Каждый понедельник в 9:00 |
| `0 0 1 * *` | Первого числа каждого месяца |

**Жизненные примеры:**

```
# Подготовка к ежедневному стендапу
cline schedule create "Standup prep" \
  --cron "0 8 * * MON-FRI" \
  --prompt "Summarize: (1) PRs merged yesterday, (2) PRs in review, (3) open issues assigned to team"

# Еженедельная проверка зависимостей
cline schedule create "Dependency check" \
  --cron "0 10 * * MON" \
  --prompt "Check for outdated npm dependencies. For any with security vulnerabilities, create a branch with the update and open a PR."

# Еженедельный отчёт о состоянии кодовой базы
cline schedule create "Code health" \
  --cron "0 6 * * MON" \
  --prompt "Analyze the codebase for: (1) files with no test coverage, (2) TODO/FIXME comments older than 30 days, (3) functions longer than 100 lines."
```

**Комбинирование с Chat Connectors — магия:** Результаты scheduled agents могут автоматически приходить в ваш Telegram или Slack. Сначала подключаете мессенджер, потом создаёте расписание:

```
cline connect telegram -k $BOT_TOKEN

cline schedule create "Morning briefing" \
  --cron "0 8 * * *" \
  --prompt "Summarize overnight activity in the repo"
```

Всё. Каждое утро в 8:00 — дайджест в телефоне.

**Как это устроено технически:**

- Расписания живут в hub (`cline hub`), который запускается автоматически при создании первого расписания.
- Они сохраняются между перезапусками.
- История выполнения включает: статус, длительность, токены, стоимость.
- Статистика: процент успехов, среднее время, последняя ошибка.

**Советы с опытом:**

- Начните с одного — ежедневного дайджеста PR. Самый очевидный и полезный кандидат.
- Всегда указывайте `--workspace` явно. Агент должен знать, где его репозиторий.
- Контролируйте стоимость через `--model`: для рутинных задач достаточно лёгкой модели.
- Комбинируйте с connectors — пусть результаты приходят туда, где вы уже сидите.
- Раз в неделю проверяйте историю. Задача регулярно падает? Уточните промпт.

---

## 7.3. Multi-Agent Teams — Координация агентов

Сложные задачи — новая фича с тестами, рефакторинг модуля, настройка CI/CD — часто слишком велики для одной сессии. Контекстное окно забивается, агент теряет нить, качество падает.

**Multi-Agent Teams** решают это через декомпозицию. Один агент-координатор понимает общую картину, разбивает работу на подзадачи и делегирует их специалистам. Каждый специалист работает в своём изолированном контексте, фокусируясь на одной конкретной задаче. Координатор собирает результаты и управляет зависимостями.

Запускается это так:

```
# Новая команда
cline --team-name auth-sprint "Plan and implement user authentication with tests"

# Возобновить работу команды
cline --team-name auth-sprint "Continue with incomplete tasks"

# Из интерактивного режима
/team Plan and implement a REST API with tests

# Отключить команды для конкретного запуска
cline --no-teams "your prompt"
```

**Как это выглядит на практике:**

Координатор получает задачу:
"Implement user authentication with OAuth, JWT tokens, and tests"

Координатор декомпозирует:
├── **Специалист 1:** "Implement OAuth integration with corporate IdP"
├── **Специалист 2:** "Create JWT token service with refresh logic"
├── **Специалист 3:** "Write integration tests for auth endpoints"
└── **Специалист 4:** "Update API documentation for auth routes"

Каждый работает независимо и параллельно. Координатор собирает результаты и следит, чтобы никто никого не заблокировал. У координатора для этого есть дополнительные инструменты; специалисты работают с тем же набором, что и обычный агент, но в своём песочнице.

**Состояние команды живёт между сессиями** в `~/.cline/data/teams/[team-name]/`:

| Компонент | Что хранит |
|-----------|------------|
| Task board | Текущие задачи и их статус |
| Inter-agent mailbox | Сообщения между агентами |
| Mission log | Лог всей активности |

Это значит, что вы можете прервать команду, уйти на обед, вернуться через три часа и продолжить с того же места.

**Sub-agents vs. Teams:** Для простых сценариев делегирования внутри одной сессии используйте sub-agents. Они легче, но не сохраняют состояние:

| | Sub-agents | Teams |
|------------|------------|-------|
| Сохраняют состояние | Нет | Да |
| Для чего | Параллельное исследование | Многодневные задачи |
| Настройка | Не нужна (встроено) | Флаг `--team-name` |
| Контекст | Общий с основным агентом | Изолированный |

**Сигналы, что пора развернуть Teams:**

| Нужны Teams | Достаточно одной сессии |
|-------------|------------------------|
| Задача на несколько часов | Решается за 30 минут |
| Чётко делится на подзадачи | Монолитная, неделимая |
| Нужна параллельная работа | Можно делать последовательно |
| Придётся прерываться и возвращаться | Одна непрерывная сессия |

**Практические советы:**

- Давайте командам осмысленные имена (auth-sprint, api-v2, refactor-payments) — они понадобятся для возобновления.
- Формулируйте задачу координатору конкретно: технологии, ограничения, критерии приёмки.
- Проверяйте mission log: `~/.cline/data/teams/[team-name]/`.
- Если между подзадачами есть жёсткие зависимости — присмотритесь к Kanban (раздел 7.4), dependency chains там управляются визуально.

---

## 7.4. Kanban — Параллельный запуск с изолированными worktrees

**Kanban** — это веб-доска, где каждая задача запускает агента в отдельном git worktree.

> **Статус:** Research preview. Возможности могут меняться.

Зачем это нужно? Параллельная работа агентов в одном репозитории — это боль: они редактируют одни и те же файлы, перетирают изменения друг друга, создают merge conflicts. Kanban решает проблему радикально — изоляцией. Каждая задача получает собственный **ephemeral git worktree** — полную, но лёгкую копию репозитория.

Результат: десять агентов могут работать над десятью разными задачами одновременно и никогда не столкнутся лбами.

**Запуск — одна команда:**

```
cd /path/to/your/repo
npx kanban
```

Откроется доска в браузере. Дальше:

1. **Create** → Создаёте карточку задачи (сами или через sidebar chat)
2. **Link** → ⌘+click для dependency chain
3. **Start** → Play — создаётся worktree, агент начинает работу
4. **Monitor** → Карточка показывает последнее действие агента
5. **Review** → Diff с checkpoint system + inline comments
6. **Ship** → Commit или Open PR
7. **Clean up** → Trash — worktree удаляется, resume ID сохраняется

**Как работает изоляция:** Когда вы нажимаете Play, Kanban создаёт ephemeral git worktree. `node_modules` и другие gitignored файлы не копируются — они симлинкуются из основного репозитория. Агент работает в своём терминале, карточка показывает его последние сообщения. И так — для каждой задачи параллельно. Никаких merge conflicts между агентами.

**Dependency chains:** Связывайте карточки через ⌘+click (Mac) / Ctrl+click (Windows/Linux). Когда одна задача завершается и попадает в trash, следующая стартует автоматически. В комбинации с auto-commit это полностью автономный конвейер: результат одной задачи становится входом для другой.

**Inline review:** Дифф вьюер показывает изменения не только суммарно, но и в разрезе диапазонов сообщений. Кликните на любую строку — оставьте комментарий. Комментарий уходит агенту как обратная связь:

```
"Use a different approach here — avoid mutation"
"This edge case isn't handled: empty array"
"Extract this into a separate function"
```

**Отправка изменений:**

| Действие | Что происходит |
|----------|----------------|
| Commit | Агент мержит changes из worktree в коммит на base branch |
| Open PR | Создаёт ветку и открывает Pull Request |

В обоих случаях merge conflicts обрабатываются интеллектуально.

**Когда Kanban — ваш инструмент, а когда — избыточен:**

| Используйте Kanban | Не нужен |
|--------------------|----------|
| 3+ задач одновременно | Всего одна задача |
| Чёткие dependency chains | Монолитная реализация без разбиения |
| Нужен визуальный контроль | Headless / CI подход |
| Хотите ревью с комментариями | Полностью автономный режим |

**На практике:**

- Sidebar chat — отличный способ декомпозиции: попросите агента разбить задачу на карточки и запустить их.
- Dependency chains + auto-commit = настраиваете один раз, и конвейер работает сам.
- Inline comments — самый быстрый способ направить агента: он видит обратную связь прямо в контексте изменения.
- Всегда отправляйте карточки в trash после завершения — worktrees занимают место.
- Хотите вернуться к задаче? Используйте resume ID — он сохраняется при удалении.

---

## 7.5. Chat Connectors — Агент в мессенджерах

Вы уже живёте в мессенджерах. Переключаться в IDE ради каждого вопроса агенту — лишнее трение.

**Chat Connectors** убирают это переключение: пишете агенту в Telegram или Slack, получаете ответ там же, продолжаете диалог в том же треде. Агент работает на вашей машине, но вы управляете им из телефона.

**Где это особенно полезно:**

- Быстрый вопрос о кодовой базе без открытия IDE
- Мониторинг результатов scheduled agents
- Делегирование задач с мобильного
- Командный доступ к агенту через корпоративный Slack

**Что подключается:**

| Платформа | Команда | Что нужно |
|-----------|---------|-----------|
| Telegram | `cline connect telegram` | Bot token |
| Slack | `cline connect slack` | Bot token + signing secret + base URL |
| Discord | `cline connect discord` | App ID + bot token + public key + base URL |
| Google Chat | `cline connect gchat` | Service account JSON + base URL |
| WhatsApp | `cline connect whatsapp` | Phone ID + access token + app secret + verify token + base URL |
| Linear | `cline connect linear` | API key + webhook signing secret + base URL |

**Быстрый старт (один шаг):**

```
cline connect
```

Wizard проведёт через выбор платформы, ввод credentials, настройку безопасности, модели и системного промпта.

**Telegram — самый простой путь:**

```
# 1. Создаёте бота через @BotFather в Telegram
# 2. Запускаете коннектор
cline connect telegram -k $BOT_TOKEN
# 3. Пишете боту — агент отвечает
```

Всё. Один токен — и агент у вас в кармане.

**Slack — для команд:**

```
cline connect slack \
  --token $SLACK_BOT_TOKEN \
  --signing-secret $SLACK_SIGNING_SECRET \
  --base-url https://your-server.example.com
```

Каждый Slack-тред маппируется на отдельную сессию — агент помнит контекст внутри треда.

**Контроль доступа — критически важно:**

По умолчанию любой, кто нашёл вашего бота, может слать ему сообщения и выполнять задачи на вашей машине. В production так нельзя. Используйте `--hook-command`:

```
# Telegram: разрешить только одному пользователю
cline connect telegram -k $BOT_TOKEN \
  --hook-command 'jq -r ".payload.actor.participantKey" | grep -q "telegram:id:12345" && echo "{\"action\":\"allow\"}" || echo "{\"action\":\"deny\",\"message\":\"unauthorized\"}"'
```

**Как работает hook command:**

Скрипт получает JSON через stdin:

```json
{
  "payload": {
    "actor": {
      "participantKey": "telegram:id:12345",
      "displayName": "User Name"
    },
    "message": "The incoming message text"
  }
}
```

Возвращает `{"action": "allow"}` или `{"action": "deny", "message": "reason"}`. Без `--hook-command` доступ открыт всем.

**Несколько коннекторов — легко:**

```
# Terminal 1
cline connect telegram -k $TELEGRAM_TOKEN

# Terminal 2
cline connect slack --token $SLACK_TOKEN --signing-secret $SECRET --base-url $URL
```

Все коннекторы работают через один hub. Остановить можно все сразу или по одному:

```
cline connect --stop              # Все
cline connect telegram --stop     # Только Telegram
```

**Самый мощный паттерн — Scheduled Agents + Connectors:**

```
cline connect telegram -k $BOT_TOKEN

cline schedule create "Daily standup" \
  --cron "0 8 * * MON-FRI" \
  --prompt "Summarize: PRs merged yesterday, PRs in review, open blockers"
```

Агент запускается по расписанию и присылает результат в Telegram. Вы просто читаете.

**Что важно помнить:**

- Всегда настраивайте `--hook-command` до того, как поделитесь ботом с командой.
- Для командного Slack-бота используйте отдельную машину, а не свой ноутбук.
- Telegram — простейший старт: один токен, никаких публичных URL.
- Slack и Discord требуют публичный URL для webhook — используйте ngrok для локального тестирования.
- Контролируйте стоимость через `--model`: для чат-запросов хватит лёгкой модели.
- Если коннектор не стартует — проверьте `cline hub start`.

---

## Практические выводы

- **Headless режим — ваш главный инструмент CI/CD.** `git diff | cline "review"`, `--json` для парсинга, `--auto-approve true` для безостановочных пайплайнов. Всегда ограничивайте команды через `CLINE_COMMAND_PERMISSIONS` и ставьте `--timeout`.
- **Scheduled Agents превращают рутину в фоновый процесс.** Ежедневный дайджест PR, еженедельная проверка зависимостей, ежемесячный аудит кода — настройте один раз и получайте результаты в Telegram или Slack. Начните с одного расписания и уточняйте промпты по мере использования.
- **Multi-Agent Teams — для задач, которые не влезают в одну сессию.** Если задача длится часами, чётко делится на подзадачи и требует параллельной работы — создавайте команду. Координатор возьмёт декомпозицию на себя, а специалисты поработают в изолированных контекстах.
- **Kanban — когда агентов больше одного.** Изолированные git worktree решают проблему merge conflicts между параллельными задачами. Dependency chains + auto-commit строят полностью автономные конвейеры. Используйте для 3+ одновременных задач с визуальным контролем.
- **Chat Connectors убирают трение переключения контекста.** Telegram — простейший старт (один токен). Всегда настраивайте `--hook-command` для контроля доступа. Комбинируйте с расписаниями — и агент будет сам присылать отчёты туда, где вы уже находитесь.
- **Общий принцип: автоматизация начинается с одной задачи.** Не пытайтесь автоматизировать всё сразу. Выберите одну повторяющуюся боль, настройте её — и только потом переходите к следующей.

