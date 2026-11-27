---
name: tmpmail
description: "Temporary email for pentesting. Use when user needs disposable/temp email, check inbox, or read temp emails. Triggers on: tmpmail, tempmail, temp mail, temporary email, disposable email, burner email, temp inbox. Args: new, list, read N, wait, status, delete."
---

# Temporary Email Skill

Multi-provider temporary email for pentesting. Prioritizes TempMail.lol (domains not on common blocklists).

## Args Reference

| Command | Description |
|---------|-------------|
| `new` | Create new temp email (auto-select best provider) |
| `new lol` | Create via TempMail.lol (uncommon domains, recommended) |
| `new tm` | Create via Mail.tm |
| `new gm` | Create via Guerrilla Mail |
| `list` | List inbox messages (numbered) |
| `read N` | Read message number N (from list) |
| `wait` | Poll inbox every 5s until new message arrives (max 2min) |
| `wait 10` | Poll every 10 seconds |
| `status` | Show current email, provider, session info |
| `delete` | Destroy current email and clear session |

If no args, show status if session exists, otherwise show this help. The skill is invoked as `/tmpmail`.

## JSON Parsing

Use `python3 -c` for all JSON parsing (jq may not be installed). Pattern:

```bash
echo "$JSON" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['key'])"
```

## Session File

Path: `/tmp/tempmail_session.json`

```json
{
  "provider": "lol|tm|gm",
  "email": "user@domain.com",
  "created_at": "2026-11-27T10:00:00",
  "token": "...",
  "password": "...",
  "account_id": "...",
  "sid_token": "...",
  "cached_emails": []
}
```

Always read session at start of `list`, `read`, `wait`, `status`, `delete`. If no session exists, tell user to run `new` first.

## Provider Details

### TempMail.lol (Primary - `lol`)
- Domains like `gloatinggoat.com`, `repairhubde.com`, `brimac.net` are NOT on common blocklists
- Each address gets unique random subdomain (e.g. `user@abc.gloatinggoat.com`)
- **IMPORTANT**: Emails are consumed/deleted on API fetch. MUST cache them in session file after fetching.
- Endpoints:
  - `POST https://api.tempmail.lol/v2/inbox/create` -> `{"address":"...","token":"..."}`
  - `GET https://api.tempmail.lol/v2/inbox?token=TOKEN` -> `{"emails":[...],"expired":false}`

### Mail.tm (Secondary - `tm`)
- Current domain: rotates (check `GET /domains`)
- Domains ARE on common blocklists but rotate periodically
- Best API with full CRUD
- Endpoints:
  - `GET https://api.mail.tm/domains` -> domain in `hydra:member[0].domain`
  - `POST https://api.mail.tm/accounts` body: `{"address":"USER@DOMAIN","password":"PASS"}`
  - `POST https://api.mail.tm/token` body: `{"address":"...","password":"..."}` -> `{"token":"..."}`
  - `GET https://api.mail.tm/messages` with `Authorization: Bearer TOKEN` -> messages in `hydra:member[]`
  - `GET https://api.mail.tm/messages/ID` -> full message with `.text` and `.html[]`

### Guerrilla Mail (Tertiary - `gm`)
- Domain `guerrillamailblock.com` IS on blocklists
- Addresses expire in 60 min (extendable +1hr)
- Endpoints (all `https://api.guerrillamail.com/ajax.php`):
  - `?f=get_email_address&ip=127.0.0.1&agent=claude-tempmail` -> `{"email_addr":"...","sid_token":"..."}`
  - `?f=check_email&seq=0&sid_token=SID` -> `{"list":[...]}`
  - `?f=fetch_email&email_id=ID&sid_token=SID` -> message with `mail_body`

## Implementation Instructions

### `new` / `new [provider]`

Default provider: `lol` (TempMail.lol) unless specified.

**TempMail.lol (`lol`):**
```bash
RESULT=$(curl -s -X POST https://api.tempmail.lol/v2/inbox/create)
EMAIL=$(echo "$RESULT" | python3 -c "import sys,json; print(json.load(sys.stdin)['address'])")
TOKEN=$(echo "$RESULT" | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
```

**Mail.tm (`tm`):**
```bash
DOMAIN=$(curl -s https://api.mail.tm/domains | python3 -c "import sys,json; print(json.load(sys.stdin)['hydra:member'][0]['domain'])")
USER=$(cat /dev/urandom | tr -dc 'a-z0-9' | head -c 8)
PASS=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 16)
EMAIL="${USER}@${DOMAIN}"
ACCT=$(curl -s -X POST https://api.mail.tm/accounts -H "Content-Type: application/json" -d "{\"address\":\"${EMAIL}\",\"password\":\"${PASS}\"}")
ACCT_ID=$(echo "$ACCT" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
TOKEN=$(curl -s -X POST https://api.mail.tm/token -H "Content-Type: application/json" -d "{\"address\":\"${EMAIL}\",\"password\":\"${PASS}\"}" | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
```

