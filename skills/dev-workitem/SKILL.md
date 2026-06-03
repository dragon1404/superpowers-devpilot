---
name: dev-workitem
description: "Start a DevPilot automated delivery workflow for an Azure DevOps work item. Usage: /dev-workitem <workItemId>"
---

# DevPilot: Start Workflow

**Announce:** "Starting DevPilot workflow for work item {workItemId}."

## Step 1 — Extract Work Item ID

Extract the workItemId from the user's message (the integer following `/dev-workitem`).

If no workItemId is provided, stop and say: "Please provide a work item ID. Usage: /dev-workitem <workItemId>"

## Step 1b — Check Prerequisites

Before doing anything else, verify both required dependencies are available.

**Check 1 — superpowers plugin:**

Check whether the superpowers skills directory exists at:
`~/.claude/plugins/cache/claude-plugins-official/superpowers/`

Run:
```bash
ls ~/.claude/plugins/cache/claude-plugins-official/superpowers/ 2>/dev/null && echo "FOUND" || echo "MISSING"
```

**Check 2 — Azure DevOps MCP:**

Check whether `mcp__azure-devops__wit_get_work_item` is available as a tool in the current session. If Azure DevOps MCP tools are listed in your available tools, the server is configured. If not, it is missing.

**If either dependency is missing**, stop and say:

> **DevPilot prerequisites not met.**
>
> The following are required before using DevPilot:
>
> | Dependency | Status | Install |
> |---|---|---|
> | superpowers plugin | {FOUND/MISSING} | https://github.com/obra/superpowers |
> | Azure DevOps MCP | {FOUND/MISSING} | https://github.com/microsoft/azure-devops-mcp |
>
> Run `/dev-setup` for a guided installation walkthrough.

If both dependencies are present, continue to Step 2.

## Step 2 — Check for Existing Workflow

Check if `.devpilot/state/{workItemId}.json` exists in the current working directory.

If it exists, stop and say: "A DevPilot workflow for work item {workItemId} already exists. Use `/dev-resume {workItemId}` to continue."

## Step 3 — Parse ADO Connection from Git Remote

Run: `git remote get-url origin`

Parse the Azure DevOps connection details from the URL using these patterns:

- `https://dev.azure.com/{org}/{project}/_git/{repo}` → adoOrg = `https://dev.azure.com/{org}`
- `git@ssh.dev.azure.com:v3/{org}/{project}/{repo}` → adoOrg = `https://dev.azure.com/{org}`
- `https://{org}.visualstudio.com/{project}/_git/{repo}` → adoOrg = `https://{org}.visualstudio.com`

Extract and store: `adoOrg`, `adoProject`, `adoRepo`.

If the command fails (non-zero exit code or no output), stop and say: "Could not read a git remote named 'origin'. Please ensure this repository has an Azure DevOps origin configured."

If the remote URL does not match any Azure DevOps pattern, stop and say: "This repository does not appear to be hosted on Azure DevOps. DevPilot requires an Azure DevOps git remote."

## Step 4 — Fetch Work Item

Call `mcp__azure-devops__wit_get_work_item` with:
- id: {workItemId}

Read and note:
- `System.Title` → title
- `System.Description` → description
- `Microsoft.VSTS.Common.AcceptanceCriteria` → acceptanceCriteria
- `System.WorkItemType` → workItemType
- `System.State` → state

If the work item has relations listed, call `mcp__azure-devops__wit_get_work_items_batch_by_ids` for any linked work item IDs to gather additional context.

## Step 5 — Create Feature Branch

Generate a slug from the work item title:
- Take the first 5 words
- Lowercase all characters
- Replace spaces with hyphens
- Remove non-alphanumeric characters except hyphens

Branch name: `feature/{workItemId}-{slug}`

Run:
```bash
git checkout -b feature/{workItemId}-{slug}
```

If branch already exists (exit code non-zero), run:
```bash
git checkout feature/{workItemId}-{slug}
```

## Step 6 — Initialize State File

Create directory: `.devpilot/state/`

Write `.devpilot/state/{workItemId}.json` with this exact structure (substituting real values):

