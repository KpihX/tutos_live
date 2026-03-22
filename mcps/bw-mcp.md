# 🔐 bw-mcp — Bitwarden AI-Blind Vault Proxy

> Give agents full organizational control over your Bitwarden vault — while making it physically impossible for them to read your secrets.

The problem is straightforward: I want Claude to reorganize my vault, rename items, move them between folders, update metadata. But I absolutely cannot let the model see my passwords, TOTP seeds, or API keys. `bw-mcp` is the solution: a security-hardened proxy that sits between the LLM and the Bitwarden CLI, redacting all secret values before they ever reach the model.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Claude Code                               │
│                    (model — AI-BLIND)                            │
└──────────────────────────┬──────────────────────────────────────┘
                           │  MCP tools
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                        bw-mcp proxy                              │
│                                                                  │
│  ┌──────────────┐  ┌────────────────┐  ┌─────────────────────┐  │
│  │   server.py  │  │ transaction.py │  │      wal.py         │  │
│  │  (MCP tools) │  │ (ACID engine)  │  │  (Write-Ahead Log)  │  │
│  └──────────────┘  └────────────────┘  └─────────────────────┘  │
│         │                  │                    │                │
│  ┌──────────────┐  ┌────────────────┐           │                │
│  │  scrubber.py │  │   models.py    │           │                │
│  │  (redaction) │  │ (Pydantic val.)│           │                │
│  └──────────────┘  └────────────────┘           │                │
│                           │                     │                │
│              Human approval (Zenity popup)       │                │
│                           │                     │                │
│                           ▼                     ▼                │
│                     Bitwarden CLI ←── WAL (Fernet+PBKDF2) ──────┘
│                       (bw CLI)                                   │
└─────────────────────────────────────────────────────────────────┘
```

**7-layer defense in depth:**
1. Pydantic schema validation (`extra="forbid"`) on all inputs
2. `force_redact()` — overwrites all secret fields with `[REDACTED_BY_PROXY_POPULATED]`
3. Human-in-the-loop: Zenity GUI popup requiring Master Password before every transaction
4. ACID transaction engine (all-or-nothing, max 25 ops per batch)
5. Write-Ahead Log (WAL) — Fernet-encrypted, PBKDF2 key, survives crashes
6. LIFO rollback — automatic reversal if any operation fails
7. Full audit trail — every op logged with status, rollback command, timestamp

---

## 📦 Install

```bash
cd ~/Work/AI/MCPs/bw_mcp
uv tool install --editable .

# Two binaries installed:
bw-mcp serve       # MCP server (stdio transport)
bw-admin           # Rich+Typer admin CLI
```

Dependencies: Python 3.12+, `mcp`, `pydantic`, `cryptography`, `rich`, `typer`, `pyyaml`.

**Prerequisites:** Bitwarden CLI (`bw`) must be installed and authenticated:
```bash
sudo snap install bw
bw login
```

---

## 🔌 Claude Code config

`bw-mcp` uses **Zenity for password prompts** — it does NOT need `bw-env` secrets. The `bw` CLI handles its own auth separately.

```json
// In ~/.claude.json → projects["/home/kpihx"]["mcpServers"]
"bw-mcp": {
  "command": "/home/kpihx/.local/bin/bw-mcp",
  "args": ["serve"]
}
```

No login shell wrapper needed — `bw-mcp` prompts interactively via Zenity when auth is required.

---

## 🛠️ Key operations

### Explore the vault

```
get_vault_map           — full vault structure (folders, items, collections)
compare_secrets_batch   — compare items without revealing values
get_template            — fetch item schemas for creation
```

### Propose a transaction

```
propose_vault_transaction  — submit a batch of operations for human review
get_proxy_audit_context    — current proxy state and WAL status
inspect_transaction_log    — review past transaction history
```

### Transaction flow

```
1. Model calls propose_vault_transaction({ ops: [...] })
       ↓
2. Pydantic validates each op against strict schema
       ↓
3. Zenity popup: "Approve this vault transaction? [Master Password]"
       ↓  (human types password)
4. WAL writes encrypted intent to disk
       ↓
5. Ops execute sequentially (max 25 per batch)
       ↓
6a. All succeed → WAL marks COMMITTED, audit log updated
6b. Any failure → LIFO rollback of completed ops, WAL marks ROLLED_BACK
```

---

## 🔧 Admin CLI (`bw-admin`)

```bash
bw-admin status          # proxy state, WAL health, last transaction
bw-admin logs            # recent transaction log (redacted)
bw-admin wal inspect     # raw WAL contents (encrypted headers shown)
bw-admin wal recover     # replay interrupted transaction from WAL
```

---

## ⚠️ Known limitations

| Limitation | Root cause | Mitigation |
|---|---|---|
| No atomic COMMIT | Bitwarden API has no transaction mode | WAL + LIFO rollback (Saga pattern) |
| Race conditions | External clients can modify vault during batch | 25-op cap per batch |
| `delete_attachment` is irreversible | Bitwarden doesn't trash attachments | Isolated in its own transaction automatically |
| Session timeout | `BW_SESSION` can expire during long batches | Keep batches short; WAL survives for recovery on next boot |

---

## 🗺️ Roadmap

- HTTP transport for homelab deployment (Traefik)
- Telegram admin bridge (same pattern as tick-mcp and whats-mcp)
- Operator dashboard (`/admin/status`, `/admin/help`)

**Repos:** [GitHub](https://github.com/KpihX/bw-mcp) · [GitLab](https://gitlab.com/kpihx/bw-mcp)
