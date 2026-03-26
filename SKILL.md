---
name: setup-claude-remote
description: Set up or repair a long-lived Claude Code Remote Control session on Linux using tmux + systemd --user + loginctl enable-linger. Use when the user wants Claude Code to survive SSH disconnects and auto-start after reboot.
argument-hint: [session-name] [working-directory]
disable-model-invocation: true
effort: medium
---

# setup-claude-remote

You are setting up a durable Claude Code Remote Control environment on a Linux machine.

Use this skill only when the user explicitly wants Claude Code Remote Control to:
1. survive SSH disconnects,
2. restart automatically after reboot,
3. remain available after logout,
4. or be repaired when the existing tmux/systemd setup is broken.

This skill performs real system changes. Be deliberate, idempotent, and transparent.

## Arguments

- `$0` = desired remote session name (optional)
- `$1` = working directory (optional)

If arguments are omitted:
- session name defaults to a slug derived from the current working directory name
- working directory defaults to the current working directory

## Goal

Set up this stack:

- `tmux` keeps the Claude process alive when SSH disconnects
- `systemd --user` recreates the tmux session automatically after reboot
- `loginctl enable-linger` keeps the user manager alive after logout and allows boot-time startup
- Claude Code Remote Control runs in an interactive session by default:
  - prefer `claude --remote-control "<name>"`
  - only use `claude remote-control --name "<name>"` if the user explicitly wants server mode

Do not assume that resume flags and remote-control flags can be safely combined unless the installed CLI help on the target machine confirms it.

## Required behavior

### 1) Inspect first, mutate second

Before creating or changing anything:

- detect OS and init system
- verify whether systemd user services are available
- verify whether `tmux` exists
- verify whether `claude` exists
- verify Claude auth status
- verify whether linger is enabled
- check whether an existing service/script/session already exists

Use commands like:

```bash
uname -a
ps -p 1 -o comm=
whoami
echo "$HOME"
pwd
command -v tmux || true
command -v claude || true
claude --version || true
claude auth status --text || true
systemctl --user --version || true
systemctl --user status --no-pager || true
loginctl show-user "$USER" -p Linger --value || true
tmux ls || true
```

If systemd user services are unavailable, stop and explain clearly that this skill is written for Linux with systemd. Do not fake success. In that case, offer a fallback recommendation only:
- tmux-only mode, or
- a platform-native alternative such as launchd on macOS

### 2) Resolve names deterministically

Derive these values:

- `REMOTE_NAME` = `$0` if present, otherwise basename of current working directory
- `WORKDIR` = `$1` if present, otherwise current working directory
- `SLUG` = lowercase kebab-case derived from `REMOTE_NAME`
- `TMUX_SESSION` = `claude-${SLUG}`
- `SCRIPT_PATH` = `$HOME/bin/start-claude-remote-${SLUG}.sh`
- `SERVICE_NAME` = `claude-remote-${SLUG}.service`
- `SERVICE_PATH` = `$HOME/.config/systemd/user/${SERVICE_NAME}`

Never use spaces in tmux session names or filenames.

### 3) Show a concise action plan before changes

Before editing files or running privileged commands, tell the user exactly what you are about to do in 3 parts:

- files to create/update
- commands to run
- whether sudo is needed

Keep this short and concrete.

### 4) Create an idempotent start script

Ensure these directories exist if missing:

```bash
mkdir -p "$HOME/bin"
mkdir -p "$HOME/.config/systemd/user"
```

Create `SCRIPT_PATH` with executable permissions.

The script must:

- use `#!/usr/bin/env bash`
- use `set -euo pipefail`
- verify `tmux` and `claude` are available
- verify `WORKDIR` exists
- exit successfully if the tmux session already exists
- create exactly one named tmux session for this setup
- run Claude Code inside tmux in a restart loop
- sleep briefly before restarting if Claude exits

Generate script content in this pattern:

```bash
#!/usr/bin/env bash
set -euo pipefail

SESSION="{{TMUX_SESSION}}"
WORKDIR="{{WORKDIR}}"
REMOTE_NAME="{{REMOTE_NAME}}"
CLAUDE_BIN="${CLAUDE_BIN:-$(command -v claude || true)}"
TMUX_BIN="${TMUX_BIN:-$(command -v tmux || true)}"

if [ -z "${TMUX_BIN}" ]; then
  echo "tmux not found in PATH" >&2
  exit 1
fi

if [ -z "${CLAUDE_BIN}" ]; then
  echo "claude not found in PATH" >&2
  exit 1
fi

if [ ! -d "${WORKDIR}" ]; then
  echo "working directory not found: ${WORKDIR}" >&2
  exit 1
fi

if "${TMUX_BIN}" has-session -t "${SESSION}" 2>/dev/null; then
  exit 0
fi

"${TMUX_BIN}" new-session -d -s "${SESSION}" -c "${WORKDIR}"   "while true; do \"${CLAUDE_BIN}\" --remote-control \"${REMOTE_NAME}\"; sleep 2; done"
```

After writing it, run:

```bash
chmod +x "{{SCRIPT_PATH}}"
```

### 5) Create a user-level systemd service

Create `SERVICE_PATH` with this structure:

```ini
[Unit]
Description=Claude Code Remote Control session ({{REMOTE_NAME}}) in tmux
After=default.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=%h/bin/start-claude-remote-{{SLUG}}.sh
ExecStop=-/usr/bin/tmux kill-session -t {{TMUX_SESSION}}
TimeoutStopSec=10

[Install]
WantedBy=default.target
```

Rules:

- use `Type=oneshot` and `RemainAfterExit=yes`
- kill only the specific tmux session in `ExecStop`
- do not kill all tmux sessions
- do not write secrets into the service file
- keep the unit minimal and readable

### 6) Enable linger when needed

Check whether linger is already enabled.

If it is not enabled, explain why it matters:
- without linger, user services may not survive logout or start at boot as intended

Then run:

```bash
sudo loginctl enable-linger "$USER"
```

If sudo is unavailable or denied, say clearly:
- tmux will still help with SSH disconnects while the machine is running
- but reboot/logout persistence is not guaranteed without linger

### 7) Enable and start the service

Run:

```bash
systemctl --user daemon-reload
systemctl --user enable --now "{{SERVICE_NAME}}"
```

Then verify with:

```bash
systemctl --user status "{{SERVICE_NAME}}" --no-pager || true
tmux ls || true
```

If setup succeeds, also show the user these useful commands:

```bash
tmux attach -t {{TMUX_SESSION}}
systemctl --user status {{SERVICE_NAME}}
journalctl --user -u {{SERVICE_NAME}} -f
systemctl --user restart {{SERVICE_NAME}}
```

### 8) Validation checklist

Do not claim success until you verify all of the following:

- the start script exists
- the service file exists
- the user service is enabled
- the tmux session exists
- the Claude process was launched from inside that tmux session
- linger is either confirmed enabled or explicitly reported as not enabled

If one of these fails, diagnose the failed step and continue from there instead of restarting everything blindly.

### 9) Safety and change control

Follow these rules strictly:

- never overwrite unrelated user services
- never kill unrelated tmux sessions
- if an existing script or service differs, summarize the delta before replacing it
- prefer in-place repair over delete-and-recreate
- do not silently switch to server mode
- do not claim reboot persistence unless both:
  - `systemd --user enable` succeeded, and
  - linger is enabled or already active
- if `claude auth status` suggests the user is not logged in, say so clearly and stop before pretending the remote session will work

### 10) Final response format

At the end, always report:

- remote name
- working directory
- tmux session name
- script path
- service path
- whether linger is enabled
- the exact commands to inspect or attach
- any remaining manual step, if one still exists

If a privileged step was skipped or failed, explicitly say the setup is partial.