```json
{
  "workItemId": 0,
  "branch": "feature/0-slug",
  "adoOrg": "https://dev.azure.com/org",
  "adoProject": "Project",
  "adoRepo": "Repo",
  "status": "DESIGNING",
  "designCompleted": false,
  "designApproved": false,
  "planCompleted": false,
  "planApproved": false,
  "implementationCompleted": false,
  "reviewCompleted": false,
  "testingCompleted": false,
  "prCreated": false,
  "prUrl": null,
  "lastUpdated": "2026-01-01T00:00:00Z"
}
```

Replace `0`, `"feature/0-slug"`, org/project/repo, and timestamp with the actual values. Use ISO 8601 format for `lastUpdated`.

## Step 7 — Post ADO Comment: Design Started

Call `mcp__azure-devops__wit_add_work_item_comment` with:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Started: Design
  ```

If the tool call fails or errors, print a warning and continue:
> "⚠️ [DevPilot] Warning: Could not post 'Design Started' comment to work item {workItemId} — {error}. Workflow will continue; fix the Azure DevOps MCP connection when convenient."

## Step 8 — Verify State File Written

Confirm that `.devpilot/state/{workItemId}.json` was written successfully by checking it exists. The file was written with `status: "DESIGNING"` in Step 6.

## Step 9 — Run Design Stage

Invoke `superpowers:brainstorming` using the Skill tool, passing the following block as the `args` parameter:

> **Work Item {workItemId}: {title}**
>
> **Description:**
> {description}
>
> **Acceptance Criteria:**
> {acceptanceCriteria}
>
> **Instructions for brainstorming:** Treat this work item as the feature requirement. The design document must include: Work Item ID and title, assumptions, impacted modules, database impact, API impact, and testing impact. Save the design to `docs/design/{workItemId}-design.md`.

The brainstorming skill will run its interactive process. After it completes, verify that `docs/design/{workItemId}-design.md` exists.

## Step 9b — Clarification Check: Design

Review `docs/design/{workItemId}-design.md` for any open questions or unresolved assumptions that require a developer decision before the design can be finalised.

If you identify any questions:

1. Format them as a numbered list. For each question, include your own suggested answer.
2. Commit the draft design:
   ```bash
   git add docs/design/{workItemId}-design.md
   git commit -m "docs: add draft design document for work item {workItemId}"
   ```
3. Call `mcp__azure-devops__wit_add_work_item_comment` with:
   - workItemId: {workItemId}
   - comment:
     ```
     [DevPilot] Design Clarifications Needed

     The following questions need answers before the design can be finalised. Suggested answers are provided — update the work item description with your decisions, then run `/dev-resume {workItemId}`.

     {numbered list of questions with suggested answers}
     ```
4. Update `.devpilot/state/{workItemId}.json`:
   - Set `status` to `"WAITING_FOR_DESIGN_CLARIFICATION"`
   - Set `lastUpdated` to current ISO 8601 timestamp
5. Commit:
   ```bash
   git add .devpilot/state/{workItemId}.json
   git commit -m "chore: devpilot state — waiting for design clarification on {workItemId}"
   ```
6. Tell the developer:
   > Design clarifications needed for work item {workItemId}.
   >
   > Questions have been posted as a comment on the ADO work item. Update the work item description with your decisions, then run `/dev-resume {workItemId}` to continue.

   **STOP.**

If you have no open questions, continue to Step 10.

## Step 10 — Commit Design Document

```bash
git add docs/design/{workItemId}-design.md
git commit -m "docs: add design document for work item {workItemId}"
```

## Step 11 — Post ADO Comment: Design Completed

Call `mcp__azure-devops__wit_add_work_item_comment` with:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Completed: Design
  Document: docs/design/{workItemId}-design.md
  ```

If the tool call fails or errors, print a warning and continue:
> "⚠️ [DevPilot] Warning: Could not post 'Design Completed' comment to work item {workItemId} — {error}. Workflow will continue; fix the Azure DevOps MCP connection when convenient."

## Step 12 — Update State and Pause for Approval

Update `.devpilot/state/{workItemId}.json`:
- Set `designCompleted` to `true`
- Set `status` to `"WAITING_FOR_DESIGN_APPROVAL"`
- Set `lastUpdated` to current ISO 8601 timestamp

Commit the updated state file:
```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — waiting for design approval on {workItemId}"
```

**Tell the developer:**

> Design complete for work item {workItemId}.
>
> Review the design document at: `docs/design/{workItemId}-design.md`
>
> When you are satisfied with the design, run `/dev-resume {workItemId}` to approve it and continue to the Implementation Plan stage.
