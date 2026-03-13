# Processing Uncaught Bounce Notifications

When Mailman 2 can't parse a bounce, it forwards the original bounce message as a `message/rfc822` MIME attachment inside an "Uncaught bounce notification" email to the list owner. The bounced email address is buried in that attachment, not in the top-level body.

## Prerequisites

- `gog` CLI ([gogcli](https://github.com/steipete/gogcli)) installed and authenticated
- Python 3 available

## Step-by-step

### 1. Search for bounce notifications

```bash
gog gmail messages search 'subject:"Uncaught bounce notification" after:2026/3/12' --json
```

This returns message objects with `id` fields. Collect all the message IDs.

### 2. Get the full message and extract bounced addresses

The message MIME structure looks like:

```
multipart/mixed
  text/plain          ← Mailman boilerplate (useless)
  message/rfc822      ← The actual bounce report
    text/plain        ← Contains the bounced address and error
```

Use `gog gmail get <messageId> --format=full --json` to get the full payload, then walk the MIME tree to find the `text/plain` nested inside the `message/rfc822` part.

### 3. Parse bounced emails from the attachment text

The bounce text varies by mail server but common patterns include:

- `Recipient: [SMTP:user@domain.com]` (MailEnable format)
- `Final-Recipient: rfc822; user@domain.com` (DSN format)
- `Original-Recipient: rfc822; user@domain.com` (DSN format)

### Complete extraction script

Save as `/tmp/extract_bounced.py`:

```python
import json, sys, base64, re

data = json.load(sys.stdin)
payload = data['message']['payload']

def extract_nested(part, is_rfc822_child=False):
    mime = part.get('mimeType', '')
    body = part.get('body', {})
    b64data = body.get('data', '')
    if is_rfc822_child and b64data:
        return base64.urlsafe_b64decode(b64data + '==').decode('utf-8', errors='replace')
    result = ''
    for p in part.get('parts', []):
        child_is_rfc822 = is_rfc822_child or (mime == 'message/rfc822')
        result += extract_nested(p, child_is_rfc822) or ''
    return result

text = extract_nested(payload) or ''

emails = set()
for m in re.finditer(r'Recipient:\s*\[SMTP:([^\]]+)\]', text):
    emails.add(m.group(1).lower())
for m in re.finditer(r'Final-Recipient:\s*rfc822;\s*(\S+)', text):
    emails.add(m.group(1).lower())
for m in re.finditer(r'Original-Recipient:\s*rfc822;\s*(\S+)', text):
    emails.add(m.group(1).lower())

# Filter out the list address itself
emails = {e for e in emails if 'lists.injuryfree.org' not in e}

for e in sorted(emails):
    print(e)
```

### Run it across all bounce messages

```bash
for id in <space-separated message IDs>; do
  gog gmail get "$id" --format=full --json 2>/dev/null \
    | python3 /tmp/extract_bounced.py
done | sort -u
```

This produces a deduplicated list of bounced email addresses ready for removal.

## Important notes

- `--format=raw` does NOT work reliably for these messages (base64 decode errors on some). Use `--format=full --json` instead.
- The `--format=full` JSON nests the attachment content as base64url-encoded `data` fields in the `payload.parts` tree.
- The Claude.ai Gmail MCP connector does NOT surface attachment contents — only `gog` works for this.
- All bounces seen so far are 550 permanent failures (address rejected / user unknown), which means the addresses should be removed, not just disabled.
