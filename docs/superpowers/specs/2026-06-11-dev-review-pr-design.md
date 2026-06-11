# Design: `/dev-review-pr` Skill

**Date:** 2026-06-11
**Status:** Draft

---

## Overview

A standalone DevPilot skill that reviews any Azure DevOps pull request and posts findings as inline PR thread comments. Given either a full ADO PR URL or a work item ID, it locates the PR, fetches the diff via ADO MCP tools, reviews the code at "high" effort, and creates a PR thread for each finding — mirroring what a human code reviewer would do.

It is independent of any DevPilot workflow state file. Any ADO PR can be reviewed, not just ones created by `/dev-workitem`.

---

## Command

```
/dev-review-pr <prUrl | workItemId>
```

**Input — two accepted forms:**

**Form 1 — PR URL.** A full ADO PR URL:
- `https://dev.azure.com/{org}/{project}/_git/{repo}/pullrequest/{prId}`
- `https://{org}.visualstudio.com/{project}/_git/{repo}/pullrequest/{prId}`

Parsed fields: `adoOrg`, `adoProject`, `adoRepo`, `prId`.

**Form 2 — Work item ID.** A plain integer (e.g. `42138`). The skill resolves it to the latest linked active PR via ADO MCP tools (see Step 1b).

If the argument is neither a valid ADO PR URL nor a plain integer, stop immediately with:
> "Invalid input. Provide a full ADO PR URL or a work item ID (integer). Example: `/dev-review-pr 42138` or `/dev-review-pr https://dev.azure.com/org/project/_git/repo/pullrequest/123`"

---

## Review Modes

Before fetching anything, the skill prompts the developer:

> "How would you like to review this PR?
> **[A] Pure ADO** (recommended) — fetches diff via API, works from any directory
> **[L] Local checkout** — checks out the branch locally, more powerful but requires a clean working tree
>
> Press Enter to accept [A]."

### Mode A — Pure ADO (default)
Fetches all diff content via ADO MCP tools. No git operations. Works from any directory.

### Mode L — Local Checkout
Runs `git status` to verify a clean working tree (stops with instructions if dirty). Checks out the PR's source branch. Invokes the built-in `code-review` CLI skill on the local diff. Findings are parsed into the shared finding structure before posting.

Both modes converge at the same posting step — thread format is identical.

---

## Step-by-Step Flow

### Step 1 — Detect Input Type

Determine whether the argument is a PR URL (contains `pullrequest/`) or a work item ID (plain integer).

- **PR URL path:** extract `adoOrg`, `adoProject`, `adoRepo`, `prId` directly from the URL. Skip Step 1b.
- **Work item ID path:** store the integer as `workItemId`. Proceed to Step 1b.

### Step 1b — Resolve Work Item ID to PR (work item path only)

1. Call `wit_get_work_item` with `workItemId`. Read:
   - `System.TeamProject` → `adoProject`
   - `relations` array → filter for entries where `rel = "ArtifactLink"` and `attributes.name = "Pull Request"`

2. Extract the PR number from each matching relation's `url` field. ADO artifact link URLs end with the numeric PR ID as the last path segment (e.g. `vstfs:///Git/PullRequestId/{collectionId}/{projectId}/{prId}`).

3. If no PR links are found, stop and say:
   > "Work item {workItemId} has no linked pull requests. Link a PR to the work item in ADO first, or provide the PR URL directly."

4. Call `repo_list_pull_requests_by_repo_or_project` scoped to `adoProject` to retrieve full PR details. Match the returned PRs against the extracted PR IDs.

5. From matching PRs, keep only those with `status: active`. If none are active, stop and say:
   > "Work item {workItemId} has linked PRs but none are currently active ({list of PR IDs with their statuses})."

6. If multiple active PRs are found, take the one with the most recent `creationDate`. Store its `pullRequestId` as `prId`, `repository.name` as `adoRepo`, and construct the full URL as `prUrl`.

7. Announce: "Found PR #{prId}: {title} — proceeding with review."

### Step 2 — Prompt Review Mode

Ask the developer to choose Mode A or Mode L (default: A). Record the choice as `reviewMode`.

### Step 3 — Fetch PR Metadata

Call `repo_get_pull_request_by_id`:
- Extract: PR `title`, `description`, `sourceBranch`, `targetBranch`, `status`
- If `status` is not active (i.e., abandoned or completed), stop and say:
  > "PR #{prId} is {status}. Only active PRs can be reviewed."

### Step 4 — Fetch Diff (Mode A only)