**Guerrilla Mail (`gm`):**
```bash
RESULT=$(curl -s "https://api.guerrillamail.com/ajax.php?f=get_email_address&ip=127.0.0.1&agent=claude-tempmail")
EMAIL=$(echo "$RESULT" | python3 -c "import sys,json; print(json.load(sys.stdin)['email_addr'])")
SID=$(echo "$RESULT" | python3 -c "import sys,json; print(json.load(sys.stdin)['sid_token'])")
```

After creating, save to `/tmp/tempmail_session.json` and display:
```
[TempMail] Email created!
[TempMail] Provider : TempMail.lol
[TempMail] Address  : user@abc.gloatinggoat.com
[TempMail] Status   : Active
```

### `list`

Fetch inbox based on provider. Display numbered list.

**TempMail.lol**: Fetch and CACHE emails in session (they get deleted from server):
```bash
RESULT=$(curl -s "https://api.tempmail.lol/v2/inbox?token=${TOKEN}")
```
Parse `emails` array. Each email has: `from`, `subject`, `body`, `date`, `html`. Append new emails to `cached_emails` in session file.

**Mail.tm**: Fetch messages list:
```bash
RESULT=$(curl -s https://api.mail.tm/messages -H "Authorization: Bearer ${TOKEN}")
```
Parse `hydra:member` array. Each has: `id`, `from.address`, `from.name`, `subject`, `intro`, `createdAt`.

**Guerrilla**: Check inbox:
```bash
RESULT=$(curl -s "https://api.guerrillamail.com/ajax.php?f=check_email&seq=0&sid_token=${SID}")
```
Parse `list` array. Each has: `mail_id`, `mail_from`, `mail_subject`, `mail_excerpt`, `mail_date`.

Display format:
```
[TempMail] Inbox: user@abc.gloatinggoat.com (TempMail.lol)

  #  | From                | Subject                    | Date
 ----+---------------------+----------------------------+--------------------
  1  | noreply@target.com  | Verify your email          | 2026-11-27 10:30
  2  | info@target.com     | Welcome!                   | 2026-11-27 10:31

  2 messages
```

If no messages: `[TempMail] Inbox empty.`

### `read N`

N is the message number from `list` output (1-indexed).

**TempMail.lol**: Read from `cached_emails[N-1]` in session file. Body is in `body` (text) or `html` field.

**Mail.tm**: Get message ID from last list, then:
```bash
MSG=$(curl -s "https://api.mail.tm/messages/${MSG_ID}" -H "Authorization: Bearer ${TOKEN}")
```
Body in `.text` (prefer) or `.html[]`.

**Guerrilla**: Get mail_id from last list, then:
```bash
MSG=$(curl -s "https://api.guerrillamail.com/ajax.php?f=fetch_email&email_id=${MAIL_ID}&sid_token=${SID}")
```
Body in `mail_body`.

Display format:
```
[TempMail] Message #1

  From    : noreply@target.com
  Subject : Verify your email
  Date    : 2026-11-27 10:30

  ----
  [message body - plain text preferred, strip HTML tags if only HTML available]
  ----
```

For HTML-only content, strip tags with: `python3 -c "import sys,html,re; print(re.sub('<[^>]+>','',html.unescape(sys.stdin.read())))"`

### `wait` / `wait N`

Poll inbox every N seconds (default 5) for up to 2 minutes. Use a bash loop:

```bash
for i in $(seq 1 $MAX_ITERATIONS); do
  # check inbox based on provider
  # if new message found, display it and break
  sleep $INTERVAL
done
```

For TempMail.lol, be careful: each fetch consumes emails. Cache everything.

Display while waiting: `[TempMail] Waiting for email... (30s elapsed)`
On receive: `[TempMail] New message received!` then show the message.

### `status`

Read session and display:
```
[TempMail] Session Info
  Provider : TempMail.lol
  Address  : user@abc.gloatinggoat.com
  Created  : 2026-11-27 10:00
  Cached   : 3 emails
```

### `delete`

Delete account (if supported by provider), remove `/tmp/tempmail_session.json`.

```
[TempMail] Session destroyed.
```

## Error Handling

- If API call fails (non-200, timeout, empty response), report the error and suggest trying another provider
- If session file missing, tell user to run `/tmpmail new`
- If provider is down, auto-fallback to next provider in order: lol -> tm -> gm
- Curl timeout: use `curl -s --max-time 10` for all requests

## Important Rules

1. NEVER use jq - always use python3 for JSON parsing
2. ALWAYS cache TempMail.lol emails after fetching (they are deleted from server)
3. ALWAYS use `--max-time 10` with curl
4. For `read N`, N is 1-indexed from the `list` output
5. When creating Mail.tm accounts, generate random 8-char lowercase username and 16-char password
6. Display output in the exact format shown above - clean, aligned, readable
7. Store the message list from last `list` command in session file so `read N` can reference it
