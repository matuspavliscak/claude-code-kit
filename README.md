# Claude Code Kit

A collection of reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills. Use at your own risk.

## `.claude/skills/` — Slash Commands

| Skill | Trigger | What it does |
|-------|---------|--------------|
| **[VPN - Cisco](.claude/skills/vpn-cisco/)** | `/vpn-cisco:vpn` | Connect/disconnect Cisco Secure Client (AnyConnect) VPN via `openconnect` — certificate auth, macOS Keychain, no GUI needed |
| **[VPN - FortiClient](.claude/skills/vpn-forti/)** | `/vpn-forti:vpn-forti` | Connect/disconnect FortiClient VPN via `openfortivpn` — username/password auth, macOS Keychain, no GUI needed |

## Installation

### Option A: Plugin install (recommended, requires [Claude Code](https://docs.anthropic.com/en/docs/claude-code))

```bash
# Add this repo as a marketplace
/plugin marketplace add matuspavliscak/claude-code-kit

# Install skills
/plugin install vpn@matuspavliscak-claude-code-kit          # Cisco VPN
/plugin install vpn-forti@matuspavliscak-claude-code-kit    # FortiClient VPN
```

### Option B: Copy what you need

```bash
git clone https://github.com/matuspavliscak/claude-code-kit.git
cd claude-code-kit

# Copy a single skill
cp -r .claude/skills/vpn-cisco ~/.claude/skills/vpn-cisco
# or
cp -r .claude/skills/vpn-forti ~/.claude/skills/vpn-forti
```

### Option C: Just tell Claude

```
Clone https://github.com/matuspavliscak/claude-code-kit and copy
.claude/skills/vpn-cisco to ~/.claude/skills/vpn-cisco
```

Then use `/vpn-cisco` or `/vpn-forti` in Claude Code.

## Adding Your Own Skills

Create a new directory in `.claude/skills/` with a `SKILL.md` file:

```
.claude/skills/my-skill/
├── SKILL.md          # Required — frontmatter + instructions
├── scripts/          # Optional — executable code
└── references/       # Optional — documentation
```

## License

MIT
