---
name: mailman2-cli
description: This skill should be used when the user asks to "manage mailing lists", "add member to list", "subscribe someone to a list", "remove member from list", "unsubscribe someone from a list", "list members", "who is on a list", "find user on lists", "show all mailing lists", "sync list membership", "change member settings", "set digest mode", "set nomail", "bulk add members", "clone list membership", "mailman administration", or mentions Mailman 2 CLI utilities, mailing list administration, email list management, or running mailman commands via SSH on a remote server.
version: 0.1.0
---

# Mailman 2 CLI Administration

Provide guidance and execute Mailman 2 mailing list management operations via SSH on remote servers.

## Configuration

Read server configuration from `.claude/mailman2-admin.local.md` in the project directory. The YAML frontmatter contains:

```yaml
default_server: my-server
servers:
  my-server:
    ssh_alias: my-ssh-alias
    mailman_bin: /usr/lib/mailman/bin
    default_domain: lists.example.com
```

If no settings file exists, inform the user and provide the template from `${CLAUDE_PLUGIN_ROOT}/settings-template.md`.

## Command Execution Pattern

All Mailman commands run remotely via SSH:

```
ssh <ssh_alias> <mailman_bin>/<command> [arguments]
```

## Mailman 2 CLI Reference

### list_lists — Enumerate All Lists

```bash
ssh ALIAS MAILMAN_BIN/list_lists
```

Output format: one list per line with description. To get bare list names:

```bash
ssh ALIAS MAILMAN_BIN/list_lists -b
```

### list_members — Show List Membership

Basic (emails only):

```bash
ssh ALIAS MAILMAN_BIN/list_members LISTNAME
```

With delivery status info, use flags:
- `--regular` — regular (non-digest) members only
- `--digest` — digest members only
- `--nomail` — members with delivery disabled
- `--fullnames` — include display names

```bash
ssh ALIAS MAILMAN_BIN/list_members --regular LISTNAME
ssh ALIAS MAILMAN_BIN/list_members --digest LISTNAME
ssh ALIAS MAILMAN_BIN/list_members --nomail LISTNAME
```

### add_members — Add Members to a List

Add members via stdin. One email per line.

```bash
echo "user@example.com" | ssh ALIAS MAILMAN_BIN/add_members -r - LISTNAME
```

Key flags:
- `-r -` — read regular members from stdin
- `-d -` — read digest members from stdin
- `-w n` — welcome message: y(es) or n(o)
- `-a n` — admin notification: y(es) or n(o)

For batch adds, pipe multiple emails:

```bash
printf "alice@example.com\nbob@example.com\ncarol@example.com" | ssh ALIAS MAILMAN_BIN/add_members -r - -w n -a n LISTNAME
```

### remove_members — Remove Members from a List

**DESTRUCTIVE: Confirm with user before executing.**

Remove single member:

```bash
ssh ALIAS MAILMAN_BIN/remove_members -n -N LISTNAME user@example.com
```

Remove multiple members:

```bash
ssh ALIAS MAILMAN_BIN/remove_members -n -N LISTNAME user1@example.com user2@example.com
```

Remove from stdin:

```bash
printf "user1@example.com\nuser2@example.com" | ssh ALIAS MAILMAN_BIN/remove_members -n -N -f - LISTNAME
```

Key flags:
- `-n` — no goodbye message to removed member
- `-N` — no admin notification
- `-f -` — read addresses from stdin
- `--all` — remove ALL members (very destructive, always confirm)

### find_member — Search Across All Lists

Find which lists a member belongs to:

```bash
ssh ALIAS MAILMAN_BIN/find_member PATTERN
```

PATTERN is a case-insensitive regex. Examples:

```bash
ssh ALIAS MAILMAN_BIN/find_member user@example.com
ssh ALIAS MAILMAN_BIN/find_member ".*@example.com"
```

Output groups results by matching address, listing each list they belong to.

### sync_members — Sync Membership with File

**DESTRUCTIVE: Confirm with user before executing.**

Synchronize list membership with a file of addresses. Adds missing members, removes unlisted members.

```bash
printf "alice@example.com\nbob@example.com" | ssh ALIAS MAILMAN_BIN/sync_members -w=no -g=no -a=no -f - LISTNAME
```

Key flags:
- `-f -` — read desired member list from stdin
- `-w=no` — no welcome messages
- `-g=no` — no goodbye messages
- `-a=no` — no admin notifications
- `-d=no` — don't remove (additions only)

### withlist — Change Member Settings

Change individual member options (digest, nomail, moderation, hide, nodupes, ack, plain text) using Python scripts executed via SSH. Supported operations include:

- Setting digest mode on/off
- Disabling/enabling delivery (nomail)
- Enabling/disabling moderation for a member
- Concealing subscription (hide from member list)
- Suppressing duplicate messages
- Querying all settings for a member
- Bulk setting changes across all members

Refer to `references/member-settings.md` for complete withlist patterns covering all member options, bulk changes, and querying.

### change_member_address — Change a Member's Email

```bash
ssh ALIAS MAILMAN_BIN/change_member_address -l LISTNAME old@example.com new@example.com
```

Flags:
- `-l LISTNAME` — specific list (omit to change across all lists)
- `-n` — dry run (no changes made)

### clone_member — Copy Settings Between Lists

Not a built-in utility. To replicate membership from one list to another:

```bash
ssh ALIAS MAILMAN_BIN/list_members SRCLIST | ssh ALIAS MAILMAN_BIN/add_members -r - -w n -a n DESTLIST
```

## Safety Rules

1. **Always confirm before executing:** `remove_members`, `sync_members`, `--all` flags
2. **Show the user what will happen** before running destructive commands
3. **For batch operations**, show the count and a sample of addresses before proceeding
4. **Never run** `remove_members --all` without explicit user confirmation
5. **For sync_members**, show who will be added AND removed before executing

## List Name Resolution

When the user provides a list name:
- If just a name (e.g., "announce"), use it directly as the LISTNAME argument
- If a full address (e.g., "announce@lists.example.com"), extract just the local part before the @
- If ambiguous, use `list_lists` to find matching lists and confirm with the user

## Error Handling

Common errors and solutions:
- **"No such list"** — list name is wrong; run `list_lists -b` to find correct name
- **"No such member"** — email not on list; run `find_member` to check
- **SSH connection failure** — verify SSH alias is configured and server is reachable

## Additional Resources

### Reference Files

For detailed member settings manipulation patterns:
- **`references/member-settings.md`** - Complete withlist patterns for all member options
