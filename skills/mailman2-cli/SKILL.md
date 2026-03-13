---
name: mailman2-cli
description: This skill should be used when the user asks to "manage mailing lists", "add member to list", "subscribe someone to a list", "remove member from list", "unsubscribe someone from a list", "list members", "who is on a list", "find user on lists", "show all mailing lists", "sync list membership", "change member settings", "set digest mode", "set nomail", "bulk add members", "clone list membership", "rename list", "duplicate list", "copy list", "mailman administration", or mentions Mailman 2 CLI utilities, mailing list administration, email list management, or running mailman commands via SSH on a remote server.
version: 0.1.0
---

# Mailman 2 CLI Administration

Provide guidance and execute Mailman 2 mailing list management operations via SSH on remote servers.

## Configuration

Read server configuration from `.claude/mailman.local.md` in the project directory. The YAML frontmatter contains:

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

Not a built-in utility. To replicate membership from one list to another (single SSH connection):

```bash
ssh ALIAS "MAILMAN_BIN/list_members SRCLIST | MAILMAN_BIN/add_members -r - -w n -a n DESTLIST"
```

### newlist — Create a New List

```bash
ssh ALIAS sudo MAILMAN_BIN/newlist -q --urlhost=DOMAIN --emailhost=DOMAIN LISTNAME ADMIN_EMAIL PASSWORD
```

Key flags:
- `-q` — quiet mode (suppress alias output to stdout)
- `--urlhost=DOMAIN` — set the list's web host (e.g., `lists.example.com`)
- `--emailhost=DOMAIN` — set the list's email domain (e.g., `lists.example.com`)

**Important:** After creating a list, the Mailman alias entries must be added to `/etc/aliases` and `sudo newaliases` must be run. The alias lines follow this pattern:

```
## LISTNAME mailing list
LISTNAME:              "|MAILMAN_BIN/../mail/mailman post LISTNAME"
LISTNAME-admin:        "|MAILMAN_BIN/../mail/mailman admin LISTNAME"
LISTNAME-bounces:      "|MAILMAN_BIN/../mail/mailman bounces LISTNAME"
LISTNAME-confirm:      "|MAILMAN_BIN/../mail/mailman confirm LISTNAME"
LISTNAME-join:         "|MAILMAN_BIN/../mail/mailman join LISTNAME"
LISTNAME-leave:        "|MAILMAN_BIN/../mail/mailman leave LISTNAME"
LISTNAME-owner:        "|MAILMAN_BIN/../mail/mailman owner LISTNAME"
LISTNAME-request:      "|MAILMAN_BIN/../mail/mailman request LISTNAME"
LISTNAME-subscribe:    "|MAILMAN_BIN/../mail/mailman subscribe LISTNAME"
LISTNAME-unsubscribe:  "|MAILMAN_BIN/../mail/mailman unsubscribe LISTNAME"
```

### config_list — Export/Import List Configuration

Export configuration to a file:

```bash
ssh ALIAS MAILMAN_BIN/config_list -o /tmp/LISTNAME.cfg LISTNAME
```

Import configuration from a file:

```bash
ssh ALIAS MAILMAN_BIN/config_list -i /tmp/LISTNAME.cfg LISTNAME
```

**Note:** `config_list` exports/imports list-level settings (posting policy, moderation defaults, subject prefix, etc.). Per-member settings stored in the list pickle (individual nomail flags, per-member moderation flags) are NOT preserved by config export/import.

### rmlist — Delete a List

**DESTRUCTIVE: Always confirm with user before executing.**

```bash
ssh ALIAS sudo MAILMAN_BIN/rmlist LISTNAME
```

Key flags:
- `--archives` — also delete the list's archive files

Without `--archives`, the mbox and HTML archives are preserved on disk.

### arch — Regenerate HTML Archives

Rebuild a list's HTML archives from its mbox file:

```bash
ssh ALIAS MAILMAN_BIN/arch --wipe LISTNAME
```

Key flags:
- `--wipe` — clear existing HTML archives and regenerate from scratch

## Compound Operations

### duplicate_list — Copy a List (Without Removing Original)

This is not a built-in Mailman command. It is a multi-step procedure. Steps are batched to minimize SSH connections (4 connections total instead of ~9).

