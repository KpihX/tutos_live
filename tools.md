# 🧰 KpihX Ubuntu — Tools & Apps

> A living inventory of the tools installed on KpihX-Ubuntu (25.10).
> Each entry: what problem it solves + how to install.
> Short installs are inline. Complex setups link to a dedicated tutorial.

---

## ✏️ Editors & Terminal

### micro

A modern terminal text editor. Replaces `nano` and avoids the learning curve
of `vim` for quick edits — mouse support, familiar shortcuts (`Ctrl+S`,
`Ctrl+Z`, `Ctrl+C` to copy), syntax highlighting out of the box.

```bash
sudo apt install micro
```

Set as default editor:

```bash
echo 'export EDITOR=micro' >> ~/.kshrc
```

---

## 🔍 File Search

### plocate

Ultra-fast file search by filename. Replaces `find` for "where is that file"
queries — results in milliseconds via a pre-built index.

```bash
sudo apt install plocate

# Update the index (runs automatically via systemd timer)
sudo updatedb

# Search
plocate tailscale.conf
```

---

## 🧹 System Maintenance

### clean — Unified Cleanup Suite

Modular disk cleanup system for Ubuntu. Five specialized scripts orchestrated
by a single entry point, with four flag levels from read-only analysis to deep
system purge. Covers Snap revisions, package caches (uv, npm, pnpm), Docker,
AI models (Ollama, HuggingFace), and `~/Work/` project artifacts.

```bash
~/Work/sh/clean/clean_all.sh --infos    # full analysis (safe, read-only)
~/Work/sh/clean/clean_all.sh --safe     # weekly safe cleanup
~/Work/sh/clean/clean_all.sh --purge    # deep clean (prompted)
```

→ **Full guide:** [clean.md](clean.md) — module breakdown, flag levels, automation via systemd.

---

## 🖼️ Media & Images

### ImageMagick (`magick`)

Swiss-army knife for image manipulation from the CLI. Crop, resize, convert
format, annotate, compose — without opening a GUI. The go-to tool for preparing
screenshots and assets for tutorials.

```bash
sudo apt install imagemagick

magick input.png -crop WxH+X+Y output.png   # crop to region
magick input.png -resize 1280x output.png   # resize (keep ratio)
magick input.png -resize 50% output.png     # scale by percent
magick *.png output.pdf                     # merge images to PDF
magick convert input.png output.jpg         # format conversion
```

---

## 🌐 Network & Connectivity

### Tailscale

WireGuard-based mesh VPN. Makes any device (homelab, phone, server) reachable
via a stable `hostname` from anywhere — no port forwarding, no public IP,
no VPN server to maintain. Each device gets a `100.x.x.x` IP and a MagicDNS
hostname resolvable from all other Tailscale nodes.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

→ **Full guide:** [tailscale.md](tailscale.md) — MagicDNS fix, SSH config,
Bitwarden SSH agent, FQDN hardening.

---

## 📦 Node.js

### nvm — Node Version Manager

Manages multiple Node.js versions in isolation. Each project pins its Node
version via `.nvmrc`. Global packages stay per-version — no system Node, no
sudo required.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.zshrc
nvm install 22
nvm alias default 22
```

→ **Full guide:** [npm-prefix.md](npm-prefix.md) — nvm setup, `.nvmrc` auto-switch hook, the system npm EACCES trap, and the `~/.npm-global` fix for daily-driver CLI tools that auto-update.

**⚠️ Gotcha — `.npmrc` prefix vs `.kshrc` universal hub:**

If you use a `prefix=~/.npm-global` line in `~/.npmrc` **and** source nvm from a
universal shell hub (`.kshrc`), nvm will warn on every shell start:

```
Your .npmrc has a prefix setting which is incompatible with nvm.
Run `nvm use --delete-prefix` to unset it.
```

**Keep `prefix=` in `~/.npmrc` and ignore the warning.** The warning is cosmetic.
Removing `prefix=` and using only `NPM_CONFIG_PREFIX` in `~/.kshrc` seems cleaner
but breaks tool auto-updaters (Codex, Gemini CLI…): they spawn `npm install -g`
in subprocesses that don't inherit env vars → fall back to system prefix `/usr/local`
→ EACCES. The `~/.npmrc` file is read by ALL npm invocations unconditionally.

```bash
# ~/.npmrc — keep this, always:
# ⚠️ nvm warns about this line — harmless, do NOT remove it.
prefix=/home/kpihx/.npm-global
```

Also add `NPM_CONFIG_PREFIX` in `~/.kshrc` as belt-and-suspenders for interactive
shells, plus the idempotency guard to prevent double-sourcing:

```bash
# In ~/.kshrc — BEFORE the interactive guard:

