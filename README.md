# tmpmail-skill

A Claude Code skill for temporary/disposable email management. Built for pentesting workflows.

## Features

- **Multi-provider**: TempMail.lol (primary), Mail.tm, Guerrilla Mail
- **Smart defaults**: Prioritizes providers with domains not on common blocklists
- **Auto-caching**: Emails from TempMail.lol are cached locally (they're consumed on fetch)
- **Simple CLI**: Create, list, read, wait for, and delete temp emails

## Installation

Copy the `SKILL.md` file to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/tmpmail
cp SKILL.md ~/.claude/skills/tmpmail/SKILL.md
```

## Usage

Once installed, use the `/tmpmail` slash command in Claude Code:

| Command | Description |
|---------|-------------|
| `/tmpmail new` | Create new temp email (auto-selects best provider) |
| `/tmpmail new lol` | Create via TempMail.lol (recommended) |
| `/tmpmail new tm` | Create via Mail.tm |
| `/tmpmail new gm` | Create via Guerrilla Mail |
| `/tmpmail list` | List inbox messages |
| `/tmpmail read 1` | Read message #1 |
| `/tmpmail wait` | Poll inbox until new message arrives |
| `/tmpmail status` | Show current session info |
| `/tmpmail delete` | Destroy session |

## Providers

| Provider | Blocklist Status | Notes |
|----------|-----------------|-------|
| TempMail.lol | Not on common blocklists | Uses unique subdomains, emails consumed on fetch |
| Mail.tm | On common blocklists | Full CRUD API, domains rotate |
| Guerrilla Mail | On blocklists | 60min expiry, extendable |

## Requirements

- `curl` with `--max-time` support
- `python3` (for JSON parsing)

## License

[MIT](LICENSE)
