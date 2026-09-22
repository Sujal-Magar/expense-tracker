# Feature Implementation Playbook — Multi-Agent Workflow

This document is the operational guide for developing any feature (`[[FEATURE]]`, e.g. `expense-crud`) under the **Staged Dual-Validation Workflow with Multi-Agent Parallel Execution**.

Replace `[[FEATURE]]` with the actual feature name (e.g. `expense-crud`). `[[VERSION]]` is the plan version (e.g. `1.0.0`).

Small, simple features do not require the full multi-agent pipeline — see [When to Use the Simple Path Instead](#when-to-use-the-simple-path-instead) at the end. Base prompts (`.ai/prompts/build-mode.md` and `.ai/prompts/validation-prompt.md`) remain available for single-agent or sequential workflows.

---

## Workflow Lifecycle at a Glance

```
0. Spec & Catalog Registration ──► pnpm index:sync ──► pnpm index:verify
   (FDS, Behavior, Visuals/Figma)                               │
                                                                ▼
1. Plan: Frontend Fragment ══╗
                              ║  run in parallel
   Plan: Backend Fragment  ══╝
                │
                ▼
2. Plan Synthesizer  ──────────► plan-v[[VERSION]].md
                │                contract-v[[VERSION]].md (technology-agnostic API Contract spec)
                ▼
3. Plan Review (independent agent)  ──► plan-v[[VERSION]]-review.md (reviews both files)
                │
                ▼
4. Developer Approval Gate  (Commit Plan + Contract)
                │            freezes: FDS, behavior, visual specs, plan, contract-v[[VERSION]].md
                │            neither agent below may edit contract-v[[VERSION]].md
                │
    ┌───────────┴────────────┐
    ▼                        ▼
5. Frontend Build        Backend Build         (run in parallel, each its own commit)
    │                        │  (generates @expense-tracker/contracts
    │                        │   from contract-v[[VERSION]].md first, if required)
    ▼                        │
6. UI Review & Freeze        │        (can happen as soon as Frontend commits — frontend
    │                        │         still runs against mocks, independent of Backend)
    └───────────┬────────────┘
                ▼
7. Integration Build  (Commit Integration)
                │
                ▼
   7b. Code Validation 1 — SonarQube Static Gate + Fix Loop
                │
    ┌───────────┴────────────┐
    ▼                        ▼
8. Unit/API Test        UI/End-to-End Test     (run in parallel)
    │                        │
    └───────────┬────────────┘
                ▼
       (Commit Tests & Fixes)
                │
                ▼
   8b. Diagnosis & Fix Loop (If Testing or Validation Finds Unresolved Failures)
                │
                ▼
   8c. Code Validation 2 — SonarQube Full Gate + Fix Loop
                │
                ▼
9. Validation Mode  (Commit Validation Report)
```

---

## A Note on Committing Parallel Work

Whenever two agents work in the same working tree concurrently, running `git add -A` is unsafe as it may capture incomplete files from another agent. Every commit instruction below stages **only the paths the specific phase owns**:

- **Frontend Build**: stages only `frontend/`
- **Backend Build**: stages only `backend/` and `packages/contracts/`
- **Testing**: stages `backend/`, `frontend/`, and `features/[[FEATURE]]/plans/`

Always run `git status` prior to committing to verify staged changes stay strictly within the assigned boundary.

---

## A Note on the Activity Log

Every agent appends one line to `features/[[FEATURE]]/plans/activity-log.md` upon completion (creating the file if it does not exist). This provides an audit trail:
`- <Phase Name> | <YYYY-MM-DD HH:mm> | files touched: <paths> | result: <done / failed / notes>`

For phases that use base prompts (Integration, Validation), add the log entry manually before committing.

---

## Phase 0: Specification Definition & Catalog Registration

Before invoking any AI planning agent, the developer establishes the source of truth for the feature by authoring its specifications and registering it in the project catalog.

### 1. Author Feature Specifications

Create the feature directory under `features/[[FEATURE]]/`:

- **Feature Design Specification (`features/[[FEATURE]]/fds.md`)**:
  Must include standard YAML frontmatter, functional requirements, validation rules, data model, and acceptance criteria:

  ```markdown
  ---
  id: [[FEATURE]]
  version: 1.0.0
  status: draft
  changelog:
    - version: 1.0.0
      date: YYYY-MM-DD
      changes: "Initial draft of [[FEATURE]] specification"
  ---

  # Feature Design Specification: [[FEATURE]]

  ## Description

  ...

  ## Functional Requirements

  - REQ-[[PREFIX]]-01: ...
  - REQ-[[PREFIX]]-02: ...

  ## Validation Rules

  1. Field constraints (types, min/max lengths, regex, positive numbers)...

  ## Data Model

  | Field | Type        | Required | Description |
  | :---- | :---------- | :------- | :---------- |
  | id    | UUID/string | Yes      | Primary key |

  ## Acceptance Criteria

  - [ ] Criteria 1...
  ```

- **Behavioral Specification (`features/[[FEATURE]]/behavior.md`)**:
  Documents user interactions and edge states:
  - Form submission, cancellation, and validation UX (inline error states).
  - Empty states (with CTA) and loading states (skeleton loaders, mutation spinners).
  - Toast notifications for success and failure.
  - Confirmation dialogs for destructive actions (e.g. deletion).

- **Visual & Design Specifications (`features/[[FEATURE]]/visuals/`)**:
  - `features/[[FEATURE]]/visuals/figma.md`: Layout dimensions, spacing, grid, color tokens, typography, and component states.
  - UI assets: Screenshots, wireframes, or mockup diagrams (e.g. `features/[[FEATURE]]/visuals/[[FEATURE]].png`).

---

### 2. Register Feature in `features/index.json`

Add the feature entry under `active_features` in `features/index.json`:

```json
"[[FEATURE]]": {
  "id": "[[FEATURE]]",
  "title": "Feature Title",
  "domain": "finance",
  "status": "active",
  "version": "1.0.0",
  "path": "features/[[FEATURE]]/fds.md",
  "behavior_spec": "features/[[FEATURE]]/behavior.md",
  "figma_spec": "features/[[FEATURE]]/visuals/figma.md",
  "visual_spec": [
    "features/[[FEATURE]]/visuals/[[FEATURE]].png"
  ],
  "owner": "feature-team",
  "dependencies": []
}
```

---

### 3. Run Index Automation Scripts

Run the index scripts to update metrics and verify that all specifications are completely consistent:

```bash
# 1. Recalculates total, active, and archived counts and updates features/index.json
pnpm index:sync

# 2. Validates that all files exist on disk, versions match the FDS frontmatter, and references are consistent
pnpm index:verify
```

> [!IMPORTANT]
> `pnpm index:verify` must exit with `0` (clean) before you proceed to Phase 1. If any missing files, version mismatches, or invalid dependencies are reported, fix them first.

---

### 4. Commit Feature Specifications

Once verification passes, commit the specifications:

```bash
git add features/[[FEATURE]]/ features/index.json
git commit -m "docs([[FEATURE]]): add feature specifications and register in catalog"
```

---

## Phase 1: Plan Mode — Two Parallel Fragments

Frontend and backend planning require different contextual focus (UI/interaction vs data model/domain rules). Drafting them independently and merging surfaces interface discrepancies before code is written.

### Session A — Frontend Fragment

Open a new session:

- **System Prompt**:
  ```text
  Read the file .ai/prompts/plan/plan-fe.md and follow it exactly. That is your system prompt.
  ```
- **User Message**:
  ```text
  Feature ID = expense-crud
  ```

### Session B — Backend Fragment

Concurrently, in a separate session:

- **System Prompt**:
  ```text
  Read the file .ai/prompts/plan/plan-be.md and follow it exactly. That is your system prompt.
  ```
- **User Message**:
  ```text
  Feature ID = expense-crud
  ```

### What Happens:

- Session A produces `features/expense-crud/plans/plan-v[[VERSION]]-fragment-frontend.md`.
- Session B produces `features/expense-crud/plans/plan-v[[VERSION]]-fragment-backend.md`.
- Both sessions are read-only with respect to source code and specifications.

---

## Phase 2: Plan Synthesizer

Run in a new session after both fragments are generated:

- **System Prompt**:
  ```text
  Read the file .ai/prompts/plan/plan-synthesizer.md and follow it exactly. That is your system prompt.
  ```
- **User Message**:
  ```text
  Feature ID = expense-crud
  ```

### What Happens:

- Agent reconciles both fragments and produces:
  1. `features/expense-crud/plans/plan-v[[VERSION]].md` (the unified implementation plan)
  2. `features/expense-crud/plans/contract-v[[VERSION]].md` (canonical, technology-agnostic API Contract specification: endpoints, request/response structures, and status codes in Markdown prose).

---

## Phase 3: Plan Review

Run in a new session with an independent agent:

- **System Prompt**:
  ```text
  Read the file .ai/prompts/plan/plan-review.md and follow it exactly. That is your system prompt.
  ```
- **User Message**:
  ```text
  Feature ID = expense-crud
  ```

### What Happens:

- Evaluates the synthesized plan and contract against `rules/architecture.md`, `rules/conventions.md`, and `rules/tech-stack.md`.
- Produces `features/expense-crud/plans/plan-v[[VERSION]]-review.md` with a verdict (`PASS` or `CHANGES REQUIRED`).

### Action:

- If `CHANGES REQUIRED`: Address findings via Plan Synthesizer or fragments, then re-review.
- If `PASS`: Proceed to Developer Approval Gate.

---

## Phase 4: Developer Approval Gate

### Action:

1. Review `plan-v[[VERSION]].md`, `contract-v[[VERSION]].md`, and `plan-v[[VERSION]]-review.md`.
2. Confirm the Plan Review verdict is `PASS`.
3. Confirm `contract-v[[VERSION]].md` is complete and technology-agnostic.

### Commit:

```bash
git add -A && git commit -m "docs(expense-crud): add approved implementation plan and contract"
```

All specifications (`fds.md`, `behavior.md`, `visuals/`, `plan-v[[VERSION]].md`, and `contract-v[[VERSION]].md`) are now **frozen**.

---

## Phase 5: Frontend Build and Backend Build — Parallel Execution

Because `contract-v[[VERSION]].md` is frozen, Frontend and Backend agents build in parallel without waiting on each other.

### Session A — Frontend Build

- **System Prompt**:
  ```text
  Read the file .ai/prompts/build/build-mode-frontend.md and follow it exactly. That is your system prompt.
  ```
- **User Message**:
  ```text
  Feature ID = expense-crud
  ```

### Session B — Backend Build

Concurrently, in a separate session:

- **System Prompt**:
  ```text
  Read the file .ai/prompts/build/build-mode-backend.md and follow it exactly. That is your system prompt.
  ```
- **User Message**:
  ```text
  Feature ID = expense-crud
  ```

### What Happens:

- **Frontend Agent**: Builds UI components against mock data shaped to match `contract-v[[VERSION]].md`. Writes strictly to `frontend/`.
- **Backend Agent**: Generates/updates ts-rest contract definitions under `packages/contracts/` from `contract-v[[VERSION]].md`, then implements Drizzle schemas, repositories, services, and Express routes. Writes strictly to `backend/` and `packages/contracts/`.

### Commits:

Each session commits independently upon completion:

```bash
# Frontend (once finished and UI Review in Phase 6 is complete):
pnpm format
git status
git add frontend/
git commit -m "feat(expense-crud): frontend build complete and UI frozen"
```

```bash
# Backend (once finished):
pnpm format
git status
git add backend/ packages/contracts/
git commit -m "feat(expense-crud): backend build complete"
```

---

## Phase 6: UI Review & Freeze

Can be executed as soon as Frontend Build is complete, without waiting for Backend Build (since the frontend runs on mocks):

1. Start frontend dev server (`pnpm --filter @expense-tracker/frontend dev`) and inspect the feature pages.
2. Verify visual styling, responsive design, modals, form states, and empty states against visual specs.
3. Once verified, freeze UI and complete the Frontend commit shown in Phase 5.

---

## Phase 7: Integration Build Mode

Runs sequentially after both Frontend and Backend are committed.

### Claude Code Prompt:

- **System Prompt**:
  ```text
  Read the file .ai/prompts/build-mode.md and follow it exactly. That is your system prompt.
  ```
- **User Message**:
  ```text
  Feature ID = expense-crud
  Phase = Integration
  ```

### What Happens:

- Replaces frontend mock data with real API calls using the `@expense-tracker/contracts` ts-rest client.
- Connects React Hook Form submissions, TanStack Query mutations, and error toasts.
- Purges mock data files from production paths.

### Commit:

```bash
pnpm format
git add -A && git commit -m "feat(expense-crud): integration build complete"
```

### Action:

Append to `features/expense-crud/plans/activity-log.md`:
`- Build: Integration | <YYYY-MM-DD HH:mm> | files touched: frontend/**, backend/** | result: done`

---

## Phase 7b: Code Validation 1 — SonarQube Static Gate

Runs after Integration Build is committed, prior to writing tests:

1. Run the local SonarQube **Static Analysis Gate** scan (no coverage threshold required at this phase).
2. If zero blocker or critical issues are found, proceed to Phase 8.
3. If issues exist, address them within the 3-attempt retry limit.
4. Record status in `features/expense-crud/plans/activity-log.md`:
   `- Code Validation 1 | <YYYY-MM-DD HH:mm> | sonar static gate: pass | result: done`

---

## Phase 8: Test Build Mode — Parallel Sessions

Post-implementation test generation driven by `fds.md` and `behavior.md`.

### Session A — Unit & API Integration Tests

- **System Prompt**:
  ```text
  Read the file .ai/prompts/test/test-build-mode-unit-api.md and follow it exactly. That is your system prompt.
  ```
- **User Message**:
  ```text
  Feature = expense-crud
  ```

### Session B — UI & End-to-End Tests

Concurrently, in a separate session:

- **System Prompt**:
  ```text
  Read the file .ai/prompts/test/test-build-mode-ui-e2e.md and follow it exactly. That is your system prompt.
  ```
- **User Message**:
  ```text
  Feature = expense-crud
  ```

### Commit:

Once both test sessions finish and all tests pass:

```bash
pnpm format
git status
git add backend/ frontend/ e2e/ features/expense-crud/plans/
git commit -m "test(expense-crud): test build complete (unit/API + UI/E2E, parallel)"
```

---

## Phase 8b: Diagnosis & Fix Loop (On Unresolved Failures)

If Phase 8 reports failures that cannot be resolved within the test agents' bounded retries, or if cross-boundary defects are surfaced:

### Claude Code Prompt:

- **System Prompt**:
  ```text
  Read the file .ai/prompts/diagnosis/diagnosis-mode.md and follow it exactly. That is your system prompt.
  ```
- **User Message**:
  ```text
  Feature ID = expense-crud
  ```

### What Happens:

Produces `features/expense-crud/plans/plan-v[[VERSION]]-diagnosis.md`, classifying each failure into one of six categories with concrete Suggested Next Steps:

| Category                      | Routing & Action                                                                               |
| :---------------------------- | :--------------------------------------------------------------------------------------------- |
| **Backend defect**            | Run `.ai/prompts/build/build-mode-backend.md` in **Fix Mode**                                  |
| **Frontend defect**           | Run `.ai/prompts/build/build-mode-frontend.md` in **Fix Mode**                                 |
| **Integration-wiring defect** | Re-run `.ai/prompts/build-mode.md` (`Phase = Integration`) scoped to the wiring issue          |
| **Bad test**                  | Re-run responsible test prompt to rewrite the faulty test in place                             |
| **Contract mismatch**         | Amend `contract-v[[VERSION]].md`, re-approve at Approval Gate, run both Dev Agents in Fix Mode |
| **FDS ambiguity**             | Escalate to human per `rules/workflow.md §4` (Clarification vs Extension vs Contradiction)     |

### Running Fix Mode:

- **Frontend Fix**:
  - System prompt: `.ai/prompts/build/build-mode-frontend.md`
  - User message:
    ```text
    Feature ID = expense-crud
    Mode = Fix
    Findings = <finding IDs from diagnosis report>
    ```
- **Backend Fix**:
  - System prompt: `.ai/prompts/build/build-mode-backend.md`
  - User message:
    ```text
    Feature ID = expense-crud
    Mode = Fix
    Findings = <finding IDs from diagnosis report>
    ```

Commit fixes with path-scoped staging (`git add frontend/` or `git add backend/`), re-run Integration (Phase 7), and re-run tests (Phase 8).

---

## Phase 8c: Code Validation 2 — SonarQube Full Gate

Runs once Phase 8 and any defect fixes are committed:

1. Run the local SonarQube **Full Quality Gate** scan (static analysis + line coverage per `coverage_target` in `fds.md` frontmatter, typically >80%).
2. Resolve any static analysis issues. If coverage is short, route back to the appropriate test agent.
3. Record status in `features/expense-crud/plans/activity-log.md`:
   `- Code Validation 2 | <YYYY-MM-DD HH:mm> | sonar full gate: pass | result: done`

---

## Phase 9: Validation Mode (Final Audit)

Final compliance audit verifying all requirements and acceptance criteria.

### Claude Code Prompt:

- **System Prompt**:
  ```text
  Read the file .ai/prompts/validation-prompt.md and follow it exactly. That is your system prompt.
  ```
- **User Message**:
  ```text
  Feature = expense-crud
  ```

### What Happens:

- Agent checks all FDS criteria, validates architecture adherence, executes automated checks, and generates `features/expense-crud/validation-report.md`.

### Commit:

```bash
pnpm format
git add -A && git commit -m "docs(expense-crud): add final validation report"
```

Append final entry to `features/expense-crud/plans/activity-log.md`:
`- Validation | <YYYY-MM-DD HH:mm> | output: validation-report.md | result: pass`

---

## When to Use the Simple Path Instead

Use the complexity scoring rubric in `rules/workflow.md §5` to select the workflow depth:

- **Low complexity (Score 0–2)**: Single component, no API contract changes. Use the simple sequential path:
  - Plan: `.ai/prompts/plan.md`
  - Build: `.ai/prompts/build-mode.md` (Frontend → Backend → Integration sequentially)
  - Validate: `.ai/prompts/validation-prompt.md`
- **Medium complexity (Score 3–4)**: Adopt Plan Review (`plan-review.md`) and parallel Testing (`test-build-mode-unit-api.md`, `test-build-mode-ui-e2e.md`), but keep Build sequential.
- **High complexity (Score 5–6)**: Full multi-agent pipeline detailed above with parallel fragments, plan review, parallel builds, and parallel test execution.

---

## Summary of All AI Prompts

| File                                           | Role                                                                            |
| :--------------------------------------------- | :------------------------------------------------------------------------------ |
| `.ai/prompts/plan/plan-fe.md`                  | Frontend planning fragment (parallel with `plan-be.md`)                         |
| `.ai/prompts/plan/plan-be.md`                  | Backend planning fragment (parallel with `plan-fe.md`)                          |
| `.ai/prompts/plan/plan-synthesizer.md`         | Synthesizes fragments into `plan-v<version>.md` and `contract-v<version>.md`    |
| `.ai/prompts/plan/plan-review.md`              | Independent pre-approval review of plan and API contract                        |
| `.ai/prompts/build/build-mode-frontend.md`     | Parallel frontend build against frozen contract mocks                           |
| `.ai/prompts/build/build-mode-backend.md`      | Parallel backend build (generates `packages/contracts` and API)                 |
| `.ai/prompts/build-mode.md`                    | Baseline build prompt used for Phase 7 (Integration) and sequential simple path |
| `.ai/prompts/test/test-build-mode-unit-api.md` | Backend & API integration tests (parallel with UI/E2E)                          |
| `.ai/prompts/test/test-build-mode-ui-e2e.md`   | Frontend component & Playwright E2E tests (parallel with Unit/API)              |
| `.ai/prompts/diagnosis/diagnosis-mode.md`      | Classifies test/validation failures into 6 categories with concrete routing     |
| `.ai/prompts/validation-prompt.md`             | Baseline validation prompt used for Phase 9 final audit and compliance report   |
