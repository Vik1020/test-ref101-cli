---
name: new-feature
description: Создаёт фичу по процессу feature_full С human approvals на каждом этапе. Используй при "Создай фичу", "new feature", "фича с approvals", "feature with approvals".
---

# New Feature Skill (MCP-first v1.20.0)

Создание фичи по процессу `feature_full` - 9 фаз, 4 approvals.

**ВАЖНО:** Этот Skill использует MCP-first подход для гарантированной записи workflow состояния.

## Trigger Keywords

- "Создай фичу v{X.Y.Z}: {description}"
- "new feature v{X.Y.Z}: {description}"
- "фича с approvals v{X.Y.Z}: {description}"
- "feature with approvals v{X.Y.Z}: {description}"

## Process Overview

```
RELEASE → BC_DRAFT → AC_DRAFT → PLAN_FINALIZE → PC → IC → QA → APPLY → DEPLOY
             ↑          ↑            ↑                     ↑
           [PO]       [TL]        [Team]                [QA Lead]
```

**Approvals:**
- BC_DRAFT → Product Owner
- AC_DRAFT → Tech Lead
- PLAN_FINALIZE → Team
- QA_TESTING → QA Lead

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
- **version**: vX.Y.Z format (e.g., v1.20.0)
- **description**: короткое описание фичи
- **feature_name**: snake_case версия description (e.g., "critical_auth")

If version not provided, ask user.

### 2. Initialize Release via MCP (CRITICAL)

**ОБЯЗАТЕЛЬНО используй pcc_init_release вместо Edit для создания RELEASE.md:**

```
Call: pcc_init_release
Args: {
  version: "v{X.Y.Z}",
  feature_name: "{feature_name}",
  process_id: "feature_full",
  problem_statement: "{description}",
  solution_summary: "TBD in BC/AC"
}
```

Этот вызов создаёт:
- `.pcc/releases/v{X.Y.Z}.json` (SSOT для workflow)
- `docs/releases/v{X.Y.Z}/RELEASE_v{X_Y_Z}_{feature_name}.md`
- Устанавливает `current_phase: RELEASE`

### 3. Execute Workflow Phases (with approvals, MCP-first)

Follow `feature_full` process (with human approvals):

#### Phase: BC_DRAFT (⏸️ APPROVAL: Product Owner)

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

1. **ASK USER:** "BC готов. Approve для перехода к AC? (Product Owner)"
2. Wait for user confirmation

#### Phase: AC_DRAFT (⏸️ APPROVAL: Tech Lead)

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

1. **ASK USER:** "AC готов. Approve для перехода к PLAN? (Tech Lead)"
2. Wait for user confirmation

#### Phase: PLAN_FINALIZE (⏸️ APPROVAL: Team)

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

1. **ASK USER:** "Plan готов. Approve для начала разработки? (Team)"
2. Wait for user confirmation

#### Phase: PC_DEVELOPMENT

1. Transition:
```
Call: pcc_workflow_transition
Args: { release_id: "v{X.Y.Z}", to_phase: "PC_DEVELOPMENT" }
```

2. Implement the feature (write code via Edit)

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

Validate: Code quality, Security, Testing

#### Phase: QA_TESTING (⏸️ APPROVAL: QA Lead)

```
Call: pcc_create_artifact
Args: {
  release_id: "v{X.Y.Z}",
  artifact_type: "QA",
  content: "<full QA_TESTING markdown content>",
  auto_transition: true
}
```

1. Test the implementation
2. **ASK USER:** "QA passed. Approve для деплоя? (QA Lead)"
3. Wait for user confirmation

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
- Approvals received: 4/4

## MCP Tools Reference (v1.20.0)

| Tool | Purpose | Example |
|------|---------|---------|
| `pcc_init_release` | **Create release with state** | `{ version: "v1.20.0", feature_name: "auth", process_id: "feature_full", problem_statement: "..." }` |
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

### Approval Rejected

If user says "No" or "Reject" at approval point:
1. Ask: "Что нужно исправить?"
2. Make requested changes
3. Re-present for approval

## Example Session (MCP-first)

```
User: Создай фичу v1.20.0: критичная авторизация

Claude: Начинаю фичу v1.20.0 по процессу feature_full (с approvals).

[pcc_init_release] → success
  - State: .pcc/releases/v1.20.0.json
  - File: docs/releases/v1.20.0/RELEASE_v1_20_0_auth.md
  - Phase: RELEASE

[pcc_create_artifact BC_delta, auto_transition: true] → success
  - File: docs/releases/v1.20.0/BC_delta_auth.md
  - Transitioned: RELEASE → BC_DRAFT

⏸️ BC готов. Approve для перехода к AC? (Product Owner)

User: Approve

[pcc_create_artifact AC_delta, auto_transition: true] → success
  - Transitioned: BC_DRAFT → AC_DRAFT

⏸️ AC готов. Approve для перехода к PLAN? (Tech Lead)

User: Approve

... continues with approvals at each checkpoint ...

[git commit, tag, push]

✅ Feature v1.20.0 deployed!
- Files: 7 created
- Tag: v1.20.0
- Approvals: 4/4 ✓
- Workflow state: 100% tracked in .pcc/
```
