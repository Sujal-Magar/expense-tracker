# Expense Tracker — AI Agent Guide

This project uses the **Staged Dual-Validation Workflow** with Multi-Agent Parallel Execution. All AI agents must read this file first before doing anything.

Detailed operations are documented in [WORKFLOW_PLAYBOOK.md](file:///home/sujal/programming/work/expense-tracker/WORKFLOW_PLAYBOOK.md).

---

## Workflow Phases & Prompts

### Multi-Agent Pipeline (Default for Medium & High Complexity)

| Phase                               | When                             | System Prompt / Action                                                                                             |
| :---------------------------------- | :------------------------------- | :----------------------------------------------------------------------------------------------------------------- |
| **0 — Spec & Catalog Registration** | Before planning begins           | Author FDS, behavior, visual specs; run `pnpm index:sync` & `pnpm index:verify`                                    |
| **1 — Plan Fragments (Parallel)**   | Before any code is written       | `.ai/prompts/plan/plan-fragments.md` (`Phase = Both`, or individual `Frontend` / `Backend`)                        |
| **2 — Plan Synthesizer**            | After both fragments exist       | `.ai/prompts/plan/plan-synthesizer.md`                                                                             |
| **3 — Plan Review**                 | Before developer approval        | `.ai/prompts/plan/plan-review.md`                                                                                  |
| **4 — Developer Approval Gate**     | After plan review passes         | Human reviews & freezes `plan-v<version>.md` and `contract-v<version>.md`                                          |
| **5 — Build Mode (Parallel)**       | After approval gate              | `.ai/prompts/build-mode.md` (`Phase = Both`, or individual `Frontend` / `Backend`)                                 |
| **6 — UI Review & Freeze**          | After frontend build completes   | Human confirms UI against visual specs; freezes UI                                                                 |
| **7 — Integration Build Mode**      | After both build sessions commit | `.ai/prompts/build-mode.md` with `Phase = Integration`                                                             |
| **7b — Code Validation 1**          | After integration commits        | SonarQube Static Analysis Gate (no coverage threshold)                                                             |
| **8 — Test Build Mode (Parallel)**  | Post-integration spec tests      | `.ai/prompts/test/test-build-mode-unit-api.md` (Unit/API)<br>`.ai/prompts/test/test-build-mode-ui-e2e.md` (UI/E2E) |
| **8b — Diagnosis & Fix Loop**       | On unresolved test/gate failures | `.ai/prompts/diagnosis/diagnosis-mode.md`                                                                          |
| **8c — Code Validation 2**          | After tests pass                 | SonarQube Full Quality Gate (static analysis + coverage target)                                                    |
| **9 — Validation Mode**             | Final compliance audit           | `.ai/prompts/validation-prompt.md`                                                                                 |

### Simple / Sequential Baseline Prompts (Low Complexity)

For low-complexity features (Score 0–2 per `rules/workflow.md §5`):

- **Plan**: `.ai/prompts/plan-mode.md`
- **Build**: `.ai/prompts/build-mode.md` (`Phase = Frontend` → `Backend` → `Integration` sequentially)
- **Validation**: `.ai/prompts/validation-prompt.md`

---

## How to Invoke AI Workflow Modes

### Phase 1: Plan Fragments (Multi-Agent Subagents & Pragmatic Choice)

```text
# Multi-Agent Parallel Execution (Agent rolls up Frontend & Backend subagents):
System prompt : .ai/prompts/plan/plan-fragments.md
User message  : Feature ID = expense-crud
                Phase = Both

# Pragmatic Choice (When remaining session tokens are low):
# Draft one fragment individually in a single focused session without subagents:
System prompt : .ai/prompts/plan/plan-fragments.md
User message  : Feature ID = expense-crud
                Phase = Frontend (or Backend)
```

### Phase 2: Plan Synthesizer

```text
System prompt : .ai/prompts/plan/plan-synthesizer.md
User message  : Feature ID = expense-crud
```

### Phase 3: Plan Review

```text
System prompt : .ai/prompts/plan/plan-review.md
User message  : Feature ID = expense-crud
```

### Phase 5: Build Mode (Multi-Agent Subagents & Pragmatic Choice)

```text
# Multi-Agent Parallel Execution (Agent rolls up Frontend & Backend subagents):
System prompt : .ai/prompts/build-mode.md
User message  : Feature ID = expense-crud
                Phase = Both

# Pragmatic Choice (When remaining session tokens are low):
# Run one layer individually in a single focused session without subagents:
System prompt : .ai/prompts/build-mode.md
User message  : Feature ID = expense-crud
                Phase = Frontend (or Backend)
```

### Phase 7: Integration Build Mode

```text
System prompt : .ai/prompts/build-mode.md
User message  : Feature ID = expense-crud
                Phase = Integration
```

### Phase 8: Parallel Test Build Mode

```text
# Session A (Unit & API Integration):
System prompt : .ai/prompts/test/test-build-mode-unit-api.md
User message  : Feature = expense-crud

# Session B (UI & E2E):
System prompt : .ai/prompts/test/test-build-mode-ui-e2e.md
User message  : Feature = expense-crud
```

### Phase 8b: Diagnosis Mode (When Failures Occur)

```text
System prompt : .ai/prompts/diagnosis/diagnosis-mode.md
User message  : Feature ID = expense-crud
```

### Phase 9: Validation Mode

```text
System prompt : .ai/prompts/validation-prompt.md
User message  : Feature = expense-crud
```

---

## Project Rules (Read Before Any Task)

| Rule File                                                                                            | Summary & Critical Sections                                                                                                                                                           |
| :--------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [`rules/workflow.md`](file:///home/sujal/programming/work/expense-tracker/rules/workflow.md)         | Full 10-phase workflow, Rollback Decision Tree (§3), Change Classification (§4), Complexity Scoring (§5), Living Specs (§6), SonarQube Profiles (§7), Bounded Retry Policy (§8)       |
| [`rules/architecture.md`](file:///home/sujal/programming/work/expense-tracker/rules/architecture.md) | Layer hierarchy: Presentation → Service → Repository → Database (`data/app.db`); shared ts-rest contracts under `packages/contracts`; frontend Next.js 14 + TanStack Query + Tailwind |
| [`rules/tech-stack.md`](file:///home/sujal/programming/work/expense-tracker/rules/tech-stack.md)     | Next.js 14, Tailwind CSS, Radix UI, TanStack Query, Express.js, Drizzle ORM, better-sqlite3, ts-rest, Zod, Vitest, Playwright (No AWS SES)                                            |
| [`rules/conventions.md`](file:///home/sujal/programming/work/expense-tracker/rules/conventions.md)   | Naming conventions, strict typing without `any`, structured domain error handling, and test design                                                                                    |

---

## Feature Specs Location

```
features/
└── <feature-id>/
    ├── fds.md                   # Feature Design Specification (source of truth)
    ├── behavior.md              # Interaction & behavioral spec
    ├── plans/                   # Housing for versioned plans & contracts
    │   ├── plan-v1.0.0.md       # Implementation plan (frozen after approval)
    │   ├── contract-v1.0.0.md   # API contract specification (frozen after approval)
    │   ├── plan-v1.0.0-review.md# Plan review report
    │   └── activity-log.md      # Multi-agent audit trail
    ├── validation-report.md     # Generated in Validation Mode
    └── visuals/                 # Visual design reference and screenshots
```

### Active Features

| Feature               | FDS                                   | Status                 |
| :-------------------- | :------------------------------------ | :--------------------- |
| `expense-crud`        | `features/expense-crud/fds.md`        | active — awaiting-plan |
| `category-mgmt`       | `features/category-mgmt/fds.md`       | active — awaiting-plan |
| `budget-limits`       | `features/budget-limits/fds.md`       | active — awaiting-plan |
| `spend-reports`       | `features/spend-reports/fds.md`       | active — awaiting-plan |
| `transaction-filters` | `features/transaction-filters/fds.md` | active — awaiting-plan |

---

## Key Constraints

- **Never modify** `fds.md`, `behavior.md`, `visuals/`, approved plans in `plans/`, `contract-v<version>.md`, or `rules/` during Build or Test modes.
- **Bounded retry**: max 3 attempts per task, max 5 attempts for full suite fixes. Beyond 3 attempts, stop and escalate.
- **Strict path boundaries in parallel phases**: Frontend Build touches only `frontend/`; Backend Build touches only `backend/` and `packages/contracts/`.
- **No new libraries** without explicit approval.
- **No production code changes** in Test Build Mode unless a defect is formally documented or diagnosed.
- **All API contracts** originate in `packages/contracts` via ts-rest — never defined inline or duplicated between applications.
