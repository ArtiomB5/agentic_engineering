# Приложение 1: Руководство по созданию скиллов для Cline

> **Назначение:** Полное руководство по проектированию, написанию и публикации скиллов для Cline. Помогает понять, как работают скиллы технически, как структурировать SKILL.md и избежать типичных ошибок.

---

## Оглавление

1. [Как скиллы работают технически](#1-как-скиллы-работают-технически)
2. [Структура скилла](#2-структура-скилла)
3. [Анатомия SKILL.md](#3-анатомия-skillmd)
4. [Правило 1: Имя скилла — kebab-case, конкретное](#4-правило-1-имя-скилла--kebab-case-конкретное)
5. [Правило 2: Description — это триггер, а не заголовок](#5-правило-2-description--это-триггер-а-не-заголовок)
6. [Правило 3: Важное — в начало SKILL.md](#6-правило-3-важное--в-начало-skillmd)
7. [Правило 4: Конкретные команды, не абстрактные описания](#7-правило-4-конкретные-команды-не-абстрактные-описания)
8. [Правило 5: Держи SKILL.md под 5k токенов](#8-правило-5-держи-skillmd-под-5k-токенов)
9. [Правило 6: Скрипты — для детерминированных операций](#9-правило-6-скрипты--для-детерминированных-операций)
10. [Правило 7: Decision trees для сложных скиллов](#10-правило-7-decision-trees-для-сложных-скиллов)
11. [Правило 8: Глобальный vs проектный скилл](#11-правило-8-глобальный-vs-проектный-скилл)
12. [Полный пример: скилл для Next.js деплоя](#12-полный-пример-скилл-для-nextjs-деплоя)
13. [Чеклист перед публикацией скилла](#13-чеклист-перед-публикацией-скилла)

---

## 1. Как скиллы работают технически

Прежде чем писать скилл — понять механику. Скиллы используют прогрессивную загрузку:

| Уровень | Когда загружается | Стоимость | Содержимое |
|---------|-------------------|-----------|------------|
| Метаданные | Всегда при старте | ~100 токенов на скилл | `name` + `description` из frontmatter |
| Инструкции | Когда скилл активирован | До 5k токенов | Тело SKILL.md |
| Ресурсы | По мере необходимости | Практически без ограничений | Файлы из `docs/`, `templates/`, `scripts/` |

Когда ты отправляешь сообщение, Cline видит в системном промпте список скиллов с их описаниями:

```
SKILLS

The following skills provide specialized instructions for specific tasks.
When a user's request matches a skill description, use the use_skill tool
to load and activate the skill.

Available skills:
  - "create-pull-request": Create a GitHub pull request following project conventions...
  - "nextjs-deploy": Deploy Next.js app to Vercel...
```

Если запрос совпадает с описанием — Cline вызывает `use_skill`, получает полный текст SKILL.md и следует инструкциям. После активации Cline получает сообщение: «IMPORTANT: The skill is now loaded. Do NOT call use_skill again for this task» — то есть скилл активируется один раз на задачу.

---

## 2. Структура скилла

```
.cline/skills/
└── my-skill/                  # Имя директории = name в frontmatter
    ├── SKILL.md               # Обязательно: метаданные + инструкции
    ├── docs/                  # Опционально: детальная документация
    │   ├── setup.md
    │   └── troubleshooting.md
    ├── templates/             # Опционально: шаблоны файлов
    │   └── config.yaml
    └── scripts/               # Опционально: исполняемые скрипты
        └── validate.sh
```

Если `name` в frontmatter не совпадает с именем директории — скилл молча игнорируется.

---

## 3. Анатомия SKILL.md

```markdown
---
name: my-skill
description: Описание что делает скилл и когда использовать. Используй глаголы действия.
---

# My Skill

Инструкции для Cline...
```

### Обязательные поля frontmatter

| Поле | Описание |
|------|----------|
| `name` | Должен точно совпадать с именем директории (kebab-case) |
| `description` | Максимум 1024 символа, определяет когда скилл активируется |

### Опциональные поля frontmatter

| Поле | Описание |
|------|----------|
| `disabled: true` | Отключить скилл без удаления файла |

---

## 4. Правило 1: Имя скилла — kebab-case, конкретное

### Хорошо

- `aws-cdk-deploy`
- `pr-review-checklist`
- `database-migration`
- `nextjs-vercel-deploy`

### Плохо

- `aws` — слишком расплывчато
- `my_skill` — подчёркивания вместо дефисов
- `DeployToAWS` — PascalCase
- `misc-helpers` — слишком общее

---

## 5. Правило 2: Description — это триггер, а не заголовок

`description` — это то, что Cline читает чтобы решить: активировать скилл или нет. Это самое важное поле.

### Формула хорошего description

```
[Что делает скилл, глагол действия]. Use when [конкретные сценарии использования].
```

### Хорошо

```yaml
description: Deploy applications to AWS using CDK. Use when deploying, updating infrastructure, or managing AWS resources.

description: Create a GitHub pull request following project conventions. Use when the user asks to create a PR, submit changes for review, or open a pull request. Handles commit analysis, branch management, PR template usage, and PR creation using the gh CLI tool.

description: Generate release notes from git commits. Use when preparing releases, writing changelogs, or summarizing recent changes.
```

### Плохо

```yaml
description: Helps with AWS stuff.

description: Data analysis helper.

description: Useful for releases.
```

### Что включать в description

- Глаголы действия в начале («Deploy», «Generate», «Create», «Analyze»)
- Фразы-триггеры которые пользователь может написать («Use when deploying», «Use when the user asks to create a PR»)
- Конкретные инструменты, файлы, домены («using CDK», «using the gh CLI tool», «CSV, Excel, or JSON data files»)
- Перечисление подзадач если скилл охватывает несколько сценариев («Handles commit analysis, branch management, PR template usage»)

**Тест description:** попробуй разные формулировки запроса и проверь активируется ли скилл. Если нет — добавь эти формулировки в description.

---

## 6. Правило 3: Важное — в начало SKILL.md

Cline читает файл последовательно. Самые критичные инструкции должны быть первыми.

### Структура тела SKILL.md

```markdown
# Название скилла

Одна строка: что делает этот скилл (опционально, если понятно из названия).

## Critical Rules
(Если есть жёсткие ограничения — сюда, первыми)

- Никогда не делать X
- Всегда проверять Y перед Z

## Prerequisites Check
(Что проверить перед началом работы)

## Workflow / Steps
(Основные шаги, пронумерованные)

## Error Handling
(Типичные ошибки и как их решать)

## Summary Checklist
(Финальный чеклист — опционально)
```

---

## 7. Правило 4: Конкретные команды, не абстрактные описания

### Плохо

```markdown
## Steps
1. Проверь что gh CLI установлен
2. Убедись что ты авторизован
3. Создай PR
```

### Хорошо

```markdown
## Steps

### 1. Check if `gh` CLI is installed
```bash
gh --version
```

If not installed, inform the user:
- macOS: `brew install gh`
- Other: https://cli.github.com/

### 2. Check authentication
```bash
gh auth status
```

If not authenticated, guide the user to run `gh auth login`.
```

Реальные команды, ожидаемый вывод, что делать если что-то пошло не так — это то, что делает скилл полезным.

---

## 8. Правило 5: Держи SKILL.md под 5k токенов

Если скилл требует больше — выноси детали в `docs/` и ссылайся на них:

```markdown
## Advanced Configuration
For detailed setup options, see [setup.md](docs/setup.md).

## Troubleshooting
For edge cases and debugging, see [troubleshooting.md](docs/troubleshooting.md).
```

Cline читает файлы из `docs/` через `read_file` только когда инструкции на них ссылаются. Это позволяет хранить неограниченный объём документации без постоянного потребления токенов.

### Когда что использовать

| Поместить в | Когда |
|-------------|-------|
| SKILL.md | Основной процесс, критичные правила, частые случаи |
| `docs/` | Детальная документация, edge cases, платформо-специфичные инструкции |
| `templates/` | Конфиг-файлы, шаблоны кода, boilerplate |
| `scripts/` | Валидация, детерминированные операции, сложные вычисления |

---

## 9. Правило 6: Скрипты — для детерминированных операций

Скрипты токен-эффективны: в контекст попадает только их вывод, не код. 500-строчный скрипт валидации производит простое «Passed» или детальные ошибки без потребления токенов на сам код.

```markdown
## Validate Configuration
Run the validation script before proceeding:

```bash
python scripts/validate.py
```

Expected output: «Configuration valid» or a list of errors to fix.
```

---

## 10. Правило 7: Decision trees для сложных скиллов

Если скилл охватывает несколько сценариев — используй деревья решений. Это помогает Cline быстро найти нужный путь без чтения всего документа:

```markdown
## Which approach to use?

Deploy target?
+-- Vercel -> see ## Vercel Deploy
+-- AWS -> see ## AWS Deploy
+-- Docker -> see ## Docker Deploy
```

---

## 11. Правило 8: Глобальный vs проектный скилл

| | Проектный `.cline/skills/` | Глобальный `~/.cline/skills/` |
|---|---|---|
| Область | Один проект | Все проекты на машине |
| Шаринг | Коммитится в репо, шарится с командой | Только твоя машина |
| Приоритет | Ниже | Выше (перекрывает проектный при совпадении имён) |

**Проектный** — для специфики проекта (деплой на конкретный сервер, работа с конкретной БД).

**Глобальный** — для универсальных задач (создание PR, code review, анализ данных).

---

## 12. Полный пример: скилл для Next.js деплоя

### Структура

```
.cline/skills/
└── nextjs-vercel-deploy/
    ├── SKILL.md
    └── docs/
        └── env-vars.md
```

### `.cline/skills/nextjs-vercel-deploy/SKILL.md`

```markdown
---
name: nextjs-vercel-deploy
description: Deploy Next.js application to Vercel. Use when deploying to production, setting up Vercel project, managing environment variables, or troubleshooting Vercel build failures.
---

# Next.js Vercel Deploy

## Critical Rules

1. Always run tests before deploying: `npm run test`
2. Always run build check before deploying: `npm run build`
3. Never deploy with TypeScript errors: `npx tsc --noEmit`
4. Ask before pushing to production — always confirm with user

## Prerequisites Check

### 1. Verify Vercel CLI
```bash
vercel --version
```

If not installed: `npm i -g vercel`

### 2. Verify authentication
```bash
vercel whoami
```

If not authenticated: `vercel login`

### 3. Run pre-deploy checks
```bash
npx tsc --noEmit && npm run test && npm run build
```

Stop if any check fails. Do not proceed to deploy.

## Deploy Workflow

### 1. Deploy to preview
```bash
vercel
```

Share the preview URL with the user for review.

### 2. Deploy to production (only after user confirms)
```bash
vercel --prod
```

### 3. Verify deployment
```bash
curl -I https://your-app.vercel.app/api/health
```

Expected: HTTP 200

## Environment Variables

For managing env vars, see [env-vars.md](docs/env-vars.md).

## Error Handling

**Build fails:**
- Check TypeScript errors: `npx tsc --noEmit`
- Check for missing env vars: compare `.env.example` with Vercel dashboard

**Deploy hangs:**
- Check Vercel dashboard for build logs
- Common cause: infinite loop in `getStaticProps` or `generateStaticParams`

**404 after deploy:**
- Verify `vercel.json` rewrites configuration
- Check that `next.config.js` output mode matches deployment target

## Post-Deploy

- Report the production URL to the user
- Update `memory-bank/progress.md` with deploy date and version
- Remind about monitoring: Vercel Analytics dashboard
```

---

## 13. Чеклист перед публикацией скилла

- [ ] Имя директории точно совпадает с `name` в frontmatter
- [ ] `name` в kebab-case, конкретное и описательное
- [ ] `description` начинается с глагола действия
- [ ] `description` содержит фразы-триггеры («Use when...»)
- [ ] `description` упоминает конкретные инструменты/файлы/домены
- [ ] Критичные правила — в начале SKILL.md
- [ ] Все шаги содержат реальные команды, не абстрактные описания
- [ ] Есть секция Error Handling с типичными проблемами
- [ ] SKILL.md не превышает 5k токенов
- [ ] Детальная документация вынесена в `docs/`
- [ ] Скилл протестирован: разные формулировки запроса активируют его