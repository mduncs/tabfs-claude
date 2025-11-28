# TabFS for Claude Code

> Fork of [TabFS](https://github.com/osnr/TabFS) by [Omar Rizwan](https://omar.website/tabfs/), adapted for Claude Code integration.

Browser tabs as a filesystem. `ls` your tabs. `rm` to close. `cat` to read content.

## What This Fork Adds

Claude Code integration via:
- **Skill** (`.claude/skills/tabfs-browser-access/`) - teaches Claude how to use TabFS
- **Command** (`.claude/commands/tabfs.md`) - quick reference for tab operations

Additional routes for development workflows:
- `document.html` - full page HTML including head
- `selection.txt` - currently selected text
- `meta.json` - page metadata as JSON
- `console.json` - captured console output
- `errors.json` - JavaScript errors

## Quick Start

```bash
# 1. Install macFUSE
brew install macfuse
# Allow in System Settings → Privacy & Security, reboot

# 2. Compile
cd TabFS/fs && make

# 3. Install browser extension
# Firefox: about:debugging → Load Temporary Add-on → TabFS/extension/manifest.json

# 4. Register native messaging host
cd TabFS && ./install.sh firefox

# 5. Use it
ls TabFS/fs/mnt/tabs/by-id/
```

## Documentation

- [PROJECT.md](PROJECT.md) - full documentation, architecture, usage examples
- [CONTRIBUTING.md](CONTRIBUTING.md) - development guide, adding routes

## License

GPLv3 - see [LICENSE](LICENSE)

Original TabFS by Omar Rizwan, modifications for Claude Code integration.