# Idempotency guard — prevents double-sourcing when shell configs chain
if [ -z "$NVM_LOADED" ]; then
    export NVM_DIR="$HOME/.nvm"
    [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
    nvm use default --silent 2>/dev/null
    export NVM_LOADED=1
fi

# Redundant with ~/.npmrc prefix= but reinforces for interactive shells
export NPM_CONFIG_PREFIX="$HOME/.npm-global"
export PATH="$HOME/.npm-global/bin:$PATH"
```

Also: if `.kshrc` has a `case $- in *i*) ;; *) return ;; esac` guard near the
top (leftover from `.bashrc` boilerplate), it will silently exit before setting
any exports in non-interactive shells — breaking MCPs, scripts, and `zsh -l -c`.
Move that guard to **after** all `export` statements.

---

## 🐍 Python

### uv

Ultra-fast Python package manager and project tool. Single binary, replaces
`pip`, `pipx`, `virtualenv`, `poetry` — with a consistent opinionated
workflow for scripts, tools, and full projects.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Common use:

```bash
uv tool install <package>          # install CLI tool globally (~/.local/bin)
uv tool install --editable .       # editable install for own packages
uv run --with <lib> script.py      # run script with ad-hoc dependency
uv init --package myproject        # scaffold a new project
```

**Templates:**
- [`pyproject.toml`](https://github.com/kpihx/tutos_live/blob/master/templates/pyproject.toml) — KpihX project template: `uv_build` backend, `src/` layout, Typer+Rich, editable install. Copy to project root and fill in the placeholders.

---

### pyenv — Python Version Manager

The Python equivalent of nvm. Manages multiple Python versions on the same
machine, completely isolated from the system Python. Each version lives in
`~/.pyenv/versions/<version>/` — no sudo, no apt, no system pollution.

```bash
curl https://pyenv.run | bash
```

Add to `~/.kshrc` (universal hub — before the interactive guard):

```bash
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - $(basename "$SHELL"))"
```

Common use:

```bash
pyenv install 3.14          # install a specific version
pyenv install --list        # list all available versions
pyenv global 3.14           # set default for all shells
pyenv local 3.12            # pin a project (writes .python-version)
pyenv versions              # list installed versions
pyenv version               # show current active version
```

The `.python-version` file at the project root is the pyenv equivalent of
nvm's `.nvmrc` — pyenv switches automatically when you `cd` into the directory.

```bash
# In a project that needs Python 3.12:
echo "3.12" > .python-version

# Auto-switch fires on cd — no manual command needed:
cd myproject/
# → pyenv activates 3.12 silently
python --version
# → Python 3.12.x
```

Unlike nvm (which needs an explicit `nvm use` hook wired in `~/.zshrc`),
pyenv's auto-switch is built into `eval "$(pyenv init ...)"` — it installs
a `chpwd` hook in zsh and a `cd` wrapper in bash automatically.

Works seamlessly with `uv`: `uv` respects the active pyenv Python when you
run `uv venv` or `uv run` inside a project that has `.python-version`.

**⚠️ Gotcha — shell-specific `pyenv init` in a shell-agnostic `.kshrc`**

`eval "$(pyenv init - zsh)"` generates **zsh-specific** shell code. When
`.kshrc` is sourced from bash (e.g. via `~/.bashrc`), this produces syntax
errors. Fix: detect the active shell dynamically:

```bash
# In ~/.kshrc — shell-agnostic, works for both zsh and bash:
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - $(basename "$SHELL"))"
```

```
Without the fix:                   With the fix:
  .kshrc sourced from bash           .kshrc sourced from bash
       ↓                                  ↓
  eval "$(pyenv init - zsh)"        eval "$(pyenv init - bash)"
       ↓                                  ↓
  zsh-specific function syntax       bash-compatible cd wrapper
  → syntax error in bash ❌          → auto-switch works ✅
