# `gemvm` Architecture & Development Context

## Objective
`gemvm` is a Python-based CLI tool designed to manage isolated ARM Linux VMs on macOS (Apple Silicon). It provides a high-permission sandbox (via Lima's `vz` backend) tailored for safe execution of the Gemini CLI in "yolo mode".

## Design Philosophy & Architecture
`gemvm` was designed with simplicity and self-containment in mind, heavily inspired by (but explicitly diverging from) a similar Bash-based tool called `claudevm`.

- **Self-Contained Configuration:** Unlike `claudevm` which relies on external YAML templates, `gemvm` generates its `lima.yaml` dynamically within the Python script using template strings. This makes the tool entirely portable as a single executable.
- **Native Tooling:** Wherever possible, `gemvm` leverages Lima's native capabilities (`limactl copy` instead of parsing SSH configs for `scp`, and `ssh -F` for direct routing) to avoid brittle shell parsing.
- **Smart Session Management:** Integrates terminal window title management (`xterm` control sequences) and intelligent `tmux` attachment. If a user exits the Gemini CLI, the tool will automatically reinject `gemini --yolo` into the pane upon the next connection.

## Core Features
- **Automated Lifecycle:** `create`, `stop`, `delete`, and `list` operations wrapper around `limactl`.
- **Pre-Authenticated "Yolo" Sandbox:**
  - Installs Node.js v20 (via NodeSource) and the Gemini CLI globally.
  - Automatically trusts the `/` directory in `~/.gemini/trustedFolders.json` to enable promptless execution safely inside the VM.
  - Syncs the host's `oauth_creds.json` so the VM session is pre-logged into the user's Google AI "Pro" subscription. (Note: Google OAuth refresh tokens do not require keep-alive pings; they remain valid indefinitely).
- **Environment Seeding:** Supports a `~/.gemvm/skeleton/` directory. Files are synced using `bsdtar` with `--no-xattrs`, `--no-mac-metadata`, and explicit exclusions for `.DS_Store` to prevent macOS junk from polluting the Linux VM.
- **Native Scrollback:** Provisions a default `~/.tmux.conf` that disables the alternate screen (`set -ga terminal-overrides ',xterm*:smcup@:rmcup@'`), allowing macOS terminal emulators (like iTerm2) to retain native scrollback history.

## Knowledge Base & Workarounds

### Networking & SSH Delay (vzNAT)
When using the `vz` virtualization framework, the default network is completely isolated. To allow the VM to reach the local network (e.g., 192.168.x.x), `vzNAT: true` is enabled in the config.

**The Bug:** The NAT gateway lacks reverse DNS. When a user connects via SSH, `sshd` attempts a DNS lookup on the NAT IP, which hangs for ~4 seconds before timing out and identifying the host as `UNKNOWN`.
**The Fix:** The provisioning script automatically appends `127.0.0.1 UNKNOWN` to `/etc/hosts` and disables `UseDNS` in `/etc/ssh/sshd_config` to short-circuit this lookup and restore instant PTY login times.
- References:
  - [Lima GitHub Issue #1282: SSH timeout with vzNAT](https://github.com/lima-vm/lima/issues/1282)
  - [Lima Networking Documentation](https://github.com/lima-vm/lima/blob/master/docs/network.md)

### SSH Agent Forwarding
Lima's generated SSH configuration includes `ControlMaster auto`. If a background process establishes the initial multiplexed connection without agent forwarding, subsequent interactive connections (`ssh -A`) will reuse that socket and silently drop the forwarding request. 
**The Fix:** The `gemvm ssh` command explicitly passes `-o ControlPath=none` to bypass the multiplexer and guarantee a fresh connection for agent forwarding.
