---
name: dev-review-pr
description: "Review an Azure DevOps pull request and post findings as inline PR comments. Usage: /dev-review-pr <prUrl | workItemId>"
---

# DevPilot: Review Pull Request

**Announce:** "Starting PR review for {arg}."

---

## Step 1 — Detect Input Type

Extract the argument from the user's message (everything after `/dev-review-pr`). Trim whitespace.

- If the argument contains `pullrequest/` → it is a PR URL. Parse using these patterns:
  - `https://dev.azure.com/{org}/{project}/_git/{repo}/pullrequest/{prId}` → `adoOrg = https://dev.azure.com/{org}`
  - `https://{org}.visualstudio.com/{project}/_git/{repo}/pullrequest/{prId}` → `adoOrg = https://{org}.visualstudio.com`
  - Extract and store: `adoOrg`, `adoProject`, `adoRepo`, `prId` (integer at the end of the URL)
  - Skip Step 1b. Proceed to Step 2.

- If the argument is a plain integer (digits only) → treat it as a work item ID. Store as `workItemId`. Proceed to Step 1b.

- If neither condition applies → stop and say:
  > "Invalid input. Provide a full ADO PR URL or a work item ID (integer).
  > Examples:
  > - `/dev-review-pr 42138`
  > - `/dev-review-pr https://dev.azure.com/org/project/_git/repo/pullrequest/123`"

## Step 1b — Resolve Work Item ID to PR (work item path only)

Call `mcp__azure-devops__wit_get_work_item` with:
- id: {workItemId}

Read:
- `System.TeamProject` → store as `adoProject`
- `relations` array → filter entries where `rel = "ArtifactLink"` AND `attributes.name = "Pull Request"`

If no such entries are found, stop and say:
> "Work item {workItemId} has no linked pull requests. Link a PR to the work item in ADO first, or provide the PR URL directly: `/dev-review-pr <prUrl>`"

For each PR artifact link entry, extract the PR number from its `url` field. The URL has the format `vstfs:///Git/PullRequestId/{collectionId}/{projectId}/{prId}` — the PR number is the last path segment (the rightmost `/`-delimited integer). Collect all extracted PR numbers as `linkedPrIds`.

Call `mcp__azure-devops__repo_list_pull_requests_by_repo_or_project` with:
- project: {adoProject}

From the returned list, keep only PRs whose `pullRequestId` is in `linkedPrIds` AND whose `status` is `"active"`.

If no active PRs remain, stop and say:
> "Work item {workItemId} has linked PRs but none are currently active. PR statuses: {list each linked prId with its status}."

If multiple active PRs remain, take the one with the most recent `creationDate`.

Store:
- `prId` = `pullRequestId` of the chosen PR
- `adoRepo` = `repository.name` of the chosen PR
- `prUrl` = `repository.remoteUrl` + `/pullrequest/` + `prId`

Announce: "Found PR #{prId}: {title} — proceeding with review."

Proceed to Step 2.

## Step 2 — Prompt Review Mode

Ask the developer:

> "How would you like to review this PR?
> **[A] Pure ADO** (recommended) — fetches diff via API, works from any directory
> **[L] Local checkout** — checks out the branch locally, more powerful but requires a clean working tree
>
> Press Enter or type A to use Pure ADO."

Wait for a response. If the response is empty or `A` (case-insensitive), set `reviewMode = "A"`. If `L` (case-insensitive), set `reviewMode = "L"`.

## Step 3 — Fetch PR Metadata

Call `mcp__azure-devops__repo_get_pull_request_by_id` with:
- repositoryId: {adoRepo}
- pullRequestId: {prId}
- project: {adoProject}

Read and store:
- `title` → PR title
- `description` → PR description  
- `sourceRefName` → `sourceBranch` (e.g. `refs/heads/feature/123-my-feature`)
- `targetRefName` → `targetBranch` (e.g. `refs/heads/main`)
- `status`

If `status` is not `"active"`, stop and say:
> "PR #{prId} is {status}. Only active PRs can be reviewed."

## Step 4 — Fetch Diff (Mode A only)

*Skip this entire step if `reviewMode = "L"`. Proceed to Step 5.*

### Step 4a — Get Changed Files

Call `mcp__azure-devops__repo_get_pull_request_changes` with:
- repositoryId: {adoRepo}
- pullRequestId: {prId}
- project: {adoProject}

If the call fails, stop and say: "Could not fetch changed files for PR #{prId} — {error}. Cannot proceed without the file list."

From the returned `changes` array, extract each entry's `item.path` and `changeType` (values: `add`, `edit`, `delete`). Ignore entries where `item.isFolder = true`.

