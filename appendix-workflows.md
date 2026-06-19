# Приложение В: Руководство по созданию воркфлоу для Cline

> **Назначение:** Полное руководство по проектированию, написанию и поддержке воркфлоу для Cline (`.cline/workflows/`). Помогает понять, как воркфлоу работают технически, когда они нужны, как их структурировать и избежать типичных ошибок.

---

## Оглавление

1. [Главный вопрос](#1-главный-вопрос)
2. [Как воркфлоу работают технически](#2-как-воркфлоу-работают-технически)
3. [Структура хранения и приоритеты](#3-структура-хранения-и-приоритеты)
4. [Анатомия файла воркфлоу](#4-анатомия-файла-воркфлоу)
5. [Правила именования](#5-правила-именования)
6. [Карта категорий воркфлоу](#6-карта-категорий-воркфлоу)
7. [Дерево решений для нового воркфлоу](#7-дерево-решений-для-нового-воркфлоу)
8. [Правила написания содержимого](#8-правила-написания-содержимого)
9. [Полные примеры](#9-полные-примеры)
10. [Диагностика проблем](#10-диагностика-проблем)
11. [Аудит существующих воркфлоу](#11-аудит-существующих-воркфлоу)
12. [Чеклист перед сохранением воркфлоу](#12-чеклист-перед-сохранением-воркфлоу)

---

## 1. Главный вопрос

Перед созданием воркфлоу задай себе один вопрос:

**«Есть ли у меня повторяющаяся многошаговая задача, которую я запускаю явной командой?»**

Если да — это кандидат на воркфлоу. Если задача разовая или не имеет чёткой последовательности шагов — воркфлоу не нужен.

---

## 2. Как воркфлоу работают технически

Понимание механики критично для написания эффективных воркфлоу.

### Механизм активации

Когда ты пишешь `/release.md` в чате, происходит следующее:

1. Cline находит файл `release.md` среди включённых воркфлоу
2. Читает содержимое файла целиком
3. Оборачивает его в XML-тег и **вставляет в начало твоего сообщения**:

```
<explicit_instructions type="release.md">
{содержимое файла release.md}
</explicit_instructions>
{твоё сообщение без /release.md}
```

4. Слэш-команда удаляется из текста сообщения

**Это означает:**

- **Воркфлоу — это явные инструкции**, которые Cline получает как часть твоего сообщения, а не как фоновый контекст
- **Содержимое читается целиком** при каждом вызове — нет прогрессивной загрузки
- **Один воркфлоу за раз** — только первая слэш-команда в сообщении обрабатывается
- **Слэш-команда может быть в любом месте сообщения**, не только в начале: `/release.md сделай релиз 3.5.0` работает так же как `сделай релиз 3.5.0 /release.md`

### Имя команды = имя файла (с расширением)

```
release.md              →  /release.md
pr-review.md            →  /pr-review.md
hotfix-release.md       →  /hotfix-release.md
address-pr-comments.md  →  /address-pr-comments.md
```

Расширение `.md` является частью команды. Не пиши `/release` — пиши `/release.md`.

### Встроенные команды имеют приоритет

Если имя файла совпадает со встроенной командой — встроенная команда выполнится, воркфлоу проигнорируется.

Зарезервированные имена: `newtask`, `smol`, `compact`, `newrule`, `reportbug`, `deep-planning`, `explain-changes`.

---

## 3. Структура хранения и приоритеты

### Пути хранения

```
Проектные воркфлоу:
  .cline/workflows/

Глобальные воркфлоу:
  ~/Documents/Cline/Workflows/   (macOS / Windows)
  ~/.cline/workflows/            (Linux / через CLINE_DIR)
```

### Приоритет при совпадении имён

```
Локальный (.cline/workflows/)
  > Глобальный
    > Удалённый (Enterprise)
```

> **Примечание:** Cline также поддерживает legacy-путь `.clinerules/workflows/`. Если у тебя есть файлы в этой директории — они продолжат работать, но рекомендуется перенести их в `.cline/workflows/`.

### Глобальные vs проектные воркфлоу

> **Это нужно во ВСЕХ моих проектах?**
> ├── Да → глобальный воркфлоу (`~/.cline/workflows/`)
> │         Примеры: ревью PR, анализ веток, поиск ревьюеров
> └── Нет → проектный воркфлоу (`.cline/workflows/`)
>           Примеры: релиз этого проекта, деплой на конкретный сервер

Проектные воркфлоу коммитятся в репозиторий и шарятся с командой. Глобальные — только на твоей машине.

### Управление через UI

Воркфлоу можно включать и отключать через панель Rules в интерфейсе Cline. Отключённый воркфлоу не появляется в автодополнении и не выполняется при вводе команды.

---

## 4. Анатомия файла воркфлоу

Воркфлоу — это обычный markdown-файл. Никакого обязательного frontmatter нет.

### Минимальный воркфлоу

```markdown
# Название задачи

Краткое описание что делает этот воркфлоу.

## Шаги

1. Первый шаг с конкретной командой
2. Второй шаг
3. Финальный шаг
```

### Полная структура

```markdown
# Название воркфлоу

Одна строка: что делает этот воркфлоу и когда его использовать.

## Обзор

Краткий список того, что воркфлоу делает (опционально, для сложных процессов).

## Шаг 1: Название шага

Описание + конкретная команда:

```bash
команда --флаги
```

Что делать если команда завершилась с ошибкой.

## Шаг 2: Название шага

...

## Важные замечания

- Ограничения и предупреждения
- Что НЕ делать в этом воркфлоу
```

### Нет frontmatter — это нормально

Воркфлоу не требуют frontmatter. Файл без frontmatter работает корректно.

### XML-теги для структурирования сложных воркфлоу

Для длинных воркфлоу с несколькими режимами можно использовать XML-теги как логические секции:

```markdown
# PR Review

<detailed_steps>
## 1. Gather PR Information
...
</detailed_steps>

<example_review>
## Example: Reviewing PR #1234
...
</example_review>

<guidelines>
Keep comments short and friendly. Thank the author.
</guidelines>
```

Это помогает Cline понять структуру и находить нужную секцию без чтения всего файла.

---

## 5. Правила именования

Имя файла = имя слэш-команды. Выбирай имена которые легко набирать.

### Хорошо

```
release.md              →  /release.md
pr-review.md            →  /pr-review.md
hotfix-release.md       →  /hotfix-release.md
address-pr-comments.md  →  /address-pr-comments.md
find-pr-reviewers.md    →  /find-pr-reviewers.md
deploy-staging.md       →  /deploy-staging.md
db-migration.md         →  /db-migration.md
```

### Плохо

```
my workflow.md    →  пробелы не работают в слэш-командах
Release.md        →  регистр имеет значение, будь последователен
workflow1.md      →  непонятно что делает
misc.md           →  слишком общее
newtask.md        →  конфликт со встроенной командой
```

### Правила

- Используй `kebab-case` (дефисы вместо пробелов и подчёркиваний)
- Имя должно описывать действие, а не тему: `release.md` лучше чем `versioning.md`
- Короткие имена удобнее набирать: `release.md` лучше чем `create-new-release-version.md`
- Избегай конфликтов со встроенными командами

---

## 6. Карта категорий воркфлоу

### Git и релизы

#### `release.md`
**Когда нужен:** Есть стандартный процесс релиза с несколькими шагами (changelog, версия, тег, публикация).

Нужен если процесс релиза включает больше двух команд и требует конкретной последовательности.

#### `hotfix-release.md`
**Когда нужен:** Хотфиксы требуют cherry-pick конкретных коммитов и отличаются от обычного релиза.

#### `git-branch-analysis.md`
**Когда нужен:** Регулярно нужен анализ изменений в ветке перед PR или ревью.

### Code Review

#### `pr-review.md`
**Когда нужен:** Есть стандарт ревью PR с конкретными шагами (получить diff, проанализировать, оставить комментарий).

#### `address-pr-comments.md`
**Когда нужен:** Процесс обработки комментариев к PR требует конкретной последовательности (получить комментарии, применить, ответить).

#### `find-pr-reviewers.md`
**Когда нужен:** Нужно находить подходящих ревьюеров на основе git-истории и экспертизы.

### Деплой и инфраструктура

#### `deploy-staging.md`
**Когда нужен:** Деплой на staging требует конкретных шагов (проверки, команды, верификация).

#### `deploy-production.md`
**Когда нужен:** Деплой в production отличается от staging и требует дополнительных проверок и подтверждений.

#### `db-migration.md`
**Когда нужен:** Миграции БД требуют конкретной последовательности (бэкап, применить, верифицировать).

### Документация и контент

#### `writing-documentation.md`
**Когда нужен:** Есть стандарт написания документации (стиль, структура, компоненты).

#### `update-changelog.md`
**Когда нужен:** Changelog обновляется по конкретному формату и требует анализа коммитов.

### Проектно-специфичные

#### `onboarding.md`
**Когда нужен:** Новые разработчики регулярно проходят одни и те же шаги настройки окружения.

#### `code-review-checklist.md`
**Когда нужен:** Есть внутренний чеклист ревью, который нужно применять последовательно.

---

## 7. Дерево решений для нового воркфлоу

```
Есть идея для воркфлоу
│
├── Это повторяющаяся задача с конкретными шагами?
│   └── Нет → Не создавай воркфлоу
│
├── Пользователь явно запускает это командой?
│   └── Нет → Рассмотри скилл (автоактивация) или правило (всегда активно)
│
├── Это применимо ко всем проектам?
│   ├── Да → Глобальный воркфлоу (~/.cline/workflows/)
│   └── Нет → Проектный воркфлоу (.cline/workflows/)
│
├── Шаги требуют подтверждения пользователя в середине?
│   ├── Да → Явно укажи точки остановки в воркфлоу
│   └── Нет → Воркфлоу может выполняться линейно
│
├── Воркфлоу охватывает одну задачу или несколько?
│   ├── Одну → Один файл
│   └── Несколько → Разбить на отдельные файлы
│
└── Шаги конкретные (команды, пути, форматы)?
    ├── Да → Создавай
    └── Нет → Уточни шаги перед созданием
```

---

## 8. Правила написания содержимого

### 1. Конкретные команды, не абстрактные описания

**Плохо** — Cline не знает что именно делать:

```markdown
## Шаг 1: Обновить версию
Обнови версию в package.json и changelog.
```

**Хорошо** — конкретные команды и форматы:

```markdown
## Шаг 1: Обновить версию

```bash
cat package.json | grep '"version"'
```

Обнови `package.json` → поле `"version"` до целевой версии.
Обнови `CHANGELOG.md` → добавь секцию `## [X.Y.Z]` в начало файла.
```

### 2. Явные точки остановки для подтверждения

Воркфлоу часто включают необратимые действия. Явно указывай где нужно подтверждение:

```markdown
## Шаг 3: Создать тег и запушить

**Подожди подтверждения пользователя перед этим шагом.**

```bash
git tag v{VERSION}
git push origin v{VERSION}
```
```

Или через вопрос:

```markdown
Покажи пользователю список коммитов и спроси какие включить в релиз.
Продолжай только после получения ответа.
```

### 3. Обработка ошибок для критичных шагов

```markdown
## Шаг 2: Запустить тесты

```bash
npm run test
```

Если тесты упали — остановись и сообщи пользователю. Не продолжай деплой.
```

### 4. Финальный отчёт

Хороший воркфлоу заканчивается итоговым сообщением:

```markdown
## Финальный отчёт

Предоставь:
- Версию/тег релиза
- Ссылку на страницу релиза
- Краткое описание ключевых изменений
```

### 5. Ссылки на файлы проекта вместо описаний

**Плохо:**

```markdown
Обнови файл с версией проекта и файл с историей изменений.
```

**Хорошо:**

```markdown
Обнови `package.json` (поле `version`) и `CHANGELOG.md`.
```

### 6. Размер воркфлоу

Воркфлоу читается целиком при каждом вызове.

**Целевой размер:** 30–100 строк. Если воркфлоу длиннее — проверь нет ли лишних объяснений или примеров которые можно убрать.

Исключение: воркфлоу с примерами команд могут быть длиннее, если примеры реально нужны Cline для правильного выполнения.

---

## 9. Полные примеры

### Пример 1: Воркфлоу релиза

**Файл:** `.cline/workflows/release.md`

```markdown
# Release

Prepare and publish a release directly from `main`.

## Overview

1. Select/confirm the target version
2. Curate CHANGELOG.md entries for end users
3. Ensure package.json version matches the changelog
4. Create and push a release commit + tag
5. Trigger publish workflow
6. Update GitHub release notes and share a summary

## Step 1: Sync and determine version

```bash
git checkout main
git pull origin main
cat package.json | grep '"version"'
```

Confirm the release version with the maintainer (patch/minor/major).

## Step 2: Curate changelog and version

- Edit `CHANGELOG.md` for the target version using human-friendly release notes.
- Ensure version headers use bracket format: `## [3.66.1]`.
- Update `package.json` version to the same value.

## Step 3: Commit and tag

```bash
git add CHANGELOG.md package.json package-lock.json
git commit -m "v<version> Release Notes"
git push origin main
git tag v<version>
git push origin v<version>
```

## Step 4: Trigger publish workflow

Tell the maintainer to run:
https://github.com/org/repo/actions/workflows/publish-stable.yml

Use `v<version>` as the release tag.

## Step 5: Update GitHub release notes

```bash
gh release edit v<version> --notes "<final curated release notes>"
```

## Step 6: Final summary

Provide:
- Released version/tag
- Link to release page
- Summary of top end-user changes
```

---

### Пример 2: Воркфлоу с точками остановки

**Файл:** `.cline/workflows/deploy-production.md`

```markdown
# Deploy to Production

Deploy the current branch to production. Requires explicit confirmation at each critical step.

## Step 1: Pre-deploy checks

```bash
npx tsc --noEmit
npm run test
npm run build
```

Stop if any check fails. Report the failure to the user.

## Step 2: Confirm deployment

**Ask the user to confirm before proceeding.**

Show the current branch, last commit hash and message, and ask:
"Ready to deploy to production. Confirm?"

Wait for explicit confirmation. Do not proceed without it.

## Step 3: Deploy

```bash
vercel --prod
```

## Step 4: Verify

```bash
curl -I https://your-app.com/api/health
```

Expected: HTTP 200. If not — report to user immediately.

## Step 5: Summary

Report:
- Deployment URL
- Commit hash deployed
- Time of deployment
```

---

### Пример 3: Аналитический воркфлоу без деплоя

**Файл:** `~/.cline/workflows/find-pr-reviewers.md`

```markdown
# Find Best Reviewers for Current Branch

Analyze the current branch to find the best reviewers based on domain expertise and git history.

## Steps

1. Get the current branch and verify it's not `main`
2. Get changed files: `git diff origin/main...HEAD --name-only`
3. Understand the nature of changes: `git diff origin/main...HEAD`
4. Identify the domain/feature area being changed
5. Find all files related to that domain (not just changed files)
6. Get contributors for related files:
   `git log --format="%an <%ae>" -- <related-files> | sort | uniq -c | sort -rn`
7. Exclude yourself: `git config user.email`
8. Score by: domain expertise (highest) > direct file expertise > line ownership

## Output Format

Ordered list of top 5 reviewers:

1. **Name** - Domain expert: 15 commits to related files, authored core logic
2. **Name** - 8 commits to affected files, recently added the feature being modified

Do NOT ask questions — analyze and output the list directly.
```

---

### Пример 4: Минимальный воркфлоу

**Файл:** `.cline/workflows/address-pr-comments.md`

```markdown
# Address PR Comments

Review and address all comments on the current branch's PR.

## Steps

1. Get the current branch name and find the associated PR:
   ```bash
   gh pr view --json number,title,body
   ```

2. Understand the PR context:
   - Get the full diff: `git diff origin/main...HEAD`
   - Read the changed files to understand what the PR is doing

3. Fetch all PR comments:
   - Inline comments: `gh api repos/{owner}/{repo}/pulls/{pr_number}/comments`
   - General comments: `gh pr view {pr_number} --json comments,reviews`

4. Present a summary of all comments with your recommendation for each
   (apply, skip, or respond). Ignore bot noise.

5. **Wait for approval before proceeding.**

6. After approval:
   - Apply code changes and commit
   - Reply to comments that were addressed or intentionally skipped
   - Push commits
```

---

## 10. Диагностика проблем

### Воркфлоу не активируется при вводе команды

- Убедись что воркфлоу включён в панели Rules (toggle)
- Проверь точное имя команды: файл `release.md` → команда `/release.md` (с расширением `.md`)
- Убедись что слэш-команда окружена пробелами или находится в начале/конце сообщения: `/release.md` работает, `do/release.md` — нет
- Проверь нет ли конфликта со встроенной командой

### Воркфлоу активируется, но Cline игнорирует шаги

- Содержимое воркфлоу вставляется как `<explicit_instructions>` — это явные инструкции, Cline должен им следовать
- Если шаги слишком абстрактны, Cline может интерпретировать их по-своему — сделай команды конкретнее
- Проверь что файл не пустой и сохранён в UTF-8

### Воркфлоу выполняется не полностью

- Добавь явные точки остановки с подтверждением пользователя
- Разбей длинный воркфлоу на несколько файлов с более узкой областью

### Конфликт имён между локальным и глобальным воркфлоу

Локальный (`.cline/workflows/`) всегда побеждает глобальный. Если глобальный воркфлоу перестал работать — проверь нет ли файла с таким же именем в `.cline/workflows/` проекта.

---

## 11. Аудит существующих воркфлоу

Воркфлоу нужно периодически пересматривать. Устаревший воркфлоу с неправильными командами хуже отсутствующего.

**Вопросы для аудита каждого файла:**

- Команды в воркфлоу всё ещё актуальны? (инструменты не изменились?)
- Пути к файлам в воркфлоу соответствуют текущей структуре проекта?
- Воркфлоу дублирует другой воркфлоу или скилл?
- Воркфлоу используется? (если нет — возможно, его стоит удалить)
- Файл длиннее 100 строк? (возможно, нужно разбить или сократить)
- Есть ли необратимые шаги без явного подтверждения пользователя?

---

## 12. Чеклист перед сохранением воркфлоу

- [ ] Имя файла в `kebab-case`, описывает действие
- [ ] Имя не конфликтует со встроенными командами (`newtask`, `smol`, `compact`, `newrule`, `reportbug`, `deep-planning`, `explain-changes`)
- [ ] Каждый шаг содержит конкретные команды или действия, не абстрактные описания
- [ ] Необратимые шаги (push, deploy, tag) требуют явного подтверждения
- [ ] Есть обработка ошибок для критичных шагов
- [ ] Воркфлоу заканчивается финальным отчётом
- [ ] Файл не длиннее 100 строк (или длина оправдана)
- [ ] Нет дублирования с другими воркфлоу или скиллами
- [ ] Проектный воркфлоу — в `.cline/workflows/`, глобальный — в `~/.cline/workflows/`
- [ ] Воркфлоу протестирован: команда `/имя-файла.md` активирует его корректно
```
