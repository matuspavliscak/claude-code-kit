---
name: vpn
description: Connect or disconnect Cisco Secure Client VPN via CLI
user_invocable: true
---

Manage VPN connections via openconnect CLI (no GUI automation).

## Usage
- `/vpn` or `/vpn connect` - connect to VPN
- `/vpn disconnect` - disconnect from VPN
- `/vpn status` - show current VPN state
- `/vpn setup` - run full setup from scratch

## Prerequisites
- macOS with Homebrew
- Cisco Secure Client installed (used for status checks and as fallback)
- A PFX certificate file from IT + its password (the PFX password protects the certificate file - it is separate from your VPN login password)
- VPN credentials (username + password) - your regular VPN login credentials
- VPN profile name - open Cisco Secure Client GUI and check the dropdown to find yours (e.g. "VPN MyCompany")
- VPN hostname - provided by IT, or check the Cisco profile XML on your machine (typically under `/opt/cisco/secureclient/profile/`)

## Setup

### Step 1: Install openconnect
```bash
brew install openconnect
```

### Step 2: Store VPN password in macOS Keychain
`<keychain_service>` is an arbitrary label you choose (e.g. "vpn-mycompany"). It is only used to look up the password later - pick something descriptive. This value must match `KEYCHAIN_SERVICE` in Step 4's config file.
```bash
security add-generic-password -s "<keychain_service>" -a "<username>" -w
# Prompts for the VPN password interactively
```

### Step 3: Extract cert and key from PFX
macOS `security import` can't handle newer PKCS12 format (AES-256-CBC). Use openssl to extract directly to PEM:
```bash
mkdir -p ~/.vpn/certs

# Extract client certificate
openssl pkcs12 -in <path-to-pfx> -passin 'pass:<pfx_password>' \
  -clcerts -nokeys -out ~/.vpn/certs/<username>.pem

# Extract private key (unencrypted)
openssl pkcs12 -in <path-to-pfx> -passin 'pass:<pfx_password>' \
  -nocerts -nodes -out ~/.vpn/certs/<username>.key

# Restrict key permissions
chmod 600 ~/.vpn/certs/<username>.key
```
**Security note:** The private key is stored unencrypted on disk. `chmod 600` restricts read access to your macOS user only, but anyone with access to your macOS account (or root) can read it.

### Step 4: Write config file
Create `~/.vpn/config`:
```
VPN_PROFILE=<vpn_profile_name>
VPN_USERNAME=<username>
KEYCHAIN_SERVICE=<keychain_service>
VPN_CLI=/opt/cisco/secureclient/bin/vpn
VPN_HOST=<vpn_hostname>
VPN_CERT=$HOME/.vpn/certs/<username>.pem
VPN_KEY=$HOME/.vpn/certs/<username>.key
```
Placeholder guide:
- `VPN_PROFILE` - the profile name shown in the Cisco Secure Client dropdown (e.g. "VPN MyCompany")
- `VPN_USERNAME` - your VPN login username
- `KEYCHAIN_SERVICE` - the arbitrary label you chose in Step 2 (e.g. "vpn-mycompany")
- `VPN_HOST` - the VPN server hostname (ask IT or check your Cisco profile XML)

### Step 5: Set up passwordless sudo
openconnect needs root for tun device and routing. Without this, Claude Code can't run connect/disconnect autonomously.
```bash
sudo visudo -f /etc/sudoers.d/openconnect
```
Add these lines (replace `<user>` with your macOS username):
```
<user> ALL=(ALL) NOPASSWD: /opt/homebrew/bin/openconnect
<user> ALL=(ALL) NOPASSWD: /usr/bin/killall openconnect
```
Note: `/usr/bin/kill` doesn't exist on macOS - use `/usr/bin/killall` instead.

