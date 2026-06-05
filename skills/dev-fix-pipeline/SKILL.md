---
name: dev-fix-pipeline
description: "Check a failed CI pipeline linked to a PR and automatically fix the errors. Usage: /dev-fix-pipeline <workItemId>"
---

# DevPilot: Fix Pipeline Failure

**Announce:** "Investigating pipeline failure for work item {workItemId}."

---

## Step 1 — Extract Work Item ID

Extract the workItemId from the user's message (the integer following `/dev-fix-pipeline`).

If no workItemId is provided, stop and say: "Please provide a work item ID. Usage: /dev-fix-pipeline <workItemId>"

## Step 2 — Read State File

Read `.devpilot/state/{workItemId}.json`.

If the file does not exist, stop and say: "No DevPilot workflow found for work item {workItemId}. Start one with `/dev-workitem {workItemId}`."

Store: `branch`, `adoOrg`, `adoProject`, `adoRepo`, `prUrl`.

If `prUrl` is null, stop and say: "No PR has been created for work item {workItemId} yet. The pipeline check requires an open PR. Run `/dev-resume {workItemId}` to complete the workflow first."

## Step 3 — Find the Failed Build

Call `mcp__azure-devops__pipelines_get_builds` with:
- project: {adoProject}
- branchName: `refs/heads/{branch}`
- statusFilter: `failed`
- top: 3

Find the most recent failed build. Store its `id` as `buildId` and `buildNumber`.

If no results are returned, try `mcp__azure-devops__pipelines_list_runs` for the same branch to check YAML pipeline runs. Look for any run with `result: failed`.

If no failed build is found at all, stop and say: "No failed builds found for branch `{branch}`. The pipeline may still be running, or it may have already passed."

## Step 4 — Get the Build Logs

Call `mcp__azure-devops__pipelines_get_build_log` with:
- project: {adoProject}
- buildId: {buildId}

Read the log and identify:
- The step or task that failed
- The specific error message(s)
- Any file paths and line numbers referenced in the error

If the top-level log lacks detail, call `mcp__azure-devops__pipelines_get_build_log_by_id` for each failed timeline record to get granular output.

Store a concise `errorSummary` (1–3 sentences describing what failed and why).

## Step 5 — Post ADO Comment: Investigating

Call `mcp__azure-devops__wit_add_work_item_comment` with:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Investigating pipeline failure

  Build: {buildId} ({buildNumber})
  Error: {errorSummary}
  ```

If the tool call fails, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post investigating comment — {error}. Continuing."

## Step 6 — Fix the Error

Invoke `superpowers:systematic-debugging` using the Skill tool, passing the following as context:

> **Pipeline failure on work item {workItemId}**
>
> **Branch:** {branch}
>
> **Build:** {buildId} ({buildNumber})
>
> **Error:**
> {errorSummary}
>
> **Relevant log output:**
> {key failing log lines, trimmed to what is actionable}
>
> Diagnose the root cause and fix it. Do not modify code unrelated to the failure.

## Step 7 — Commit and Push the Fix

Stage only the files that were modified during the fix:

```bash
git diff --name-only
```

Stage and commit those files:

```bash
git add <changed files>
git commit -m "fix: resolve pipeline failure for work item {workItemId}

{one-line description of what was fixed}"
git push
```

## Step 8 — Post ADO Comment: Fix Applied

Call `mcp__azure-devops__wit_add_work_item_comment` with:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Pipeline fix applied

  Error: {errorSummary}
  Fix: {brief description of the change}
  Branch: {branch}

  The fix has been pushed. The pipeline should re-run automatically on the PR.
  If it fails again, run `/dev-fix-pipeline {workItemId}` to retry.
  ```

If the tool call fails, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post fix comment — {error}. Continuing."

## Step 9 — Report to Developer

Tell the developer:

> Pipeline fix applied for work item {workItemId}.
>
> **Error:** {errorSummary}
> **Fix:** {brief description of the change}
>
> The fix has been pushed to `{branch}`. Monitor the pipeline to confirm it passes.
> If it fails again, run `/dev-fix-pipeline {workItemId}` to retry.
