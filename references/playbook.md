# Playbook: setup-agent-harness

Пошаговое руководство для агента при вызове `/setup-agent-harness`. Читать целиком перед фазой Analysis.

## Фаза 1: Discovery (1-2 минуты)

Собрать факты о проекте. Ничего не менять.

### 1.1 Стек

Пробежаться по манифестам:

| Файл | Значит |
|------|--------|
| `package.json` | Node.js/JS/TS |
| `pnpm-lock.yaml` / `yarn.lock` / `package-lock.json` | конкретный менеджер |
| `tsconfig.json` | TypeScript |
| `pyproject.toml` / `requirements.txt` / `setup.py` / `Pipfile` | Python |
| `Cargo.toml` | Rust |
| `go.mod` | Go |
| `build.gradle` / `build.gradle.kts` / `settings.gradle*` | Gradle (обычно JVM: Kotlin/Java/Scala) |
| `pom.xml` | Maven (Java/Kotlin) |
| `mix.exs` | Elixir |
| `Gemfile` | Ruby |
| `composer.json` | PHP |
| `pubspec.yaml` | Dart/Flutter |
| `Package.swift` | Swift |
| `*.csproj` / `*.sln` | .NET/C# |

Определить **primary stack**. Возможен monorepo — тогда фиксировать список. Полный набор правил — `stack-detectors.md`.

### 1.2 Профиль

Понять, что за приложение (эвристики):
- Spring/Express/FastAPI/Rails/etc. в зависимостях → backend service.
- React/Vue/Svelte/Next → frontend.
- CLI → одна entry point и стандартные CLI-либы (clap, cobra, click, commander).
- Библиотека → есть `main` в package.json/`lib.rs`/`__init__.py`, публикация настроена.
- Инфраструктура/скрипты → Terraform, Ansible, docker-compose как основное.

### 1.3 Git

```
git rev-parse --is-inside-work-tree
git branch --show-current
git remote -v
git log -1 --format='%an <%ae>' 2>/dev/null || true
git status --short
```

- Есть ли git-репо вообще?
- Какая ветка по умолчанию (`main`, `master`, `develop`, что-то другое)?
- Есть ли remote?
- Идентити для commit'ов (из последнего коммита или `git config user.*`) — понадобится в фазе 8.

### 1.4 Существующая инфраструктура агента

Проверить наличие:
- `CLAUDE.md` в корне.
- `.claude/skills/` и его содержимое.
- `docs/`, `docs/tasks/`.
- `.gitignore` (для фазы 5 — учесть при добавлении tracker-файлов).

### 1.5 Известные проблемы

Быстрое сканирование:
- `TODO:` / `FIXME:` / `XXX:` через `Grep` — сколько их, где.
- Явные заметки типа `README-DRAFT.md`, `NOTES.md`.

Держать список в уме для фазы 5 (seed-задачи).

## Фаза 2: Preflight (10 секунд)

Одной репликой пользователю: «Обнаружил `<stack>` проект на ветке `<branch>` (либо: репозиторий не инициализирован). План: анализ → docs → трекер → skills → CLAUDE.md → commit. Существующий CLAUDE.md/docs/skills [нашёл/не нашёл]. Остановить, если что-то не так — иначе поехали.»

Ждать 3-5 секунд. Если пользователь не остановил — продолжать без явного `y`. Если возразил — уточнить и адаптировать.

Особые случаи:
- Если репозитория нет вообще (`.git` отсутствует) — сообщить, спросить: инициализировать `git init` или пропустить всё, связанное с git?
- Если ветка не `main` — сообщить фактическое имя, дальше в CLAUDE.md подставить его как дефолт.

## Фаза 3: Analysis (для непустого проекта)

Пропустить, если проект действительно пустой (нет `src/`, `lib/`, `app/`, `internal/` — только манифесты). Заменить на короткий шаблон в фазе 4.

Иначе — 4 параллельных Explore-агента. Задания зависят от профиля, но общий каркас:

| # | Тема | Что спрашиваем у агента |
|---|------|------------------------|
| 1 | Модель данных | Основные сущности/типы/схемы, связи, персистентность, миграции, инварианты |
| 2 | API/интерфейс | HTTP endpoints, CLI команды, экспортируемые функции, форматы I/O |
| 3 | Бизнес-логика | Основные модули/сервисы, публичные функции, workflow, состояния |
| 4 | Инфраструктура и безопасность | Конфигурация, env-переменные, аутентификация, деплой, зависимости |

