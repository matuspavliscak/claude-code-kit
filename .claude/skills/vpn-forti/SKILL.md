---
name: vpn-forti
description: Connect or disconnect FortiClient VPN via CLI
user_invocable: true
---

Manage FortiClient VPN connections via openfortivpn CLI (no GUI automation).

## Usage
- `/vpn-forti:vpn-forti` or `/vpn-forti:vpn-forti connect` - connect to VPN
- `/vpn-forti:vpn-forti disconnect` - disconnect from VPN
- `/vpn-forti:vpn-forti status` - show current VPN state
- `/vpn-forti:vpn-forti setup` - run full setup from scratch

Note: Plugin skills use the `/{skill}:{plugin}` format. After installing, start a fresh `claude` session — `/reload-plugins` may not load skills correctly.

## Prerequisites
- macOS with Homebrew
- FortiClient installed (optional, used as status fallback)
- VPN credentials (username + password) - your regular VPN login credentials
- VPN hostname and port - check FortiClient GUI or ask IT (typically port 443 or 10443)
- Trusted certificate SHA256 hash - obtained on first connection attempt (Step 3)

## Setup

### Step 1: Install openfortivpn
```bash
brew install openfortivpn
```

### Step 2: Store VPN password in macOS Keychain
`<keychain_service>` is an arbitrary label you choose (e.g. "vpn-forti-mycompany"). It is only used to look up the password later - pick something descriptive. This value must match `KEYCHAIN_SERVICE` in Step 4's config file.
```bash
security add-generic-password -s "<keychain_service>" -a "<username>" -w
# Prompts for the VPN password interactively
```

### Step 3: Get trusted certificate hash
On first connection attempt, openfortivpn shows the server certificate hash and refuses to connect. Run this to capture it:
```bash
sudo openfortivpn <vpn_host>:<port> -u <username> 2>&1 | grep "trusted-cert"
```
It outputs something like: `--trusted-cert <sha256hash>`. Copy the hash value for Step 4. Verify with IT if needed.

### Step 4: Write config file
Create `~/.vpn/forti-config`:
```
VPN_HOST=<vpn_hostname>
VPN_PORT=<vpn_port>
VPN_USERNAME=<username>
KEYCHAIN_SERVICE=<keychain_service>
TRUSTED_CERT=<sha256_certificate_hash>
```
Placeholder guide:
- `VPN_HOST` - the FortiGate VPN server hostname (check FortiClient GUI or ask IT)
- `VPN_PORT` - the server port (typically `443` or `10443`)
- `VPN_USERNAME` - your VPN login username
- `KEYCHAIN_SERVICE` - the arbitrary label you chose in Step 2 (e.g. "vpn-forti-mycompany")
- `TRUSTED_CERT` - the SHA256 hash from Step 3

### Step 5: Set up passwordless sudo
openfortivpn needs root for tun device and routing. Without this, Claude Code can't run connect/disconnect autonomously.
```bash
sudo visudo -f /etc/sudoers.d/openfortivpn
```
Add these lines (replace `<user>` with your macOS username):
```
<user> ALL=(ALL) NOPASSWD: /opt/homebrew/bin/openfortivpn
<user> ALL=(ALL) NOPASSWD: /usr/bin/killall openfortivpn
```

