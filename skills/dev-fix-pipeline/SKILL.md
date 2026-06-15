---
name: dev-fix-pipeline
description: "Investigate a failed CI pipeline on an open PR and automatically fix the error. Usage: /dev-fix-pipeline <workItemId>"
---

# DevPilot: Fix Pipeline Failure

**Announce:** "Investigating pipeline failure for work item {workItemId}."

---

## Step 1 — Extract Work Item ID

Extract the workItemId from the user's message (the integer following `/dev-fix-pipeline`).

If no workItemId is provided, stop and say: "Please provide a work item ID. Usage: /dev-fix-pipeline <workItemId>"

## Step 2 — Read State File

Read `.devpilot/state/{workItemId}.json`.

**If the file exists:** store all fields. Note `branch`, `adoOrg`, `adoProject`, `adoRepo`, `prCreated`, and `pipelineFixCount` (treat as 0 if missing). Set `stateFileExists = true`.

**If the file does not exist:** set `stateFileExists = false` and attempt to recover from ADO:

1. Call `mcp__azure-devops__wit_get_work_item` with:
   - id: {workItemId}
   - expand: "relations"

2. From `relations`, filter entries where `rel = "ArtifactLink"` AND `attributes.name = "Pull Request"`. Extract the PR number from each entry's `url` field (`vstfs:///Git/PullRequestId/{collectionId}/{projectId}/{prId}` — rightmost `/`-delimited integer). Store as `linkedPrIds`.

3. If no linked PR entries are found, stop and say:
   > "No DevPilot state file and no linked pull requests found for work item {workItemId}. Either start a workflow with `/dev-workitem {workItemId}` or link a PR to the work item in ADO."

4. Read `System.TeamProject` from the work item → store as `adoProject`.

5. Call `mcp__azure-devops__repo_list_pull_requests_by_repo_or_project` with:
   - project: {adoProject}
   - status: "Active"

   Keep only PRs whose `pullRequestId` is in `linkedPrIds`. If multiple, take the most recent by `creationDate`. If none, stop and say:
   > "Work item {workItemId} has linked PRs but none are currently active. Cannot determine the source branch."

6. From the chosen PR, store:
   - `branch` = `sourceRefName` with `refs/heads/` stripped
   - `adoRepo` = `repository.name`
   - `adoProject` = `repository.project.name` (overwrite if more specific than above)
   - `prCreated = true`
   - `pipelineFixCount = 0`

   Announce: "No local state file found — recovered branch `{branch}` from linked PR #{prId}."

## Step 3 — Precondition Checks

**Check 1 — PR must be open:**
If `prCreated` is not `true`, stop and say:
> "Work item {workItemId} does not have an open PR yet. `/dev-fix-pipeline` only works after the PR is created. Run `/dev-resume {workItemId}` to complete the workflow first."

**Check 2 — Branch match:**
Run:
```bash
git rev-parse --abbrev-ref HEAD
```
Store the result as `currentBranch`.

If `currentBranch` does not match `{branch}` from the state file, stop and say:
> "You are on branch `{currentBranch}` but work item {workItemId} is on `{branch}`. Switch to the correct branch before running this command."

## Step 4 — Find Latest Build

Call `mcp__azure-devops__pipelines_get_builds` with:
- project: {adoProject}
- branchName: `refs/heads/{branch}`
- top: 1

If no results are returned, call `mcp__azure-devops__pipelines_list_runs` for the same branch to cover YAML pipelines. Take the most recent run.

If still no results, stop and say: "No pipeline builds found for branch `{branch}`."

Store the result as `latestBuild`. Record its `id` as `buildId` and `buildNumber`.

## Step 5 — Check Build Status

Read the `status` and `result` fields of `latestBuild`:

- **Failed / partially succeeded** → continue to Step 6
- **In progress / queued / not started** → stop and say: "The latest build ({buildNumber}) is still running. Wait for it to complete, then retry."
- **Succeeded / any other non-failed status** → stop and say: "The latest build ({buildNumber}) passed. No pipeline failure found on branch `{branch}`."

## Step 6 — Get Build Log

Call `mcp__azure-devops__pipelines_get_build_log` with:
- project: {adoProject}
- buildId: {buildId}

If the top-level log does not contain actionable error detail, call `mcp__azure-devops__pipelines_get_build_log_by_id` for each failed timeline record to get granular output.

Extract and store:
- `failingStep` — name of the step or task that failed
- `errorMessages` — the specific error message(s)
- `errorLocations` — file paths and line numbers referenced in the error (if any)
- `errorSummary` — 1–3 sentence summary of what failed and why

## Step 7 — Classify Failure

Classify using the extracted log details:

**Automatable** — the error points directly at code that can be changed:
- Compile error (syntax error, type mismatch, missing import)
- Test assertion failure
- Lint or formatting violation
- Missing file referenced in code

**Not automatable** — the error is environmental or infrastructural:
- Missing secret or environment variable
- Pipeline agent offline or unavailable
- Network timeout or DNS failure
- Permission denied on infrastructure resource
- External dependency down

