# codex-profile

[![English](https://img.shields.io/badge/lang-English-blue)](README.md)
[![繁體中文](https://img.shields.io/badge/lang-繁體中文-lightgrey)](README.zh-TW.md)

Switch between multiple Codex CLI accounts via `CODEX_HOME` isolation. Each profile keeps its own `auth.json` and sessions; `config.toml` (MCP servers, features) is shared across profiles by default.

## Install

```bash
mkdir -p ~/.local/bin
mv codex-profile ~/.local/bin/
chmod +x ~/.local/bin/codex-profile

# Auto-apply active profile to every new shell (recommended)
echo 'eval "$(codex-profile shell-init)"' >> ~/.zshrc
exec zsh
```

## Commands

### Profile management

| Command | What it does |
|---|---|
| `list` (alias: `ls`) | Show all profiles, marking the active one with `*` and indicating login status |
| `current` (alias: `active`) | Print the active profile name |
| `add <name>` | Create an empty profile. First `codex` run in it triggers OAuth |
| `import-current [name]` | Copy existing `~/.codex` into a new profile (default name: `default`). On first import, promotes its `config.toml` to the shared file |
| `switch <name>` (alias: `use`) | Make `<name>` the active profile. Does **not** touch auth — no re-login |
| `remove <name>` (alias: `rm`) | Delete a profile and all its credentials (requires typing the name to confirm) |

### Using the active profile

| Command | What it does |
|---|---|
| `env` | Print `export CODEX_HOME=...` for the active profile. Use with `eval "$(codex-profile env)"` |
| `run -- <cmd> [args]` (alias: `exec`) | Run `<cmd>` with `CODEX_HOME` set to the active profile |
| `shell-init` (alias: `init`) | Print a snippet to add to `~/.zshrc` / `~/.bashrc` — every new shell auto-applies the active profile |

### Shared config (`config.toml`)

`config.toml` (MCP servers, `[features]`, model defaults) is symlinked from each profile to `~/.codex-profiles/_shared/config.toml`. Edit once, applies everywhere.

| Command | What it does |
|---|---|
| `config status` | Show shared config path and per-profile link state (linked / local / missing / broken) |
| `config path` | Print the shared `config.toml` path |
| `config edit` | Open the shared `config.toml` in `$EDITOR` |
| `config link <profile> [--force]` | Re-link a profile's `config.toml` to shared. `--force` overwrites a local copy |
| `config unlink <profile>` | Detach a profile from shared (make a private copy) |
| `config relink-all [--force]` | Re-link every profile to shared |
| `config help` | Show `config` subcommand usage |

### Help

| Command | What it does |
|---|---|
| `help` (alias: `-h`, `--help`) | Show top-level usage |

## Usage examples

### 1. Getting started

**You already have a logged-in `~/.codex`:**

```bash
# Adopt it as a named profile
codex-profile import-current work
codex-profile list
# *  work    logged in
```

**You're starting fresh:**

```bash
codex-profile add work
codex-profile switch work
codex                       # browser OAuth flow
```

### 2. Login / switch between profiles

```bash
# Add a second profile (empty -> first codex run triggers OAuth)
codex-profile add personal
codex-profile switch personal
codex                       # OAuth for the second ChatGPT account

# Switch back later — no re-login, auth is cached per profile
codex-profile switch work
codex
```

> ⚠️ **Kill running codex before switching.** Otherwise you may hit
> `refresh token was already used` (Codex CLI bug, OAuth race condition):
> ```bash
> pkill -f codex; sleep 2
> codex-profile switch <name>
> ```

### 3. Importing config from existing `~/.codex`

When you run `import-current`, the existing `config.toml` is automatically promoted to the shared location, so any future profiles you create inherit it:

```bash
codex-profile import-current work
# -> Detects ~/.codex/config.toml, moves it to ~/.codex-profiles/_shared/config.toml
# -> Replaces ~/.codex-profiles/work/config.toml with a symlink to the shared file

codex-profile add personal
# personal/config.toml is also a symlink to the shared one
# -> All your MCP servers are immediately available in 'personal' too
```

### 4. Editing shared config

```bash
codex-profile config edit
# Opens ~/.codex-profiles/_shared/config.toml in $EDITOR
# Save -> applies to ALL profiles immediately (no relink needed)

codex-profile config status
# Shared config: /home/you/.codex-profiles/_shared/config.toml
#   42 lines, 3 MCP server(s)
#
# Per-profile config.toml:
#   work        linked    -> shared
#   personal    linked    -> shared
```

If one profile needs different settings (rare, but the option is there):

```bash
codex-profile config unlink personal       # personal/config.toml becomes a private copy
$EDITOR ~/.codex-profiles/personal/config.toml   # edit independently

# Decide later it should follow shared again:
codex-profile config link personal --force
```

### 5. Checking auth token

```bash
# Where is the active profile's auth.json?
codex-profile current
ls -la "$CODEX_HOME/auth.json"

# Inspect token structure (mask sensitive values)
python3 <<'EOF'
import json, os
d = json.load(open(f"{os.environ['CODEX_HOME']}/auth.json"))
print("auth_mode:", "chatgpt" if d.get("tokens") else "apikey")
t = d.get("tokens", {})
print("account_id:", t.get("account_id"))
print("last_refresh:", d.get("last_refresh"))
print("access_token len:", len(t.get("access_token", "")))
print("refresh_token len:", len(t.get("refresh_token", "")))
EOF

# Compare two profiles — same account_id means they share the same OAuth chain (BAD)
for p in ~/.codex-profiles/*/auth.json; do
    echo "=== $p ==="
    python3 -c "import json; print(json.load(open('$p'))['tokens']['account_id'])"
done
```

If the token is stuck in a refresh-race state, the fix is usually:

```bash
pkill -f codex; sleep 2
codex     # 90% of the time this re-acquires a working token without re-login
```

Only if that fails:

```bash
rm "$CODEX_HOME/auth.json"
codex     # full OAuth re-login for this profile
```

## What's shared vs private

| File | Scope |
|---|---|
| `config.toml` | **Shared** via symlink |
| `auth.json` | Private per profile |
| `sessions/`, `history.jsonl`, `log/` | Private per profile |

## Layout on disk

```
~/.codex-profiles/
├── _shared/
│   └── config.toml              # The real shared config
├── .active                      # Plain text: name of active profile
├── work/
│   ├── auth.json
│   ├── config.toml -> ../_shared/config.toml
│   ├── sessions/
│   └── ...
└── personal/
    ├── auth.json
    ├── config.toml -> ../_shared/config.toml
    └── ...
```

## Environment variables

| Variable | Purpose | Default |
|---|---|---|
| `CODEX_HOME` | Where Codex CLI reads its state. Set by `codex-profile` | `~/.codex` |
| `CODEX_PROFILES_DIR` | Where `codex-profile` stores profile directories | `~/.codex-profiles` |
| `EDITOR` / `VISUAL` | Used by `config edit` | `vi` |

## Uninstall

```bash
# Remove the shell-init line from ~/.zshrc / ~/.bashrc, then:
unset CODEX_HOME
exec zsh

rm -rf ~/.codex-profiles   # optional, wipes all profiles
rm ~/.local/bin/codex-profile
```

Codex falls back to `~/.codex` once `CODEX_HOME` is unset.
