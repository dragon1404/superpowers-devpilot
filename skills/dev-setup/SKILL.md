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
> **Installation:**
>
> **Option A — Install via npm (recommended):**
> ```bash
> npm install -g @claude-plugins/superpowers
> ```
> Then restart Claude Code.
>
> **Option B — Install from source:**
> ```bash
> git clone https://github.com/obra/superpowers ~/.claude/plugins/superpowers
> ```
> Then add the plugin path in Claude Code settings → Plugins.
>
> **Verify:**
> After restarting Claude Code, run `/dev-setup` again to confirm the status shows FOUND.
>
> Repository: https://github.com/obra/superpowers

---

## Step 5 — Install Missing: Azure DevOps MCP

**Only run this section if `ado_status = MISSING`.**

Tell the developer:

> ### Installing Azure DevOps MCP
>
> The Azure DevOps MCP server gives DevPilot access to your work items, repositories, and pull requests via the `mcp__azure-devops__*` tools.
>
> **Step 1 — Install the server:**
> ```bash
> npm install -g @microsoft/azure-devops-mcp
> ```
>
> **Step 2 — Configure Claude Code:**
>
> Add the following to your Claude Code MCP configuration (usually `~/.claude/mcp_servers.json` or via Claude Code Settings → MCP Servers):
>
> ```json
> {
>   "azure-devops": {
>     "command": "azure-devops-mcp",
>     "env": {
>       "AZURE_DEVOPS_ORG_URL": "https://dev.azure.com/your-org",
>       "AZURE_DEVOPS_AUTH_TYPE": "pat",
>       "AZURE_DEVOPS_TOKEN": "your-personal-access-token"
>     }
>   }
> }
> ```
>
> Replace `your-org` with your Azure DevOps organization name and `your-personal-access-token` with a PAT that has the following scopes:
> - **Work Items:** Read & Write
> - **Code:** Read
> - **Pull Requests:** Read & Write
>
> **Generate a PAT:** Azure DevOps → User Settings → Personal Access Tokens → New Token
>
> **Step 3 — Restart Claude Code** to load the new MCP server.
>
> **Verify:**
> After restarting, run `/dev-setup` again to confirm the status shows FOUND.
>
> Repository: https://github.com/microsoft/azure-devops-mcp

---

## Step 6 — Final Status

After presenting all installation instructions, say:

> Once you have installed the missing dependencies, restart Claude Code and run `/dev-setup` again to confirm everything is ready.
>
> When both show FOUND, run `/dev-workitem <workItemId>` to start your first DevPilot workflow.
