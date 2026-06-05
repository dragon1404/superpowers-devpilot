# Design: `/dev-fix-pipeline` Skill

**Date:** 2026-06-05
**Plugin:** superpowers-devpilot
**Command:** `/dev-fix-pipeline <workItemId>`

---

## Overview

A new DevPilot skill that investigates a failed CI pipeline on an open PR and automatically applies a fix. The skill only operates after the PR has been created (`prCreated: true` in the state file). It fetches the latest build, retrieves the log, classifies the failure as automatable or not, attempts an automated fix via `systematic-debugging`, verifies locally (tests first, build as fallback), and pushes only when verification passes.

---

## Preconditions

All must pass before any action is taken:

1. `.devpilot/state/{workItemId}.json` exists
2. `prCreated` is `true` in the state file
3. Current branch matches the `branch` field in the state file

If any precondition fails, stop with a clear message — no changes made.

---

## Stages

### Stage 1 — Build Discovery

1. Call `mcp__azure-devops__pipelines_get_builds` for `refs/heads/{branch}`, top 1 (latest build only)
2. If no results, fall back to `mcp__azure-devops__pipelines_list_runs` for the same branch (YAML pipelines)
3. Check the status of the latest build/run:
   - **Failed** → continue to Stage 2
   - **In progress / queued** → stop, tell developer to wait for the build to complete
   - **Passed / other** → stop, tell developer no failed build was found

### Stage 2 — Log Retrieval

1. Call `mcp__azure-devops__pipelines_get_build_log` with the build ID
2. If the top-level log lacks actionable detail, call `mcp__azure-devops__pipelines_get_build_log_by_id` for failed timeline records to get granular output
3. Extract: failing step name, error message(s), referenced file paths and line numbers

### Stage 3 — Failure Classification

Classify the failure into one of two buckets:

| Class | Examples |
|---|---|
| **Automatable** | Compile error, test failure, lint/format violation — clear error message pointing at code |
| **Not automatable** | Missing secret or env var, pipeline agent offline, network timeout, infra permission denied |

If **not automatable**: post the full error analysis as an ADO work item comment and stop. Do not attempt a fix.

### Stage 4 — Fix

Invoke `superpowers:systematic-debugging` with:
- The error summary and relevant log lines as context
- Branch and work item ID for reference
- Explicit instruction: touch only files directly related to the failure

After the skill completes, run `git diff --name-only`. If no files were modified, systematic-debugging could not determine a fix — post findings to ADO and stop without pushing.

### Stage 5 — Local Verification

1. **Test run:** Identify test files related to changed files and run them.
   - If tests fail due to **sandbox restrictions** (network blocked, external service unreachable, setup exits with a connection/environment error rather than a test assertion failure) → skip to step 2
   - If tests fail due to **actual test failures** → do not push, report to developer and stop
2. **Build check:** Run the project build command.
   - If build fails → do not push, report to developer and stop
3. Proceed to Stage 6 only when build passes (tests may have been sandbox-skipped).

### Stage 6 — Push & Reporting

1. Stage only changed files (from `git diff --name-only`)
2. Commit: `fix: resolve pipeline failure for work item {workItemId} — {one-line summary of what was fixed}`
3. Push to the feature branch
4. Post ADO comment (see ADO Comments section below)
5. Update state file: increment `pipelineFixCount`, set `lastPipelineFixAt` to current ISO 8601 timestamp
6. Tell the developer: fix pushed, monitor the pipeline; if it fails again, run `/dev-fix-pipeline {workItemId}` to retry

---

## State File Changes

Two new fields added to `.devpilot/state/{workItemId}.json`:

```json
{
  "pipelineFixCount": 0,
  "lastPipelineFixAt": null
}
```

`pipelineFixCount` is incremented on each successful push. `lastPipelineFixAt` is set to the timestamp of the most recent fix push.

---

## ADO Comments

| Event | Comment |
|---|---|
| Not automatable failure | `[DevPilot] Pipeline failure requires manual intervention — {error analysis}` |
| No files changed by debugging | `[DevPilot] Pipeline fix attempted but no change could be determined — {error summary}` |
| Verification failed | `[DevPilot] Pipeline fix attempted but local verification failed — {details}` |
| Fix pushed successfully | `[DevPilot] Pipeline fix applied — Error: {summary} / Fix: {description} / Verification: {tests passed \| build-only}` |

---

## Error Handling Summary

| Scenario | Behaviour |
|---|---|
| No build found | Stop — inform developer, no action |
| Build still running | Stop — tell developer to wait |
| Build passed | Stop — no fix needed |
| Not automatable failure | Post analysis to ADO, stop |
| Debugging makes no changes | Post findings to ADO, stop |
| Local tests fail (actual failure) | Post details to ADO, stop without pushing |
| Local build fails | Post details to ADO, stop without pushing |
| ADO comment tool call fails | Warn and continue — do not block workflow |
