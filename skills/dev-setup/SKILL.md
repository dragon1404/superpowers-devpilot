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
> **Step 1 — Register the Superpowers marketplace:**
> ```
> /plugin marketplace add obra/superpowers-marketplace
> ```
>
> **Step 2 — Install the plugin:**
> ```
> /plugin install superpowers@superpowers-marketplace
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
> claude mcp add azure-devops --scope user -- npx -y @azure-devops/mcp {org} --authentication pat
> ```
>
> This registers the Azure DevOps MCP server for your user account using your org `{org}`.
>
> **Authentication:** You need to set `PERSONAL_ACCESS_TOKEN` in `~/.claude/settings.json`.
> Generate a PAT at: Azure DevOps → User Settings → Personal Access Tokens → New Token, with these scopes:
> - **Work Items:** Read & Write
> - **Code:** Read
> - **Pull Requests:** Read & Write
>
> **Option A — Automatic (recommended):** Run this script in your terminal:
> ```bash
> read -p "ADO email: " ado_email && read -sp "ADO PAT: " ado_pat && echo && \
> token=$(echo -n "$ado_email:$ado_pat" | base64) && \
> python3 - <<EOF
> import json, os
> path = os.path.expanduser("~/.claude/settings.json")
> cfg = json.load(open(path)) if os.path.exists(path) else {}
> cfg.setdefault("env", {})["PERSONAL_ACCESS_TOKEN"] = "$token"
> json.dump(cfg, open(path, "w"), indent=2)
> print("✅ PERSONAL_ACCESS_TOKEN written to ~/.claude/settings.json")
> EOF
> ```
>
> **Option B — Manual:** Edit `~/.claude/settings.json` and add:
> ```json
> {
>   "env": {
>     "PERSONAL_ACCESS_TOKEN": "<base64encoded email:pat>"
>   }
> }
> ```
> The value is your email and PAT joined with `:` and base64-encoded:
> ```bash
> echo -n "you@example.com:yourPAT" | base64
> ```
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