```

Note: auto-switch only fires in interactive shells where `pyenv init` ran.
Scripts and non-interactive subprocesses use whichever version `pyenv global`
set — same constraint as nvm in non-sourced contexts.

---

## 🐙 Git & Forges

### gh — GitHub CLI

GitHub's official CLI. Create repos, manage PRs, issues, releases, and access
the full GitHub REST API from the terminal. Replaces most github.com browser
interactions.

```bash
# Add apt repo first, then:
sudo apt install gh
gh auth login
```

→ **Full guide:** [gh.md](gh.md) — repos, PRs, issues, releases, API, CI.

### glab — GitLab CLI

GitLab's official CLI. Same philosophy as `gh` but for GitLab — including
native CI/CD pipeline management, job logs, artifacts, and protected variables.

```bash
# Add apt repo first, then:
sudo apt install glab
glab auth login
```

→ **Full guide:** [glab.md](glab.md) — repos, MRs, pipelines, CI/CD, secrets.

---

## 🌐 Web & Publishing

### Docsify (via npx)

Zero-build Markdown site generator. Drop one `index.html` in a repo of
Markdown files, push to GitHub Pages, and it's a full website. No build step,
no CI, no compiled output — Docsify fetches and renders `.md` files
client-side at runtime.

```bash
# No install needed — npx runs it on demand
npx docsify-cli serve .
```

→ **Full guide:** [github-pages.md](github-pages.md) — setup, `.nojekyll` trap, local preview, GitHub Pages activation.

**Templates:**
- [`github-pages-index.html`](https://github.com/kpihx/tutos_live/blob/master/templates/github-pages-index.html) — ready-to-use `index.html`: VS Code dark theme, search, sidebar, inline code color fix. Copy to repo root, rename to `index.html`, replace the 4 `SITE_*` placeholders.
- [`_sidebar.md`](https://github.com/kpihx/tutos_live/blob/master/templates/_sidebar.md) — generic Docsify sidebar with `.nojekyll` reminder. Copy to repo root as `_sidebar.md`.

### grip — GitHub-flavored Markdown preview

Renders any `.md` file locally in the browser exactly as GitHub would display it
— same CSS, same syntax highlighting, same table and checkbox rendering. No
Docsify setup needed, no repo push required. Instant sanity-check before
committing docs.

```bash
uv tool install grip

grip README.md          # opens http://localhost:6419
grip README.md 8080     # custom port
grip .                  # serves the whole directory
```

Works offline after first run (caches GitHub CSS locally). Authenticates via
`~/.config/grip/settings.py` if you hit the GitHub API rate limit (60 req/h
unauthenticated).

---

## 🔐 Secrets & Auth

### bw-env

Shell wrapper that injects Bitwarden vault secrets into RAM at unlock time.
Secrets exist only while the session is active — lock the vault or sleep the
machine and they're purged. The vault lives in a self-hosted Vaultwarden
instance, reachable only via Tailscale.

```bash
bw-env unlock    # prompt for password → inject secrets to RAM
bw-env status    # check lock state and visible secrets
bw-env lock      # purge secrets from RAM
```

→ **Full guide:** [bw-env.md](bw-env.md) — architecture, commands, auto-lock on sleep.
**Repos:** [GitHub](https://github.com/KpihX/bw-env) · [GitLab](https://gitlab.com/kpihx/bw-env)

### Bitwarden CLI (`bw`)

Command-line interface for the Bitwarden / Vaultwarden password manager.
Used by `bw-env` to inject secrets into RAM. The CLI authenticates against
a self-hosted Vaultwarden instance.

```bash
# Download from Bitwarden releases or via snap
sudo snap install bw

