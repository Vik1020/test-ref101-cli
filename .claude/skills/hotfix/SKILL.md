---
name: hotfix
description: Создаёт хотфикс по упрощённому процессу bugfix_simple (5 фаз, 1 approval). Используй при "Хотфикс", "hotfix", "быстрый фикс", "bugfix", "quick fix", "исправить баг".
---

# Hotfix Skill (MCP-first v1.20.0)

Быстрое исправление бага по процессу `bugfix_simple` - 5 фаз, 1 approval.

**ВАЖНО:** Этот Skill использует MCP-first подход для гарантированной записи workflow состояния.

## Trigger Keywords

- "Хотфикс v{X.Y.Z}: {description}"
- "hotfix v{X.Y.Z}: {description}"
- "быстрый фикс v{X.Y.Z}: {description}"
- "bugfix v{X.Y.Z}: {description}"
- "quick fix v{X.Y.Z}: {description}"
- "исправить баг v{X.Y.Z}: {description}"

## Process Overview

```
RELEASE → PC → IC → QA → DEPLOY
                    ↑
                 [QA Lead]
```

**Skipped phases:** BC_DRAFT, AC_DRAFT, PLAN_FINALIZE, APPLY_DELTAS

**Approvals:** Only QA_TESTING (QA Lead)

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

## When to Use

✅ Use hotfix when:
- Bug fix that doesn't change API contracts
- No new business requirements
- No architectural changes
- Patch version (v1.X.**1**)

❌ Don't use hotfix when:
- New feature (use new-feature or new-release-auto)
- Changes to BC/AC domains
- Breaking changes

## Instructions

### 0. Start Exploration Tracking (v1.35.0)

**ОБЯЗАТЕЛЬНО выполни в самом начале скилла:**

```bash
bash "$(git rev-parse --show-toplevel)/tools/observability/hooks/start-exploration.sh" "{краткое описание бага}"
```

Этот скрипт сохраняет snapshot текущих токенов. При вызове `pcc_init_release` метрики EXPLORATION будут автоматически импортированы в phase_history[0].

**Важно:** Выполни этот шаг СРАЗУ после загрузки скилла, до любого исследования кодовой базы.

### 1. Parse Request

Extract from user message:
- **version**: vX.Y.Z format (typically patch: v1.19.**1**)
- **description**: описание бага
- **fix_name**: snake_case версия description (e.g., "auth_empty_token")

If version not provided, suggest patch increment.

### 2. Initialize Release via MCP (CRITICAL)

**ОБЯЗАТЕЛЬНО используй pcc_init_release вместо Edit для создания RELEASE.md:**

```
Call: pcc_init_release
Args: {
  version: "v{X.Y.Z}",
  feature_name: "{fix_name}",
  process_id: "bugfix_simple",
  problem_statement: "{bug description}"
}
```

Этот вызов создаёт:
- `.pcc/releases/v{X.Y.Z}.json` (SSOT для workflow)
- `docs/releases/v{X.Y.Z}/RELEASE_v{X_Y_Z}_{fix_name}.md`
- Устанавливает `current_phase: RELEASE`

### 3. Execute Workflow Phases (5 phases only, MCP-first)

Follow `bugfix_simple` process:

#### Phase: PC_DEVELOPMENT

1. Transition to PC phase:
```
Call: pcc_workflow_transition
Args: { release_id: "v{X.Y.Z}", to_phase: "PC_DEVELOPMENT" }
```

2. Implement the fix (write code via Edit)
3. Create `src/{component}/context.md` if needed

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

Validate: Fix correctness, No regressions, Security

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

1. Test the fix
2. **ASK USER:** "QA passed. Approve для деплоя? (QA Lead)"
3. Wait for user confirmation

#### Phase: DEPLOYED

1. Transition:
```
Call: pcc_workflow_transition
Args: { release_id: "v{X.Y.Z}", to_phase: "DEPLOYED" }
```

2. `git add . && git commit -m "fix: {description}"`
3. `git tag v{X.Y.Z}`
4. `git push && git push --tags`

### 4. Report Completion

Show summary:
- Hotfix deployed
- Files modified
- Git tag created
- Approval: 1/1 ✓
- Workflow state: 100% tracked

## MCP Tools Reference (v1.20.0)

| Tool | Purpose | Example |
|------|---------|---------|
| `pcc_init_release` | **Create release with state** | `{ version: "v1.19.1", feature_name: "auth_fix", process_id: "bugfix_simple", problem_statement: "..." }` |
| `pcc_create_artifact` | **Create artifact + auto-transition** | `{ release_id: "v1.19.1", artifact_type: "IC", content: "...", auto_transition: true }` |
| `pcc_workflow_status` | Check current phase | `{ release_id: "v1.19.1" }` |
| `pcc_workflow_transition` | Manual phase change | `{ release_id: "v1.19.1", to_phase: "DEPLOYED" }` |

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

### Fix Introduces New Bug

If testing reveals new issues:
1. Return to PC_DEVELOPMENT
2. Fix the regression
3. Re-run IC_VALIDATION
4. Re-submit for QA

## Example Session (MCP-first)

```
User: Хотфикс v1.19.1: баг авторизации при пустом токене

Claude: Начинаю хотфикс v1.19.1 по процессу bugfix_simple (5 фаз).

[pcc_init_release] → success
  - State: .pcc/releases/v1.19.1.json
  - File: docs/releases/v1.19.1/RELEASE_v1_19_1_auth_empty_token.md
  - Phase: RELEASE

[pcc_workflow_transition] → PC_DEVELOPMENT

Реализую исправление...

[Edit] src/auth/tokenValidator.ts

[pcc_create_artifact IC, auto_transition: true] → success
  - File: docs/releases/v1.19.1/IC_VALIDATION_v1_19_1.md
  - Transitioned: PC_DEVELOPMENT → IC_VALIDATION

[pcc_create_artifact QA, auto_transition: true] → success
  - File: docs/releases/v1.19.1/QA_TESTING_v1_19_1.md
  - Transitioned: IC_VALIDATION → QA_TESTING

⏸️ QA passed. Approve для деплоя? (QA Lead)

User: Approve

[pcc_workflow_transition] → DEPLOYED

[git commit, tag, push]

✅ Hotfix v1.19.1 deployed!
- Files: 4 created/modified
- Tag: v1.19.1
- Approval: 1/1 ✓
- Workflow state: 100% tracked in .pcc/
- Time: ~15 min
```

## Comparison with Full Release

| Aspect | Hotfix (bugfix_simple) | Feature (feature_full) |
|--------|----------------------|----------------------|
| Phases | 5 | 9 |
| Approvals | 1 (QA) | 4 (PO, TL, Team, QA) |
| BC/AC docs | Skipped | Required |
| Domain updates | Skipped | Required |
| Typical time | 15-30 min | 1-2 hours |
| Use case | Bug fixes | New features |
