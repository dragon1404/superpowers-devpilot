# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A Claude Code plugin (`superpowers-devpilot`) distributed via a self-hosted marketplace. It contains no compiled code — everything is SKILL.md instruction files that Claude Code loads and executes as natural-language programs.

## Architecture

Each user-facing command is a skill in `skills/<name>/SKILL.md`. Claude Code reads and follows these files step-by-step when the user invokes the corresponding slash command.

| Skill directory | Command | Purpose |
|---|---|---|
| `skills/dev-workitem/` | `/dev-workitem <id>` | Start a new delivery workflow for an ADO work item |
| `skills/dev-resume/` | `/dev-resume [id]` | Resume or approve a paused workflow; auto-detects work item ID from state files if omitted |
| `skills/dev-fix-pipeline/` | `/dev-fix-pipeline [id\|prUrl]` | Diagnose and fix a failed CI pipeline; accepts work item ID or PR URL; auto-detects from state files if omitted |
| `skills/dev-setup/` | `/dev-setup` | Guided prerequisite installation wizard |
| `skills/pr-review/` | `/pr-review <prUrl\|workItemId>` | Review a PR and post findings as inline ADO threads |
| `skills/my-prs/` | `/my-prs [--refresh]` | List active PRs involving you, split by vote and reviewed status |
| `skills/pr-review-all/` | `/pr-review-all` | Review all `waiting` PRs in parallel via ADO mode; reads from `.devpilot/my-prs.json` |

Skills call other skills (e.g. `superpowers:brainstorming`, `superpowers:writing-plans`) and ADO MCP tools (`mcp__azure-devops__*`) to do their work.

## Cross-Skill Contract

`/pr-review` posts a summary thread whose content starts with `[DevPilot Review] Summary`. `/my-prs` detects this prefix via `mcp__azure-devops__repo_list_pull_request_threads` to mark unvoted reviewer PRs as `[reviewed]`. Do not change this prefix.

`/pr-review-all` reads `.devpilot/my-prs.json` (written by `/my-prs`) and spawns one Agent subagent per `"waiting"` PR, each running the full `/pr-review --ado` flow. The `[DevPilot Review] Summary` detection contract between `/pr-review` and `/my-prs` remains unchanged.

## State Files

Both state files live in the **target project** (not this repo):

- **Workflow state** — `.devpilot/state/<workItemId>.json`: persists stage progress for `/dev-workitem` and `/dev-resume`. Schema and status values defined in `skills/dev-workitem/SKILL.md` Step 6; branching logic in `skills/dev-resume/SKILL.md`.
- **PR list state** — `.devpilot/my-prs.json`: persists resolved email/project config and the last-run PR set (used for `[NEW]` tagging on the next run). Schema defined in `skills/my-prs/SKILL.md` Step 7.

## docs/ Convention

Design specs and implementation plans produced during development of this plugin live in `docs/superpowers/specs/` and `docs/superpowers/plans/`. These are committed history, not generated output — they document why skills were built the way they are.

## Releasing a New Version

Every release requires all four steps — do not skip any:

1. Bump the version to the same string in all three files:
   - `.claude-plugin/plugin.json` — `"version"` field (read by the plugin manager UI)
   - `.claude-plugin/marketplace.json` — `"version"` inside the `plugins[0]` entry
   - `package.json` — `"version"` field
2. Update `README.md` if the release adds or changes user-facing behaviour.
3. Commit and push.
4. Tag the release (`git tag vX.Y.Z && git push origin vX.Y.Z`) and create a GitHub release with notes.

The plugin manager reads `plugin.json` for the displayed version — if only the git tag is updated, the UI will show the old version.

## Dependencies (runtime, not in this repo)

- `superpowers` plugin — provides skills invoked by DevPilot (`brainstorming`, `writing-plans`, `subagent-driven-development`, `requesting-code-review`, `test-driven-development`, `finishing-a-development-branch`, `systematic-debugging`)
- Azure DevOps MCP server — provides `mcp__azure-devops__*` tools used throughout every skill
