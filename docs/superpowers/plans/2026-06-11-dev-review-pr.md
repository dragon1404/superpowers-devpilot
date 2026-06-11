# dev-review-pr Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a standalone `/dev-review-pr` skill that accepts an ADO PR URL or work item ID, fetches the PR diff via ADO MCP tools, reviews the code at high effort, and posts findings as inline PR thread comments.

**Architecture:** Single `skills/dev-review-pr/SKILL.md` natural-language program. Two input paths (PR URL vs work item ID resolution) and two review modes (pure ADO API vs local checkout) converge at a shared posting step using `mcp__azure-devops__repo_create_pull_request_thread`. No state file — stateless and idempotent.

**Tech Stack:** Claude Code SKILL.md, Azure DevOps MCP tools (`mcp__azure-devops__*`), built-in `code-review` CLI skill (Mode L only).

**Spec:** `docs/superpowers/specs/2026-06-11-dev-review-pr-design.md`

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `skills/dev-review-pr/SKILL.md` | Create | Full skill — all steps |
| `.claude-plugin/plugin.json` | Modify | Add `/dev-review-pr` to description command list |
| `.claude-plugin/marketplace.json` | Modify | Update description to include new command |

---

## Task 1: Scaffold skill directory + write header through Step 1

**Files:**
- Create: `skills/dev-review-pr/SKILL.md`

- [ ] **Step 1: Create the skill directory and file with frontmatter, announce line, and Step 1**

```bash
mkdir -p skills/dev-review-pr
```

Write `skills/dev-review-pr/SKILL.md` with this exact content:

````markdown
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
````

- [ ] **Step 2: Self-review Step 1 for ambiguity**

Verify:
- Both URL patterns are explicitly listed with their respective `adoOrg` derivations
- "Plain integer" is unambiguous (digits only, no spaces)
- Error message shows concrete examples
- The skip-to-Step-2 instruction is explicit

- [ ] **Step 3: Commit**

```bash
git add skills/dev-review-pr/SKILL.md
git commit -m "feat: scaffold dev-review-pr skill — Step 1 input detection"
```

---

## Task 2: Write Step 1b — work item ID → PR resolution

**Files:**
- Modify: `skills/dev-review-pr/SKILL.md`

- [ ] **Step 1: Append Step 1b to the SKILL.md**

Append after Step 1:

````markdown
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
````

- [ ] **Step 2: Self-review Step 1b for ambiguity**

Verify:
- The vstfs URL parsing instruction identifies the last integer segment unambiguously
- The "no active PRs" error lists the actual statuses to help the developer understand the state
- `prUrl` construction covers the case where `repository.remoteUrl` is available (standard ADO response field)
- The step ends with an explicit "Proceed to Step 2" instruction so the PR URL path and work item path both enter Step 2 with the same stored variables

- [ ] **Step 3: Commit**

```bash
git add skills/dev-review-pr/SKILL.md
git commit -m "feat: dev-review-pr — Step 1b work item ID to PR resolution"
```

---

## Task 3: Write Steps 2–3 — mode prompt and PR metadata fetch

**Files:**
- Modify: `skills/dev-review-pr/SKILL.md`

- [ ] **Step 1: Append Steps 2 and 3 to the SKILL.md**

````markdown
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
````

- [ ] **Step 2: Self-review Steps 2–3 for ambiguity**

Verify:
- Mode prompt waits for a response before continuing (not assumed)
- Empty response defaults to "A" explicitly
- Step 3 re-checks active status (needed for PR URL path — Step 1b already filtered for active, but PR URL path skips Step 1b)
- `sourceBranch` and `targetBranch` retain the `refs/heads/` prefix (later steps strip it explicitly)

- [ ] **Step 3: Commit**

```bash
git add skills/dev-review-pr/SKILL.md
git commit -m "feat: dev-review-pr — Steps 2–3 mode prompt and PR metadata fetch"
```

---

