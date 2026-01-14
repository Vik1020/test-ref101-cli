---
name: new-release-auto
description: Создаёт релиз по процессу feature_full_auto (БЕЗ human approvals, полностью автоматически). Используй при "Создай релиз", "new release", "/release", "автоматический релиз".
---

# New Release Auto Skill (MCP-first v1.20.0)

Создание релиза по процессу `feature_full_auto` - 9 фаз, 0 approvals.

**ВАЖНО:** Этот Skill использует MCP-first подход для гарантированной записи workflow состояния.

## Trigger Keywords

- "Создай релиз v{X.Y.Z}: {description}"
- "new release v{X.Y.Z}: {description}"
- "/new-release v{X.Y.Z}: {description}"

## ⚠️ Plan Mode Constraint

**Этот Skill нельзя запускать в plan mode.**

Plan mode ограничивает инструменты до read-only, но Skill использует:
- `pcc_init_release` (создание файлов)
- `pcc_create_artifact` (запись артефактов)
- `Edit` (изменение кода)
- `Bash(git commit)` (коммиты)

**Правильный workflow:**
1. Завершите планирование (ExitPlanMode)
2. Получите подтверждение плана от пользователя
3. Запустите Skill вне plan mode

---

## Instructions

### 0. Start Exploration Tracking (v1.35.0)

**ОБЯЗАТЕЛЬНО выполни в самом начале скилла:**

```bash
bash "$(git rev-parse --show-toplevel)/tools/observability/hooks/start-exploration.sh" "{краткое описание фичи}"
```

Этот скрипт сохраняет snapshot текущих токенов. При вызове `pcc_init_release` метрики EXPLORATION будут автоматически импортированы в phase_history[0].

**Важно:** Выполни этот шаг СРАЗУ после загрузки скилла, до любого исследования кодовой базы.

### 1. Parse Request

Extract from user message:
- **version**: vX.Y.Z format (e.g., v1.19.0)
- **description**: короткое описание фичи
- **feature_name**: snake_case версия description (e.g., "dark_theme")

If version not provided, ask user.

### 2. Initialize Release via MCP (CRITICAL)

**ОБЯЗАТЕЛЬНО используй pcc_init_release вместо Edit для создания RELEASE.md:**

```
Call: pcc_init_release
Args: {
  version: "v{X.Y.Z}",
  feature_name: "{feature_name}",
  process_id: "feature_full_auto",
  problem_statement: "{description}",
  solution_summary: "TBD in BC/AC"
}
```

Этот вызов создаёт:
- `.pcc/releases/v{X.Y.Z}.json` (SSOT для workflow)
- `docs/releases/v{X.Y.Z}/RELEASE_v{X_Y_Z}_{feature_name}.md`
- Устанавливает `current_phase: RELEASE`

### 3. Execute Workflow Phases (MCP-first)

**ВАЖНО:** Для каждого артефакта используй `pcc_create_artifact` с `auto_transition: true`.
НЕ используй Edit для создания BC_delta, AC_delta и других workflow-файлов.

#### Phase: BC_DRAFT

```
Call: pcc_create_artifact
Args: {
  release_id: "v{X.Y.Z}",
  artifact_type: "BC_delta",
  content: "<full BC_delta markdown content>",
  auto_transition: true
}
```

Content должен включать: Goals, Actors, Features, Scenarios

#### Phase: AC_DRAFT

```
Call: pcc_create_artifact
Args: {
  release_id: "v{X.Y.Z}",
  artifact_type: "AC_delta",
  content: "<full AC_delta markdown content>",
  auto_transition: true
}
```

Content должен включать: Use Cases, Data Flow, Contracts

#### Phase: PLAN_FINALIZE

```
Call: pcc_create_artifact
Args: {
  release_id: "v{X.Y.Z}",
  artifact_type: "PLAN_FINALIZE",
  content: "<full PLAN_FINALIZE markdown content>",
  auto_transition: true
}
```

**ВАЖНО:** Content ДОЛЖЕН включать таблицу Tasks в формате:

