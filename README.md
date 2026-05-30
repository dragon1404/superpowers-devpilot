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
```

At each automated stage, DevPilot posts a `[DevPilot]` progress comment to your Azure DevOps work item, commits artifacts to a feature branch, and updates a local state file so the workflow can be resumed at any time.

---

## Requirements

| Dependency | Purpose | Install |
|---|---|---|
| [superpowers](https://github.com/obra/superpowers) | Provides the skills DevPilot orchestrates (brainstorming, TDD, code review, etc.) | `/plugin install superpowers@claude-plugins-official` |
| [Azure DevOps MCP](https://github.com/microsoft/azure-devops-mcp) | Connects DevPilot to your Azure DevOps org (work items, repos, PRs) | See below |
| Azure DevOps git remote | DevPilot parses your ADO org/project/repo from `origin` | — |

---

## Installation

### 1. Install this plugin

```bash
git clone https://github.com/<your-org>/superpowers-devpilot-plugin \
  ~/.claude/plugins/superpowers-devpilot
```

Or add it to your Claude Code `settings.json` plugins list by path.

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

**Azure DevOps MCP:**
```bash
npm install -g @microsoft/azure-devops-mcp
```

Add to your Claude Code MCP configuration:
```json
{
  "azure-devops": {
    "command": "azure-devops-mcp",
    "env": {
      "AZURE_DEVOPS_ORG_URL": "https://dev.azure.com/your-org",
      "AZURE_DEVOPS_AUTH_TYPE": "pat",
      "AZURE_DEVOPS_TOKEN": "your-personal-access-token"
    }
  }
}
```

Required PAT scopes: **Work Items** (Read & Write), **Code** (Read), **Pull Requests** (Read & Write).

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
5. Runs `superpowers:brainstorming` with the work item as requirements context
6. Saves the design to `docs/design/<workItemId>-design.md` and commits it
7. Posts `[DevPilot] Stage Completed: Design` to the work item
8. Pauses and prompts you to review the design

**Example:**
```
/dev-workitem 21238
```

---

### `/dev-resume <workItemId>`

Resumes the workflow from the last completed stage.

**At an approval gate** (`WAITING_FOR_DESIGN_APPROVAL` or `WAITING_FOR_PLAN_APPROVAL`), running `/dev-resume` **is** the approval — it advances the workflow automatically.

**After an interruption**, it detects the current state and picks up exactly where it left off.

**Example:**
```
/dev-resume 21238
```

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

There are exactly two human checkpoints:

**Gate 1 — Design**

After Stage 2, DevPilot pauses with:
> *"Design complete. Review at `docs/design/<id>-design.md`. Run `/dev-resume <id>` to approve and continue."*

Review the design document. When satisfied, run `/dev-resume <id>` to proceed to the implementation plan.

**Gate 2 — Implementation Plan**

After Stage 3, DevPilot pauses with:
> *"Implementation plan complete. Review at `docs/plan/<id>-plan.md`. Run `/dev-resume <id>` to approve and begin implementation."*

Review the plan. When satisfied, run `/dev-resume <id>` to kick off automated implementation through to PR.

---

## State File

DevPilot persists workflow state at `.devpilot/state/<workItemId>.json`:

```json
{
  "workItemId": 21238,
  "branch": "feature/21238-add-payment-gateway",
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
  "lastUpdated": "2026-05-30T10:00:00Z"
}
```

### Status values

| Status | Meaning |
|--------|---------|
| `DESIGNING` | Stage 2 in progress |
| `WAITING_FOR_DESIGN_APPROVAL` | Paused — awaiting `/dev-resume` |
| `PLANNING` | Stage 3 in progress |
| `WAITING_FOR_PLAN_APPROVAL` | Paused — awaiting `/dev-resume` |
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
