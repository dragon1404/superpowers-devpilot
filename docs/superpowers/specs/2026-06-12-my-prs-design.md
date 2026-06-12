# Design: `/my-prs` Skill

**Date:** 2026-06-12
**Status:** Draft

---

## Overview

A standalone DevPilot skill that lists every active Azure DevOps pull request that involves the current user in a given project, grouped into three buckets so the developer can quickly spot reviews they are missing:

1. **Waiting for my review** — PRs where I am a reviewer and have not voted yet
2. **Assigned to me** — PRs where I am a reviewer and have already voted
3. **PRs I created** — PRs I authored

It is read-only and stateless. It performs no write actions (no voting, no commenting).

---

## Command

```
/my-prs [project] [email]
```

Both arguments are optional and **order-independent**. They are disambiguated by content:

- The token containing `@` → `email` (identity override).
- The other token → `project` (project name).

Examples:

- `/my-prs` — uses saved/derived values
- `/my-prs Payments` — explicit project
- `/my-prs me@corp.com` — explicit email
- `/my-prs Payments me@corp.com` — explicit project and email (any order)

### Persisted state

The skill reads and writes a small JSON file at `./.devpilot/my-prs.json` (in the current working directory where the skill runs). It holds both the resolved config and the last run's PR set:

```json
{
  "email": "me@corp.com",
  "project": "Payments",
  "lastCheck": {
    "checkedAt": "2026-06-12T09:00:00Z",
    "prIds": [1423, 1488, 1402, 1500]
  }
}
```

- `email` / `project` — remembered config so a bare `/my-prs` reuses them. Resolution precedence (highest wins):
  - **email**: argument → config file → session email (Claude Code `userEmail`)
  - **project**: argument → config file → git `origin` remote
- `lastCheck.prIds` — the flat set of pull request IDs surfaced on the previous run, used to tag PRs that are **new since you last checked**.

The file is rewritten at the end of each run with the resolved config and the current run's PR IDs.

### No organization needed

The Azure DevOps MCP server is configured with its organization at setup time. None of the MCP tools used here (`repo_list_pull_requests_by_repo_or_project`, `core_get_identity_ids`) accept an `org` parameter — the org is implicit in the server. The skill therefore never resolves or passes an org.

---

## Step-by-Step Flow

### Step 0 — Check Prerequisites

Confirm the Azure DevOps MCP tools are available in the session (e.g. `mcp__azure-devops__repo_list_pull_requests_by_repo_or_project`). If not, stop:

> "Azure DevOps MCP tools are not available. Run `/dev-setup` to configure prerequisites."

### Step 1 — Parse Arguments

Take everything after `/my-prs`, trim, and split on whitespace into at most two tokens.

- A token containing `@` → store as `argEmail`.
- A token not containing `@` → store as `argProject`.

### Step 1b — Load Persisted State

If `./.devpilot/my-prs.json` exists, read it and parse:
- `email` and `project` as `configEmail` and `configProject`
- `lastCheck.prIds` as `previousPrIds` (the set of PR IDs from the previous run)

If the file is missing or unparseable, treat all three as unset (do not error). When `previousPrIds` is unset (first run), nothing is tagged `[NEW]`.

### Step 2 — Resolve Email and Project

Apply precedence (highest wins):

- `email` = `argEmail` → `configEmail` → session email (Claude Code `userEmail` context value)
- `project` = `argProject` → `configProject` → git `origin`

To derive the project from git, read the remote:

```bash
git remote get-url origin
```

Parse the project segment using the same patterns as `/dev-workitem`:
- `https://dev.azure.com/{org}/{project}/_git/{repo}` → `project`
- `git@ssh.dev.azure.com:v3/{org}/{project}/{repo}` → `project`
- `https://{org}.visualstudio.com/{project}/_git/{repo}` → `project`

If `project` cannot be resolved from any source (no argument, no config, no parseable git remote) → stop:

> "No project specified and no Azure DevOps git remote found. Run `/my-prs <projectName>` — e.g. `/my-prs Payments`."

### Step 3 — Resolve Identity (vote split only)

Call `mcp__azure-devops__core_get_identity_ids` with `searchFilter: {email}`.

- On success, store the returned identity GUID as `myIdentityId`.
- This identity is used **only** to find my own vote within each reviewer PR's `reviewers[]` array.
- If the call fails or returns no match, set `myIdentityId = null` and set a flag `identityUnresolved = true` (handled in Step 5).

### Step 4 — Query Pull Requests

All queries use `status: "Active"` and are scoped to the resolved `project` (no `repositoryId` — project scope returns PRs across all repos in the project).

**Query A — PRs I created:**

```
repo_list_pull_requests_by_repo_or_project(
  project: {project},
  status: "Active",
  created_by_me: true,
  top: 100
)
```

Store as `createdByMe`.

