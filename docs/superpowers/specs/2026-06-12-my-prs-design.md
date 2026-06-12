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

- The token containing `@` → `email` (identity override). When omitted, defaults to the session email.
- The other token → `project` (project name). When omitted, derived from the git `origin` remote.

Examples:

- `/my-prs` — current repo's project, session email
- `/my-prs Payments` — explicit project, session email
- `/my-prs me@corp.com` — current repo's project, explicit email
- `/my-prs Payments me@corp.com` — explicit project and email (any order)

### No organization needed

The Azure DevOps MCP server is configured with its organization at setup time. None of the MCP tools used here (`repo_list_pull_requests_by_repo_or_project`, `core_get_identity_ids`) accept an `org` parameter — the org is implicit in the server. The skill therefore never resolves or passes an org.

---

## Step-by-Step Flow

### Step 0 — Check Prerequisites

Confirm the Azure DevOps MCP tools are available in the session (e.g. `mcp__azure-devops__repo_list_pull_requests_by_repo_or_project`). If not, stop:

> "Azure DevOps MCP tools are not available. Run `/dev-setup` to configure prerequisites."

### Step 1 — Parse Arguments

Take everything after `/my-prs`, trim, and split on whitespace into at most two tokens.

- A token containing `@` → store as `email`.
- A token not containing `@` → store as `project`.
- If `email` is not provided, set `email` to the session email (Claude Code `userEmail` context value).

### Step 2 — Resolve Project

1. If `project` was supplied as an argument → use it.
2. Otherwise, read the git `origin` remote:

   ```bash
   git remote get-url origin
   ```

   Parse the project segment using the same patterns as `/dev-workitem`:
   - `https://dev.azure.com/{org}/{project}/_git/{repo}` → `project`
   - `git@ssh.dev.azure.com:v3/{org}/{project}/{repo}` → `project`
   - `https://{org}.visualstudio.com/{project}/_git/{repo}` → `project`

3. If no `project` argument and the git remote cannot be read or parsed → stop:

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

### Step 6 — Render Output

Print three sections in this order. Within each section, sort PRs **oldest-first** by `creationDate`. Each PR is one compact line:

```
#{pullRequestId}  {title} — {createdBy.displayName}, {age}{voteLabel}  {prUrl}
```

- `age` is computed from `creationDate` relative to now, rendered compactly (e.g. `3h`, `2d`, `5w`).
- `voteLabel` appears only in the "Assigned to me" section, formatted as ` ({label})`.
- `prUrl` is the web URL of the PR, built from the PR payload's `repository.webUrl` as `{repository.webUrl}/pullrequest/{pullRequestId}`. This keeps the skill org-agnostic — the org is whatever the payload already contains.

If `identityUnresolved` is true, print a one-line note under the "Waiting for my review" header:

> _Note: could not resolve identity for `{email}` — all reviewer PRs shown here; vote status unknown._

Example:

```
my-prs — project: Payments

Waiting for my review (2)
  #1423  Fix token refresh race        — alice, 5d   https://dev.azure.com/org/Payments/_git/api/pullrequest/1423
  #1488  Add retry to webhook sender   — bob,   2d   https://dev.azure.com/org/Payments/_git/api/pullrequest/1488

Assigned to me (1)
  #1402  Bump SDK to 4.x  — carol, 8d (approved)  https://dev.azure.com/org/Payments/_git/sdk/pullrequest/1402

PRs I created (1)
  #1500  Refactor cache layer  — 1d  https://dev.azure.com/org/Payments/_git/api/pullrequest/1500
```

Empty buckets render as `(none)`:

```
Assigned to me (none)
```

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| Azure DevOps MCP tools unavailable | Stop, point to `/dev-setup` |
| No project arg and no parseable git origin | Stop with usage example |
| `core_get_identity_ids` fails / no match | Continue; all reviewer PRs go to "Waiting for my review" with a note; vote labels omitted |
| `repo_list_pull_requests_by_repo_or_project` fails (either query) | Stop with the error — no partial list |
| A reviewer PR has no matching reviewer entry for my identity | Place it in "Waiting for my review" (fallback) |

---

## Constraints

- Read-only: no votes, comments, or any write operations
- Active PRs only
- Single project per invocation (scope is the resolved project)
- Stateless: no state file, idempotent, safe to re-run

---

## Out of Scope

- Completed or abandoned PRs
- Org-wide scan across all projects
- Draft PRs handled differently from active ones
- Any write actions (voting, commenting, approving)
- Pagination beyond `top: 100` per bucket