## Task 4: Write Step 4 — diff fetch (Mode A)

**Files:**
- Modify: `skills/dev-review-pr/SKILL.md`

- [ ] **Step 1: Append Step 4 to the SKILL.md**

````markdown
## Step 4 — Fetch Diff (Mode A only)

*Skip this entire step if `reviewMode = "L"`. Proceed to Step 5.*

### Step 4a — Get Changed Files

Call `mcp__azure-devops__repo_get_pull_request_changes` with:
- repositoryId: {adoRepo}
- pullRequestId: {prId}
- project: {adoProject}

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
````

- [ ] **Step 2: Self-review Step 4 for ambiguity**

Verify:
- Mode A skip instruction is explicit at the top of Step 4
- Folder entries are excluded (`isFolder = true`)
- Binary extension list matches the spec
- `refs/heads/` stripping is called out explicitly — this is easy to forget and would cause API errors
- Failure to fetch a file is non-fatal (skip + warn, don't abort)

- [ ] **Step 3: Commit**

```bash
git add skills/dev-review-pr/SKILL.md
git commit -m "feat: dev-review-pr — Step 4 diff fetch (Mode A)"
```

---

## Task 5: Write Step 5 — code review (both modes)

**Files:**
- Modify: `skills/dev-review-pr/SKILL.md`

- [ ] **Step 1: Append Step 5 to the SKILL.md**

````markdown
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
````

- [ ] **Step 2: Self-review Step 5 for ambiguity**

Verify:
- Mode A and Mode L sections have explicit skip instructions at the top of each section
- Finding field names are exact and consistent with what Steps 6–7 will reference
- Severity guide is concrete (not just "high/medium/low")
- Step 5L explicitly uses the `original input argument` (not the resolved prId) in error messages so the user can retry easily
- `refs/heads/` stripping reminder is in Step 5L-2

- [ ] **Step 3: Commit**

```bash
git add skills/dev-review-pr/SKILL.md
git commit -m "feat: dev-review-pr — Step 5 code review (Mode A inline and Mode L local checkout)"
```

---

## Task 6: Write Steps 6–7 — post findings and summary thread

**Files:**
- Modify: `skills/dev-review-pr/SKILL.md`

- [ ] **Step 1: Append Steps 6 and 7 to the SKILL.md**

````markdown
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
````

- [ ] **Step 2: Self-review Steps 6–7 for ambiguity**

Verify:
- JSON field names (`repositoryId`, `pullRequestId`, `project`, etc.) are consistent across Steps 6a, 6b, and 7
- Minor batch thread uses the first and last `lineNumber` of the group for `rightFileStart`/`rightFileEnd`
- Summary thread omits `threadContext` explicitly (not just silently)
- `assessment` has a defined value for all four combinations of Critical/Important counts
- `"status": "closed"` vs `"status": "active"` distinction is intentional and noted

- [ ] **Step 3: Commit**

```bash
git add skills/dev-review-pr/SKILL.md
git commit -m "feat: dev-review-pr — Steps 6–7 post findings and summary thread"
```

---

## Task 7: Write Step 8 and complete the skill

**Files:**
- Modify: `skills/dev-review-pr/SKILL.md`

- [ ] **Step 1: Append Step 8 to the SKILL.md**

````markdown
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
````

- [ ] **Step 2: Final self-review of the complete SKILL.md**

Read the full file. Check against the spec (`docs/superpowers/specs/2026-06-11-dev-review-pr-design.md`):

- Step 1 covers both URL formats with explicit org derivation ✓
- Step 1b covers work item resolution, no-PR-links error, no-active-PR error, multi-PR tie-breaking ✓
- Step 2 covers mode prompt with default-A behavior ✓
- Step 3 covers PR metadata fetch and active-status guard ✓
- Step 4 covers 30-file cap, binary file filtering, `refs/heads/` stripping, per-changeType fetch logic, per-file failure handling ✓
- Step 5A covers four review categories and exact finding structure ✓
- Step 5L covers clean-tree check, checkout failure, skill invocation, severity mapping, no-line-number fallback ✓
- Step 6a covers individual Critical/Important threads with optional suggestion ✓
- Step 6b covers batched Minor threads per file ✓
- Step 7 covers summary thread with all four assessment cases ✓
- Step 8 covers terminal report with zero-finding message ✓
- Error table in spec: all failure modes have a handling instruction in the skill ✓

Fix any gaps found before committing.

- [ ] **Step 3: Commit**

```bash
git add skills/dev-review-pr/SKILL.md
git commit -m "feat: dev-review-pr — Step 8 terminal report, skill complete"
```

---

## Task 8: Update plugin registration files

**Files:**
- Modify: `.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Update plugin.json description to include the new command**

In `.claude-plugin/plugin.json`, change the `description` field from:

```json
"description": "Automated software delivery workflow: Azure DevOps integration + superpowers skills orchestration. Commands: /dev-workitem <id>, /dev-resume <id>, /dev-fix-pipeline <id>, /dev-setup"
```

To:

```json
"description": "Automated software delivery workflow: Azure DevOps integration + superpowers skills orchestration. Commands: /dev-workitem <id>, /dev-resume <id>, /dev-fix-pipeline <id>, /dev-review-pr <prUrl|workItemId>, /dev-setup"
```

- [ ] **Step 2: Verify plugin.json is valid JSON**

```bash
cat .claude-plugin/plugin.json | python3 -m json.tool > /dev/null && echo "VALID" || echo "INVALID"
```

Expected: `VALID`

- [ ] **Step 3: Update marketplace.json description**

In `.claude-plugin/marketplace.json`, change the plugin `description` field from:

```json
"description": "Automate your Azure DevOps delivery lifecycle — from work item to pull request — with two commands and two approval checkpoints, powered by Claude Code and superpowers skills."
```

To:

```json
"description": "Automate your Azure DevOps delivery lifecycle — from work item to pull request — with AI-powered code review, automated pipeline fixes, and two approval checkpoints, powered by Claude Code and superpowers skills."
```

- [ ] **Step 4: Verify marketplace.json is valid JSON**

```bash
cat .claude-plugin/marketplace.json | python3 -m json.tool > /dev/null && echo "VALID" || echo "INVALID"
```

Expected: `VALID`

- [ ] **Step 5: Commit**

```bash
git add .claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "chore: register dev-review-pr skill in plugin manifests"
```

---

## Self-Review Notes

**Spec coverage:** All spec sections are covered across Tasks 1–8. Error table rows:
- "Invalid input (neither URL nor integer)" → Step 1 ✓
- "Work item has no linked PR artifact links" → Step 1b ✓
- "Work item has linked PRs but none are active" → Step 1b ✓
- "PR is not active" → Step 3 ✓
- "`repo_get_pull_request_changes` fails" → Step 4a (no explicit handling — add warn+stop there if needed; the spec says this is fatal)
- "`repo_get_file_content` fails for a file" → Step 4b ✓
- "`repo_create_pull_request_thread` fails for a finding" → Steps 6a, 6b ✓
- "Local checkout fails (dirty working tree)" → Step 5L-1 ✓
- "Local checkout fails (other reason)" → Step 5L-2 ✓

**Note on Step 4a failure:** The spec says `repo_get_pull_request_changes` failure should stop the skill. The plan currently does not add a stop instruction there. The implementer should add: "If the call fails, stop and say: 'Could not fetch changed files for PR #{prId} — {error}. Cannot proceed without the file list.'"

**Placeholders:** None. All steps contain exact content. ✓

**Type consistency:** `findings` array uses consistent field names across Steps 5, 6, and 7. `fileDiffs` is produced in Step 4 and consumed in Step 5. `adoRepo`, `adoProject`, `prId` are set in Steps 1/1b and used consistently throughout. ✓
