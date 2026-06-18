---
name: pr-review-all
description: "Review all pending Azure DevOps PRs in parallel using ADO mode. Usage: /pr-review-all"
---

# DevPilot: Review All PRs

**Announce:** "Reviewing all pending pull requests."

---

## Step 0 — Check Prerequisites

Confirm the Azure DevOps MCP tools are available in this session (e.g. `mcp__azure-devops__repo_get_pull_request_by_id` is listed in your available tools). If they are not available, stop and say:

> "Azure DevOps MCP tools are not available. Run `/dev-setup` to configure prerequisites."

## Step 1 — Load PR State

Use the `Read` tool to read `.devpilot/my-prs.json` from the target project root.

If the file does not exist or cannot be parsed as valid JSON, stop and say:

> "No PR state file found. Run `/my-prs` first to populate the list, then retry `/pr-review-all`."

Parse and store:
- `stateProject` = top-level `project` field
- `checkedAt` = `lastCheck.checkedAt`
- `allPrs` = `lastCheck.prs` array

## Step 2 — Check Staleness

Compute the age of `checkedAt` relative to now (in minutes). If older than 60 minutes, warn:

> "⚠️ PR list was last refreshed {age} ago. It may be out of date.
> [C]ontinue with existing list / [R]efresh with /my-prs first — press Enter to continue."

If the response is `R` (case-insensitive) → stop and say: "Run `/my-prs` to refresh, then retry `/pr-review-all`."
Anything else (including Enter) → continue.

## Step 3 — Extract Waiting PRs

Filter `allPrs` to entries where `state === "waiting"`. Store as `waitingPrs`.

If `waitingPrs` is empty, stop and say:

> "No PRs waiting for your review. (Project: {stateProject}, last checked: {checkedAt})"

## Step 4 — Parallel Review

Display the PRs about to be reviewed:

> "Launching parallel reviews for {N} PR(s):
>   #{id}  {url}
>   ..."

Spawn one Agent subagent per PR in `waitingPrs`. **All subagents must be launched in a single message** so they run concurrently.

Each subagent receives the following prompt (substitute the actual values for that PR):

> "You are a PR review subagent. Run the full `/pr-review` skill in `--ado` mode for this PR.
>
> PR URL: {pr.url}
> PR ID: {pr.id}
>
> Follow every step of the pr-review skill (Steps 1 through 8) exactly as written, using `--ado` mode throughout (do NOT do a local checkout). After posting the summary thread, return a JSON result on the very last line of your output in exactly this format (no trailing text after it):
> ```json
> { "prId": {pr.id}, "url": "{pr.url}", "countCritical": N, "countImportant": N, "countMinor": N, "posted": true|false, "error": "<omit field if no error>" }
> ```
> Set `posted: true` if the summary thread was successfully posted (Step 7 of pr-review succeeded). Set `error` only if the review could not be completed; omit the field on success."

Wait for all subagents to complete before proceeding.

## Step 5 — Collect Results

Parse the last JSON line from each subagent's output into `reviewResults`.

For any subagent whose output cannot be parsed into the expected JSON, synthesize a fallback result:
```json
{ "prId": {pr.id}, "url": "{pr.url}", "countCritical": 0, "countImportant": 0, "countMinor": 0, "posted": false, "error": "Could not parse subagent output" }
```

## Step 6 — Print Summary

Print a results table:

```
DevPilot reviewed {N} PR(s) — project: {stateProject}

  PR       Findings (C / I / M)   Posted   Notes
  ──────────────────────────────────────────────────────────────────────
  #{prId}  {C} / {I} / {M}        ✓        {url}
  #{prId}  {C} / {I} / {M}        ✗        {error}
```

Column guide:
- **PR** — `#{prId}`
- **Findings (C / I / M)** — Critical / Important / Minor counts
- **Posted** — `✓` if `posted: true`, `✗` if `posted: false`
- **Notes** — the PR URL on success; the `error` message on failure

After the table, print totals:

```
Total: {sumCritical} Critical · {sumImportant} Important · {sumMinor} Minor across {N} PR(s)
```

If any result has `posted: false`, append:

```
⚠️ One or more reviews could not be fully posted. Check individual PR URLs for details.
```
