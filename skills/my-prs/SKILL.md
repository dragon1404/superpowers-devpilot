---
name: my-prs
description: "List active Azure DevOps PRs that involve you — waiting for your review, already voted, and ones you created. Usage: /my-prs [project] [email]"
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
- `identityId` → `cachedIdentityId` (may be absent on older state files)
- `lastCheck.prs` → `previousPrs`; derive `previousPrIds` = the set of `id` values in that array
- `previousReviewedPrIds` = the set of `id` values where `state === "reviewed"` in that array
- `prCache` = a map of `id` → `{ state, checkedAt, vote }` built from `lastCheck.prs` (only entries that have a `checkedAt` field)

If the file is missing or does not parse, treat `configEmail`, `configProject`, `cachedIdentityId`, `previousPrIds`, `previousReviewedPrIds`, and `prCache` as unset. Do not error. When `previousPrIds` is unset (first run), no PR will be tagged `[NEW]` later.

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

## Step 3 — Resolve Identity

If `cachedIdentityId` is set, use it directly as `myIdentityId` and set `identityUnresolved = false`. Skip the API call.

Otherwise, call `mcp__azure-devops__core_get_identity_ids` with:
- searchFilter: {email}

If it returns an identity, store its GUID as `myIdentityId` and set `identityUnresolved = false`. This identity is used to match your reviewer entry in each PR's `reviewers[]` array.

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

## Step 5 — Enrich Reviewer PRs with Vote and Reviewed Status

For each PR in `reviewerPrs`:

**Cache hit check (skip fetches if fresh):**

Look up `prCache[pr.pullRequestId]`. If an entry exists with `state` of `reviewed` or `voted`, AND its `checkedAt` timestamp is within 30 minutes of now:
- Use cached `state` and `vote` directly. Assign to the appropriate bucket. Skip all API calls for this PR.

Otherwise, proceed with fetches:

**Fetch full details to read the vote:**

Call `mcp__azure-devops__repo_get_pull_request_by_id` with:
- repositoryId: {pr.repository.name}
- pullRequestId: {pr.pullRequestId}
- project: {project}

From the returned `reviewers[]` array, find the entry whose `id` equals `myIdentityId` and read its `vote`. If `identityUnresolved` is true or no matching reviewer entry is found, treat `vote = 0`.

Vote labels:
- `10` → `approved`
- `5` → `approved with suggestions`
- `-5` → `waiting for author`
- `-10` → `rejected`

**Bucket assignment:**
- `vote != 0` → **Already voted** (with vote label). Skip thread check for this PR.
- `vote == 0` → check reviewed status (below), then assign to **Reviewed, pending vote** or **Waiting for my review**.

**Reviewed status check (vote = 0 PRs only):**

First check if this PR's id is in `previousReviewedPrIds`. If yes, mark `reviewed = true` immediately — skip the thread fetch entirely (a DevPilot review thread never disappears, so the cached state is reliable).

Otherwise, call `mcp__azure-devops__repo_list_pull_request_threads` with:
- repositoryId: {pr.repository.name}
- pullRequestId: {pr.pullRequestId}
- project: {project}

If any thread's first comment (`comments[0].content`) starts with `[DevPilot Review] Summary`, mark this PR as `reviewed = true` → bucket: **Reviewed, pending vote**. Otherwise `reviewed = false` → bucket: **Waiting for my review**.

If the thread fetch fails, treat `reviewed = false` → **Waiting for my review**.

## Step 5b — Tag New PRs

Build `currentPrIds` = the set of all `pullRequestId` values across the three buckets. A PR is **new** if its id is in `currentPrIds` but not in `previousPrIds`. Record which PRs are new for rendering. If `previousPrIds` is unset (first run), no PR is new.

## Step 6 — Render Output

Print a header line `my-prs — project: {project}`, then four sections in this order: **Waiting for my review**, **Reviewed, pending vote**, **Already voted**, **PRs I created**. Within each section, sort PRs oldest-first by `creationDate`. Show the count after each section header, e.g. `Waiting for my review (2)`. Empty buckets render as `(none)`, e.g. `Already voted (none)`.

