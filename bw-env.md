# 🔐 bw-env — Bitwarden-backed Secret Injection

> **Machine:** KpihX-Ubuntu (Ubuntu 25.10)
> **Lived on:** 2026-03 · **Status:** Production-stable

---

I don't keep secrets in `.env` files, shell history, or environment variables
that persist across reboots. The attack surface for those is too large —
files can leak, history can be read, exported vars survive in process trees
longer than expected.

The approach I use: secrets live in **Vaultwarden** (self-hosted Bitwarden),
and get injected into RAM at unlock time via `tmpfs` (`/run/user/...`).
They exist only while the session is active. Lock the vault — or sleep the
machine — and they're gone.

That's what **`bw-env`** does. It's a thin shell layer on top of the
Bitwarden CLI that handles the unlock/inject/revoke lifecycle.

---

## The full documentation lives in the repo

Everything — architecture, install, config, commands, daemon setup,
auto-lock on sleep — is documented in the project README:

**GitHub:** [github.com/KpihX/bw-env](https://github.com/KpihX/bw-env)
**GitLab:** [gitlab.com/kpihx/bw-env](https://gitlab.com/kpihx/bw-env)

---

## Quick reference

```bash
bw-env unlock    # prompt for password → sync vault → inject secrets to RAM
bw-env status    # check lock state and which secrets are visible
bw-env sync      # force re-sync with Vaultwarden without full unlock
bw-env lock      # purge secrets from RAM + trigger global revocation
bw-env restart   # restart the background sync daemon (systemd)
bw-env logs      # view recent log entries (alias: bw-env logs -n 50)
```

---

## How it fits into the shell architecture

`bw-env unlock` is called once per session — from `~/.kshrc`, which is
sourced by every shell (Zsh via `~/.zshenv`, Bash via `~/.bashrc`).
After unlock, secrets are available as environment variables in all
subsequent shells and subprocesses.

```
┌─────────────────────────────────────────────────────┐
│  bw-env unlock                                      │
│       │                                             │
│       ▼                                             │
│  Bitwarden CLI ──── HTTPS ────► Vaultwarden         │
│  (bw sync + bw unlock)          (self-hosted)       │
│       │                                             │
│       ▼                                             │
│  Secrets written to RAM (tmpfs, not disk)           │
│  └── available as $ENV_VARS in all shells           │
│                                                     │
│  On lock / sleep / screen-lock:                     │
│  └── D-Bus hook triggers bw-env lock               │
│      └── secrets purged from RAM                   │
└─────────────────────────────────────────────────────┘
```

> **Vaultwarden** (`vault.kpihx-labs.com`) is reachable via Tailscale only —
> intentionally private. If Tailscale is down, `bw-env unlock` will fail.
> See [tailscale.md](tailscale.md) for the connectivity setup.