# Or via npm (latest)
npm install -g @bitwarden/cli

# Authenticate
bw login
bw unlock
```

### Bitwarden Desktop

Desktop app that acts as a hardware-backed **SSH agent** — SSH keys live in
the vault, exposed through a local socket (`~/.bitwarden-ssh-agent.sock`)
when unlocked. Replaces `ssh-agent` and key files on disk.

Download: https://bitwarden.com/download/ (AppImage or `.deb`)

Wire the SSH agent in `~/.kshrc`:

```bash
export SSH_AUTH_SOCK="$HOME/.bitwarden-ssh-agent.sock"
```

---

## 🖥️ Terminals

### WaveTerm

A modern GPU-accelerated terminal with a block-based layout — each command,
SSH session, or web view lives in its own independently scrollable block.
Sidebar widgets launch AI CLIs (claude, codex, gemini) or open URLs in one
click. BYOK AI chat built in via Groq, Mistral, or any OpenAI-compatible API.

Download the `.deb` from [waveterm.dev](https://waveterm.dev):

```bash
sudo dpkg -i waveterm_*.deb
```

→ **Full guide:** [waveterm.md](waveterm.md) — sidebar widgets, SSH connections, BYOK AI modes (Groq, Mistral), wsh secrets.

**Templates:**
- [`waveterm-widgets.json`](https://github.com/kpihx/tutos_live/blob/master/templates/waveterm-widgets.json) — AI CLI sidebar widgets (claude, codex, gemini, copilot, vibe) + GitHub/GitLab web shortcuts. Copy to `~/.config/waveterm/widgets.json`, fill `BINARY_PATH_*` and `SESSION_ID_*`.
- [`waveterm-ai-modes.json`](https://github.com/kpihx/tutos_live/blob/master/templates/waveterm-ai-modes.json) — BYOK AI Modes (Groq Scout, Groq Maverick, Mistral Large, Codestral, Pixtral). Set secrets via `wsh secret set`, paste into Wave Config → Wave AI Modes.

### Warp

AI-first terminal for Ubuntu — built-in AI command suggestions, natural-language
command generation (`Ctrl+I`), and collaborative Warp Drive for sharing runbooks.

Download the `.deb` from [warp.dev](https://warp.dev):

```bash
sudo dpkg -i warp-terminal_*.deb
```

---

## 🖥️ Shell

### Zsh + Oh My Zsh

`zsh` is the interactive shell. `oh-my-zsh` adds plugins, themes, and sane
defaults. `~/.kshrc` is the universal hub for secrets and aliases, sourced
by both `.zprofile` (login shells) and `.zshrc` (interactive shells).

```bash
sudo apt install zsh
chsh -s $(which zsh)

# Oh My Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

→ **Full guide:** [zsh_env.md](zsh_env.md) — `.zshenv` / `.zprofile` / `.zshrc` triptych, when each file is sourced, the `zsh -l -c` secret injection pattern, and the GUI app gap (WaveTerm, VS Code).

---

---

## 🤖 Messaging & APIs

### tg — Telegram CLI

Like `gh` for GitHub but for Telegram. Manages bots (Bot API, auto-discovers tokens from `TELEGRAM_*_TOKEN`) and personal conversations (Telethon MTProto, one-time OTP setup). Two independent layers, zero magic.

```bash
git clone git@github.com:KpihX/tg.git ~/Work/tools/tg
cd ~/Work/tools/tg
uv tool install --editable .
```

