# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A Claude Code plugin (`superpowers-devpilot`) distributed via a self-hosted marketplace. It contains no compiled code — everything is SKILL.md instruction files that Claude Code loads and executes as natural-language programs.

## Architecture

Each user-facing command is a skill in `skills/<name>/SKILL.md`. Claude Code reads and follows these files step-by-step when the user invokes the corresponding slash command.

| Skill directory | Command | Purpose |
|---|---|---|
| `skills/dev-workitem/` | `/dev-workitem <id>` | Start a new delivery workflow for an ADO work item |
| `skills/dev-resume/` | `/dev-resume <id>` | Resume or approve a paused workflow |
| `skills/dev-fix-pipeline/` | `/dev-fix-pipeline <id>` | Diagnose and fix a failed CI pipeline |
| `skills/dev-setup/` | `/dev-setup` | Guided prerequisite installation wizard |

Skills call other skills (e.g. `superpowers:brainstorming`, `superpowers:writing-plans`) and ADO MCP tools (`mcp__azure-devops__*`) to do their work.

## State File

Workflow state is persisted at `.devpilot/state/<workItemId>.json` in the **target project** (not this repo). The schema is defined in `skills/dev-workitem/SKILL.md` Step 6. Status values drive the branching logic in `skills/dev-resume/SKILL.md`.

## Releasing a New Version

Three files must all be updated to the same version string whenever a release is made:

1. `.claude-plugin/plugin.json` — `"version"` field (read by the plugin manager UI)
2. `.claude-plugin/marketplace.json` — `"version"` inside the `plugins[0]` entry
3. `package.json` — `"version"` field

After committing, tag the release (`git tag vX.Y.Z`) and create a GitHub release. The plugin manager reads `plugin.json` for the displayed version — if only the git tag is updated, the UI will show the old version.

## Dependencies (runtime, not in this repo)

- `superpowers` plugin — provides skills invoked by DevPilot (`brainstorming`, `writing-plans`, `subagent-driven-development`, `requesting-code-review`, `test-driven-development`, `finishing-a-development-branch`, `systematic-debugging`)
- Azure DevOps MCP server — provides `mcp__azure-devops__*` tools used throughout every skill
