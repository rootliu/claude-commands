# Claude Code Commands

Personal collection of [Claude Code](https://claude.com/claude-code) slash commands, shareable across machines.

## Commands

| Command | Description |
|---------|-------------|
| `/sync-zotero` | Sync Zotero library: download missing PDFs from arXiv, discover new papers by topic, import with user confirmation |
| `/commit` | Create well-formatted commits with conventional commit messages |

## Installation

Copy the command files to your Claude Code user commands directory:

### Windows
```powershell
# Clone this repo
git clone https://github.com/rootliu/claude-commands.git

# Copy commands to Claude Code config
copy claude-commands\commands\*.md %USERPROFILE%\.claude\commands\
```

### macOS / Linux
```bash
# Clone this repo
git clone https://github.com/rootliu/claude-commands.git

# Copy commands to Claude Code config
cp claude-commands/commands/*.md ~/.claude/commands/
```

Or create a symlink to auto-sync:
```bash
# Remove existing commands dir and symlink
rm -rf ~/.claude/commands
ln -s /path/to/claude-commands/commands ~/.claude/commands
```

## Usage

In Claude Code, use the slash command:

```
/sync-zotero              # Sync missing PDFs
/sync-zotero discover     # Search for new papers and ask one-by-one
/sync-zotero full         # Sync + discover

/commit                   # Create conventional commit
/commit --style=full      # Detailed commit message
```

## Requirements

### sync-zotero
- [Zotero](https://www.zotero.org/) installed with a local library
- Python 3.x with sqlite3 module
- `curl` (or `curl.exe` on Windows) for arXiv downloads
- Papers must have arXiv IDs in their metadata

### commit
- Git repository
- Standard git CLI