```markdown
## Tasks

| ID | Title | File | Complexity |
|----|-------|------|------------|
| T1 | Task description | path/to/file.ts | Low/Medium/High |
| T2 | Another task | path/to/other.ts | Low |
```

Валидатор `plan_has_tasks` ищет строки `| T1 |`, `| T2 |` и т.д. Без этой таблицы переход в PC_DEVELOPMENT будет заблокирован.

Также включить: Task Breakdown (детали), Dependencies, Risks

#### Phase: PC_DEVELOPMENT

1. Implement the feature (write code via Edit)
2. Transition manually after implementation:

```
Call: pcc_workflow_transition
Args: { release_id: "v{X.Y.Z}", to_phase: "IC_VALIDATION" }
```

#### Phase: IC_VALIDATION

```
Call: pcc_create_artifact
Args: {
  release_id: "v{X.Y.Z}",
  artifact_type: "IC",
  content: "<full IC_VALIDATION markdown content>",
  auto_transition: true
}
```

#### Phase: QA_TESTING

```
Call: pcc_create_artifact
Args: {
  release_id: "v{X.Y.Z}",
  artifact_type: "QA",
  content: "<full QA_TESTING markdown content>",
  auto_transition: true
}
```

#### Phase: APPLY_DELTAS

1. Update domain files (BC_DOMAIN_*, AC_DOMAIN_*) via Edit
2. Transition:

```
Call: pcc_workflow_transition
Args: { release_id: "v{X.Y.Z}", to_phase: "DEPLOYED" }
```

#### Phase: DEPLOYED

1. `git add . && git commit`
2. `git tag v{X.Y.Z}`
3. `git push && git push --tags`

### 4. Report Completion

Show summary:
- Version deployed
- Files created
- Git tag created

## MCP Tools Reference (v1.20.0)

| Tool | Purpose | Example |
|------|---------|---------|
| `pcc_init_release` | **Create release with state** | `{ version: "v1.20.0", feature_name: "dark_theme", process_id: "feature_full_auto", problem_statement: "..." }` |
| `pcc_create_artifact` | **Create artifact + auto-transition** | `{ release_id: "v1.20.0", artifact_type: "BC_delta", content: "...", auto_transition: true }` |
| `pcc_workflow_status` | Check current phase | `{ release_id: "v1.20.0" }` |
| `pcc_workflow_transition` | Manual phase change | `{ release_id: "v1.20.0", to_phase: "DEPLOYED" }` |

## Error Handling

### MCP Server Unavailable

If `pcc_init_release` fails with connection error:
1. Inform user: "MCP сервер недоступен"
2. Suggest: "Запустите: node tools/command-center/dist/mcp/server.js"
3. Offer: "Продолжить без MCP enforcement? (не рекомендуется)"

### Release Already Exists

If `pcc_init_release` returns `RELEASE_EXISTS`:
1. Show current phase
2. Ask: "Продолжить с текущей фазы или начать заново?"

### Validation Failed

If `pcc_workflow_transition` returns blocking validators:
1. Show which validators failed
2. Fix the issue (add missing sections)
3. Retry transition

## Example Session (MCP-first)

```
User: Создай релиз v1.21.0: добавить темную тему

Claude: Начинаю релиз v1.21.0 по процессу feature_full_auto.

[pcc_init_release] → success
  - State: .pcc/releases/v1.21.0.json
  - File: docs/releases/v1.21.0/RELEASE_v1_21_0_dark_theme.md
  - Phase: RELEASE

[pcc_create_artifact BC_delta, auto_transition: true] → success
  - File: docs/releases/v1.21.0/BC_delta_dark_theme.md
  - Transitioned: RELEASE → BC_DRAFT

[pcc_create_artifact AC_delta, auto_transition: true] → success
  - Transitioned: BC_DRAFT → AC_DRAFT

... continues through all phases ...

[git commit, tag, push]

✅ Release v1.21.0 deployed!
- Files: 7 created
- Tag: v1.21.0
- Workflow state: 100% tracked in .pcc/
- Pushed to origin/main
```