Инструкции агенту:
- Русский язык (или как выбрано в фазе 2).
- Формат: таблицы, ссылки `file:line`, минимум прозы.
- В конце — «известные проблемы и TODO».

Ждать все 4 отчёта. Держать их как источник для фаз 4 и 5.

## Фаза 4: Docs generation

### 4.1 Решить набор doc-файлов

Правила выбора — в `docs-structure.md`. Базовый набор для сервиса:

```
docs/
  README.md                # индекс
  01-overview.md           # обязательно всегда
  02-domain-model.md       # если есть модель данных
  03-state-machines.md     # если есть явные state machines
  04-services.md           # если есть выраженные бизнес-модули
  05-api.md                # если есть внешний API (REST/GraphQL/RPC/CLI)
  06-async-api.md          # если есть асинхронный интерфейс (WS/SSE/queues)
  07-data-contracts.md     # если есть DTO/schemas/proto — по-другому 07-dto-and-mappers.md
  08-security.md           # если есть auth/permissions
  09-error-handling.md     # если есть централизованная обработка ошибок
  10-configuration.md      # env, конфиги, деплой
  11-known-gaps.md         # обязательно (даже если пусто)
```

Для frontend/CLI/library профилей — адаптировать имена и убрать нерелевантные. См. `docs-structure.md`.

### 4.2 Написать файлы

Для каждого — использовать материал из фазы 3. Стиль:

- Плотные таблицы вместо прозы.
- `file:line` ссылки везде, где применимо.
- Минимум эмоций и оценочных суждений.
- Русский язык (или как выбрано).

Шаблон `docs/README.md` — `templates/docs-readme.template.md`. Наполняется списком созданных файлов.

### 4.3 Пустой проект — заглушки

Если фаза 3 пропущена, создать только:
- `docs/README.md`
- `docs/01-overview.md` — с секциями, но плейсхолдерами `_(заполнить когда появится код)_`.
- `docs/11-known-gaps.md` — с одной строкой «Проект в фазе инициализации».

## Фаза 5: Tracker setup

### 5.1 Создать структуру

```
docs/tasks/
  README.md                # шаблон templates/tasks-readme.template.md
  INDEX.md                 # шаблон templates/tasks-index.template.md
```

### 5.2 Seed-задачи

Пройтись по:
- пунктам из `docs/11-known-gaps.md` (максимум 3 наиболее приоритетных),
- заметным `TODO`/`FIXME` из фазы 1.5.

Создать до 3 seed-задач в формате `T-001-*.md` — `T-003-*.md`. Файлы:
- frontmatter обязателен (см. `tasks-readme.template.md` → шаблон таск-файла);
- `## Контекст` — 2-3 строки;
- `## Acceptance criteria` — хотя бы один пункт;
- `## План` — 2-4 шага;
- `## Лог` — первая запись `- YYYY-MM-DD: заведена автоматически при setup-agent-harness.`

Добавить строки в `INDEX.md`. Если seed'ов нет — оставить таблицу «Открытые» пустой с пометкой `_(пока пусто)_`.

## Фаза 6: Skills install

Скопировать 8 файлов из `references/skills/` в `.claude/skills/` целевого проекта:

**Таск-трекер:**
- `task-add.md`
- `task-done.md`
- `task-sync.md`
- `docs-sync.md`

**Само-улучшение агента:**
- `wheel-check.md` — проверка «не изобретаю ли велосипед» перед новой реализацией
- `mid-retro.md` — пауза для само-осмотра внутри задачи
- `self-review.md` — ревью своего diff'а после `task-done`
- `pre-flight.md` — 3 обязательных вопроса (assumptions/risks/reversibility) между дизайном и кодингом для non-trivial и high-stakes

Если файлы уже существуют — не перезаписывать без причины. Если у пользователя своя версия — оставить как есть, в отчёте упомянуть «skills уже есть, не трогал».

**Идемпотентный докат:** если из 8 файлов часть уже есть, а часть отсутствует — доложить недостающие. Это применимо к старым проектам, где harness был поставлен до появления skill'ов само-улучшения.

Если `.claude/settings.local.json` уже существует — не трогать вообще.

## Фаза 7: CLAUDE.md

### 7.1 Backup существующего

