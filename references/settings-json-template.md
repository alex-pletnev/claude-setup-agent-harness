# settings.json — рекомендованные hooks

Опциональный компонент harness'а. Устанавливается в `.claude/settings.json` целевого проекта (project-scope, коммитится в git для team-consistency).

**НЕ устанавливать в user-scope `~/.claude/settings.json`** — hooks сработают во всех проектах, blast radius.

> **Source of truth:** JSON-файлы в `references/settings-hooks/*.json`. Часть C `/harness-update` читает только их. Этот markdown — человеко-читаемая документация «зачем каждый hook нужен».

## Hook 1 — Stop-hook: reminder про self-review после commit'а

**Source:** `references/settings-hooks/self-review-reminder.json`

**Что делает:** на событии `Stop` (когда Claude завершает итерацию) сравнивает current `HEAD` с сохранённым в `.claude/last-committed-head.txt`. Если различаются — значит в этой итерации был commit → возвращает `{"decision":"block","reason":"…"}`. Claude Code интерпретирует это как «не останавливайся, вот причина» → модель получает reminder про self-review и продолжает работу в той же итерации.

**Форма output'а для события Stop:** только top-level поля `continue|suppressOutput|stopReason|decision|reason|systemMessage`. `hookSpecificOutput.additionalContext` разрешён лишь для `UserPromptSubmit`/`PostToolUse` — для Stop он даёт ошибку валидации. JSON собираем через `python3 -m json` для корректного escape'а commit-message'а с любыми символами.

**Почему безопасен:**
- Guard `state_file`: следующий Stop на том же HEAD ничего не выводит → нет loop'а.
- Обёрнут в `|| exit 0` / `2>/dev/null` — падение shell'а не разрушает hook.
- Время выполнения ~50ms (git rev-parse + один file read + python3 startup).
- State в gitignored `.claude/last-committed-head.txt` — не создаёт конфликтов между разработчиками.

**settings.json (fragment):**

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "cd \"$(git rev-parse --show-toplevel 2>/dev/null || pwd)\" || exit 0; current=$(git rev-parse HEAD 2>/dev/null || echo \"\"); state_file=\".claude/last-committed-head.txt\"; prev=$(cat \"$state_file\" 2>/dev/null || echo \"\"); if [ -n \"$current\" ] && [ \"$current\" != \"$prev\" ]; then echo \"$current\" > \"$state_file\"; short=$(git rev-parse --short \"$current\"); msg=$(git log -1 --format=%s \"$current\"); python3 -c 'import json,sys; print(json.dumps({\"decision\":\"block\",\"reason\":f\"В этой итерации был commit {sys.argv[1]}: {sys.argv[2]}. По правилу harness (self-review в Auto-mode) — прогони /self-review до ответа пользователю.\"}))' \"$short\" \"$msg\"; fi",
            "timeout": 5,
            "statusMessage": "Checking for new commit"
          }
        ]
      }
    ]
  }
}
```

**Обязательно в `.gitignore`:**

```
# harness hook state (per-clone)
.claude/last-committed-head.txt
```

## Как активировать

**Новый проект** (через `/setup-agent-harness` — см. playbook Фаза 6.5): установщик спрашивает «включить рекомендованные hooks? (y/n)», при `y` записывает settings.json + добавляет строку в .gitignore.

**Существующий проект:** `/harness-update` Часть C — sync hooks из `references/settings-hooks/*.json` в проектный `settings.json` с backup'ом, detect'ом local-customization и tracking'ом через `.claude/harness-sync-state.json`.

**После install:** hook применится после первого `/hooks` открытия в UI или после перезапуска сессии — Claude Code watches settings.json только для директорий, где settings.json существовал при старте сессии.

## Что НЕ включаем (осознанно)

Из анализа T-030 — эти hooks рассмотрены и отклонены пока:

- **PreToolUse(Bash git commit)** — блокирует нормальные commit'ы. Слишком тесная связь.
- **PreToolUse(Write) для wheel-check** — ложные срабатывания, шум.
- **User-scope hooks** — blast radius. Только project-scope.
