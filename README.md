# setup-agent-harness

Универсальный Claude Code skill для одноразовой bootstrap-настройки agent harness в любом проекте — существующем или пустом.

Собирает в проекте:

- `CLAUDE.md` — правила для AI-агента, git-политика, проактивные триггеры.
- `docs/` — компактную документацию с плотными таблицами и ссылками `file:line`.
- `docs/tasks/` — файловый task-трекер (одна задача = один MD-файл).
- `.claude/skills/` — 4 slash-skills: `task-add`, `task-done`, `task-sync`, `docs-sync`.

Работает **языко-агностично**: определяет стек по манифесту (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `build.gradle*`, `pom.xml`, `mix.exs`, …) и адаптирует шаблоны.

## Установка

```bash
git clone git@github.com:alex-pletnev/claude-setup-agent-harness.git \
  ~/.claude/skills/setup-agent-harness
```

Или, если репо уже склонирован куда-то ещё, symlink:

```bash
ln -s /path/to/claude-setup-agent-harness ~/.claude/skills/setup-agent-harness
```

Перезапустить сессию Claude Code — скилл появится в списке доступных.

## Использование

В любом проекте:

```
/setup-agent-harness
```

Skill сам:
1. Определит стек и git-состояние.
2. Одной репликой подтвердит план.
3. Проанализирует код 4 параллельными Explore-агентами (если проект непустой).
4. Сгенерирует `docs/` под профиль проекта.
5. Установит `docs/tasks/` + `.claude/skills/`.
6. Сгенерирует `CLAUDE.md`.
7. Соберёт первый commit.
8. **Спросит пользователя перед `git push`.**

## Дефолты поведения

- **Интерактивность:** «делай сам, отчёт в конце». Пользователя дёргает только перед push.
- **Существующий CLAUDE.md:** бэкапится в `CLAUDE.md.bak-YYYY-MM-DD-HHMMSS`, переписывается.
- **Существующие `docs/`, `docs/tasks/`, `.claude/skills/`:** сливаются с новыми шаблонами; ничего не удаляется без разрешения.
- **Git-политика:** `main`, без подтверждений (кроме push), Conventional Commits + `Refs:` на таск.
- **Язык агента:** RU по умолчанию; спрашивает, если код и README явно EN.
- **Идемпотентность:** повторный вызов на настроенном проекте не ломает — предлагает точечное обновление.

## Структура

```
setup-agent-harness/
├── SKILL.md                          # оркестратор + фазы
└── references/
    ├── playbook.md                   # детальный чек-лист 8 фаз
    ├── stack-detectors.md            # правила для 10+ стеков
    ├── docs-structure.md             # матрица «профиль × docs-файлы»
    ├── templates/
    │   ├── claude-md.template.md
    │   ├── tasks-readme.template.md
    │   ├── tasks-index.template.md
    │   └── docs-readme.template.md
    └── skills/
        ├── task-add.md
        ├── task-done.md
        ├── task-sync.md
        └── docs-sync.md
```

## После установки в проекте

Skill устанавливает 4 slash-команды в `.claude/skills/` целевого проекта:

| Команда | Действие | Git |
|---------|----------|-----|
| `/task-add "название"` | Создать таск-карточку | commit (без push) |
| `/task-done T-XXX` | Закрыть задачу | тесты → commit → push |
| `/task-sync` | Аудит трекера | нет |
| `/docs-sync` | Актуализировать `docs/*.md` | commit → push (если правки) |

Проактивные триггеры (без явной команды) описаны в сгенерированном `CLAUDE.md`.

## Лицензия

MIT.