1. Call `repo_get_pull_request_changes` to get the list of changed files with their `changeType` (add / edit / delete).
2. **Cap at 30 files.** If more, warn: "PR has {total} changed files — reviewing first 30 (edits first, then adds, then deletes)."
3. Skip binary files (detected by extension: `.png`, `.jpg`, `.gif`, `.pdf`, `.zip`, `.exe`, `.dll`, `.wasm`, and similar non-text formats).
4. For each file in scope, call `repo_get_file_content` twice:
   - Base version: `targetBranch`
   - Head version: `sourceBranch`
5. Compute a unified diff for each file (base → head). Store as in-memory diff representation.

For **Mode L**: skip Step 4 — diff is produced by the local `code-review` skill.

### Step 5 — Code Review

**Mode A:** Claude reviews the full diff inline at "high" effort, focusing on:
- Correctness bugs (logic errors, off-by-one, null dereferences, race conditions)
- Security issues (injection, secret exposure, missing auth checks)
- Breaking API or contract changes
- Missing or incorrect error handling

**Mode L:** Invoke the built-in `code-review` skill on the checked-out branch diff. The skill produces a structured list of findings in its response — extract each finding's file path, line number (if present), severity label, and description from that output and map them into the shared finding structure. If a finding lacks a line number, post it as a file-level thread (omit `rightFileStart`/`rightFileEnd` from `threadContext`).

Each finding is recorded with:
- `filePath` — file the issue is in
- `lineNumber` — specific line in the head (source branch) version
- `severity` — `Critical` | `Important` | `Minor`
- `title` — one-line description
- `detail` — explanation of the issue
- `suggestion` — concrete fix recommendation (when determinable)

**Minor findings are grouped by file** — all minor issues in the same file are batched into one thread to reduce PR noise.

### Step 6 — Post Findings as PR Threads

For each **Critical** and **Important** finding, call `repo_create_pull_request_thread` with:

```
repositoryId: {adoRepo}
pullRequestId: {prId}
threadContext: {
  filePath: {filePath},
  rightFileStart: { line: lineNumber },
  rightFileEnd:   { line: lineNumber }
}
comments: [{
  content: "[DevPilot Review] {Severity}: {title}\n\n{detail}\n\n💡 {suggestion}"
}]
```

For each file with **Minor** findings, post one batched thread:
```
[DevPilot Review] Minor issues in {filePath}

{bullet list of minor findings with line numbers}
```

If any `repo_create_pull_request_thread` call fails, log a warning and continue — do not abort the full review.

### Step 7 — Post Summary Thread

After all finding threads are posted, call `repo_create_pull_request_thread` with no `threadContext` (top-level) and comment:

```
[DevPilot Review] Summary

PR: {title}
Files reviewed: {n} reviewed{" (first 30 of {total})" if capped}
Findings: {x} Critical · {y} Important · {z} Minor

Overall: {one-sentence assessment}
```

Assessment examples:
- "No issues found — looks good to merge."
- "Minor issues only — safe to merge after review."
- "Important issues found — fix before merging."
- "Critical issues found — do not merge until resolved."

### Step 8 — Report to Developer

Print to terminal:

```
DevPilot review complete for PR #{prId}: {title}

Findings posted as inline PR comments:
  Critical:  {x}
  Important: {y}
  Minor:     {z} (batched by file)
  Files reviewed: {n}{" of first 30" if capped}

PR: {prUrl}
```

If zero findings across all severities, report "No issues found at high effort." and confirm the summary thread was posted.

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| Invalid input (neither URL nor integer) | Stop immediately with format error |
| Work item has no linked PR artifact links | Stop with instructions to link a PR |
| Work item has linked PRs but none are active | Stop listing PR IDs and statuses |
| PR is not active (abandoned/completed) | Stop with status message |
| `repo_get_pull_request_changes` fails | Stop with error — cannot proceed without file list |
| `repo_get_file_content` fails for a file | Skip that file, warn, continue |
| `repo_create_pull_request_thread` fails for a finding | Log warning, continue to next finding |
| Local checkout fails (Mode L, dirty working tree) | Stop with instructions to stash changes |
| Local checkout fails (other reason) | Stop with error, suggest Mode A |

---

## Constraints

- Maximum 30 files reviewed per PR (edits prioritised over adds/deletes)
- Binary files are skipped automatically
- Review effort is fixed at "high" — not user-configurable
- Skill has no state file — it is stateless and idempotent (re-running posts duplicate threads; the developer can dismiss them)

---

## Out of Scope

- Re-reviewing only changed files since last run (no incremental review)
- Approving or requesting changes on the PR via ADO vote API
- Reviewing draft PRs differently from active ones
- Per-language review rules or custom rule sets