Remove entries whose path ends with any of these extensions (binary files):
`.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.ico`, `.pdf`, `.zip`, `.tar`, `.gz`, `.exe`, `.dll`, `.so`, `.dylib`, `.wasm`, `.mp4`, `.mp3`, `.ttf`, `.woff`, `.woff2`

Sort the remaining entries: edits first, then adds, then deletes.

If more than 30 entries remain, warn:
> "⚠️ PR has {total} changed files — reviewing first 30 (edits first, then adds, then deletes)."

Take only the first 30 entries. Store as `filesToReview`.

### Step 4b — Fetch File Contents

For each file in `filesToReview`, fetch file content as follows:

**Strip the `refs/heads/` prefix** from `sourceBranch` and `targetBranch` before using them as version descriptors. Example: `refs/heads/feature/123-foo` → `feature/123-foo`.

For files with `changeType = "add"`:
- Fetch head only. Set base content to empty string `""`.
- Call `mcp__azure-devops__repo_get_file_content` with:
  - repositoryId: {adoRepo}, project: {adoProject}, path: {file.path}
  - versionDescriptor: `{ "versionType": "branch", "version": "{sourceBranch}" }`

For files with `changeType = "delete"`:
- Fetch base only. Set head content to empty string `""`.
- Call `mcp__azure-devops__repo_get_file_content` with:
  - repositoryId: {adoRepo}, project: {adoProject}, path: {file.path}
  - versionDescriptor: `{ "versionType": "branch", "version": "{targetBranch}" }`

For files with `changeType = "edit"`:
- Fetch both versions.
- Head: versionDescriptor `{ "versionType": "branch", "version": "{sourceBranch}" }`
- Base: versionDescriptor `{ "versionType": "branch", "version": "{targetBranch}" }`

If a fetch fails for any file, warn: "⚠️ Could not fetch {file.path} — skipping." Remove it from `filesToReview` and continue.

Store collected results as `fileDiffs`: an array of `{ path, baseContent, headContent, changeType }`.

## Step 5 — Code Review

### Step 5A — Inline Review (Mode A)

*Skip this section if `reviewMode = "L"`. Jump to Step 5L.*

Review all entries in `fileDiffs` at high effort. For each file, compare `baseContent` and `headContent`. Identify issues in these categories:

- **Correctness bugs:** logic errors, off-by-one, null/undefined dereferences, race conditions, incorrect calculations
- **Security issues:** injection vulnerabilities (SQL, shell, XSS), secret or credential exposure, missing authentication or authorization checks
- **Breaking changes:** modified public API signatures, changed return types, removed exported methods or fields
- **Error handling:** missing try/catch, unhandled promise rejections, swallowed exceptions, missing input validation at system boundaries

For each issue found, record a finding with these exact fields:

```json
{
  "filePath": "<exact path as it appears in fileDiffs>",
  "lineNumber": "<line number in headContent; for deletes, use line number in baseContent>",
  "severity": "Critical | Important | Minor",
  "title": "<one-line description, max 80 chars>",
  "detail": "<explanation of why this is a problem>",
  "suggestion": "<concrete fix; omit field if not determinable>"
}
```

Severity guide:
- **Critical** — will cause data loss, security breach, crash in production, or incorrect results affecting users
- **Important** — degrades reliability, maintainability, or correctness in non-trivial ways; should be fixed before merge
- **Minor** — style, naming, minor inefficiency, or low-risk improvement; safe to defer

Store all findings as `findings`. If no issues found, set `findings = []`.

Proceed to Step 6.

### Step 5L — Local Checkout Review (Mode L)

*Skip this section if `reviewMode = "A"`.*

**Step 5L-1:** Verify clean working tree:

```bash
git status --porcelain
```

If the output is non-empty, stop and say:
> "Your working tree has uncommitted changes. Stash or commit them before reviewing:
> - `git stash` to stash changes
> - `git commit -am 'WIP'` to commit in-progress work
>
> Then retry: `/dev-review-pr {original input argument}`"

**Step 5L-2:** Fetch and check out the source branch (strip `refs/heads/` prefix first):

```bash
git fetch origin
git checkout {sourceBranch}
```

If checkout fails for any reason (branch not found, conflicts, etc.), stop and say:
> "Could not check out branch `{sourceBranch}`. Try Mode A instead by running `/dev-review-pr {original input argument}` and choosing [A]."

**Step 5L-3:** Invoke the built-in `code-review` skill using the Skill tool with no additional arguments. It will review the diff of the checked-out branch against the base.

**Step 5L-4:** Parse the `code-review` skill's output into the shared finding structure. For each finding in the output:

Map severity:
- "Critical", "Error", "High" → `Critical`
- "Warning", "Medium", "Important" → `Important`
- "Info", "Low", "Minor", "Note", "Suggestion" → `Minor`
- Any unlabelled finding → `Important`