Если `CLAUDE.md` уже был:
```
mv CLAUDE.md CLAUDE.md.bak-YYYY-MM-DD-HHMMSS
```

### 7.2 Рендер шаблона

Взять `templates/claude-md.template.md`. Подставить:

- `{{PROJECT_NAME}}` — имя из манифеста или папки.
- `{{STACK}}` — короткое описание, например «Kotlin/Spring Boot 3 (JDK 21) + PostgreSQL».
- `{{PRIMARY_BRANCH}}` — обнаруженная default-ветка.
- `{{TEST_COMMAND}}` — команда тестов для стека (см. `stack-detectors.md` → «commands»).
- `{{BUILD_COMMAND}}` — команда сборки.
- `{{LANGUAGE}}` — язык общения агента (RU/EN).
- `{{DURATION_BASELINES}}` — таблица baseline'ов длительности стек-специфичных команд (см. `stack-detectors.md` → «durations»). Заполняется дефолтами по определённому стеку.

### 7.3 Дополнить контекстными правилами

Если анализ фазы 3 выявил специфику (например, обязательный запуск миграций, специфичная система CI, критические инварианты) — добавить компактным списком в раздел «Проектные особенности».

## Фаза 8: Bootstrap commit + push

### 8.1 Проверка

```
git status --short
```

Отсеять:
- `settings.local.json` в `.claude/` — не коммитить.
- Локальные env-файлы, `.DS_Store`, IDE-конфиги — не коммитить (если `.gitignore` их не покрывает, упомянуть в отчёте как отдельный TODO).

### 8.2 Stage

Только именованные пути. Пример:
```
git add CLAUDE.md \
  .claude/skills/*.md \
  docs/README.md docs/*.md \
  docs/tasks/README.md docs/tasks/INDEX.md docs/tasks/T-*.md
```

### 8.3 Safety scan (обязательно)

```
git diff --cached | grep -Ei '(password|secret|token|api[_-]?key|jwt|credential)\s*[:=]'
```

Ложноположительные (в шаблонах слова могут встречаться) — глазами отсеять. Реальные секреты — блокировать commit, показать строки пользователю.

### 8.4 Commit

Формат:
```
chore(harness): bootstrap agent harness — docs, tasks, skills, CLAUDE.md

- docs/*: <N> файлов документации (перечислить)
- docs/tasks/: файловый task-трекер (T-001..T-00X)
- .claude/skills/: task-add, task-done, task-sync, docs-sync
- CLAUDE.md: правила агента, Auto-mode, Git-автоматизация

Refs: setup-agent-harness

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

Использовать identity из последнего коммита репозитория:
```
git -c user.name="<name>" -c user.email="<email>" commit -m "$(cat <<'EOF'
...
EOF
)"
```

Если identity не определяется никак — попросить пользователя одной фразой.

### 8.5 Ждать пользователя перед push

**Не пушить автоматически.** Вывести пользователю фразу:

> Готово. Собрано: <N> файлов, коммит `<short_hash>`. Пушу в `origin <branch>`? (y/n)

Если `y` — `git push origin <branch>`, вывести хэш и завершить.
Если `n` — оставить коммит локальным, отчитаться какие файлы, где, как позже запушить.

## Финальный отчёт

Одним блоком:

- **Стек:** …
- **Создано файлов:** …
- **Skills установлены:** task-add, task-done, task-sync, docs-sync
- **Задачи-seed:** T-001, T-002, …
- **CLAUDE.md:** новый (backup: имя файла)
- **Git:** commit `<hash>`, push [выполнен / ждёт подтверждения / пропущен]

## Идемпотентный повторный вызов

Если `CLAUDE.md`, `docs/`, `.claude/skills/` уже настроены (по маркеру «setup-agent-harness» в CLAUDE.md или наличию всех skills):

- Проверить, всех ли **8 skills** хватает. Если недостаёт — предложить доложить недостающие (без перезаписи существующих).
- Спросить пользователя: «Harness уже установлен. Что обновить?»
  - (a) Только skills (перезаписать 8 файлов из свежих шаблонов).
  - (b) Только CLAUDE.md (с backup).
  - (c) Только docs, если появились новые файлы в исходниках.
  - (d) Полный переустановщик — всё пересобрать.
  - (e) Отмена.

Не перезаписывать таск-файлы **никогда**. `INDEX.md` — только дополнение, никаких удалений.
