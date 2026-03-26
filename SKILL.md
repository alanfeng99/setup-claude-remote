---
name: setup-claude-remote
description: Set up or repair a long-lived Claude Code Remote Control session using platform-native persistence (systemd on Linux, launchd on macOS, Task Scheduler on Windows). Use when the user wants Claude Code to survive disconnects, restart after reboot, or remain available after logout.
argument-hint: [session-name] [working-directory]
disable-model-invocation: true
effort: medium
license: MIT
compatibility: Linux with systemd, macOS with launchd, or Windows with Task Scheduler. Requires claude CLI (and tmux on Linux/macOS).
---

# setup-claude-remote

You are setting up a durable Claude Code Remote Control environment on a Linux, macOS, or Windows machine.

Use this skill only when the user explicitly wants Claude Code Remote Control to:
1. survive SSH disconnects (Linux/macOS) or terminal closures (Windows),
2. restart automatically after reboot,
3. remain available after logout,
4. or be repaired when the existing setup is broken.

This skill performs real system changes. Be deliberate, idempotent, and transparent.

## Arguments

- `$0` = desired remote session name (optional)
- `$1` = working directory (optional)

If arguments are omitted:
- session name defaults to a slug derived from the current working directory name
- working directory defaults to the current working directory

## Goal

Set up this stack using the native service manager on each platform:

| Layer | Linux | macOS | Windows |
|-------|-------|-------|---------|
| Process keeper | tmux | tmux | Hidden PowerShell window |
| Service manager | systemd `--user` | launchd (LaunchAgents) | Task Scheduler |
| Survive logout | `loginctl enable-linger` | Built-in | Task Scheduler setting |
| Auto-start | `systemctl enable` | `RunAtLoad` in plist | "At log on" trigger |
| Attach to session | `tmux attach` | `tmux attach` | View log file |

- Claude Code Remote Control runs in an interactive session by default:
  - prefer `claude --remote-control "<name>"`
  - only use `claude remote-control --name "<name>"` if the user explicitly wants server mode

Do not assume that resume flags and remote-control flags can be safely combined unless the installed CLI help on the target machine confirms it.

## Required behavior

### 1) Inspect first, mutate second

Before creating or changing anything:

- detect OS and init system
- determine the platform: **Linux+systemd**, **macOS+launchd**, or **Windows+TaskScheduler**
- verify whether `tmux` exists (Linux/macOS only)
- verify whether `claude` exists
- verify Claude auth status
- check whether an existing service/script/session already exists

**Cross-platform detection commands:**

On Linux/macOS, use:

```bash
uname -s
ps -p 1 -o comm=
whoami
echo "$HOME"
pwd
command -v tmux || true
command -v claude || true
claude --version || true
claude auth status --text || true
tmux ls || true
```

On Windows (PowerShell), use:

```powershell
$env:OS
whoami
$env:USERPROFILE
Get-Location
Get-Command claude -ErrorAction SilentlyContinue | Select-Object -ExpandProperty Source
claude --version 2>&1
claude auth status --text 2>&1
Get-ScheduledTask -TaskName "Claude Remote*" -ErrorAction SilentlyContinue
```

**Platform-specific inspection:**

On Linux, also run:
```bash
systemctl --user --version || true
systemctl --user status --no-pager || true
loginctl show-user "$USER" -p Linger --value || true
```

On macOS, also run:
```bash
sw_vers || true
launchctl list | grep claude || true
ls ~/Library/LaunchAgents/com.claude.remote* 2>/dev/null || true
```

On Windows, also run:
```powershell
Get-ScheduledTask -TaskName "Claude Remote*" -ErrorAction SilentlyContinue | Format-Table TaskName, State
Get-Process -Name claude -ErrorAction SilentlyContinue
Test-Path "$env:USERPROFILE\bin\start-claude-remote-*.ps1"
```

**Platform decision logic:**

- If `uname -s` returns `Linux` and `systemctl --user` is available → use the **Linux+systemd** path
- If `uname -s` returns `Darwin` → use the **macOS+launchd** path
- If `$env:OS` equals `Windows_NT` → use the **Windows+TaskScheduler** path
- If running inside WSL2 on Windows, `uname -s` returns `Linux` → use the **Linux+systemd** path (WSL2 is a full Linux environment)
- If none of the above conditions are met, stop and explain. Offer tmux-only mode as a fallback on Unix-like systems.

### 2) Resolve names deterministically

Derive these values (shared across all platforms):

- `REMOTE_NAME` = `$0` if present, otherwise basename of current working directory
- `WORKDIR` = `$1` if present, otherwise current working directory
- `SLUG` = lowercase kebab-case derived from `REMOTE_NAME`

