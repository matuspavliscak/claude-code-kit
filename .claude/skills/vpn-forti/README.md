# VPN — FortiClient VPN via CLI

Connect and disconnect FortiClient VPN from Claude Code using `openfortivpn` — no GUI automation needed.

## Why?

| Approach | Problem |
|----------|---------|
| FortiClient GUI | Requires manual clicks, can't be automated by Claude Code |
| FortiClient CLI tools | Undocumented, unreliable for scripting |
| **openfortivpn** | Clean CLI, well-documented, works reliably |

## Quick Start

```
/vpn-forti:vpn-forti connect       # connect to VPN
/vpn-forti:vpn-forti disconnect    # disconnect
/vpn-forti:vpn-forti status        # check connection state
/vpn-forti:vpn-forti setup         # guided first-time setup
```

> After installing, start a fresh `claude` session — `/reload-plugins` may not load skills correctly.

## How It Works

- `openfortivpn` runs as a backgrounded process with PID tracking
- Credentials passed via temporary config file (deleted after process starts)
- VPN password is retrieved from macOS Keychain at connect time
- Tunnel verified by checking for `ppp0` interface
- FortiClient GUI is quit before connecting (releases the connect lock)
- Disconnect sends SIGTERM, falls back to SIGKILL

## Setup Overview

1. `brew install openfortivpn`
2. Store VPN password in macOS Keychain
3. Get trusted certificate hash (first connection attempt)
4. Create config file at `~/.vpn/forti-config`
5. Configure passwordless `sudo` for `openfortivpn`
6. Deploy connect/disconnect scripts to `~/.local/bin/`

Full step-by-step instructions are in [SKILL.md](SKILL.md).

## Requirements

- macOS with Homebrew
- FortiClient installed (optional, used as status fallback)
- VPN credentials (username + password)
- VPN hostname and port

## Permissions

To allow VPN commands to run without Claude Code prompting for permission each time, add these to your `~/.claude/settings.json` under `permissions.allow`:

```json
"Bash(~/.local/bin/vpn_forti_connect.sh*):*",
"Bash(~/.local/bin/vpn_forti_disconnect.sh*):*",
"Bash(pgrep openfortivpn*):*",
"Bash(ifconfig ppp0*):*"
```

## Security

- Passwords are never stored in files — macOS Keychain only
- Credentials passed via temporary config file, deleted after process starts
- Sudoers config is scoped to `openfortivpn` and `killall openfortivpn` only