### Step 6: Create scripts
```bash
mkdir -p ~/.local/bin
```
Ensure `~/.local/bin` is in your PATH. Add to `~/.zshrc` if not already present:
```bash
export PATH="$HOME/.local/bin:$PATH"
```
Create `~/.local/bin/vpn_forti_connect.sh`:
```bash
#!/bin/bash
# Connect to FortiClient VPN via openfortivpn (CLI, no GUI automation)
# Config: ~/.vpn/forti-config | Password: macOS Keychain
# Requires: openfortivpn (brew install openfortivpn), sudo

set -euo pipefail

CONFIG="$HOME/.vpn/forti-config"
PIDFILE="$HOME/.vpn/openfortivpn.pid"
LOGFILE="$HOME/.vpn/openfortivpn.log"

if [ ! -f "$CONFIG" ]; then
    echo "ERROR: No config found at $CONFIG"
    echo "Run /vpn-forti:vpn-forti setup to configure."
    exit 1
fi
source "$CONFIG"

VPN_HOST="${VPN_HOST:?Set VPN_HOST in $CONFIG}"
VPN_PORT="${VPN_PORT:-443}"
VPN_USERNAME="${VPN_USERNAME:?Set VPN_USERNAME in $CONFIG}"
KEYCHAIN_SERVICE="${KEYCHAIN_SERVICE:?Set KEYCHAIN_SERVICE in $CONFIG}"
TRUSTED_CERT="${TRUSTED_CERT:-}"

# Check if already connected via openfortivpn
if [ -f "$PIDFILE" ] && kill -0 "$(cat "$PIDFILE")" 2>/dev/null; then
    echo "VPN already connected (openfortivpn running, PID $(cat "$PIDFILE"))."
    exit 0
fi
if pgrep -q openfortivpn 2>/dev/null; then
    echo "VPN already connected (openfortivpn running)."
    exit 0
fi

# Check for ppp0 interface (in case FortiClient GUI connected)
if ifconfig ppp0 >/dev/null 2>&1; then
    echo "VPN already connected (ppp0 interface up)."
    exit 0
fi

# Quit FortiClient GUI if running (releases connect lock)
osascript -e 'tell application "FortiClient" to quit' 2>/dev/null || true
sleep 1

# Get password from Keychain
PASSWORD=$(security find-generic-password -s "$KEYCHAIN_SERVICE" -a "$VPN_USERNAME" -w 2>/dev/null)
if [ -z "$PASSWORD" ]; then
    echo "ERROR: No password in Keychain for service=$KEYCHAIN_SERVICE account=$VPN_USERNAME"
    exit 1
fi

echo "Connecting to $VPN_HOST:$VPN_PORT..."

# Write temporary config file (avoids password in process list via -p flag)
TMPCONF=$(mktemp)
chmod 600 "$TMPCONF"
cat > "$TMPCONF" <<EOF
host = $VPN_HOST
port = $VPN_PORT
username = $VPN_USERNAME
password = $PASSWORD
EOF
if [ -n "$TRUSTED_CERT" ]; then
    echo "trusted-cert = $TRUSTED_CERT" >> "$TMPCONF"
fi

# openfortivpn has no --background flag; background it manually
sudo nohup /opt/homebrew/bin/openfortivpn -c "$TMPCONF" \
    > "$LOGFILE" 2>&1 &

# Wait briefly then delete temp config (sudo has already read it)
sleep 2
rm -f "$TMPCONF"

# Wait for openfortivpn process to appear and capture PID
for i in {1..5}; do
    OFVPN_PID=$(pgrep -x openfortivpn 2>/dev/null || true)
    if [ -n "$OFVPN_PID" ]; then
        echo "$OFVPN_PID" > "$PIDFILE"
        break
    fi
    sleep 1
done

# Verify ppp0 interface comes up (tunnel established)
for i in {1..15}; do
    if ifconfig ppp0 >/dev/null 2>&1; then
        echo "VPN connected."
        exit 0
    fi
    # Check if openfortivpn is still running
    if ! pgrep -q openfortivpn 2>/dev/null; then
        rm -f "$TMPCONF"
        echo "VPN connection failed. Check log: $LOGFILE"
        echo "Last lines:"
        tail -5 "$LOGFILE" 2>/dev/null || true
        exit 1
    fi
    sleep 1
done

rm -f "$TMPCONF"
echo "VPN connection timed out. openfortivpn is running but tunnel not established."
echo "Check log: $LOGFILE"
tail -5 "$LOGFILE" 2>/dev/null || true
exit 1
```