→ **Full guide:** [tg.md](tg.md) — Bot API, Telethon user API, config, security.
**Repos:** [GitHub](https://github.com/KpihX/tg) · [GitLab](https://gitlab.com/kpihx/tg)

---

### m365 CLI — Microsoft 365

Official CLI for Graph API access to Outlook mail, calendar, and OneDrive. Device code auth — authenticate once in a browser, tokens auto-refresh. Works with personal Microsoft/Hotmail accounts.

```bash
npm install -g @pnp/cli-microsoft365
m365 login --authType deviceCode
```

→ **Full guide:** [m365-cli.md](m365-cli.md) — Azure App Registration, device code flow, mail/calendar/OneDrive commands.

---

### flarectl — Cloudflare CLI

Official Go CLI for Cloudflare zone and DNS management. Reads `CF_API_TOKEN` from env (injected by bw-env). Full DNS CRUD from the terminal — essential for Traefik DNS-01 challenges and homelab record management.

```bash
go install github.com/cloudflare/cloudflare-go/cmd/flarectl@latest
ln -sf ~/go/bin/flarectl ~/.local/bin/flarectl
```

→ **Full guide:** [cloudflare.md](cloudflare.md) — zone list, DNS CRUD, token scoping.

---

## 🔌 MCP Servers

### bw-mcp — Bitwarden AI-Blind Proxy

Security-hardened MCP proxy between LLMs and the Bitwarden CLI. The model can organize your entire vault (rename, move, tag) while being physically incapable of reading any secret value. 7-layer defense: Pydantic validation, force-redact, Zenity HITL, ACID engine, WAL, LIFO rollback, full audit trail.

```bash
cd ~/Work/AI/MCPs/bw_mcp
uv tool install --editable .   # installs bw-mcp + bw-admin
```

→ **Full guide:** [mcps/bw-mcp.md](mcps/bw-mcp.md) — architecture, Claude Code config, transactions, WAL recovery.
**Repos:** [GitHub](https://github.com/KpihX/bw-mcp) · [GitLab](https://gitlab.com/kpihx/bw-mcp)

---

### tick-mcp — TickTick MCP Server

71 MCP tools for full TickTick control from any agent. Covers V1 (official API) and V2 (unofficial). Dual transport: local stdio fallback + homelab HTTP at `tick.kpihx-labs.com/mcp`.

```bash
cd ~/Work/AI/MCPs/tick_mcp
uv tool install --editable .   # local fallback
# homelab: already deployed at tick.kpihx-labs.com
```

→ **Full guide:** [mcps/tick-mcp.md](mcps/tick-mcp.md) — tool categories, Claude Code config, API gotchas.
**Repos:** [GitHub](https://github.com/KpihX/tick-mcp) · [GitLab](https://gitlab.com/kpihx-labs/tick-mcp)

---

### whats-mcp — WhatsApp MCP Server

64 MCP tools for WhatsApp messaging, contacts, groups, channels, and analytics. Powered by Baileys. Intent-first entry points: `whatsup`, `daily_digest`, `find_messages`. Dual transport: stdio + homelab HTTP at `whats.kpihx-labs.com/mcp`.

```bash
cd ~/Work/AI/MCPs/whats_mcp
npm link                       # local fallback (npm global)
# homelab: already deployed at whats.kpihx-labs.com
```

→ **Full guide:** [mcps/whats-mcp.md](mcps/whats-mcp.md) — tool categories, pairing, operator surfaces.
**Repos:** [GitHub](https://github.com/KpihX/whats-mcp) · [GitLab](https://gitlab.com/kpihx-labs/whats-mcp)

### desk-mcp — Desktop Automation MCP

Gives any AI agent eyes and hands on the KpihX-Ubuntu desktop. Screenshot via XDG Desktop Portal (no dialog, native GNOME Wayland), mouse/keyboard via xdotool. Crop to window name or `{x,y,w,h}` region. 9 tools: `screenshot`, `get_windows`, `get_screen`, `click`, `double_click`, `right_click`, `move_mouse`, `type_text`, `key`, `scroll`.

```bash
cd ~/Work/AI/MCPs/desk_mcp
uv tool install --editable .   # installs desk-mcp binary
sudo apt install xdotool python3-gi python3-dbus   # system deps
```

**Repos:** [GitHub](https://github.com/KpihX/desk-mcp) · [GitLab](https://gitlab.com/kpihx/desk-mcp)

---

*Add new tools following the format above. For installs that need significant
explanation, create a dedicated `.md` with the narrative approach and link
from this file. See `.agent/AGENT.md` for the `tools.md` maintenance guide.*
