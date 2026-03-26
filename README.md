# setup-claude-remote

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that sets up durable, long-lived Claude Code Remote Control sessions on Linux servers using **tmux + systemd + loginctl linger**.

## The Problem

Running Claude Code remotely over SSH is fragile:

- **SSH disconnects** kill the Claude process
- **Server reboots** require manual restart
- **Logging out** tears down the user session and everything running in it

You end up babysitting your remote Claude sessions instead of letting them work autonomously.

## The Solution

This skill automates the entire setup to make Claude Code Remote Control survive disconnects, reboots, and logouts:

| Layer | Purpose |
|-------|---------|
| **tmux** | Keeps the Claude process alive when SSH disconnects |
| **systemd --user** | Recreates the tmux session automatically after reboot |
| **loginctl enable-linger** | Keeps the user manager alive after logout and enables boot-time startup |

The result is a Claude Code Remote Control session that is always available — you can disconnect, reconnect, reboot the server, and Claude keeps running.

## Installation

### As a Claude Code Skill

```bash
claude skill add --name setup-claude-remote --url https://github.com/alanfeng/setup-claude-remote
```

Or manually copy `SKILL.md` to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills
cp SKILL.md ~/.claude/skills/setup-claude-remote/SKILL.md
```

## Usage

Once installed, invoke the skill in Claude Code:

```
/setup-claude-remote
```

With optional arguments:

```
/setup-claude-remote my-project /home/user/projects/my-project
```

### Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| `session-name` | Name for the remote session | Derived from current directory |
| `working-directory` | Working directory for Claude | Current working directory |

## What It Does

When invoked, the skill performs these steps safely and idempotently:

1. **Inspects the environment** — checks OS, init system, tmux, claude CLI, auth status, and existing setup
2. **Shows an action plan** — lists exactly what files will be created and commands will be run
3. **Creates a start script** — an idempotent bash script in `~/bin/` that launches Claude in a tmux session with auto-restart
4. **Creates a systemd user service** — enables automatic startup and clean shutdown
5. **Enables linger** — ensures the user session persists after logout (requires sudo)
6. **Starts the service** — brings everything online and verifies it works
7. **Reports results** — shows session details and useful management commands

### Safety Guarantees

- **Inspect first, mutate second** — always checks the current state before making changes
- **Idempotent** — safe to run multiple times; won't duplicate sessions or overwrite unrelated services
- **Non-destructive** — never kills unrelated tmux sessions or overwrites unrelated systemd services
- **Transparent** — shows a clear action plan before making any changes
- **Graceful degradation** — works without sudo (tmux-only mode) and clearly reports what's missing

## Requirements

- **Linux** with systemd (Ubuntu, Debian, Fedora, RHEL, etc.)
- **tmux** installed
- **Claude Code CLI** installed and authenticated
- **sudo** access (optional, needed for loginctl linger)

## After Setup

Useful commands provided after successful setup:

```bash
# Attach to the Claude session
tmux attach -t claude-my-project

# Check service status
systemctl --user status claude-remote-my-project.service

# View logs
journalctl --user -u claude-remote-my-project.service -f

# Restart the service
systemctl --user restart claude-remote-my-project.service
```

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## License

[MIT](LICENSE)
