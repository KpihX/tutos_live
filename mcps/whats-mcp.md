# 💬 whats-mcp — WhatsApp MCP Server

> 64 MCP tools for WhatsApp — messaging, chats, contacts, groups, analytics, and intent-first overview flows.

Managing WhatsApp from the terminal felt impossible — the protocol is proprietary, the official API requires a business account, and everything unofficial felt fragile. `whats-mcp` wraps [Baileys](https://github.com/WhiskeySockets/Baileys), the most mature reverse-engineered WhatsApp library, and exposes it as a proper MCP server with dual transport and operator surfaces.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Agent (Claude)                            │
└──────────────────────────────┬──────────────────────────────────┘
                               │  64 MCP tools
                    ┌──────────┴──────────┐
                    │                     │
               stdio (local)        HTTP (homelab)
                    │                     │
          ~/.npm-global/bin/        whats.kpihx-labs.com/mcp
              whats-mcp                   │
                    │                     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │      whats-mcp      │
                    │  (Bun + Baileys)    │
                    │                     │
                    │  connection.js      │  WhatsApp session
                    │  store.js           │  local analytics
                    │  tools/             │  MCP surface
                    │  admin/             │  operator layer
                    └──────────┬──────────┘
                               │  Baileys (MTProto-like)
                    ┌──────────▼──────────┐
                    │   WhatsApp Web      │
                    │   (unofficial API)  │
                    └─────────────────────┘
```

---

## 📦 Install (local dev)

```bash
cd ~/Work/AI/MCPs/whats_mcp
npm link                  # symlinks whats-mcp to ~/.npm-global/bin/
whats-mcp serve           # stdio transport
whats-mcp serve-http      # HTTP transport (port 3000 by default)
```

Runtime: **Bun** (primary). Node.js fallback via nvm for compatibility.

### First-time pairing

```bash
whats-mcp serve
# → QR code printed in terminal
# → Scan with WhatsApp app (Settings → Linked Devices → Link a Device)
# Session saved in lancedb/ (local SQLite-like store)
```

Or use pair code if QR isn't practical:
```bash
# Via admin HTTP endpoint:
curl -X POST https://whats.kpihx-labs.com/admin/pair-code \
  -H "Content-Type: application/json" \
  -d '{"phoneNumber": "+33XXXXXXXXX"}'
```

---

## 🔌 Claude Code config

Two entries — homelab HTTP (primary) with local stdio fallback:

```json
// In ~/.claude.json → projects["/home/kpihx"]["mcpServers"]
"whats-mcp": {
  "type": "http",
  "url": "https://whats.kpihx-labs.com/mcp"
},
"whats-mcp--fallback": {
  "command": "/home/kpihx/.npm-global/bin/whats-mcp"
}
```

The stdio fallback needs no login shell wrapper — whats-mcp reads its config from `config.json`, not env vars.

---

## 🛠️ Tool categories

| Category | Examples |
|---|---|
| **Intent-first** | `whatsup`, `daily_digest`, `find_messages`, `manage_watchlist` |
| **Messaging** | `send_text`, `send_image`, `send_document`, `send_audio`, `send_poll` |
| **Chats** | `list_chats`, `get_messages`, `get_messages_multi`, `read_messages` |
| **Contacts** | `list_contacts`, `get_contact_info`, `check_phone_number` |
| **Groups** | `create_group`, `get_group_info`, `manage_group_participants` |
| **Channels** | `create_channel`, `get_channel_info`, `manage_channel` |
| **Analytics** | `analytics_overview`, `analytics_top_chats`, `analytics_timeline` |
| **Media** | `download_media`, `send_sticker`, `send_location` |
| **Reactions** | `send_reaction`, `star_message`, `forward_message` |
| **Admin** | `connection_status`, `manage_privacy`, `manage_block` |

### Start here: `whatsup`

`whatsup` is the intent-first entry point — it gives a concise overview of recent activity, unread chats, and pending watchlisted contacts. Call it first in any WhatsApp-focused session.

---

## 🔧 Operator surfaces

Beyond MCP tools, whats-mcp exposes an operator layer:

```bash
# CLI admin
whats-admin status          # connection state, session health
whats-admin reconnect       # force reconnect

# HTTP admin (homelab)
GET  /health                         # liveness probe
GET  /admin/status                   # operator dashboard
POST /admin/reconnect                # force reconnect
POST /admin/pair-code                # generate pair code for re-linking
```

Telegram admin bridge is also configured — send `/status` to the configured bot for a quick health check without opening Claude.

---

## ⚠️ Known limitations

| Limitation | Notes |
|---|---|
| Session stability | Baileys is unofficial — WhatsApp can invalidate sessions after app updates |
| No broadcast lists | Not yet implemented in Baileys |
| Rate limiting | Sending too fast triggers WhatsApp spam detection — space out bulk sends |
| Media size | Large files (>50MB) may fail on the upload path |

If the session drops: `whats-admin reconnect` or use the `/admin/reconnect` HTTP endpoint, then re-scan QR or request pair code.

---

## 🗺️ Roadmap

- Status (Stories) posting
- Scheduled message sending
- Enhanced analytics with per-contact sentiment tracking

**Repos:** [GitHub](https://github.com/KpihX/whats-mcp) · [GitLab](https://gitlab.com/kpihx-labs/whats-mcp)