If **not automatable**, call `mcp__azure-devops__wit_add_work_item_comment` with:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Pipeline failure requires manual intervention

  Build: {buildId} ({buildNumber})
  Step: {failingStep}
  Error: {errorSummary}

  This failure cannot be automatically fixed (infrastructure/configuration issue).
  ```

If the tool call fails, print a warning: "⚠️ [DevPilot] Warning: Could not post 'manual intervention' comment to work item {workItemId} — {error}." Then stop and tell the developer what was found.

If **automatable**, continue to Step 8.

## Step 8 — Fix with systematic-debugging

Invoke `superpowers:systematic-debugging` using the Skill tool, passing:

> **Pipeline failure on work item {workItemId}**
>
> **Branch:** {branch}
>
> **Build:** {buildId} ({buildNumber})
>
> **Failing step:** {failingStep}
>
> **Error:**
> {errorSummary}
>
> **Relevant log output:**
> {errorMessages and errorLocations, trimmed to what is actionable}
>
> Diagnose the root cause and fix it. Touch only files directly related to the failure. Do not modify unrelated code.

## Step 9 — Check for Changes

After the debugging skill completes, run:
```bash
git diff --name-only
```

If the output is empty (no files were modified), call `mcp__azure-devops__wit_add_work_item_comment` with:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Pipeline fix attempted but no change could be determined

  Build: {buildId} ({buildNumber})
  Error: {errorSummary}

  The failure was identified as automatable but no code change could be determined. Manual investigation required.
  ```

If the tool call fails, print a warning: `⚠️ [DevPilot] Warning: Could not post 'no change' comment to work item {workItemId} — {error}.` Then stop and tell the developer.

If files were modified, store the list as `changedFiles` and continue to Step 10.

## Step 10 — Local Verification

**Step 10a — Attempt test run:**

Identify test files related to `changedFiles` (same module, same directory, or files whose names include the changed file's base name). Run those tests.

If no related test files can be identified, skip the test run and set verification mode to `build-only`.

Evaluate the result:
- **Tests pass** → proceed to Step 10b
- **Tests fail with sandbox restrictions** (connection refused, network unreachable, service unavailable, external environment variable missing) → skip to Step 10b; note verification mode as `build-only`
- **Tests fail with actual failures** (assertion errors, wrong output, unexpected business-logic exceptions) → call `mcp__azure-devops__wit_add_work_item_comment`:
  ```
  [DevPilot] Pipeline fix attempted but local verification failed

  Build: {buildId} ({buildNumber})
  Files changed: {changedFiles}
  Verification failure: tests failed locally after the fix
  Details: {test failure output}
  ```
  If the tool call fails, warn and continue. Then stop and tell the developer.

**Step 10b — Build check:**

Run the project build command.

If no build command can be determined for this project type (no package.json, Makefile, .csproj, or equivalent), skip the build check and treat verification as passing — proceed to Step 11.

- **Build passes** → continue to Step 11
- **Build fails** → call `mcp__azure-devops__wit_add_work_item_comment`:
  ```
  [DevPilot] Pipeline fix attempted but local verification failed

  Build: {buildId} ({buildNumber})
  Files changed: {changedFiles}
  Verification failure: build failed locally after the fix
  Details: {build error output}
  ```
  If the tool call fails, warn and continue. Then stop and tell the developer.

## Step 11 — Push the Fix

Stage and commit only `changedFiles`:

Stage each file in `changedFiles` individually. For example, if `changedFiles` contains `src/foo.ts` and `src/bar.ts`, run:
```bash
git add src/foo.ts src/bar.ts
```
(Expand `changedFiles` as space-separated arguments to `git add`.)

```bash
git commit -m "fix: resolve pipeline failure for work item {workItemId} — {one-line summary of what was fixed}"
git push
```

## Step 12 — Update State File

If `stateFileExists = false`, skip this step entirely — there is no state file to update.

Otherwise, read `.devpilot/state/{workItemId}.json`. Update:
- `pipelineFixCount`: increment by 1 (set to 1 if field was missing)
- `lastPipelineFixAt`: current ISO 8601 timestamp
- `lastUpdated`: current ISO 8601 timestamp

Update your in-memory `pipelineFixCount` to this new value before continuing — subsequent steps use the updated count.

Write the updated state file back, then commit and push:

```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — pipeline fix #{pipelineFixCount} applied for {workItemId}"
git push
```

## Step 13 — Post ADO Comment: Fix Applied

Determine the verification label:
- Tests ran and passed → `tests passed`
- Tests sandbox-skipped → `build-only (tests skipped — sandbox restrictions)`

Call `mcp__azure-devops__wit_add_work_item_comment` with:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Pipeline fix applied

  Build: {buildId} ({buildNumber})
  Error: {errorSummary}
  Fix: {brief description of what changed}
  Files changed: {changedFiles}
  Verification: {verification label from Step 13}

  The fix has been pushed to `{branch}`. The pipeline should re-run automatically.
  If it fails again, run `/dev-fix-pipeline {workItemId}` to retry.
  ```

If the tool call fails, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post fix comment — {error}. Continuing."

## Step 14 — Report to Developer

Tell the developer:

> Pipeline fix #{pipelineFixCount} applied for work item {workItemId}.
>
> **Error:** {errorSummary}
> **Files changed:** {changedFiles}
> **Verification:** {verification label from Step 13}
>
> The fix has been pushed to `{branch}`. Monitor the pipeline to confirm it passes.
> If it fails again, run `/dev-fix-pipeline {workItemId}` to retry.
