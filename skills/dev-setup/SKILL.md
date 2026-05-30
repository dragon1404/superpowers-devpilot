---
name: dev-setup
description: "Check and guide installation of DevPilot's required dependencies: superpowers plugin and Azure DevOps MCP server. Run this once before using /dev-workitem."
---

# DevPilot: Setup & Dependency Check

**Announce:** "Checking DevPilot prerequisites..."

---

## Step 1 — Check superpowers Plugin

Run:
```bash
ls ~/.claude/plugins/cache/claude-plugins-official/superpowers/ 2>/dev/null && echo "FOUND" || echo "MISSING"
```

Record result as `superpowers_status` (FOUND or MISSING).

## Step 2 — Check Azure DevOps MCP

Check whether `mcp__azure-devops__wit_get_work_item` is available as a tool in the current session.

If the tool is listed in available tools → `ado_status = FOUND`
If not → `ado_status = MISSING`

## Step 3 — Report Status

Display a status table:

> **DevPilot Dependency Check**
>
> | Dependency | Status |
> |---|---|
> | superpowers plugin | {superpowers_status} |
> | Azure DevOps MCP | {ado_status} |

If both are FOUND, say:

> ✅ All dependencies are installed. You're ready to use DevPilot.
> Run `/dev-workitem <workItemId>` to start your first workflow.

Then stop.

If any are MISSING, continue to Step 4.

---

## Step 4 — Install Missing: superpowers Plugin

**Only run this section if `superpowers_status = MISSING`.**

Tell the developer:

> ### Installing superpowers
>
> superpowers is a Claude Code plugin that provides the skills DevPilot uses at each workflow stage (design, planning, implementation, review, testing, PR).
>
> **Installation — Official Marketplace (recommended):**
>
> Run this command in Claude Code:
> ```
> /plugin install superpowers@claude-plugins-official
> ```
>
> **Verify:**
> After installation, run `/dev-setup` again to confirm the status shows FOUND.
>
> Repository: https://github.com/obra/superpowers

---

## Step 5 — Install Missing: Azure DevOps MCP

**Only run this section if `ado_status = MISSING`.**

Ask the developer:

> ### Installing Azure DevOps MCP
>
> The Azure DevOps MCP server gives DevPilot access to your work items, repositories, and pull requests via the `mcp__azure-devops__*` tools.
>
> **What is your Azure DevOps organization name?**
> (The org name from your ADO URL: `https://dev.azure.com/{org}`)

Wait for the developer to provide their org name. Store it as `{org}`.

Then tell the developer:

> **Run this command in your terminal:**
>
> ```bash
> claude mcp add azure-devops --scope user -- npx -y @azure-devops/mcp {org}
> ```
>
> This registers the Azure DevOps MCP server for your user account using your org `{org}`.
>
> **Authentication:** The MCP server uses your existing Azure DevOps credentials. If prompted, authenticate via the browser flow or set the `AZURE_DEVOPS_TOKEN` environment variable to a Personal Access Token with these scopes:
> - **Work Items:** Read & Write
> - **Code:** Read
> - **Pull Requests:** Read & Write
>
> Generate a PAT at: Azure DevOps → User Settings → Personal Access Tokens → New Token
>
> **Verify:**
> After running the command, restart Claude Code and run `/dev-setup` again to confirm the status shows FOUND.
>
> Repository: https://github.com/microsoft/azure-devops-mcp

---

## Step 6 — Final Status

After presenting all installation instructions, say:

> Once you have installed the missing dependencies, restart Claude Code and run `/dev-setup` again to confirm everything is ready.
>
> When both show FOUND, run `/dev-workitem <workItemId>` to start your first DevPilot workflow.
