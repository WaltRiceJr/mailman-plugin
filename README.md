# mailman

A Claude Code plugin for managing Mailman 2 mailing lists via SSH.

## Features

- Add/remove members (single or batch)
- List all mailing lists on a server
- List members of a list
- Find which lists a user belongs to
- Sync list membership (bulk add/remove)
- Change member settings (digest, nomail, moderation, hide, etc.)
- Multi-server support via config file
- Confirmation prompts for destructive operations

## Prerequisites

- SSH access to a server running Mailman 2
- SSH alias configured in `~/.ssh/config` (or equivalent)
- Root or appropriate permissions to run Mailman CLI utilities

## Installation

```bash
claude --plugin-dir /path/to/mailman-plugin
```

## Configuration

Create `.claude/mailman.local.md` in your project directory:

```markdown
---
default_server: my-server
servers:
  my-server:
    ssh_alias: my-ssh-alias
    mailman_bin: /usr/lib/mailman/bin
    default_domain: lists.example.com
---

# Mailman2 Admin Settings

Server configuration for Mailman 2 list management.
```

Add multiple servers by adding entries under `servers:`.

## Usage

```
/mailman show all lists
/mailman list members of announce
/mailman add user@example.com to announce
/mailman add alice@example.com, bob@example.com to newsletter
/mailman remove user@example.com from announce
/mailman find user@example.com
/mailman sync announce with alice@example.com, bob@example.com, carol@example.com
/mailman set nomail for user@example.com on announce
/mailman set digest mode for user@example.com on announce
/mailman show settings for user@example.com on announce
```

Or just `/mailman` with no arguments to describe what you need interactively.

## Safety

Destructive operations (remove, sync) always ask for confirmation before executing.
