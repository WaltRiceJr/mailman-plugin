# Mailman 2 Member Settings Reference

Detailed patterns for manipulating member settings via `withlist` and related tools.

## Member Option Flags

Mailman 2 stores member options as bitwise flags. The key options are:

| Option | Flag Constant | Description |
|--------|---------------|-------------|
| Digest | `mm_cfg.Digests` | Receive digest instead of individual messages |
| DisableDelivery | N/A | Uses delivery status, not a flag |
| Moderate | `mm_cfg.Moderate` | Posts held for moderation |
| Hide | `mm_cfg.ConcealSubscription` | Hidden from member list |
| NoDupes | `mm_cfg.DontReceiveDuplicates` | Suppress duplicates |
| Ack | `mm_cfg.AcknowledgePosts` | Acknowledge posts |
| NotMeToo | `mm_cfg.DontReceiveOwnPosts` | Don't receive own posts |
| Plain | `mm_cfg.DisableMime` | Receive plain text digests |

## Using withlist for Member Settings

The `withlist` command runs Python code against a locked list object. The general pattern:

```bash
ssh ALIAS "python -c '
import paths
from Mailman import mm_cfg, MailList

m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    # operations here
    m.Save()
finally:
    m.Unlock()
' 2>/dev/null"
```

### Set Digest Mode

Enable digest for a member:

```bash
ssh ALIAS "python -c '
import paths
from Mailman import mm_cfg, MailList

m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    m.setMemberOption(\"user@example.com\", mm_cfg.Digests, 1)
    m.Save()
finally:
    m.Unlock()
'"
```

Disable digest (back to regular):

```bash
ssh ALIAS "python -c '
import paths
from Mailman import mm_cfg, MailList

m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    m.setMemberOption(\"user@example.com\", mm_cfg.Digests, 0)
    m.Save()
finally:
    m.Unlock()
'"
```

### Set Delivery Status (nomail)

Mailman uses delivery status codes:
- `MemberAdaptor.ENABLED` (0) — delivery enabled
- `MemberAdaptor.BYUSER` (1) — disabled by user
- `MemberAdaptor.BYADMIN` (2) — disabled by admin
- `MemberAdaptor.BYBOUNCE` (3) — disabled by bounce

Disable delivery (set nomail by admin):

```bash
ssh ALIAS "python -c '
import paths
from Mailman import MailList
from Mailman.MemberAdaptor import BYADMIN

m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    m.setDeliveryStatus(\"user@example.com\", BYADMIN)
    m.Save()
finally:
    m.Unlock()
'"
```

Re-enable delivery:

```bash
ssh ALIAS "python -c '
import paths
from Mailman import MailList
from Mailman.MemberAdaptor import ENABLED

m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    m.setDeliveryStatus(\"user@example.com\", ENABLED)
    m.Save()
finally:
    m.Unlock()
'"
```

### Set Moderation

Enable moderation for a member:

```bash
ssh ALIAS "python -c '
import paths
from Mailman import mm_cfg, MailList

m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    m.setMemberOption(\"user@example.com\", mm_cfg.Moderate, 1)
    m.Save()
finally:
    m.Unlock()
'"
```

### Conceal Subscription (Hide)

Hide member from member list:

```bash
ssh ALIAS "python -c '
import paths
from Mailman import mm_cfg, MailList

m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    m.setMemberOption(\"user@example.com\", mm_cfg.ConcealSubscription, 1)
    m.Save()
finally:
    m.Unlock()
'"
```

### Suppress Duplicates

```bash
ssh ALIAS "python -c '
import paths
from Mailman import mm_cfg, MailList

m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    m.setMemberOption(\"user@example.com\", mm_cfg.DontReceiveDuplicates, 1)
    m.Save()
finally:
    m.Unlock()
'"
```

### Acknowledge Posts

```bash
ssh ALIAS "python -c '
import paths
from Mailman import mm_cfg, MailList

m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    m.setMemberOption(\"user@example.com\", mm_cfg.AcknowledgePosts, 1)
    m.Save()
finally:
    m.Unlock()
'"
```

### Plain Text Digests

```bash
ssh ALIAS "python -c '
import paths
from Mailman import mm_cfg, MailList

m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    m.setMemberOption(\"user@example.com\", mm_cfg.DisableMime, 1)
    m.Save()
finally:
    m.Unlock()
'"
```

## Querying Member Settings

### Get Delivery Status

```bash
ssh ALIAS "python -c '
import paths
from Mailman import MailList

m = MailList.MailList(\"LISTNAME\", lock=False)
status = m.getDeliveryStatus(\"user@example.com\")
labels = {0: \"enabled\", 1: \"by user\", 2: \"by admin\", 3: \"by bounce\", 4: \"unknown\"}
print(\"Delivery:\", labels.get(status, \"unknown\"))
'"
```

### Get All Options for a Member

```bash
ssh ALIAS "python -c '
import paths
from Mailman import mm_cfg, MailList

m = MailList.MailList(\"LISTNAME\", lock=False)
email = \"user@example.com\"
opts = {
    \"digest\": m.getMemberOption(email, mm_cfg.Digests),
    \"moderate\": m.getMemberOption(email, mm_cfg.Moderate),
    \"hide\": m.getMemberOption(email, mm_cfg.ConcealSubscription),
    \"nodupes\": m.getMemberOption(email, mm_cfg.DontReceiveDuplicates),
    \"ack\": m.getMemberOption(email, mm_cfg.AcknowledgePosts),
    \"notmetoo\": m.getMemberOption(email, mm_cfg.DontReceiveOwnPosts),
    \"plain\": m.getMemberOption(email, mm_cfg.DisableMime),
}
status = m.getDeliveryStatus(email)
labels = {0: \"enabled\", 1: \"by user\", 2: \"by admin\", 3: \"by bounce\"}
print(\"delivery:\", labels.get(status, \"unknown\"))
for k, v in opts.items():
    print(f\"{k}: {bool(v)}\")
'"
```

## Bulk Setting Changes

### Set Option for All Members

```bash
ssh ALIAS "python -c '
import paths
from Mailman import mm_cfg, MailList

m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    for member in m.getMembers():
        m.setMemberOption(member, mm_cfg.ConcealSubscription, 1)
    m.Save()
finally:
    m.Unlock()
'"
```

### Disable Delivery for Multiple Members

```bash
ssh ALIAS "python -c '
import paths
from Mailman import MailList
from Mailman.MemberAdaptor import BYADMIN

emails = [\"user1@example.com\", \"user2@example.com\", \"user3@example.com\"]
m = MailList.MailList(\"LISTNAME\", lock=True)
try:
    for email in emails:
        m.setDeliveryStatus(email, BYADMIN)
    m.Save()
finally:
    m.Unlock()
'"
```

## Change Member Address

```bash
ssh ALIAS MAILMAN_BIN/change_member_address -l LISTNAME old@example.com new@example.com
```

Flags:
- `-l LISTNAME` — specific list (omit to change across all lists)
- `-n` — no-change mode (dry run)

## Notes

- Always use `lock=True` when modifying, `lock=False` when only reading
- The `try/finally/Unlock()` pattern prevents stale locks if errors occur
- The `import paths` line is needed to set up Mailman's Python path
- Some servers may need the full path to the Mailman Python installation
- Test commands with a single member before running bulk operations
