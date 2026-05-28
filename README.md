# 🦉 Kiro Stack

**kiro-gateway + kirolink + kiro-cli auth — unified installation**

A complete stack for accessing Kiro (Amazon Q Developer / CodeWhisperer) models through Anthropic-compatible APIs. Two complementary tools, one installer.

---

## Components

| Component | Language | Port | What it does |
|-----------|----------|------|-------------|
| **kiro-gateway** | Python | `8333` | Anthropic/OpenAI proxy for opencode TUI. Systemd-managed. Async SSE streaming, thinking blocks, tool use. |
| **kirolink** | Go | `8080` | Claude Code proxy + token manager. Translates CodeWhisperer ↔ Anthropic protocol. Built-in token refresh. |
| **kirolink data** | — | — | Token cache (`~/.aws/sso/cache/kiro-auth-token.json`), kiro-cli DB integration, env var export. |
| **kiro-cli** | Python | — | AWS Builder ID / SSO authentication. Creates the session that powers everything. |

### Architecture

```
opencode TUI ──→ kiro-gateway (8333) ──→ CodeWhisperer API ──→ Kiro/Claude models
                      ↑                        ↑
                 kiro-cli auth           kirolink token refresh
                 (SQLite DB)       ┌──── (JSON token cache)
                     │             │         │
                     └─ AWS SSO ───┘    systemd timer
                        OIDC           (hourly refresh)
                     (Builder ID)
```

### Models available (9 total)

- `claude-sonnet-4.5` — Full thinking, primary chat model
- `claude-haiku-4.5` — Fast/cheap, ideal for subagents
- `claude-sonnet-4` — Stable Sonnet
- `deepseek-3.2` — DeepSeek
- `glm-5` — GLM-5
- `minimax-m2.5` / `minimax-m2.1` — MiniMax models
- `qwen3-coder-next` — Qwen3 Coder
- `auto-kiro` — Smart default routing

---

## Quick start

### Prerequisites

