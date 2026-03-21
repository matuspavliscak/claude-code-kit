# Claude Code Kit

A collection of reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills. Use at your own risk.

## `.claude/skills/` — Slash Commands

| Skill | Trigger | What it does |
|-------|---------|--------------|
| **[VPN](.claude/skills/vpn/)** | `/vpn` | Connect/disconnect Cisco Secure Client (AnyConnect) VPN via `openconnect` — certificate auth, macOS Keychain, no GUI needed |

## Installation

### Option A: Plugin install (recommended, requires [Claude Code](https://docs.anthropic.com/en/docs/claude-code))

```bash
# Add this repo as a marketplace
/plugin marketplace add matuspavliscak/claude-code-kit

# Install the VPN skill
/plugin install vpn@matuspavliscak-claude-code-kit
```

### Option B: Copy what you need

```bash
git clone https://github.com/matuspavliscak/claude-code-kit.git
cd claude-code-kit

# Copy a single skill
cp -r .claude/skills/vpn ~/.claude/skills/vpn
```

### Option C: Just tell Claude

```
Clone https://github.com/matuspavliscak/claude-code-kit and copy
.claude/skills/vpn to ~/.claude/skills/vpn
```

Then use `/vpn` in Claude Code.

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