**Linux/macOS shared:**
- `TMUX_SESSION` = `claude-${SLUG}`
- `SCRIPT_PATH` = `$HOME/bin/start-claude-remote-${SLUG}.sh`

**Linux-specific:**
- `SERVICE_NAME` = `claude-remote-${SLUG}.service`
- `SERVICE_PATH` = `$HOME/.config/systemd/user/${SERVICE_NAME}`

**macOS-specific:**
- `PLIST_LABEL` = `com.claude.remote-control-${SLUG}`
- `PLIST_PATH` = `$HOME/Library/LaunchAgents/${PLIST_LABEL}.plist`
- `LOG_PATH` = `$HOME/Library/Logs/claude-remote-${SLUG}.log`

**Windows-specific:**
- `SCRIPT_PATH` = `$env:USERPROFILE\bin\start-claude-remote-{{SLUG}}.ps1`
- `TASK_NAME` = `Claude Remote Control ({{REMOTE_NAME}})`
- `LOG_DIR` = `$env:LOCALAPPDATA\Claude\Logs`
- `LOG_PATH` = `$env:LOCALAPPDATA\Claude\Logs\claude-remote-{{SLUG}}.log`

Never use spaces in tmux session names or filenames (except `TASK_NAME` which is a display name in Task Scheduler).

### 3) Show a concise action plan before changes

Before editing files or running privileged commands, tell the user exactly what you are about to do in 3 parts:

- files to create/update
- commands to run
- whether elevated privileges are needed (sudo on Linux, admin on Windows)

Keep this short and concrete.

### 4) Create an idempotent start script

#### 4a) Linux/macOS: Bash start script

Ensure these directories exist if missing:

```bash
mkdir -p "$HOME/bin"
```

On Linux, also:
```bash
mkdir -p "$HOME/.config/systemd/user"
```

On macOS, also:
```bash
mkdir -p "$HOME/Library/LaunchAgents"
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

"${TMUX_BIN}" new-session -d -s "${SESSION}" -c "${WORKDIR}" \
  "while true; do \"${CLAUDE_BIN}\" --remote-control \"${REMOTE_NAME}\"; sleep 2; done"
```

After writing it, run:

```bash
chmod +x "{{SCRIPT_PATH}}"
```

#### 4b) Windows: PowerShell start script

Ensure directories exist:

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\bin" -Force | Out-Null
New-Item -ItemType Directory -Path "{{LOG_DIR}}" -Force | Out-Null
```

Create `SCRIPT_PATH` as a `.ps1` file with this content:

```powershell
$ErrorActionPreference = "Stop"

$RemoteName = "{{REMOTE_NAME}}"
$WorkDir = "{{WORKDIR}}"
$LogPath = "{{LOG_PATH}}"

$ClaudeBin = (Get-Command claude -ErrorAction SilentlyContinue).Source
if (-not $ClaudeBin) {
    "$(Get-Date -Format o) ERROR: claude not found in PATH" | Out-File -FilePath $LogPath -Append -Encoding utf8
    exit 1
}

if (-not (Test-Path $WorkDir)) {
    "$(Get-Date -Format o) ERROR: working directory not found: $WorkDir" | Out-File -FilePath $LogPath -Append -Encoding utf8
    exit 1
}

Set-Location $WorkDir
"$(Get-Date -Format o) INFO: Starting Claude Remote Control '$RemoteName' in $WorkDir" | Out-File -FilePath $LogPath -Append -Encoding utf8

while ($true) {
    try {
        "$(Get-Date -Format o) INFO: Launching claude --remote-control $RemoteName" | Out-File -FilePath $LogPath -Append -Encoding utf8
        & $ClaudeBin --remote-control $RemoteName 2>&1 | ForEach-Object {
            "$(Get-Date -Format o) $_"
        } | Out-File -FilePath $LogPath -Append -Encoding utf8
    } catch {
        "$(Get-Date -Format o) ERROR: $($_.Exception.Message)" | Out-File -FilePath $LogPath -Append -Encoding utf8
    }
    "$(Get-Date -Format o) INFO: Claude exited, restarting in 2 seconds..." | Out-File -FilePath $LogPath -Append -Encoding utf8
    Start-Sleep -Seconds 2
}
```

Rules for the Windows script:

- all output goes to `LOG_PATH` with timestamps
- the restart loop mirrors the tmux bash script behavior
- use `$ErrorActionPreference = "Stop"` for strict error handling
- do not use `Start-Transcript` as it conflicts with redirection

### 5) Create the platform service

#### 5a) Linux: Create a user-level systemd service

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

#### 5b) macOS: Create a launchd plist

Create `PLIST_PATH` with this structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>{{PLIST_LABEL}}</string>

  <key>ProgramArguments</key>
  <array>
    <string>{{SCRIPT_PATH}}</string>
  </array>

  <key>RunAtLoad</key>
  <true/>

  <key>KeepAlive</key>
  <false/>

  <key>StandardOutPath</key>
  <string>{{LOG_PATH}}</string>

  <key>StandardErrorPath</key>
  <string>{{LOG_PATH}}</string>

  <key>EnvironmentVariables</key>
  <dict>
    <key>PATH</key>
    <string>{{RESOLVED_PATH}}</string>
  </dict>
</dict>
</plist>
```