- Linux (Ubuntu/Debian recommended), macOS, or WSL2
- Python 3.10+
- Go 1.21+
- Git
- An [AWS Builder ID](https://builderid.us-east-1.console.aws.amazon.com) (free, no credit card)

### Install

```bash
git clone https://github.com/marktantongco/kiro-stack.git
cd kiro-stack
chmod +x install.sh
./install.sh
```

### Install options (flags)

```bash
# Fully automated — no prompts
./install.sh --non-interactive --kirolink-service

# Reuse existing auth, skip opencode wiring
./install.sh --skip-login --skip-opencode

# Just validate configuration
./install.sh --dry-run

# Show all flags
./install.sh --help
```

| Flag | What it does |
|------|-------------|
| `--non-interactive` | Skip all prompts (defaults: reuse login, skip kirolink service, install timer) |
| `--kirolink-service` | Install kirolink as systemd service |
| `--skip-login` | Skip kiro-cli login (use existing session) |
| `--skip-opencode` | Skip opencode.jsonc wiring |
| `--dry-run` | Validate config and exit |
| `-h`, `--help` | Show usage |

### Installer steps (13 total)

1. Checks system dependencies (installs missing packages)
2. Clones kiro-gateway from GitHub (falls back to `vendor/` if offline)
3. Builds kirolink Go binary from source (falls back to `vendor/` if offline)
4. Creates Python venv with kiro-cli and dependencies
5. Guides through kiro-cli login (AWS Builder ID / browser OIDC)
6. Sets up token cache and runs kirolink refresh
7. Creates `.env` for kiro-gateway
8. Installs kiro-gateway as a systemd user service
9. Optionally installs kirolink as a systemd service
10. Creates systemd timer for hourly auth token refresh
11. Starts and verifies kiro-gateway end-to-end
12. Wires kiro provider into opencode.jsonc
13. Prints summary with management commands

### After install

```bash
# Verify gateway
curl http://localhost:8333/health
curl http://localhost:8333/v1/models -H 'Authorization: Bearer kiro-gateway-8333'

# Use in opencode
#   model: kiro/claude-sonnet-4.5
#   subagents: kiro/claude-haiku-4.5, kiro/qwen3-coder-next

# Kirolink (token management)
~/Documents/proxy/kirolink/kirolink read
~/Documents/proxy/kirolink/kirolink refresh
eval "$(~/Documents/proxy/kirolink/kirolink export)"  # for Claude Code
```

---

## Systemd management

### kiro-gateway (primary)

```bash
# Status
systemctl --user status kiro-gateway.service

# Logs (follow)
journalctl --user -u kiro-gateway.service -n 50 -f

# Restart
systemctl --user restart kiro-gateway.service

# Stop
systemctl --user stop kiro-gateway.service
```

### kirolink (optional — for Claude Code)

```bash
# Status
systemctl --user status kirolink.service

# Logs
journalctl --user -u kirolink.service -n 50 -f
```

### Auth refresh timer (prevents 504 errors)

```bash
# Status
systemctl --user status kiro-auth-refresh.timer

# Logs
journalctl --user -u kiro-auth-refresh.service -n 50 -f

# Manual trigger
systemctl --user start kiro-auth-refresh.service
```

The timer runs `kirolink refresh` hourly with a randomized 10-minute delay. This keeps your token fresh and prevents the silent 504/502 failures that happen when the ~24h SSO token expires.

---

## Auth lifecycle

Kiro uses AWS SSO OIDC via Builder ID. Tokens expire approximately every 24 hours.

### The three layers of auth defense:

1. **Hourly timer** (`kiro-auth-refresh.timer`) — proactively syncs the token from kiro-cli DB to the JSON cache. Prevents most expiry issues before they happen.
2. **kirolink auto-refresh** — if kirolink gets a 403 when proxying, it automatically refreshes the token and retries.
3. **Manual refresh** — when all else fails (e.g., the SSO session itself expired):

```bash
source ~/Documents/proxy/kiro-gateway/.venv/bin/activate
kiro-cli logout && kiro-cli login
deactivate
systemctl --user restart kiro-gateway.service
~/Documents/proxy/kirolink/kirolink refresh
```

### Quick one-liner (if kiro-cli already logged in):

```bash
kiro-cli login 2>/dev/null; systemctl --user restart kiro-gateway.service
```

---

## Using with Claude Code

kirolink can serve as a Claude Code proxy. After building:

```bash
# Start kirolink proxy
~/Documents/proxy/kirolink/kirolink server

# In another terminal, set env vars for Claude Code
eval "$(~/Documents/proxy/kirolink/kirolink export)"

# Now claude code will route through Kiro
claude
```

Or install the systemd service (`--kirolink-service` flag during install).

---

## Vendor fallback

This repo includes vendored snapshots of both upstream sources in `vendor/`:

```
vendor/
├── kiro-gateway/     # Snapshot of github.com/Jwadow/kiro-gateway
└── kirolink/         # Snapshot of github.com/alexandeism/kirolink
```

The installer tries git clone first. If that fails (no network, repo down), it falls back to the vendored copy. This makes the installer fully offline-capable.

To update the vendored sources:

```bash
# Refresh kiro-gateway
rm -rf vendor/kiro-gateway
git clone https://github.com/Jwadow/kiro-gateway.git /tmp/kiro-gateway-fresh
cp -r /tmp/kiro-gateway-fresh/* vendor/kiro-gateway/
rm -rf /tmp/kiro-gateway-fresh

# Refresh kirolink
rm -rf vendor/kirolink
git clone https://github.com/alexandeism/kirolink.git /tmp/kirolink-fresh
cp -r /tmp/kirolink-fresh/* vendor/kirolink/
rm -rf /tmp/kirolink-fresh
```

---

## Development

### Running tests

```bash
# Lint check
bash -n install.sh

# Mock fresh-machine test
test/mock_test.sh

# ShellCheck (requires shellcheck)
shellcheck install.sh
```

### CI

This repo runs ShellCheck on every push/PR via GitHub Actions (`.github/workflows/shellcheck.yml`).

---

## Directory layout

```
kiro-stack/                   # This repo
├── install.sh                # Unified installer (528 lines)
├── README.md                 # This file
├── .gitignore
├── .github/workflows/
│   └── shellcheck.yml        # CI: lint on push
├── systemd/
│   ├── kiro-auth-refresh.service   # Oneshot refresh
│   └── kiro-auth-refresh.timer     # Hourly timer
├── test/
│   └── mock_test.sh          # Sandbox test suite
└── vendor/
    ├── kiro-gateway/         # Vendored upstream source
    └── kirolink/             # Vendored upstream source

~/.config/systemd/user/
├── kiro-gateway.service      # Gateway (systemd-managed)
├── kirolink.service           # Kirolink proxy (optional)
├── kiro-auth-refresh.service  # Token refresh (oneshot)
└── kiro-auth-refresh.timer    # Hourly refresh timer
```

---

## Troubleshooting

### Gateway won't start

```bash
# Check service logs
journalctl --user -u kiro-gateway.service --no-pager -n 30

# Common issues:
# - Python deps not installed → re-run: source ~/Documents/proxy/kiro-gateway/.venv/bin/activate && pip install -r requirements.txt
# - Port conflict → check: lsof -i :8333
# - kiro-cli not logged in → re-run: kiro-cli login
```

### 504 / 502 errors from gateway

Token expired. Check the refresh timer first:

```bash
systemctl --user status kiro-auth-refresh.timer
journalctl --user -u kiro-auth-refresh.service -n 10
```

If the timer ran but the session itself expired (not just the cached token):

```bash
source ~/Documents/proxy/kiro-gateway/.venv/bin/activate
kiro-cli logout
kiro-cli login
deactivate
systemctl --user restart kiro-gateway.service
```

### No models returned

The gateway's model list reflects what CodeWhisperer/Kiro makes available. If it returns 0:
- Check gateway logs: `journalctl --user -u kiro-gateway.service -n 30`
- Verify auth isn't expired
- Test directly: `kiro-cli chat --model claude-sonnet-4.5 --message "hi" --max-tokens 10`

### Port conflicts

| Port | Service |
|------|---------|
| `8333` | kiro-gateway |
| `8080` | kirolink |
| `8082` | fcc (free-claude-code proxy) |
| `8083` | 9router-claude |
| `20129` | 9router |

### Token file not found

If `kirolink read` says token not found:
1. Run `kiro-cli login` to authenticate
2. Run `kirolink refresh` to sync the token
3. Or just hit the kiro-gateway API once — the gateway syncs the token on first call

---

## Credits

- **kiro-gateway** by [Jwadow](https://github.com/Jwadow/kiro-gateway) — Python proxy layer
- **kirolink** by [Alexandephilia](https://github.com/alexandeism/kirolink) — Go token management + Claude Code proxy
- **kiro-cli** — Kiro command-line auth tool (pip package)
- **AWS Builder ID** — Free OIDC identity provider

---

## License

MIT
