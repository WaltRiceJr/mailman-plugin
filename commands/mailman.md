---
description: Manage Mailman 2 mailing lists (add/remove members, list membership, change settings, duplicate/rename lists)
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

Read the settings file at `.claude/mailman.local.md` to get server configuration.

If the file does not exist:
1. Read the template from `${CLAUDE_PLUGIN_ROOT}/settings-template.md`
2. Show the user the template and explain they need to create `.claude/mailman.local.md` in their project
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
- **"duplicate" / "copy list" / "clone list"** → duplicate_list (CONFIRM FIRST)
- **"rename" / "rename list"** → rename_list (CONFIRM FIRST — HIGHLY DESTRUCTIVE)

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

**For duplicate operations:**
1. Verify the destination list does not already exist (run `list_lists -b` and check)
2. Ask the user for the admin email, a temporary password, and the email domain for the new list (use `default_domain` from config as default, confirm with user)
3. Show a summary: source list, destination name, admin email, domain. Warn that per-member settings (individual nomail flags, per-member moderation) are NOT preserved and must be re-applied manually if needed
4. Use AskUserQuestion to confirm before proceeding
5. Execute the full `duplicate_list` procedure from the skill reference:
   - Create new list via `newlist -q` with `--urlhost=DOMAIN` to set the list's web host and `--emailhost=DOMAIN` to set the list's email domain
   - Add aliases to `/etc/aliases` and run `sudo newaliases`
   - Export source config via `config_list -o`, import into destination via `config_list -i`
   - Copy regular members (`list_members --regular SRC | add_members -r - DEST`)
   - Copy digest members (`list_members --digest SRC | add_members -d - DEST`)
   - Copy archive mbox from `/var/lib/mailman/archives/private/SRCLIST.mbox/SRCLIST.mbox` to `/var/lib/mailman/archives/private/DESTLIST.mbox/DESTLIST.mbox`
   - Regenerate HTML archives via `arch --wipe DESTLIST`
   - Verify member counts match between source and destination

**For rename operations:**
1. Verify the destination list does not already exist (run `list_lists -b` and check)
2. Ask the user for the admin email, a temporary password, and the email domain for the new list (use `default_domain` from config as default, confirm with user)
3. Show a summary: source list, destination name, admin email, domain. Warn that the original list will be deleted and that per-member settings (individual nomail flags, per-member moderation) are NOT preserved
4. Use AskUserQuestion with an explicit YES confirmation — this is highly destructive
5. Execute the full `duplicate_list` procedure (steps above)
6. After verification succeeds (member counts match), delete the original list via `rmlist` (without `--archives` to preserve old archives as a safety net)
7. Confirm deletion succeeded and show final status

## Step 5: Report Results

After execution:
- Show the command output
- Summarize what happened (e.g., "Added 3 members to announce list")
- If errors occurred, explain and suggest fixes

If no arguments were provided ($ARGUMENTS is empty), ask the user what they want to do with their mailing lists.
