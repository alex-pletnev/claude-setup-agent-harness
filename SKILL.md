---
name: setup-agent-harness
description: Развернуть в текущем проекте (существующем или пустом) полную инфраструктуру для работы с AI-агентом — универсальную и не привязанную к языку/стеку. Устанавливает CLAUDE.md с правилами (включая карту интеграции с superpowers-skills), файловый task-трекер (docs/tasks/), 8 slash-skills (task-add, task-done, task-sync, docs-sync, wheel-check, mid-retro, self-review, pre-flight), генерирует документацию проекта из анализа кода, встраивает git-автоматизацию (commit+push). Use когда пользователь просит «настроить harness», «поставить трекер», «/setup-agent-harness», или в первый раз садится за проект и хочет системную настройку под работу с агентом.
---

# setup-agent-harness

Оркестратор bootstrap-настройки agent harness в проекте. Skill — тонкий: сам ничего не пишет напрямую, а следует playbook и берёт шаблоны из `references/`.

**Все детали алгоритма — в `references/playbook.md`. Прочитать его целиком перед началом фазы 3.**

## Ключевые файлы references

| Файл | Назначение |
|------|-----------|
| `references/playbook.md` | Полный пошаговый чек-лист всех 8 фаз |
| `references/stack-detectors.md` | Как определить стек по манифестам, что из этого следует |
| `references/docs-structure.md` | Какие docs/*.md создавать под какой профиль проекта |
| `references/templates/claude-md.template.md` | Шаблон CLAUDE.md с плейсхолдерами |
| `references/templates/tasks-readme.template.md` | Шаблон `docs/tasks/README.md` |
| `references/templates/tasks-index.template.md` | Шаблон `docs/tasks/INDEX.md` |
| `references/templates/docs-readme.template.md` | Шаблон `docs/README.md` |
| `references/skills/task-add.md` | Skill для установки в `.claude/skills/` целевого проекта |
| `references/skills/task-done.md` | ↑ |
| `references/skills/task-sync.md` | ↑ |
| `references/skills/docs-sync.md` | ↑ |
| `references/skills/wheel-check.md` | Само-улучшение: проверка «не изобретаю ли велосипед» до нового кода |
| `references/skills/mid-retro.md` | Само-улучшение: пауза для само-осмотра внутри задачи |
| `references/skills/self-review.md` | Само-улучшение: ревью своего diff'а после `task-done` |
| `references/skills/pre-flight.md` | Само-улучшение: 3 обязательных вопроса (assumptions/risks/reversibility) между дизайном и кодингом для non-trivial и high-stakes |

## Дефолты поведения (утверждены при создании skill'а)

- **Интерактивность:** «делай сам, сводный отчёт в конце». Пользователя дёргать только один раз — **перед `git push`**.
- **Существующий CLAUDE.md:** бэкапнуть в `CLAUDE.md.bak-YYYY-MM-DD-HHMMSS`, переписать с нуля.
- **Существующие `docs/`, `docs/tasks/`, `.claude/skills/`:** проверить содержимое, слить (см. фаза 8 в playbook), ничего не удалять без разрешения.
- **Git-политика:** `main`, без подтверждений (кроме push), Conventional Commits + `Refs:`. Настройки записываются в CLAUDE.md → раздел «Git-автоматизация».
- **Язык общения агента:** RU по умолчанию; спросить пользователя, если целевой репозиторий явно EN (по README, комментариям в коде).
- **Идемпотентность:** повторный вызов на настроенном проекте не ломает существующее — только обновляет skills-шаблоны и добавляет недостающее.

## Ход выполнения (обзор)

Полный чек-лист — в `references/playbook.md`. Тут — только 8 фаз с 1-2 строчками:

1. **Discovery** — детект стека, git, наличия существующей инфраструктуры.
2. **Preflight** — быстрое подтверждение плана пользователю (одна фраза), приглашение остановить, если что-то не так.
3. **Analysis** (если проект непустой) — 4 параллельных Explore-агента по чек-листу.
4. **Docs generation** — `docs/` c набором файлов, зависящим от профиля проекта (см. `docs-structure.md`).
5. **Tracker setup** — `docs/tasks/README.md` + `INDEX.md` + seed-задачи из known-gaps (если анализ их выявил).
6. **Skills install** — 8 skills копируются в `.claude/skills/` (4 таск-трекерных + 4 само-улучшения).
7. **CLAUDE.md** — рендер из шаблона с подставленными параметрами (стек, git-политика).
8. **Bootstrap commit** — commit всех новых файлов + **спросить пользователя** перед push.

## Обязательные проверки перед завершением

- `docs/README.md` ссылается на все созданные doc-файлы.
- `docs/tasks/INDEX.md` содержит хотя бы одну строку (если анализ выявил gaps) или помечен «пусто».
- `.claude/skills/` содержит ровно 8 файлов (task-add, task-done, task-sync, docs-sync, wheel-check, mid-retro, self-review, pre-flight).
- `CLAUDE.md` в корне — с корректным контекстом (стек, git-policy).
- `git status --short` — чистое дерево или только созданные skill'ом файлы.
- Если предыдущий CLAUDE.md был — backup существует.

## Финальный отчёт

Единый итог: что создано (список путей), что решено (дефолты), первый git-commit hash, и **одна фраза-вопрос пользователю: «Готово к push? (y/n)»**.

## Что этот skill НЕ делает

- Не устанавливает зависимости проекта.
- Не запускает тесты.
- Не мигрирует БД.
- Не редактирует исходный код проекта.
- Не создаёт PR, не работает с ветками кроме `main`, не мерджит.
- Не удаляет существующие файлы без явного разрешения (кроме бэкапа CLAUDE.md).
