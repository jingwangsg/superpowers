# Installing this fork via symlink

This fork can replace the official superpowers plugin in Claude Code (and Codex)
by symlinking the clone into the plugin cache directory. Edits to the repo are
picked up immediately — no copy/sync step needed.

## Prerequisites

- Claude Code with the official `superpowers` plugin previously installed
  (so the cache directory structure already exists)
- Git

## Clone

```bash
git clone https://github.com/jingwangsg/superpowers ~/WORKSPACE/superpowers
```

## Symlink into Claude Code plugin cache

```bash
# Find the cached version directory
CACHE_DIR="$HOME/.claude/plugins/cache/claude-plugins-official/superpowers"
CACHED_VERSION=$(ls "$CACHE_DIR" 2>/dev/null | head -1)

if [ -z "$CACHED_VERSION" ]; then
  echo "No cached superpowers version found. Install the official plugin first,"
  echo "then re-run this script."
  exit 1
fi

# Replace cache with symlink
rm -rf "$CACHE_DIR/$CACHED_VERSION"
ln -s ~/WORKSPACE/superpowers "$CACHE_DIR/$CACHED_VERSION"

# Verify
ls -la "$CACHE_DIR/$CACHED_VERSION"
# Should show: ... -> /Users/<you>/WORKSPACE/superpowers
```

## Symlink into Codex

Codex discovers superpowers skills via `~/.agents/skills/superpowers`.
The official install (`.codex/INSTALL.md`) clones a separate copy to
`~/.codex/superpowers` and symlinks from there. We replace that symlink
to point at our fork instead:

```bash
# Remove existing symlink (points to ~/.codex/superpowers/skills)
rm -f ~/.agents/skills/superpowers

# Point at our fork's skills directory
mkdir -p ~/.agents/skills
ln -s ~/WORKSPACE/superpowers/skills ~/.agents/skills/superpowers

# Verify
ls -la ~/.agents/skills/superpowers
# Should show: ... -> /Users/<you>/WORKSPACE/superpowers/skills
```

> **Note:** The old clone at `~/.codex/superpowers` is now unused. You can
> leave it or remove it (`rm -rf ~/.codex/superpowers`).

## Enable the plugin

Make sure `~/.claude/settings.json` has:

```json
{
  "enabledPlugins": {
    "superpowers@claude-plugins-official": true
  }
}
```

## Verify

Start a new Claude Code session and check that superpowers skills are listed:

```
/skills
```

You should see skills like `superpowers:using-git-worktrees`,
`superpowers:brainstorming`, etc.

## Uninstalling

Remove the symlinks and let each tool re-download the official version:

```bash
# Claude Code
CACHE_DIR="$HOME/.claude/plugins/cache/claude-plugins-official/superpowers"
CACHED_VERSION=$(ls "$CACHE_DIR" 2>/dev/null | head -1)
rm "$CACHE_DIR/$CACHED_VERSION"   # removes symlink only, not your repo
# Next Claude Code session will re-download the official plugin

# Codex
rm ~/.agents/skills/superpowers
# Optionally restore the official install:
#   git clone https://github.com/jingwangsg/superpowers ~/.codex/superpowers
#   ln -s ~/.codex/superpowers/skills ~/.agents/skills/superpowers
```