Rules:

- `KeepAlive` is `false` because the start script is idempotent and the tmux restart loop handles Claude restarts internally
- `RunAtLoad` is `true` so the session starts at login
- `EnvironmentVariables > PATH` must include the directories where `claude` and `tmux` are installed — resolve this by inspecting the actual paths from inspection step (e.g., `dirname $(command -v claude)` and `dirname $(command -v tmux)`) and combining with standard system paths `/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`
- do not hardcode paths like `/opt/homebrew/bin` without verifying they exist on this machine
- do not write secrets into the plist
- keep the plist minimal and readable

#### 5c) Windows: Create a Scheduled Task

Register a Scheduled Task using PowerShell. Do not create a service file on disk — Task Scheduler manages the configuration internally.

```powershell
$Action = New-ScheduledTaskAction `
  -Execute "powershell.exe" `
  -Argument "-NoProfile -WindowStyle Hidden -ExecutionPolicy RemoteSigned -File `"{{SCRIPT_PATH}}`"" `
  -WorkingDirectory "{{WORKDIR}}"

$Trigger = New-ScheduledTaskTrigger -AtLogon -User "$env:USERNAME"

$Settings = New-ScheduledTaskSettingsSet `
  -AllowStartIfOnBatteries `
  -DontStopIfGoingOnBatteries `
  -StartWhenAvailable `
  -RestartInterval (New-TimeSpan -Minutes 1) `
  -RestartCount 3 `
  -ExecutionTimeLimit ([TimeSpan]::Zero)

$Principal = New-ScheduledTaskPrincipal `
  -UserId "$env:USERNAME" `
  -LogonType S4U `
  -RunLevel Limited

Register-ScheduledTask `
  -TaskName "{{TASK_NAME}}" `
  -Action $Action `
  -Trigger $Trigger `
  -Settings $Settings `
  -Principal $Principal `
  -Description "Claude Code Remote Control session ({{REMOTE_NAME}}) with auto-restart" `
  -Force