**Query B — PRs where I am a reviewer:**

```
repo_list_pull_requests_by_repo_or_project(
  project: {project},
  status: "Active",
  i_am_reviewer: true,
  top: 100
)
```

Store as `reviewerPrs`.

If either query call fails → stop with the error (do not render a partial, misleading list):

> "Failed to fetch pull requests for project {project}: {error}."

### Step 5 — Split the Reviewer Bucket by Vote

For each PR in `reviewerPrs`:

- If `identityUnresolved` is true → place the PR in **Waiting for my review** (safe fallback — surfaces it rather than hiding it).
- Otherwise, find the entry in the PR's `reviewers[]` array whose `id` equals `myIdentityId` and read its `vote`:
  - `vote == 0` → **Waiting for my review**
  - `vote != 0` → **Assigned to me**, with a label derived from the vote value:
    - `10` → approved
    - `5` → approved with suggestions
    - `-5` → waiting for author
    - `-10` → rejected
  - If no matching reviewer entry is found → **Waiting for my review** (fallback).

### Step 5b — Tag New PRs

Compute `currentPrIds` = the set of all `pullRequestId`s across the three buckets. A PR is **new** if its ID is in `currentPrIds` but not in `previousPrIds`. Mark new PRs for a `[NEW]` tag in render. If `previousPrIds` is unset (first run), no PR is tagged new.

### Step 6 — Render Output

Print three sections in this order. Within each section, sort PRs **oldest-first** by `creationDate`. Each PR is one compact line:

```
#{pullRequestId}  {title} — {createdBy.displayName}, {age}{voteLabel}  {prUrl}{newTag}
```

- `age` is computed from `creationDate` relative to now, rendered compactly (e.g. `3h`, `2d`, `5w`).
- `voteLabel` appears only in the "Assigned to me" section, formatted as ` ({label})`.
- `prUrl` is the web URL of the PR, built from the PR payload's `repository.webUrl` as `{repository.webUrl}/pullrequest/{pullRequestId}`. This keeps the skill org-agnostic — the org is whatever the payload already contains.
- `newTag` is ` [NEW]` for PRs new since the last run, otherwise empty.

If `identityUnresolved` is true, print a one-line note under the "Waiting for my review" header:

> _Note: could not resolve identity for `{email}` — all reviewer PRs shown here; vote status unknown._

Example:

```
my-prs — project: Payments

Waiting for my review (2)
  #1423  Fix token refresh race        — alice, 5d   https://dev.azure.com/org/Payments/_git/api/pullrequest/1423
  #1488  Add retry to webhook sender   — bob,   2d   https://dev.azure.com/org/Payments/_git/api/pullrequest/1488 [NEW]

Assigned to me (1)
  #1402  Bump SDK to 4.x  — carol, 8d (approved)  https://dev.azure.com/org/Payments/_git/sdk/pullrequest/1402

PRs I created (1)
  #1500  Refactor cache layer  — 1d  https://dev.azure.com/org/Payments/_git/api/pullrequest/1500
```

Empty buckets render as `(none)`:

```
Assigned to me (none)
```

### Step 7 — Persist State

After rendering, write `./.devpilot/my-prs.json` (creating the `.devpilot` directory if needed) with the resolved config and this run's PR IDs:

```json
{
  "email": "{email}",
  "project": "{project}",
  "lastCheck": {
    "checkedAt": "{ISO-8601 now}",
    "prIds": [ ...currentPrIds ]
  }
}
```

This is best-effort — if the write fails (e.g. read-only directory), warn but continue; it does not change the listing already shown.

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| Azure DevOps MCP tools unavailable | Stop, point to `/dev-setup` |
| No project arg and no parseable git origin | Stop with usage example |
| `core_get_identity_ids` fails / no match | Continue; all reviewer PRs go to "Waiting for my review" with a note; vote labels omitted |
| `repo_list_pull_requests_by_repo_or_project` fails (either query) | Stop with the error — no partial list |
| A reviewer PR has no matching reviewer entry for my identity | Place it in "Waiting for my review" (fallback) |
| `./.devpilot/my-prs.json` missing or unparseable | Treat config and `previousPrIds` as unset; continue with arg/session/git resolution; tag no PRs `[NEW]` |
| Writing `./.devpilot/my-prs.json` fails | Warn and continue — persistence is best-effort, never blocks the listing |

---

## Constraints

- Read-only: no votes, comments, or any write operations
- Active PRs only
- Single project per invocation (scope is the resolved project)
- Read-only against ADO; the only thing it writes locally is `./.devpilot/my-prs.json` (email, default project, and the last run's PR IDs)

---

## Out of Scope

- Completed or abandoned PRs
- Org-wide scan across all projects
- Draft PRs handled differently from active ones
- Any write actions (voting, commenting, approving)
- Pagination beyond `top: 100` per bucket
