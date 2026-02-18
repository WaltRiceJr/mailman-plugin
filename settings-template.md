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
Add additional servers by adding entries under `servers:`.