Each PR is one compact line:

```
#{pullRequestId}  {title} → {targetBranch} — {createdBy.displayName}, {age}{statusTags}  {prUrl}{newTag}
```

- `targetBranch` — the PR's target branch with `refs/heads/` stripped (e.g. `main`, `develop`). Read from `targetRefName` in the PR payload.
- `age` — time from `creationDate` to now, rendered compactly (e.g. `3h`, `2d`, `5w`).
- `statusTags` — space-separated tags after the age field:
  - In **Waiting for my review**: no status tags.
  - In **Reviewed, pending vote**: no status tags (section heading is self-explanatory).
  - In **Already voted**: append ` ({voteLabel})` — e.g. ` (approved)`.
  - In **PRs I created**: no status tags.
- `prUrl` — the full web URL, built from the PR payload's `repository.webUrl` as `{repository.webUrl}/pullrequest/{pullRequestId}`.
- `newTag` — ` [NEW]` for PRs new since the last run (from Step 5b), otherwise empty.

If `identityUnresolved` is true, print this note immediately under the **Waiting for my review** header:

> _Note: could not resolve identity for `{email}` — vote status unknown, all reviewer PRs shown here._

Example output:

```
my-prs — project: Payments

Waiting for my review (1)
  #1488  Add retry to webhook sender → develop  — bob,   2d  https://dev.azure.com/org/Payments/_git/api/pullrequest/1488 [NEW]

Reviewed, pending vote (1)
  #1423  Fix token refresh race      → main     — alice, 5d  https://dev.azure.com/org/Payments/_git/api/pullrequest/1423

Already voted (1)
  #1402  Bump SDK to 4.x  → main  — carol, 8d (approved)  https://dev.azure.com/org/Payments/_git/sdk/pullrequest/1402

PRs I created (1)
  #1500  Refactor cache layer  → main  — 1d  https://dev.azure.com/org/Payments/_git/api/pullrequest/1500
```

After printing the listing, if the **Waiting for my review** bucket has one or more PRs, print this alert:

> ⚠️ You have {N} PR(s) waiting for your review.
> Run `/pr-review <prUrl>` to review one, or `/pr-review-all` to review all at once.

## Step 7 — Persist State

Create the state directory, then fully overwrite the state file (do not merge or patch — replace the entire file contents):

```bash
mkdir -p ./.devpilot
```

Write `./.devpilot/my-prs.json`, replacing any existing content, with the resolved config and one entry per PR from this run:

```json
{
  "email": "{email}",
  "project": "{project}",
  "identityId": "{myIdentityId or null}",
  "lastCheck": {
    "checkedAt": "{ISO-8601 timestamp for now}",
    "prs": [
      { "id": {pullRequestId}, "url": "{prUrl}", "targetBranch": "{targetBranch}", "state": "waiting", "checkedAt": "{ISO-8601 timestamp for now}" },
      { "id": {pullRequestId}, "url": "{prUrl}", "targetBranch": "{targetBranch}", "state": "reviewed", "checkedAt": "{ISO-8601 timestamp for now}" },
      { "id": {pullRequestId}, "url": "{prUrl}", "targetBranch": "{targetBranch}", "state": "voted", "vote": "{voteLabel}", "checkedAt": "{ISO-8601 timestamp for now}" },
      { "id": {pullRequestId}, "url": "{prUrl}", "targetBranch": "{targetBranch}", "state": "created", "checkedAt": "{ISO-8601 timestamp for now}" }
    ]
  }
}
```

For each PR, `state` is the bucket (`waiting` / `reviewed` / `voted` / `created`). `targetBranch` is always included. `checkedAt` is the ISO-8601 timestamp for this run (same value for all entries). `vote` is included only for `voted` entries. This write is best-effort — if it fails (e.g. read-only directory), warn but continue; the listing already shown is unaffected.
