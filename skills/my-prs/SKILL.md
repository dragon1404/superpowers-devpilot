---
name: my-prs
description: "List active Azure DevOps PRs that involve you — waiting for your review, assigned to you (already voted), and ones you created. Usage: /my-prs [project] [email]"
---

# DevPilot: My Pull Requests

**Announce:** "Listing your pull requests."

---

## Step 0 — Check Prerequisites

Confirm the Azure DevOps MCP tools are available in this session (e.g. `mcp__azure-devops__repo_list_pull_requests_by_repo_or_project` is listed in your available tools). If they are not available, stop and say:

> "Azure DevOps MCP tools are not available. Run `/dev-setup` to configure prerequisites."

## Step 1 — Parse Arguments

Take everything after `/my-prs`, trim it, and split on whitespace into at most two tokens.

- A token containing `@` → store as `argEmail`.
- A token not containing `@` → store as `argProject`.
- If there are no tokens, both `argEmail` and `argProject` are unset.

## Step 1b — Load Persisted State

Read the persisted state file if it exists:

```bash
cat ./.devpilot/my-prs.json 2>/dev/null
```

If the file exists and parses as JSON, extract:
- `email` → `configEmail`
- `project` → `configProject`
- `lastCheck.prs` → `previousPrs`; derive `previousPrIds` = the set of `id` values in that array

If the file is missing or does not parse, treat `configEmail`, `configProject`, and `previousPrIds` as unset. Do not error. When `previousPrIds` is unset (first run), no PR will be tagged `[NEW]` later.

## Step 2 — Resolve Email and Project

Resolve `email` by precedence (first available wins):
1. `argEmail`
2. `configEmail`
3. The session email — the user's email from your context (the `userEmail` value).

Resolve `project` by precedence (first available wins):
1. `argProject`
2. `configProject`
3. Derived from the git `origin` remote. Read it:

   ```bash
   git remote get-url origin
   ```

   Parse the project segment using these patterns:
   - `https://dev.azure.com/{org}/{project}/_git/{repo}` → `project`
   - `git@ssh.dev.azure.com:v3/{org}/{project}/{repo}` → `project`
   - `https://{org}.visualstudio.com/{project}/_git/{repo}` → `project`

If `project` cannot be resolved from any source (no argument, no config, no parseable git remote), stop and say:

> "No project specified and no Azure DevOps git remote found. Run `/my-prs <projectName>` — e.g. `/my-prs Payments`."

Note: an organization is never resolved or passed — the Azure DevOps MCP server is already configured with its organization.

## Step 3 — Resolve Identity (vote split only)

Call `mcp__azure-devops__core_get_identity_ids` with:
- searchFilter: {email}

If it returns an identity, store its GUID as `myIdentityId` and set `identityUnresolved = false`. This identity is used only to find your own vote inside each reviewer PR's `reviewers[]` array.

If the call fails or returns no match, set `myIdentityId = null` and `identityUnresolved = true`.

## Step 4 — Query Pull Requests

Both queries use `status: "Active"` and are scoped to `project`. Do not pass `repositoryId` — project scope returns PRs across all repos in the project.

**Query A — PRs I created.** Call `mcp__azure-devops__repo_list_pull_requests_by_repo_or_project` with:
- project: {project}
- status: "Active"
- created_by_me: true
- top: 100

Store the result as `createdByMe`.

**Query B — PRs where I am a reviewer.** Call `mcp__azure-devops__repo_list_pull_requests_by_repo_or_project` with:
- project: {project}
- status: "Active"
- i_am_reviewer: true
- top: 100

Store the result as `reviewerPrs`.

If either call fails, stop and say (do not render a partial list):

> "Failed to fetch pull requests for project {project}: {error}."

## Step 5 — Split the Reviewer Bucket by Vote

For each PR in `reviewerPrs`:

- If `identityUnresolved` is true → place it in **Waiting for my review** (safe fallback — surfaces it rather than hiding it).
- Otherwise, find the entry in the PR's `reviewers[]` array whose `id` equals `myIdentityId`, and read its `vote`:
  - `vote == 0` → **Waiting for my review**
  - `vote != 0` → **Assigned to me**, with a vote label derived from the vote value:
    - `10` → `approved`
    - `5` → `approved with suggestions`
    - `-5` → `waiting for author`
    - `-10` → `rejected`
  - If no reviewer entry matches `myIdentityId` → **Waiting for my review**.

The third bucket, **PRs I created**, is `createdByMe`.

## Step 5b — Tag New PRs

Build `currentPrIds` = the set of all `pullRequestId` values across the three buckets. A PR is **new** if its id is in `currentPrIds` but not in `previousPrIds`. Record which PRs are new for rendering. If `previousPrIds` is unset (first run), no PR is new.