1. **Create new list:**
   ```bash
   ssh ALIAS sudo MAILMAN_BIN/newlist -q --urlhost=DOMAIN --emailhost=DOMAIN DESTLIST ADMIN_EMAIL PASSWORD
   ```
2. **Add aliases** for the new list to `/etc/aliases` and run `sudo newaliases`
3. **Export and import config** (single connection):
   ```bash
   ssh ALIAS "MAILMAN_BIN/config_list -o /tmp/SRCLIST.cfg SRCLIST && MAILMAN_BIN/config_list -i /tmp/SRCLIST.cfg DESTLIST"
   ```
4. **Copy all members — regular and digest** (single connection):
   ```bash
   ssh ALIAS "MAILMAN_BIN/list_members --regular SRCLIST | MAILMAN_BIN/add_members -r - -w n -a n DESTLIST && MAILMAN_BIN/list_members --digest SRCLIST | MAILMAN_BIN/add_members -d - -w n -a n DESTLIST"
   ```
5. **Copy archive mbox and regenerate HTML archives** (single connection):
   ```bash
   ssh ALIAS "sudo cp /var/lib/mailman/archives/private/SRCLIST.mbox/SRCLIST.mbox /var/lib/mailman/archives/private/DESTLIST.mbox/DESTLIST.mbox && MAILMAN_BIN/arch --wipe DESTLIST"
   ```
6. **Verify member counts match** (single connection):
   ```bash
   ssh ALIAS "echo 'SOURCE:' && MAILMAN_BIN/list_members SRCLIST | wc -l && echo 'DEST:' && MAILMAN_BIN/list_members DESTLIST | wc -l"
   ```

### rename_list — Rename a List (Duplicate + Delete Original)

Same as `duplicate_list` (steps 1–6 above), plus:

7. **Delete original list** (only after verification succeeds):
    ```bash
    ssh ALIAS sudo MAILMAN_BIN/rmlist SRCLIST
    ```

**Note:** Archives of the old list are preserved on disk unless `--archives` is passed to `rmlist`. It is recommended to leave old archives in place as a safety net.

**Known limitation:** Per-member settings (individual nomail, moderation flags) stored in the list pickle are NOT preserved by `config_list` export/import. These must be re-applied manually after the rename if needed.

## SSH Connection Optimization

**IMPORTANT:** Minimize the number of SSH connections to avoid being rate-limited or blocked by the server. Batch multiple commands into a single SSH session wherever possible.

### Batching Principles

1. **Chain independent commands** with `&&` inside a quoted SSH command:
   ```bash
   ssh ALIAS "MAILMAN_BIN/cmd1 ARGS && MAILMAN_BIN/cmd2 ARGS"
   ```

2. **Pipe dependent commands** inside a single SSH session:
   ```bash
   ssh ALIAS "MAILMAN_BIN/list_members SRCLIST | MAILMAN_BIN/add_members -r - DESTLIST"
   ```

3. **Combine pipes and chains** for complex operations:
   ```bash
   ssh ALIAS "MAILMAN_BIN/cmd1 | MAILMAN_BIN/cmd2 && MAILMAN_BIN/cmd3 | MAILMAN_BIN/cmd4"
   ```

4. **Mix sudo and non-sudo** commands in one session:
   ```bash
   ssh ALIAS "sudo cmd1 && cmd2"
   ```

5. **Batch Python member setting changes** — modify multiple members or multiple settings in a single `python -c` script rather than one SSH call per change.

### When Separate Connections Are Needed

- When a step requires user confirmation before the next step
- When you need to inspect output before deciding the next action

## Safety Rules

1. **Always confirm before executing:** `remove_members`, `sync_members`, `--all` flags
2. **Show the user what will happen** before running destructive commands
3. **For batch operations**, show the count and a sample of addresses before proceeding
4. **Never run** `remove_members --all` without explicit user confirmation
5. **For sync_members**, show who will be added AND removed before executing
6. **For duplicate_list**, confirm the source list, destination name, and admin email before proceeding
7. **For rename_list**, require the user to explicitly confirm with YES before executing — this is a highly destructive operation that deletes the original list

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
