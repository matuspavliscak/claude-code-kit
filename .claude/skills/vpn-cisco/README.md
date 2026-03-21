# VPN — Cisco Secure Client (AnyConnect) via CLI

Connect and disconnect Cisco Secure Client VPN from Claude Code using `openconnect` — no GUI automation needed.

## Why?

| Approach | Problem |
|----------|---------|
| Cisco GUI | Requires manual clicks, can't be automated by Claude Code |
| Cisco CLI (`vpn -s connect`) | TLS handshake stalls indefinitely with certificate auth |
| **openconnect** | Reads cert files directly from disk, works reliably |

## Quick Start

```
/vpn-cisco:vpn connect       # connect to VPN
/vpn-cisco:vpn disconnect    # disconnect
/vpn-cisco:vpn status        # check connection state
/vpn-cisco:vpn setup         # guided first-time setup
```

> After installing, start a fresh `claude` session — `/reload-plugins` may not load skills correctly.

## How It Works

- `openconnect` runs as a background daemon (`--background`)
- Certificate and private key are read from PEM files on disk
- VPN password is retrieved from macOS Keychain at connect time
- Cisco Secure Client GUI is quit before connecting (releases the connect lock)
- Disconnect kills the `openconnect` process, falls back to Cisco CLI

## Setup Overview

1. `brew install openconnect`
2. Store VPN password in macOS Keychain
3. Extract cert and key from your PFX file using `openssl`
4. Create config file at `~/.vpn/config`
5. Configure passwordless `sudo` for `openconnect`
6. Deploy connect/disconnect scripts to `~/.local/bin/`

Full step-by-step instructions are in [SKILL.md](SKILL.md).

## Requirements

- macOS with Homebrew
- Cisco Secure Client installed (used for status checks and as fallback)
- PFX certificate from IT
- VPN credentials (username + password)

## Permissions

To allow VPN commands to run without Claude Code prompting for permission each time, add these to your `~/.claude/settings.json` under `permissions.allow`:

```json
"Bash(~/.local/bin/vpn_connect.sh*):*",
"Bash(~/.local/bin/vpn_disconnect.sh*):*",
"Bash(pgrep openconnect*):*",
"Bash(/opt/cisco/secureclient/bin/vpn state*):*"
```

## Security

- Passwords are never stored in files — macOS Keychain only
- Private key on disk is `chmod 600` (owner-read only)
- Sudoers config is scoped to `openconnect` and `killall openconnect` only