Extract:
- `filePath`: any file path mentioned in the finding (e.g. `src/foo.ts`)
- `lineNumber`: any line number mentioned; if absent, use `1`
- `title`, `detail`, `suggestion`: from the finding text; split as best as possible

Store all parsed findings as `findings`. Proceed to Step 6.

## Step 6 — Post Findings as PR Threads

Separate `findings` into:
- `criticalAndImportant` = findings where `severity` is `"Critical"` or `"Important"`
- `minorByFile` = findings where `severity` is `"Minor"`, grouped by `filePath`

### Step 6a — Post Critical and Important Threads

For each finding in `criticalAndImportant`, call `mcp__azure-devops__repo_create_pull_request_thread` with:

```json
{
  "repositoryId": "{adoRepo}",
  "pullRequestId": "{prId}",
  "project": "{adoProject}",
  "comments": [
    {
      "parentCommentId": 0,
      "content": "[DevPilot Review] {severity}: {title}\n\n{detail}\n\n💡 {suggestion}",
      "commentType": "text"
    }
  ],
  "threadContext": {
    "filePath": "{filePath}",
    "rightFileStart": { "line": {lineNumber}, "offset": 1 },
    "rightFileEnd":   { "line": {lineNumber}, "offset": 1 }
  },
  "status": "active"
}
```

If `suggestion` is absent or empty, omit the `💡 {suggestion}` line from `content`.

If the call fails, log: "⚠️ Could not post thread for {filePath}:{lineNumber} — {error}." Continue to the next finding.

### Step 6b — Post Batched Minor Threads

For each `filePath` key in `minorByFile`, call `mcp__azure-devops__repo_create_pull_request_thread` with:

```json
{
  "repositoryId": "{adoRepo}",
  "pullRequestId": "{prId}",
  "project": "{adoProject}",
  "comments": [
    {
      "parentCommentId": 0,
      "content": "[DevPilot Review] Minor issues in {filePath}\n\n{for each minor finding: '- Line {lineNumber}: {title} — {detail}'}",
      "commentType": "text"
    }
  ],
  "threadContext": {
    "filePath": "{filePath}",
    "rightFileStart": { "line": {first minor finding's lineNumber}, "offset": 1 },
    "rightFileEnd":   { "line": {last minor finding's lineNumber}, "offset": 1 }
  },
  "status": "active"
}
```

If the call fails, log: "⚠️ Could not post minor thread for {filePath} — {error}." Continue.

## Step 7 — Post Summary Thread

Count:
- `countCritical` = number of findings with `severity = "Critical"`
- `countImportant` = number of findings with `severity = "Important"`
- `countMinor` = number of findings with `severity = "Minor"`

Determine `assessment`:
- `countCritical > 0` → `"Critical issues found — do not merge until resolved."`
- `countCritical = 0` AND `countImportant > 0` → `"Important issues found — fix before merging."`
- `countCritical = 0` AND `countImportant = 0` AND `countMinor > 0` → `"Minor issues only — safe to merge after review."`
- All zero → `"No issues found — looks good to merge."`

Build `filesReviewedLabel`:
- If PR was not capped: `"{n} files reviewed"`
- If PR was capped at 30: `"30 of {total} files reviewed (first 30 only)"`

Call `mcp__azure-devops__repo_create_pull_request_thread` with:

```json
{
  "repositoryId": "{adoRepo}",
  "pullRequestId": "{prId}",
  "project": "{adoProject}",
  "comments": [
    {
      "parentCommentId": 0,
      "content": "[DevPilot Review] Summary\n\nPR: {title}\nFiles reviewed: {filesReviewedLabel}\nFindings: {countCritical} Critical · {countImportant} Important · {countMinor} Minor\n\nOverall: {assessment}",
      "commentType": "text"
    }
  ],
  "status": "closed"
}
```

Note: no `threadContext` field — this is a top-level PR comment. `"status": "closed"` marks it as informational rather than an active review thread requiring resolution.

If the call fails, log: "⚠️ Could not post summary thread — {error}." Continue to Step 8.

## Step 8 — Report to Developer

Build `filesReviewedLabel` (same logic as Step 7):
- Not capped: `"{n} files reviewed"`
- Capped: `"30 of {total} files reviewed (first 30 only)"`

Print to terminal:

```
DevPilot review complete for PR #{prId}: {title}

Findings posted as inline PR comments:
  Critical:  {countCritical}
  Important: {countImportant}
  Minor:     {countMinor} (batched by file)
  {filesReviewedLabel}

PR: {prUrl}
```

If `countCritical + countImportant + countMinor = 0`, append:
```
No issues found at high effort.
```
