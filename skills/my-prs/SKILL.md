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