Create `~/.local/bin/vpn_forti_disconnect.sh`:
```bash
#!/bin/bash
# Disconnect FortiClient VPN (supports both openfortivpn and FortiClient)

set -euo pipefail

PIDFILE="$HOME/.vpn/openfortivpn.pid"

# Check openfortivpn via PID file first
if [ -f "$PIDFILE" ]; then
    PID=$(cat "$PIDFILE")
    if kill -0 "$PID" 2>/dev/null; then
        sudo /usr/bin/killall openfortivpn 2>/dev/null || true
        sleep 2
        if ! pgrep -q openfortivpn 2>/dev/null; then
            rm -f "$PIDFILE"
            echo "VPN disconnected."
            exit 0
        fi
        # Force kill if graceful failed
        sudo /usr/bin/killall -9 openfortivpn 2>/dev/null || true
        sleep 1
        rm -f "$PIDFILE"
        echo "VPN disconnected (forced)."
        exit 0
    fi
    rm -f "$PIDFILE"
fi

# Fallback: check for any openfortivpn process
if pgrep -q openfortivpn 2>/dev/null; then
    sudo /usr/bin/killall openfortivpn
    sleep 2
    if ! pgrep -q openfortivpn 2>/dev/null; then
        echo "VPN disconnected."
        exit 0
    fi
    echo "VPN disconnect may not have completed. Check manually."
    exit 1
fi

# Check ppp0 interface (maybe FortiClient GUI is connected)
if ifconfig ppp0 >/dev/null 2>&1; then
    echo "VPN appears connected via FortiClient GUI. Disconnect from FortiClient app."
    exit 1
fi

echo "VPN already disconnected."
exit 0
```

Make both executable:
```bash
chmod +x ~/.local/bin/vpn_forti_connect.sh ~/.local/bin/vpn_forti_disconnect.sh
```

### Step 7: Verify
```bash
~/.local/bin/vpn_forti_connect.sh   # Should connect
~/.local/bin/vpn_forti_disconnect.sh # Should disconnect
~/.local/bin/vpn_forti_connect.sh   # Should reconnect
```

## Connect
Run `~/.local/bin/vpn_forti_connect.sh`. Report result concisely.

## Disconnect
Run `~/.local/bin/vpn_forti_disconnect.sh`. Report result concisely.

## Status
Check in order:
1. `pgrep openfortivpn` - if running, VPN is connected via openfortivpn
2. `ifconfig ppp0` - check if tunnel interface is up
Report Connected/Disconnected.

## How it works
- openfortivpn runs as a backgrounded process (`nohup ... &`) with PID stored in `~/.vpn/openfortivpn.pid`
- Credentials passed via temporary config file (deleted after process starts) to avoid password in process list
- Password retrieved from macOS Keychain at connect time
- Tunnel verified by checking for `ppp0` interface (PPP tunnel)
- FortiClient GUI is quit before connecting (releases connect lock)
- Disconnect sends SIGTERM via killall, falls back to SIGKILL
- Connection log stored at `~/.vpn/openfortivpn.log`

## Troubleshooting
- **"ERROR: Server certificate pinning failed!"** - The trusted certificate hash is wrong or missing. Run Step 3 to get the correct hash, then update `TRUSTED_CERT` in `~/.vpn/forti-config`.
- **"ERROR: Cannot open /dev/ppp"** - pppd is not available or the system extension is blocking it. Try restarting the machine.
- **"sudo: a terminal is required"** - passwordless sudo not configured. Run Step 5.
- **Connection times out** - Check `~/.vpn/openfortivpn.log` for details. Common causes: wrong host/port, firewall blocking, or FortiClient GUI still running.
- **Scripts missing** - recreate from the templates above (Step 6).
- **FortiClient GUI conflicts** - The connect script quits FortiClient GUI automatically. If it keeps restarting, disable the launch agent: `launchctl unload /Library/LaunchAgents/com.fortinet.fct_launcher.plist`.
- **Why not FortiClient CLI?** - The native CLI tools (`cscmd`, `fct_tunnel_ctl`) at `/Library/Application Support/Fortinet/FortiClient/bin/` are undocumented and unreliable for scripting. openfortivpn is the open-source alternative that works reliably.
