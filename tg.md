# 🤖 tg — Telegram CLI

> Like `gh` for GitHub, but for Telegram. Manage bots and personal conversations from the terminal.

I wanted a single CLI to talk to my Telegram bots and read my own conversations — without touching the Telegram web app, BotFather, or crafting raw curl commands every time. The result is `tg`: a Python CLI with two completely separate layers.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         tg CLI                               │
│                                                              │
│   ┌──────────────────────┐   ┌──────────────────────────┐   │
│   │    Bot API Layer      │   │   User API (Telethon)    │   │
│   │  (httpx, stateless)  │   │  (MTProto, session-based)│   │
│   │                      │   │                          │   │
│   │  TELEGRAM_*_TOKEN    │   │  TG_API_ID + TG_API_HASH │   │
│   │  auto-discovered     │   │  OTP auth once, stored   │   │
│   │  from env vars       │   │  at ~/.config/tg/        │   │
│   └──────────┬───────────┘   └─────────────┬────────────┘   │
│              │                             │                  │
│  tg bots     │  tg status                  │  tg user me      │
│  tg send     │  tg updates                 │  tg user chats   │
│  tg webhook  │  tg commands                │  tg user read    │
│  tg chat     │                             │  tg user send    │
└─────────────────────────────────────────────────────────────┘
```

Two layers, two authentication flows, zero overlap:

- **Bot API** — stateless, uses existing bot tokens auto-discovered from `TELEGRAM_*_TOKEN` env vars. No session file, no setup.
- **User API (Telethon)** — full MTProto access via your personal account. One-time OTP setup, then session persists at `~/.config/tg/user.session`.

---

## 📦 Install

```bash
git clone git@github.com:KpihX/tg.git ~/Work/tools/tg
cd ~/Work/tools/tg
uv tool install --editable .
tg --version
```

Editable install means changes to source take effect immediately — no reinstall needed during development.

---

## 🔐 Configuration

### Secrets via bw-env

`tg` reads its credentials from environment variables, injected by `bw-env` from Bitwarden `GLOBAL_ENV_VARS`:

```bash
# Auto-discovered bots — naming: TELEGRAM_<ALIAS>_TOKEN
# Alias is lowercased, underscores become dashes
TELEGRAM_HOMELAB_TOKEN=<token>    # → alias "homelab"
TELEGRAM_UBUNTU_TOKEN=<token>     # → alias "ubuntu"
TELEGRAM_N8N_HOMELAB_TOKEN=<tok>  # → alias "n8n-homelab"

# Default recipient for tg send
CHAT_ID=<your_chat_id>

# For tg user (Telethon) — from https://my.telegram.org
TG_API_ID=12345
TG_API_HASH=<hash>
```

### Config file (optional)

Override defaults at `~/.config/tg/config.yaml`:

```yaml
default_bot: homelab
default_chat: "<YOUR_CHAT_ID>"

bots:
  homelab:
    default_chat: "<YOUR_CHAT_ID>"
```

```
Secret resolution order (Bot API):
  TELEGRAM_*_TOKEN env vars   ← primary (bw-env injection)
       ↓
  ~/.config/tg/config.yaml    ← user overrides (optional)
       ↓
  src/tg/config.yaml          ← bundled defaults
```

---

## 🤖 Bot API

The Bot API layer is stateless — each command makes direct HTTPS requests to `api.telegram.org` using your bot token. No daemon, no session.

### List configured bots

```bash
tg bots
```

Auto-discovers all `TELEGRAM_*_TOKEN` env vars, queries Telegram for each bot's identity, and displays a Rich table with name, username, and ID.

### Bot status + webhook

```bash
tg status                    # default bot
tg status --bot n8n-homelab  # specific bot
tg webhook get               # current webhook URL
```

### Send a message

```bash
tg send "Hello world"
tg send "<b>Alert!</b>" --to CHAT_ID --bot ubuntu   # HTML supported
```

### Recent updates (messages received)

```bash
tg updates --limit 20
tg updates --bot homelab
```

### Chat/group info

```bash
tg chat info @mychannel
tg chat info -1001234567890 --bot homelab
tg chat admins @mygroup
```

### Webhook management

```bash
tg webhook get
tg webhook set https://n8n.kpihx-labs.com/webhook/xxx --bot n8n-homelab
tg webhook del --drop-pending
```

### Command list (BotFather)

```bash
tg commands get
tg commands set start "Start the bot"
```

---

## 👤 User API (Telethon)

This layer speaks MTProto — the native Telegram protocol. It acts as *your* account, not a bot. First-time setup requires a phone OTP.

### First-time setup (once only)

```bash
tg user setup
# → prompts for API ID, API Hash, phone number
# → sends OTP to your Telegram app
# → session saved at ~/.config/tg/user.session
```

Get your API credentials from `https://my.telegram.org` → "API development tools". Store `TG_API_ID` and `TG_API_HASH` in Bitwarden `GLOBAL_ENV_VARS`.

```
First-time flow:
  tg user setup
       ↓
  prompts: API ID / API Hash / phone
       ↓
  Telegram sends OTP to your phone
       ↓
  session saved: ~/.config/tg/user.session
       ↓
  all future commands reuse session silently
```

### Your identity

```bash
tg user me
```

### List conversations

```bash
tg user chats --limit 30
tg user chats --type group
tg user chats --type channel
```

### Read messages

```bash
tg user read @KpihX --limit 20
tg user read -1001234567890 --limit 50
tg user read @mychat --search "keyword"
```

**⚠️ Numeric chat ID resolution:** Telethon resolves numeric IDs via its entity cache. If you get `ValueError: Cannot find any entity`, the entity isn't cached yet. Fix: `tg user chats` first — this populates the cache.

### Send as yourself

```bash
tg user send @username "Hello!"
tg user send -1001234567890 "Group message" --reply-to 4242
```

### Edit / delete

```bash
tg user edit @username 4242 "Corrected text"
tg user delete @username 4242
```

### Contacts

```bash
tg user contacts
```

---

## 🔐 Security

```
Security model:
  Bot tokens       → bw-env (RAM injection from Bitwarden, never on disk)
  TG_API_ID/HASH   → bw-env (GLOBAL_ENV_VARS secure note)
  user.session     → ~/.config/tg/user.session (SQLite, local only)
  .env files       → excluded from git (.gitignore)
  *.session files  → excluded from git (.gitignore)
```

The session file at `~/.config/tg/user.session` is covered by `backup_workstation.sh` (backs up all of `~/.config/`). It is equivalent to a login cookie — treat it like a credential file.

---

## 🗺️ Roadmap

- `tg-mcp` — MCP server exposing tg commands as tools for Claude and other agents
- `tg user forward` — forward a message between chats
- `tg user search` — global search across all conversations
- `tg user download` — download media from messages

**Repos:** [GitHub](https://github.com/KpihX/tg) · [GitLab](https://gitlab.com/kpihx/tg)
