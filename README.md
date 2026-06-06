# superpowers-devpilot

A Claude Code plugin that automates the software delivery lifecycle — from Azure DevOps backlog to pull request — using two commands and two approval checkpoints.

---

## Overview

DevPilot orchestrates the full development workflow for a single Azure DevOps work item:

```
/dev-workitem <id>
  │
  ├─ Stage 1: Load work item from Azure DevOps
  ├─ Stage 2: Generate design document          ← you review & approve
  │
/dev-resume <id>  (design approved)
  │
  ├─ Stage 3: Generate implementation plan      ← you review & approve
  │
/dev-resume <id>  (plan approved)
  │
  ├─ Stage 4: Implement (automated)
  ├─ Stage 5: Code review (automated)
  ├─ Stage 6: Testing (automated)
  └─ Stage 7: Pull request (automated)

/dev-fix-pipeline <id>  (if pipeline fails on the PR)
  └─ Diagnose & fix CI failure, push fix, re-trigger pipeline
```

At each automated stage, DevPilot posts a `[DevPilot]` progress comment to your Azure DevOps work item, commits artifacts to a feature branch, and updates a local state file so the workflow can be resumed at any time.

---

## Requirements

| Dependency | Purpose | Install |
|---|---|---|
| [superpowers](https://github.com/obra/superpowers) | Provides the skills DevPilot orchestrates (brainstorming, TDD, code review, etc.) | `/plugin install superpowers@claude-plugins-official` |
| [Azure DevOps MCP](https://github.com/microsoft/azure-devops-mcp) | Connects DevPilot to your Azure DevOps org (work items, repos, PRs) | `claude mcp add azure-devops --scope user -- npx -y @azure-devops/mcp <org>` |
| Azure DevOps git remote | DevPilot parses your ADO org/project/repo from `origin` | — |

---

## Installation

### 1. Install this plugin

Add the marketplace and install in two commands:

* Register the marketplace:

```
/plugin marketplace add dragon1404/superpowers-devpilot
```

* Install the plugin from this marketplace:

```
/plugin install superpowers-devpilot@dragon-marketplace
```

Then reload plugins:

```
/reload-plugins
```

### 2. Run the setup wizard

Once the plugin is installed, open Claude Code and run:

```
/dev-setup
```

`/dev-setup` checks whether `superpowers` and the Azure DevOps MCP server are installed and walks you through installing any that are missing — including exact configuration steps and PAT scope requirements.

> **Note:** `/dev-workitem` also checks prerequisites at startup and will prompt you to run `/dev-setup` if anything is missing.

### Manual setup (if you prefer)

**superpowers plugin** — install from the official Claude Code marketplace:
```
/plugin install superpowers@claude-plugins-official
```

**Azure DevOps MCP** — replace `<org>` with your ADO organization name:
```bash
claude mcp add azure-devops --scope user -- npx -y @azure-devops/mcp <org>
```

If authentication is required, set a Personal Access Token with these scopes: **Work Items** (Read & Write), **Code** (Read), **Pull Requests** (Read & Write).

Generate a PAT at: Azure DevOps → User Settings → Personal Access Tokens → New Token

---

## Commands

### `/dev-setup`

Checks whether all required dependencies are installed and guides you through installing any that are missing.

Run this once after installing the plugin. It will:
1. Detect whether `superpowers` and the Azure DevOps MCP server are present
2. Show a status table (FOUND / MISSING) for each
3. For any missing dependency, provide exact installation steps including PAT configuration

```
/dev-setup
```

---

### `/dev-workitem <workItemId>`

Starts a new DevPilot workflow for the given Azure DevOps work item.

**What it does:**
1. Parses ADO org/project/repo from your git remote URL
2. Fetches the work item (title, description, acceptance criteria, linked items)
3. Creates a feature branch: `feature/<workItemId>-<title-slug>`
4. Initialises a state file at `.devpilot/state/<workItemId>.json`
5. Runs `superpowers:brainstorming` to produce a design document
6. Reviews the design for unresolved questions — if any, posts them to the work item and pauses for your input (`WAITING_FOR_DESIGN_CLARIFICATION`)
7. Saves the design to `docs/design/<workItemId>-design.md` and commits it
8. Posts `[DevPilot] Stage Completed: Design` to the work item
9. Pauses and prompts you to review the design

**Example:**
```
/dev-workitem 21238
```

---

### `/dev-resume <workItemId>`

Resumes the workflow from the last completed stage.

**At an approval gate** (`WAITING_FOR_DESIGN_APPROVAL` or `WAITING_FOR_PLAN_APPROVAL`), running `/dev-resume` **is** the approval — it advances the workflow automatically.

**At a clarification checkpoint** (`WAITING_FOR_DESIGN_CLARIFICATION` or `WAITING_FOR_PLAN_CLARIFICATION`), update the work item description in Azure DevOps with your answers, then run `/dev-resume` to continue. DevPilot re-fetches the updated description before resuming.

**After an interruption**, it detects the current state and picks up exactly where it left off.

**Example:**
```
/dev-resume 21238
```

---

### `/dev-fix-pipeline <workItemId>`

Investigates a failed CI pipeline on the PR for the given work item and automatically applies a fix.

**What it does:**
1. Checks preconditions: state file exists, PR is open (`prCreated: true`), current branch matches
2. Fetches the latest pipeline build for the feature branch (classic or YAML)
3. If the build is not failed, stops with a status message — no action taken
4. Retrieves the build log and classifies the failure as automatable or not
5. If not automatable (infra/secrets/config): posts analysis to the work item and stops
6. Invokes `superpowers:systematic-debugging` to diagnose and fix the error
7. If no files were changed by debugging, posts findings to the work item and stops
8. Runs affected tests locally; if sandbox-restricted, falls back to build-only verification
9. If verification fails, posts details to the work item and stops without pushing
10. Commits and pushes the fix; updates the state file (`pipelineFixCount`, `lastPipelineFixAt`)
11. Posts `[DevPilot] Pipeline fix applied` to the work item with error summary and verification result

**Example:**
```
/dev-fix-pipeline 21238
```

If the pipeline fails again after the fix, run the same command to retry.

---

## Workflow Stages

| # | Stage | Skill invoked | Artifact |
|---|-------|--------------|----------|
| 1 | Load work item | — | — |
| 2 | Design | `superpowers:brainstorming` | `docs/design/<id>-design.md` |
| 3 | Implementation plan | `superpowers:writing-plans` | `docs/plan/<id>-plan.md` |
| 4 | Implementation | `superpowers:subagent-driven-development` | source code changes |
| 5 | Code review | `superpowers:requesting-code-review` | `docs/review/<id>-review.md` |
| 6 | Testing | `superpowers:test-driven-development` | `docs/testing/<id>-testing.md` |
| 7 | Pull request | `superpowers:finishing-a-development-branch` | PR in Azure DevOps |

---

## Approval Gates

There are exactly two human approval checkpoints, plus optional clarification checkpoints.

**Gate 1 — Design**

After Stage 2, DevPilot pauses with:
> *"Design complete. Review at `docs/design/<id>-design.md`. Run `/dev-resume <id>` to approve and continue."*

Review the design document. When satisfied, run `/dev-resume <id>` to proceed to the implementation plan.

**Gate 2 — Implementation Plan**

After Stage 3, DevPilot pauses with:
> *"Implementation plan complete. Review at `docs/plan/<id>-plan.md`. Run `/dev-resume <id>` to approve and begin implementation."*

Review the plan. When satisfied, run `/dev-resume <id>` to kick off automated implementation through to PR.

**Clarification checkpoints (conditional)**

After brainstorming or writing-plans, if unresolved questions are found, DevPilot posts them as a numbered ADO comment with suggested answers and pauses:
> *"Design clarifications needed. Questions posted to ADO work item. Update the description with your decisions, then run `/dev-resume <id>`."*

Update the work item description in Azure DevOps with your answers, then run `/dev-resume <id>`. DevPilot re-fetches the updated description before continuing. If no questions arise, this checkpoint is skipped automatically.

---

## State File

DevPilot persists workflow state at `.devpilot/state/<workItemId>.json`:

```json
{
  "workItemId": 21238,
  "branch": "feature/21238-add-payment-gateway",
  "worktreePath": null,
  "adoOrg": "https://dev.azure.com/myorg",
  "adoProject": "MyProject",
  "adoRepo": "MyRepo",
  "status": "WAITING_FOR_DESIGN_APPROVAL",
  "designCompleted": true,
  "designApproved": false,
  "planCompleted": false,
  "planApproved": false,
  "implementationCompleted": false,
  "reviewCompleted": false,
  "testingCompleted": false,
  "prCreated": false,
  "prUrl": null,
  "pipelineFixCount": 0,
  "lastPipelineFixAt": null,
  "lastUpdated": "2026-05-30T10:00:00Z"
}
```

### Status values

| Status | Meaning |
|--------|---------|
| `DESIGNING` | Stage 2 in progress |
| `WAITING_FOR_DESIGN_CLARIFICATION` | Paused — questions posted to ADO, awaiting answers in work item description |
| `WAITING_FOR_DESIGN_APPROVAL` | Paused — awaiting `/dev-resume` to approve design |
| `PLANNING` | Stage 3 in progress |
| `WAITING_FOR_PLAN_CLARIFICATION` | Paused — questions posted to ADO, awaiting answers in work item description |
| `WAITING_FOR_PLAN_APPROVAL` | Paused — awaiting `/dev-resume` to approve plan |
| `IMPLEMENTING` | Stage 4 in progress |
| `REVIEWING` | Stage 5 in progress |
| `TESTING` | Stage 6 in progress |
| `CREATING_PR` | Stage 7 in progress |
| `COMPLETED` | Workflow finished, PR created |

---

## Repository Structure (target project)

Files DevPilot writes into the project you are working on:

```
.devpilot/
└── state/
    └── <workItemId>.json       ← workflow state

docs/
├── design/
│   └── <workItemId>-design.md
├── plan/
│   └── <workItemId>-plan.md
├── review/
│   └── <workItemId>-review.md
└── testing/
    └── <workItemId>-testing.md
```

All artifacts are committed to the feature branch as they are produced.

---

## Azure DevOps Integration

### Remote URL formats supported

DevPilot automatically parses your ADO connection from `git remote get-url origin`:

| Format | Example |
|--------|---------|
| HTTPS | `https://dev.azure.com/{org}/{project}/_git/{repo}` |
| SSH | `git@ssh.dev.azure.com:v3/{org}/{project}/{repo}` |
| Legacy | `https://{org}.visualstudio.com/{project}/_git/{repo}` |

### ADO operations

| Operation | When |
|-----------|------|
| `wit_get_work_item` | Stage 1 — fetch title, description, acceptance criteria |
| `wit_get_work_items_batch_by_ids` | Stage 1 — fetch linked work items |
| `wit_add_work_item_comment` | After every stage — progress trail |
| `repo_create_pull_request` | Stage 7 — create the PR |
| `pipelines_get_builds` | `/dev-fix-pipeline` — find latest build for the branch |
| `pipelines_list_runs` | `/dev-fix-pipeline` — fallback for YAML pipeline runs |
| `pipelines_get_build_log` | `/dev-fix-pipeline` — fetch build log |
| `pipelines_get_build_log_by_id` | `/dev-fix-pipeline` — drill into failed timeline records |

### Progress comments

Every stage posts a structured comment to your work item:

```
[DevPilot] Stage Completed: Design
Document: docs/design/21238-design.md
```

```
[DevPilot] Stage Completed: Pull Request
PR: https://dev.azure.com/myorg/MyProject/_git/MyRepo/pullrequest/42
```

```
[DevPilot] Design Clarifications Needed

The following questions need answers before the design can be finalised.
Suggested answers are provided — update the work item description with your
decisions, then run `/dev-resume 21238`.

1. Should the API be REST or GraphQL? (Suggested: REST — aligns with existing services)
2. ...
```

```
[DevPilot] Pipeline fix applied

Build: 1042 (20260605.3)
Error: Test assertion failure in PaymentServiceTests.ProcessRefund
Fix: Updated mock to return correct status code on partial refund
Files changed: src/PaymentService.cs, tests/PaymentServiceTests.cs
Verification: tests passed

The fix has been pushed to `feature/21238-add-payment-gateway`. The pipeline should re-run automatically.
```

---

## Resume Examples

### Scenario A — Resume after design approval

```
# Workflow paused at WAITING_FOR_DESIGN_APPROVAL
# You reviewed docs/design/21238-design.md and it looks good

/dev-resume 21238
# → Marks design approved, generates implementation plan, pauses again
```

### Scenario B — Resume after interruption mid-implementation

```
# Session ended while status was IMPLEMENTING

/dev-resume 21238
# → Detects IMPLEMENTING status, re-invokes subagent-driven-development with the plan
```

### Scenario C — Workflow already complete

```
/dev-resume 21238
# → "Work item 21238 workflow is already completed. PR: https://..."
```

### Scenario D — Resume after clarification checkpoint

```
# DevPilot posted questions to the ADO work item (WAITING_FOR_DESIGN_CLARIFICATION)
# You updated the work item description with your answers

/dev-resume 21238
# → Re-fetches updated work item, re-runs brainstorming with answers, continues
```

### Scenario E — Fix a failed pipeline

```
# PR is open, CI pipeline failed on the feature branch

/dev-fix-pipeline 21238
# → Fetches latest build, reads log, classifies failure, applies fix, verifies, pushes
```

---

## Git Branching

DevPilot creates and manages one branch per work item:

- **Branch name:** `feature/<workItemId>-<title-slug>`
  - Slug = first 5 words of the work item title, lowercased, hyphenated
  - Example: `feature/21238-add-payment-gateway-integration`
- **All artifact docs** are committed to this branch throughout the workflow
- **The PR** is raised from this branch to the repository's default branch

---

## Supported superpowers Skills

DevPilot requires the `superpowers` plugin and uses these skills:

| Skill | Stage |
|-------|-------|
| `superpowers:brainstorming` | Stage 2 — Design |
| `superpowers:writing-plans` | Stage 3 — Implementation plan |
| `superpowers:subagent-driven-development` | Stage 4 — Implementation |
| `superpowers:requesting-code-review` | Stage 5 — Code review |
| `superpowers:test-driven-development` | Stage 6 — Testing |
| `superpowers:finishing-a-development-branch` | Stage 7 — Pull request |
| `superpowers:systematic-debugging` | `/dev-fix-pipeline` — Pipeline failure fix |

---

## Non-Goals

- Does not replace Azure DevOps sprint/board management
- Does not auto-merge pull requests
- Does not replace human review of design and plan documents

---

## Future Enhancements

**Phase 2**
- Sprint-aware planning
- Multi-work-item orchestration
- Automatic dependency detection between work items
- Release note generation

**Phase 3**
- Deployment automation
- Environment validation
- Post-deployment verification
- Rollback recommendation generation

---

## Author

**Dragon** — tangphamtunglong1404@gmail.com

## License

MIT