### Step 6: Create scripts
```bash
mkdir -p ~/.local/bin
```
Ensure `~/.local/bin` is in your PATH. Add to `~/.zshrc` if not already present:
```bash
export PATH="$HOME/.local/bin:$PATH"
```
Create `~/.local/bin/vpn_connect.sh`:
```bash
#!/bin/bash
# Connect to VPN via openconnect (CLI, no GUI automation)
# Config: ~/.vpn/config | Password: macOS Keychain
# Requires: openconnect (brew install openconnect), sudo

set -euo pipefail

CONFIG="$HOME/.vpn/config"
if [ ! -f "$CONFIG" ]; then
    echo "ERROR: No config found at $CONFIG"
    echo "Run /vpn setup to configure."
    exit 1
fi
source "$CONFIG"

VPN_HOST="${VPN_HOST:?Set VPN_HOST in $CONFIG}"
VPN_CERT="${VPN_CERT:?Set VPN_CERT in $CONFIG}"
VPN_KEY="${VPN_KEY:?Set VPN_KEY in $CONFIG}"

# Check if already connected via openconnect
if pgrep -q openconnect 2>/dev/null; then
    echo "VPN already connected (openconnect running)."
    exit 0
fi

# Also check Cisco client state (in case GUI connected)
VPN_CLI="${VPN_CLI:-/opt/cisco/secureclient/bin/vpn}"
if [ -x "$VPN_CLI" ]; then
    STATE=$("$VPN_CLI" state 2>&1 | grep ">> state:" | tail -1 || true)
    if echo "$STATE" | grep -q "Connected"; then
        echo "VPN already connected (Cisco client)."
        exit 0
    fi
    # Quit GUI if running (releases connect lock that blocks openconnect)
    osascript -e 'tell application "Cisco Secure Client" to quit' 2>/dev/null || true
    sleep 1
fi

# Verify cert and key exist
if [ ! -f "$VPN_CERT" ]; then
    echo "ERROR: Certificate not found at $VPN_CERT"
    exit 1
fi
if [ ! -f "$VPN_KEY" ]; then
    echo "ERROR: Private key not found at $VPN_KEY"
    exit 1
fi

# Get password from Keychain
PASSWORD=$(security find-generic-password -s "$KEYCHAIN_SERVICE" -a "$VPN_USERNAME" -w 2>/dev/null)
if [ -z "$PASSWORD" ]; then
    echo "ERROR: No password in Keychain for service=$KEYCHAIN_SERVICE account=$VPN_USERNAME"
    exit 1
fi

echo "Connecting to $VPN_HOST..."

# openconnect needs sudo for tun device and routing
echo "$PASSWORD" | sudo openconnect \
    --protocol=anyconnect \
    --certificate="$VPN_CERT" \
    --sslkey="$VPN_KEY" \
    --user="$VPN_USERNAME" \
    --passwd-on-stdin \
    --background \
    --quiet \
    "$VPN_HOST" 2>&1

# Wait for process to appear (up to 10 seconds)
for i in {1..10}; do
    if pgrep -q openconnect 2>/dev/null; then
        echo "VPN connected."
        exit 0
    fi
    sleep 1
done

echo "VPN connection failed. Check credentials or try manually:"
echo "  sudo openconnect --certificate=$VPN_CERT --sslkey=$VPN_KEY --user=$VPN_USERNAME $VPN_HOST"
exit 1
```

Create `~/.local/bin/vpn_disconnect.sh` (chmod +x):
```bash
#!/bin/bash
# Disconnect VPN (supports both openconnect and Cisco client)

set -euo pipefail

CONFIG="$HOME/.vpn/config"
if [ -f "$CONFIG" ]; then
    source "$CONFIG"
fi

# Check openconnect first
if pgrep -q openconnect 2>/dev/null; then
    sudo /usr/bin/killall openconnect
    sleep 2
    if ! pgrep -q openconnect 2>/dev/null; then
        echo "VPN disconnected."
        exit 0
    fi
    echo "VPN disconnect may not have completed. Check manually."
    exit 1
fi

# Fall back to Cisco CLI
VPN_CLI="${VPN_CLI:-/opt/cisco/secureclient/bin/vpn}"
if [ -x "$VPN_CLI" ]; then
    STATE=$("$VPN_CLI" state 2>&1 | grep ">> state:" | tail -1 || true)
    if echo "$STATE" | grep -q "Disconnected"; then
        echo "VPN already disconnected."
        exit 0
    fi
    "$VPN_CLI" disconnect >/dev/null 2>&1
    sleep 2
    STATE=$("$VPN_CLI" state 2>&1 | grep ">> state:" | tail -1 || true)
    if echo "$STATE" | grep -q "Disconnected"; then
        echo "VPN disconnected."
        exit 0
    fi
    echo "VPN disconnect may not have completed. Check manually."
    exit 1
fi

echo "VPN already disconnected."
exit 0
```

Make both executable:
```bash
chmod +x ~/.local/bin/vpn_connect.sh ~/.local/bin/vpn_disconnect.sh
```

### Step 7: Verify
```bash
~/.local/bin/vpn_connect.sh   # Should connect
~/.local/bin/vpn_disconnect.sh # Should disconnect
~/.local/bin/vpn_connect.sh   # Should reconnect
```

## Connect
Run `~/.local/bin/vpn_connect.sh`. Report result concisely.

## Disconnect
Run `~/.local/bin/vpn_disconnect.sh`. Report result concisely.

## Status
Check in order:
1. `pgrep openconnect` - if running, VPN is connected via openconnect
2. Cisco CLI `state` command - check Cisco client state
Report Connected/Disconnected.

## How it works
- openconnect runs with `--background` flag as a daemon
- Reads cert+key directly from PEM files (bypasses Keychain)
- Password retrieved from macOS Keychain at connect time
- Cisco GUI is quit before connecting (releases connect lock)
- Disconnect kills openconnect process, falls back to Cisco CLI

## Troubleshooting
- **"Unexpected 404 result from server"** - harmless warning, common with Cisco AnyConnect servers. Connection works fine.
- **PFX import fails with "wrong password"** - macOS `security import` can't handle AES-256-CBC PKCS12. Use openssl to extract cert+key directly (Step 3 above).
- **Cisco CLI TLS handshake stalls** - known issue: vpnagentd stalls when presenting Keychain identity during TLS. This is why we use openconnect instead.
- **"sudo: a terminal is required"** - passwordless sudo not configured. Run Step 5.
- **Scripts missing** - recreate from the templates above (Step 6).
- **Why not Cisco CLI?** - The Cisco `vpn -s connect` command doesn't work for certificate-based auth. Even with a valid Keychain identity (cert + private key with `-A` flag), the vpnagentd daemon's TLS handshake stalls indefinitely. openconnect reads cert files directly from disk, bypassing Keychain entirely.
