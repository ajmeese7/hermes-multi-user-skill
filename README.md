> [!WARNING]
> # ⚠️ Deprecated, now an official Hermes feature
> **Multi-agent separation is built into Hermes natively via [profiles](https://github.com/hermes-agent/hermes-agent). This skill is no longer needed and is archived for historical reference.**
>
> Use the native profile system instead of the sibling-`HERMES_HOME` approach this skill teaches:
>
> ```bash
> hermes profile create <name>        # creates ~/.hermes/profiles/<name>
> hermes gateway install              # installs hermes-gateway-<name>.service
> hermes --profile <name> ...         # run any command against a profile
> ```
>
> Why the native system is better: profiles get a correctly-scoped systemd unit (`hermes-gateway-<name>.service`) automatically, so they never collide with your primary gateway's unit, a real failure mode of the sibling-`~/.hermes-<name>` layout this skill used. Data isolation (config, memory, conversations, platform sessions, Docker sandbox) is identical; profiles additionally share one Nous account and one kanban board by design.

---

# hermes-multi-user

A [Hermes Agent](https://github.com/hermes-agent/hermes-agent) skill for setting up and managing multiple user instances on a single machine.

Run separate agents for family members, coworkers, or different personas — each with their own personality, memory, conversation history, and platform connections (WhatsApp, Slack, Telegram).

## Install

```bash
hermes skills install ajmeese7/hermes-multi-user-skill/hermes-multi-user --force
```

> **Why `--force`?** The built-in security scanner flags this skill as dangerous because it references `.env` files, `.bashrc`, and systemd services — which is exactly what a multi-user setup skill needs to do. Review the [SKILL.md](hermes-multi-user/SKILL.md) yourself before installing if you'd like to verify there's nothing malicious.

## What it does

This skill teaches your Hermes agent how to:

- Create isolated `HERMES_HOME` directories for additional users
- Set up systemd services so each instance runs independently
- Configure separate WhatsApp/Slack/Telegram connections per user
- Docker-sandbox secondary agents for security
- Version-control configs while excluding secrets and bundled data
- Detect custom vs bundled skills for clean backups

## Usage

Once installed, just ask your agent:

> "Set up a new Hermes instance for my wife Alice on WhatsApp"

> "Help me version-control my multi-agent configs"

The agent will follow the skill's step-by-step guide, adapting paths and settings to your system.

## Architecture

```
~/.hermes/              # Your agent (primary)
~/.hermes-alice/        # Alice's agent (separate everything)
  config.yaml           # Her settings
  SOUL.md               # Her personality
  .env                  # API keys
  memory/               # Her memory
  sessions/             # Her conversations
  skills/               # Her skills
  whatsapp/session/     # Her WhatsApp connection
```

All instances share the same hermes-agent codebase but are completely isolated in data and behavior.

## Blog Post

For a full walkthrough of how this was built, see: [How I Set Up Multi-User AI Agents with Hermes (And You Can Too)](https://blog.aaronmeese.com/how-i-set-up-multi-user-ai-agents-with-hermes-and-you-can-too-648b52fe20e9)

## License

MIT
