---
name: hermes-multi-user
description: Set up and manage multiple Hermes Agent instances on a single machine — separate HERMES_HOME directories, systemd services, WhatsApp/Slack routing, config version control, and a sync script for tracking changes. Use when adding a second user (family member, coworker, etc.) to your Hermes setup.
tags: [hermes, multi-agent, multi-user, config, systemd, whatsapp]
---

# Multi-User Hermes Setup

## When to Use
- Adding a second (or third) Hermes user on the same machine
- Each user needs their own personality, memory, sessions, and conversation history
- Users connect via different WhatsApp numbers, Slack accounts, or Telegram chats
- You want to version-control configs without leaking secrets or bundled skills

## Architecture

Each user gets their own `HERMES_HOME` directory. All instances share the same hermes-agent codebase but are completely isolated in terms of data:

```
~/.hermes/              # Primary user (default)
~/.hermes-<name>/       # Additional user
```

### What each instance contains
```
~/.hermes-<name>/
  config.yaml           # Agent settings, model, toolsets, platform config
  SOUL.md               # Personality/persona
  .env                  # API keys (copy from primary)
  skills/               # Auto-populated with bundled skills on first run
  memory/               # User-specific memory
  sessions/             # Conversation history
  cron/                 # Scheduled jobs
  logs/                 # Runtime logs
  whatsapp/session/     # WhatsApp pairing data (if applicable)
```

## Step-by-Step Setup

### 1. Create the directory structure

```bash
NEW_USER="alice"  # Change this
HERMES_NEW="$HOME/.hermes-$NEW_USER"

mkdir -p "$HERMES_NEW"/{memory,sessions,logs,cron,skills,whatsapp/session}
```

### 2. Copy API keys

```bash
# Copy .env but review it — you may want different model settings
cp ~/.hermes/.env "$HERMES_NEW/.env"
```

### 3. Create config.yaml

Start from your primary config and adjust:

```yaml
_config_version: 10

agent:
  max_turns: 200
  reasoning_effort: medium

model:
  default: anthropic/claude-sonnet-4.6
  fallback: anthropic/claude-opus-4.6
  provider: anthropic

# Restrict toolsets per platform as needed
platform_toolsets:
  whatsapp:
  - browser
  - clarify
  - code_execution
  - cronjob
  - delegation
  - file
  - image_gen
  - memory
  - session_search
  - skills
  - terminal
  - todo
  - tts
  - vision
  - web

# Use Docker sandboxing for additional users (recommended)
terminal:
  backend: docker
  docker_image: nikolaik/python-nodejs:python3.11-nodejs20
  container_cpu: 1
  container_memory: 4096
  container_disk: 51200
  container_persistent: true
  timeout: 180

# If using WhatsApp, use a DIFFERENT bridge port than primary (default: 3000)
platforms:
  whatsapp:
    extra:
      bridge_port: 3001
      session_path: /home/<your-username>/.hermes-<name>/whatsapp/session
```

### 4. Create SOUL.md

Write a personality file tailored to the new user:

```markdown
You are a friendly, helpful AI assistant. You communicate clearly and warmly.

Keep responses concise since you're on WhatsApp — no one wants to read
an essay in a chat bubble. Use plain language, avoid jargon unless asked.
```

### 5. Migrate or disable WhatsApp on primary (if applicable)

If the new user is taking over WhatsApp from your primary instance:

```bash
# 1. Stop primary gateway first
systemctl --user stop hermes-gateway

# 2. Copy existing WhatsApp session to avoid re-pairing
cp -r ~/.hermes/whatsapp/session/* "$HERMES_NEW/whatsapp/session/"

# 3. Disable WhatsApp on primary
# In ~/.hermes/.env, set:
WHATSAPP_ENABLED=false

# 4. Restart primary
systemctl --user start hermes-gateway
```

If both users will have their own WhatsApp connections, skip the copy and just pair each one separately (step 8).

### 6. Create a systemd service

Save to `~/.config/systemd/user/hermes-<name>-gateway.service`:

```ini
[Unit]
Description=Hermes Agent Gateway - <Name>'s Instance
After=network.target
StartLimitIntervalSec=600
StartLimitBurst=5

[Service]
Type=simple
ExecStart=<HERMES_VENV>/bin/python -m hermes_cli.main gateway run --replace
WorkingDirectory=<HERMES_AGENT_DIR>
Environment="PATH=<HERMES_VENV>/bin:<NODE_BIN>:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="VIRTUAL_ENV=<HERMES_VENV>"
Environment="HERMES_HOME=<HERMES_NEW>"
Restart=on-failure
RestartSec=30
KillMode=mixed
KillSignal=SIGTERM
TimeoutStopSec=60
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```

Replace the placeholders:
- `<HERMES_VENV>` → e.g. `/home/you/.hermes/hermes-agent/venv`
- `<HERMES_AGENT_DIR>` → e.g. `/home/you/.hermes/hermes-agent`
- `<NODE_BIN>` → output of `dirname $(which node)`
- `<HERMES_NEW>` → e.g. `/home/you/.hermes-alice`
- `<Name>` → e.g. `Alice`

Key point: `HERMES_HOME` points to the new user's directory, but `WorkingDirectory` points to the **shared** hermes-agent install.

### 7. Enable and start

```bash
systemctl --user daemon-reload
systemctl --user enable hermes-<name>-gateway
systemctl --user start hermes-<name>-gateway
```

### 8. Pair WhatsApp (if applicable)

```bash
# Stop the service first (bridge can't pair while gateway owns it)
systemctl --user stop hermes-<name>-gateway

HERMES_HOME=~/.hermes-<name> hermes whatsapp
```

Scan the QR code from the new user's phone. After pairing succeeds, start the service:

```bash
systemctl --user start hermes-<name>-gateway
```

The service is `enabled`, so it will auto-start on boot going forward — this manual start is only needed after the initial pairing.

**Important:** `WHATSAPP_ALLOWED_USERS` in `.env` must use bare country+number format without a `+` prefix (e.g. `11234567890`, not `+11234567890`).

## CLI Access

The `hermes` command defaults to `~/.hermes`. Override with `HERMES_HOME`:

```bash
# One-off command
HERMES_HOME=~/.hermes-alice hermes chat

# Convenient alias (add to ~/.bashrc or ~/.zshrc)
alias hermes-alice='HERMES_HOME=~/.hermes-alice hermes'

# Then just use:
hermes-alice chat
hermes-alice whatsapp
hermes-alice gateway status
```

## Version Control Strategy

Track configs in git **without** secrets or bundled data:

### What to track
- `config.yaml` — agent configuration
- `SOUL.md` — personality
- `cron/jobs.json` — scheduled jobs (strip runtime state)
- Custom skills only (not bundled ones)

### What to exclude (.gitignore)
```gitignore
.env
*.env.*
whatsapp/
memory/
sessions/
logs/
cron/output/
.DS_Store
```

### Detecting custom vs bundled skills

Hermes maintains `skills/.bundled_manifest` listing every bundled skill with a content hash. Any skill directory NOT in this manifest is custom (user-created):

```python
from pathlib import Path

def get_custom_skills(hermes_home: Path) -> set[str]:
    manifest = hermes_home / "skills" / ".bundled_manifest"
    bundled = set()
    if manifest.exists():
        bundled = {l.split(":")[0] for l in manifest.read_text().splitlines() if ":" in l}
    all_skills = {p.parent.name for p in (hermes_home / "skills").rglob("SKILL.md")}
    return all_skills - bundled
```

## Pitfalls

1. **WhatsApp port conflicts** — each agent using WhatsApp MUST have a different `bridge_port` value, or they'll fail to bind
2. **Don't symlink skills/ directories between agents** — skills created by one agent would bleed into the other
3. **Docker sandboxing recommended for secondary users** — prevents the agent from accessing the primary user's files or other system resources
4. **Session data is NOT shared** — each agent has completely independent memory, sessions, and conversation history
5. **API key costs are shared** — all instances use the same API keys from .env, so usage bills to the same account
6. **Restart after config changes** — `systemctl --user restart hermes-<name>-gateway` to pick up config.yaml or .env changes

## Verification

After setup, confirm everything is working:

```bash
# Check service status
systemctl --user status hermes-<name>-gateway

# View logs
journalctl --user -u hermes-<name>-gateway -f

# Test CLI access
HERMES_HOME=~/.hermes-<name> hermes chat
```