```

Rules:

- `-WindowStyle Hidden` keeps the PowerShell window invisible — the user interacts via Remote Control, not the terminal
- `-ExecutionTimeLimit ([TimeSpan]::Zero)` means no time limit — Task Scheduler will not kill the long-running process
- `-LogonType S4U` ("Service for User") allows the task to run whether or not the user is logged on, enabling survive-logout behavior without storing a password. Note: S4U tasks cannot access network resources or encrypted files (EFS). If the user needs those, fall back to `-LogonType Password` and explain the tradeoff (password stored in Task Scheduler's credential store).
- `-ExecutionPolicy RemoteSigned` allows locally-created scripts to run while blocking unsigned remote scripts. Only fall back to `Bypass` if the user explicitly confirms.
- `-Force` makes the command idempotent — it replaces any existing task with the same name
- `-StartWhenAvailable` ensures the task starts even if a scheduled run was missed (e.g., laptop was asleep)
- do not use `Register-ScheduledTask` flags that require admin unless the user confirms admin access
- do not write secrets into the task definition

### 6) Enable linger (Linux only)

> Skip this section entirely on macOS and Windows. macOS LaunchAgents and Windows Task Scheduler do not need linger — they handle persistence natively.

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

#### 7a) Linux

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

#### 7b) macOS

First, remove any existing version (idempotent):

```bash
launchctl bootout gui/$(id -u) "{{PLIST_PATH}}" 2>/dev/null || true
```

Then bootstrap:

```bash
launchctl bootstrap gui/$(id -u) "{{PLIST_PATH}}"
```

Then verify with:

```bash
launchctl list | grep "{{PLIST_LABEL}}" || true
tmux ls || true
```

If the tmux session does not appear within a few seconds, check the log file:

```bash
cat "{{LOG_PATH}}" || true
```

#### 7c) Windows

Start the task immediately:

```powershell
Start-ScheduledTask -TaskName "{{TASK_NAME}}"
```

Wait a few seconds, then verify:

```powershell
Get-ScheduledTask -TaskName "{{TASK_NAME}}" | Select-Object TaskName, State
Start-Sleep -Seconds 3
Get-Process -Name claude -ErrorAction SilentlyContinue | Select-Object Id, ProcessName, StartTime
```

If the task state is not `Running`, check the log:

```powershell
if (Test-Path "{{LOG_PATH}}") { Get-Content "{{LOG_PATH}}" -Tail 20 }
```

### 8) Validation checklist

Do not claim success until you verify all of the following:

**Shared (all platforms):**
- the start script exists
- the Claude process was launched

**Linux-specific:**
- the systemd service file exists
- the user service is enabled
- the tmux session exists
- linger is either confirmed enabled or explicitly reported as not enabled

**macOS-specific:**
- the plist file exists
- the launchd agent is loaded (`launchctl list` shows it)
- the tmux session exists
- the log file path is accessible

**Windows-specific:**
- the Scheduled Task exists and its state is `Running`
- the `claude` process is running
- the log file is being written to

If one of these fails, diagnose the failed step and continue from there instead of restarting everything blindly.

### 9) Safety and change control

Follow these rules strictly:

- never overwrite unrelated user services, launchd agents, or scheduled tasks
- never kill unrelated tmux sessions or processes
- if an existing script, service, plist, or task differs, summarize the delta before replacing it
- prefer in-place repair over delete-and-recreate
- do not silently switch to server mode
- **Linux:** do not claim reboot persistence unless both `systemd --user enable` succeeded and linger is enabled or already active
- **macOS:** do not claim reboot persistence unless the plist is loaded with `RunAtLoad` set to `true` — note that macOS LaunchAgents run at login, not at cold boot before any user logs in
- **Windows:** do not claim reboot persistence unless the Scheduled Task is registered with an `AtLogon` trigger. With `LogonType S4U`, the task also survives logout. Note that the task runs at user login, not before any user logs in. For pre-login startup, a Windows Service (via NSSM) would be needed, which is out of scope for this skill.
- if `claude auth status` suggests the user is not logged in, say so clearly and stop before pretending the remote session will work

### 10) Final response format

At the end, always report:

- remote name
- working directory
- platform detected
- script path
- service path / task name (platform-appropriate)
- **Linux/macOS:** tmux session name
- **Linux:** whether linger is enabled
- **macOS/Windows:** log file location
- the exact commands to inspect or manage (platform-appropriate)
- any remaining manual step, if one still exists

**Linux useful commands:**

```bash
tmux attach -t {{TMUX_SESSION}}
systemctl --user status {{SERVICE_NAME}}
journalctl --user -u {{SERVICE_NAME}} -f
systemctl --user restart {{SERVICE_NAME}}
```

**macOS useful commands:**

```bash
tmux attach -t {{TMUX_SESSION}}
launchctl list | grep {{PLIST_LABEL}}
tail -f {{LOG_PATH}}
launchctl bootout gui/$(id -u) {{PLIST_PATH}} && launchctl bootstrap gui/$(id -u) {{PLIST_PATH}}
```

**Windows useful commands:**

```powershell
# Check task status
Get-ScheduledTask -TaskName "{{TASK_NAME}}"

# View recent logs
Get-Content "{{LOG_PATH}}" -Tail 50

# Follow logs in real-time
Get-Content "{{LOG_PATH}}" -Wait -Tail 10

# Restart the task
Stop-ScheduledTask -TaskName "{{TASK_NAME}}"
Start-ScheduledTask -TaskName "{{TASK_NAME}}"

# Check if Claude is running
Get-Process -Name claude -ErrorAction SilentlyContinue
```

**Removal commands** (include only if the user asks or for reference):

Linux:
```bash
systemctl --user disable --now {{SERVICE_NAME}}
rm {{SERVICE_PATH}}
rm {{SCRIPT_PATH}}
systemctl --user daemon-reload
```

macOS:
```bash
launchctl bootout gui/$(id -u) {{PLIST_PATH}}
rm {{PLIST_PATH}}
rm {{SCRIPT_PATH}}
```

Windows:
```powershell
Stop-ScheduledTask -TaskName "{{TASK_NAME}}" -ErrorAction SilentlyContinue
Unregister-ScheduledTask -TaskName "{{TASK_NAME}}" -Confirm:$false
Remove-Item "{{SCRIPT_PATH}}" -Force
```

If a privileged step was skipped or failed, explicitly say the setup is partial.
