# Typeless Reset Skill (Claude Code)

A [Claude Code](https://claude.com/claude-code) skill for managing [Typeless.app](https://typeless.com) trial resets, account logout, data backup, and cross-account migration on macOS.

## What it does

| Mode | Trigger | Action |
|------|---------|--------|
| **Reset** | "重置 typeless" / "reset typeless" | Clear device fingerprint + logout + reset trial |
| **Backup** | "备份 typeless" / "backup typeless" | Backup database, recordings, settings, dictionary |
| **Migrate** | "迁移 typeless" / "migrate typeless" | Backup → reset → login new account → migrate data |

## Install

Copy `SKILL.md` to your Claude Code skills directory:

```bash
# Project-level (recommended)
mkdir -p .claude/skills/reset-typeless
cp SKILL.md .claude/skills/reset-typeless/

# Or global-level
mkdir -p ~/.claude/skills/reset-typeless
cp SKILL.md ~/.claude/skills/reset-typeless/
```

Then use it in Claude Code by saying `/reset-typeless` or any trigger phrase above.

## How it works

Typeless tracks its 30-day trial server-side using a self-generated device UUID stored at:

```
~/Library/Application Support/now.typeless.desktop/device.cache
```

This UUID is **not hardware-bound** — deleting it causes the app to generate a new one, effectively appearing as a new device to the server.

The skill also handles:
- Encrypted `user-data.json` (double PBKDF2 + AES-256-CBC via electron-store)
- API signing protocol (HMAC-SHA1 + SM3 + CryptoJS AES) for cloud dictionary backup
- SQLite `user_id` replacement for cross-account history migration

## Requirements

- macOS only
- For dictionary backup/import: `pip install pycryptodome gmssl requests`

## Credits

Crypto and API signing details from [mercy719/typeless-migrator](https://github.com/mercy719/typeless-migrator).

## License

MIT
