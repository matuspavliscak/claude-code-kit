# Claude Code Kit

A collection of reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills.

## Skills

| Skill | Description |
|-------|-------------|
| [vpn](skills/vpn/) | Connect/disconnect Cisco Secure Client VPN via CLI using openconnect |

## Installation

Copy the skills you want into your Claude Code skills directory:

```bash
# Copy a single skill
cp -r skills/vpn ~/.claude/skills/vpn
```

Or clone the whole repo and symlink:

```bash
git clone https://github.com/matuspavliscak/claude-code-kit.git
ln -s "$(pwd)/claude-code-kit/skills/vpn" ~/.claude/skills/vpn
```

Then use `/vpn` in Claude Code.

## License

MIT
