# setup-claude-remote

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that sets up durable, long-lived Claude Code Remote Control sessions on **Linux**, **macOS**, and **Windows**.

## The Problem

Running Claude Code remotely is fragile:

- **SSH disconnects** (Linux/macOS) or **terminal closures** (Windows) kill the Claude process
- **Machine reboots** require manual restart
- **Logging out** tears down the session and everything running in it

You end up babysitting your remote Claude sessions instead of letting them work autonomously.

## The Solution

This skill automates the entire setup to make Claude Code Remote Control survive disconnects, reboots, and logouts — using the native service manager on each platform:

| Layer | Linux | macOS | Windows |
|-------|-------|-------|---------|
| **Process keeper** | tmux | tmux | Hidden PowerShell window |
| **Service manager** | systemd `--user` | launchd (LaunchAgents) | Task Scheduler |
| **Survive logout** | `loginctl enable-linger` | Built-in | S4U LogonType |
| **Auto-start** | `systemctl enable` | `RunAtLoad` in plist | "At log on" trigger |
| **Attach to session** | `tmux attach` | `tmux attach` | View log file |

The result is a Claude Code Remote Control session that is always available — you can disconnect, reconnect, reboot, and Claude keeps running.

## Installation

### As a Claude Code Skill

```bash
claude skill add --name setup-claude-remote --url https://github.com/alanfeng99/setup-claude-remote
```

Or manually copy `SKILL.md` to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/setup-claude-remote
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

When invoked, the skill detects your platform and performs these steps safely and idempotently:

1. **Inspects the environment** — detects OS, verifies dependencies, checks auth status, and finds existing setup
2. **Shows an action plan** — lists exactly what files will be created and commands will be run
3. **Creates a start script** — a bash script (Linux/macOS) or PowerShell script (Windows) that launches Claude with auto-restart
4. **Creates a platform service** — systemd user service on Linux, launchd plist on macOS, Scheduled Task on Windows
5. **Enables persistence** — linger on Linux (requires sudo), built-in on macOS, Task Scheduler on Windows
6. **Starts the service** — brings everything online and verifies it works
7. **Reports results** — shows session details and platform-appropriate management commands

### Safety Guarantees

- **Inspect first, mutate second** — always checks the current state before making changes
- **Idempotent** — safe to run multiple times; won't duplicate sessions or overwrite unrelated services
- **Non-destructive** — never kills unrelated tmux sessions, processes, or scheduled tasks
- **Transparent** — shows a clear action plan before making any changes
- **Graceful degradation** — works without sudo on Linux (tmux-only mode) and clearly reports what's missing

## Requirements

### Linux
- systemd (Ubuntu, Debian, Fedora, RHEL, etc.)
- tmux installed
- Claude Code CLI installed and authenticated
- sudo access (optional, needed for loginctl linger)

### macOS
- macOS 12+ (Monterey or later recommended)
- tmux installed (e.g., `brew install tmux`)
- Claude Code CLI installed and authenticated

### Windows
- Windows 10/11 or Windows Server 2016+
- PowerShell 5.1+ (included with Windows)
- Claude Code CLI installed and authenticated
- WSL2 users: the Linux path applies inside WSL

## After Setup

### Linux

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

### macOS

```bash
# Attach to the Claude session
tmux attach -t claude-my-project

# Check agent status
launchctl list | grep com.claude.remote-control-my-project

# View logs
tail -f ~/Library/Logs/claude-remote-my-project.log

# Restart the agent
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.claude.remote-control-my-project.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude.remote-control-my-project.plist
```

### Windows

```powershell
# Check task status
Get-ScheduledTask -TaskName "Claude Remote Control (my-project)"

# View recent logs
Get-Content "$env:LOCALAPPDATA\Claude\Logs\claude-remote-my-project.log" -Tail 50

# Follow logs in real-time
Get-Content "$env:LOCALAPPDATA\Claude\Logs\claude-remote-my-project.log" -Wait -Tail 10

# Restart the task
Stop-ScheduledTask -TaskName "Claude Remote Control (my-project)"
Start-ScheduledTask -TaskName "Claude Remote Control (my-project)"
```

## Platform Notes

### Windows: No tmux equivalent

Windows does not have tmux, so you cannot "attach" to the running Claude session interactively. Instead:
- Claude runs in a hidden PowerShell window managed by Task Scheduler
- All output is logged to `%LOCALAPPDATA%\Claude\Logs\`
- You interact with Claude through the Remote Control client from any terminal

### WSL2 on Windows

If you are running Claude Code inside WSL2, `uname -s` returns `Linux` and the skill will use the full Linux+systemd path. This gives you tmux attach capability and is the recommended approach for Windows users who want the richest experience.

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## License

[MIT](LICENSE)
