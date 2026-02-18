---
description: Manage Mailman 2 mailing lists (add/remove members, list membership, change settings)
argument-hint: [natural language request, e.g. "add user@example.com to announce"]
allowed-tools:
  - Read
  - "Bash(ssh:*)"
  - "Bash(printf:*)"
  - "Bash(echo:*)"
  - AskUserQuestion
---

# Mailman 2 List Management

Interpret the user's natural language request and execute the appropriate Mailman 2 operation.

## Step 1: Load Configuration

Read the settings file at `.claude/mailman2-admin.local.md` to get server configuration.

If the file does not exist:
1. Read the template from `${CLAUDE_PLUGIN_ROOT}/settings-template.md`
2. Show the user the template and explain they need to create `.claude/mailman2-admin.local.md` in their project
3. Stop and wait for the user to create the file

Parse the YAML frontmatter to extract:
- `default_server` — which server to use
- `servers.<name>.ssh_alias` — SSH connection alias
- `servers.<name>.mailman_bin` — path to Mailman binaries
- `servers.<name>.default_domain` — default email domain

## Step 2: Parse the Request

The user's request is: $ARGUMENTS

Determine the operation from the request:
- **"add"** → add_members
- **"remove" / "delete" / "unsubscribe"** → remove_members (CONFIRM FIRST)
- **"list members" / "who is on" / "show members"** → list_members
- **"find" / "search" / "which lists"** → find_member
- **"list all" / "show lists" / "what lists"** → list_lists
- **"sync"** → sync_members (CONFIRM FIRST)
- **"set" / "change" / "enable" / "disable" + setting name** → member settings change
- **"status" / "info" / "details"** → query member settings

If the request is ambiguous, ask the user to clarify using AskUserQuestion.

## Step 3: Resolve List Names

When the user mentions a list:
- If just a name like "announce", use it directly as the Mailman list name
- If a full email like "announce@lists.example.com", extract the local part ("announce")
- If unclear which list, run `list_lists -b` on the server and help the user pick

## Step 4: Execute

Build and run the appropriate SSH command using the mailman2-cli skill for correct syntax.

**For destructive operations (remove, sync):**
1. Show exactly what will happen (who will be removed/added)
2. Use AskUserQuestion to get explicit confirmation
3. Only execute after user confirms

**For batch operations (multiple emails):**
1. Show the count and list of addresses
2. Confirm before proceeding

**For add operations:**
- Default to `-w n -a n` (no welcome message, no admin notification) unless the user requests otherwise

**For remove operations:**
- Default to `-n -N` (no goodbye message, no admin notification) unless the user requests otherwise

## Step 5: Report Results

After execution:
- Show the command output
- Summarize what happened (e.g., "Added 3 members to announce list")
- If errors occurred, explain and suggest fixes

If no arguments were provided ($ARGUMENTS is empty), ask the user what they want to do with their mailing lists.
