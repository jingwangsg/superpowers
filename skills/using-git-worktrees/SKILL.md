---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - creates isolated git worktrees with smart directory selection and safety verification
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching.

**Core principle:** Systematic directory selection + safety verification = reliable isolation.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Directory Selection Process

Follow this priority order:

### 1. Check Existing Directories

```bash
# Resolve search root: main repo when inside a worktree, cwd otherwise
SEARCH_ROOT="${MAIN_REPO:-.}"

# Check in priority order
ls -d "$SEARCH_ROOT/.worktrees" 2>/dev/null     # Preferred (hidden)
ls -d "$SEARCH_ROOT/worktrees" 2>/dev/null       # Alternative
```

**If found:** Use that directory. If both exist, `.worktrees` wins.

### 2. Check CLAUDE.md

```bash
grep -i "worktree.*director" "$SEARCH_ROOT/CLAUDE.md" 2>/dev/null
```

**If preference specified:** Use it without asking.

### 3. Ask User

If no directory exists and no CLAUDE.md preference:

```
No worktree directory found. Where should I create worktrees?

1. .worktrees/ (project-local, hidden)
2. ~/.config/superpowers/worktrees/<project-name>/ (global location)

Which would you prefer?
```

## Safety Verification

### For Project-Local Directories (.worktrees or worktrees)

**MUST verify directory is ignored before creating worktree:**

```bash
# Check if directory is ignored (respects local, global, and system gitignore)
# Run from $MAIN_REPO so paths resolve correctly when inside a worktree
git -C "$MAIN_REPO" check-ignore -q .worktrees 2>/dev/null || git -C "$MAIN_REPO" check-ignore -q worktrees 2>/dev/null
```

**If NOT ignored:**

Per Jesse's rule "Fix broken things immediately":
1. Add appropriate line to .gitignore
2. Commit the change
3. Proceed with worktree creation

**Why critical:** Prevents accidentally committing worktree contents to repository.

### For Global Directory (~/.config/superpowers/worktrees)

No .gitignore verification needed - outside project entirely.

## Creation Steps

### 0. Worktree-aware context detection

**Before anything else**, detect whether you are already inside a worktree and
capture the current HEAD. This matters because directory selection (step 1–2)
may navigate to the main repo, losing the original HEAD.

```bash
# Capture current HEAD *before* any directory navigation
START_POINT=$(git rev-parse HEAD)

# Detect: am I inside a worktree (not the main checkout)?
GIT_DIR=$(git rev-parse --git-dir)
GIT_COMMON=$(git rev-parse --git-common-dir)
if [ "$GIT_DIR" != "$GIT_COMMON" ]; then
  IN_WORKTREE=true
  MAIN_REPO=$(git -C "$(git rev-parse --git-common-dir)" rev-parse --show-toplevel 2>/dev/null)
else
  IN_WORKTREE=false
  MAIN_REPO=$(git rev-parse --show-toplevel)
fi
```

**Why:** `git worktree add -b NAME` without a start-point defaults to HEAD of
whichever directory you run it from. If directory selection navigates to the
main repo, HEAD silently changes to the main branch. Capturing `START_POINT`
up front and passing it explicitly prevents this.

**When `IN_WORKTREE=true`:** directory selection (`.worktrees/`, CLAUDE.md)
should be resolved relative to `$MAIN_REPO`, not the current worktree root —
worktree directories live in the main checkout, not inside sibling worktrees.

### 1. Detect Project Name

```bash
project=$(basename "$MAIN_REPO")
```

### 2. Create Worktree

```bash
# Determine full path (resolve relative to $MAIN_REPO when in a worktree)
case $LOCATION in
  .worktrees|worktrees)
    path="$MAIN_REPO/$LOCATION/$BRANCH_NAME"
    ;;
  ~/.config/superpowers/worktrees/*)
    path="~/.config/superpowers/worktrees/$project/$BRANCH_NAME"
    ;;
esac

# Create worktree — always pass explicit start-point
git worktree add "$path" -b "$BRANCH_NAME" "$START_POINT"
cd "$path"
```

### 3. Run Project Setup

Auto-detect and run appropriate setup:

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

### 4. Verify Clean Baseline

Run tests to ensure worktree starts clean:

```bash
# Examples - use project-appropriate command
npm test
cargo test
pytest
go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### 5. Report Location

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| Already inside a worktree | Capture HEAD first, resolve dirs relative to main repo |
| `.worktrees/` exists | Use it (verify ignored) |
| `worktrees/` exists | Use it (verify ignored) |
| Both exist | Use `.worktrees/` |
| Neither exists | Check CLAUDE.md → Ask user |
| Directory not ignored | Add to .gitignore + commit |
| Tests fail during baseline | Report failures + ask |
| No package.json/Cargo.toml | Skip dependency install |

## Common Mistakes

### Skipping ignore verification

- **Problem:** Worktree contents get tracked, pollute git status
- **Fix:** Always use `git check-ignore` before creating project-local worktree

### Assuming directory location

- **Problem:** Creates inconsistency, violates project conventions
- **Fix:** Follow priority: existing > CLAUDE.md > ask

### Proceeding with failing tests

- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

### Ignoring worktree-in-worktree context

- **Problem:** When already inside a worktree, directory selection navigates to the main repo. `git worktree add -b NAME` without a start-point defaults to the main repo's HEAD, silently branching from the wrong commit.
- **Fix:** Always capture `START_POINT=$(git rev-parse HEAD)` before any navigation, and pass it explicitly: `git worktree add "$path" -b "$BRANCH_NAME" "$START_POINT"`

### Hardcoding setup commands

- **Problem:** Breaks on projects using different tools
- **Fix:** Auto-detect from project files (package.json, etc.)

## Example Workflow

```
You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Check .worktrees/ - exists]
[Verify ignored - git check-ignore confirms .worktrees/ is ignored]
[Create worktree: git worktree add .worktrees/auth -b feature/auth]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/jesse/myproject/.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

## Red Flags

**Never:**
- Create worktree without verifying it's ignored (project-local)
- Skip baseline test verification
- Proceed with failing tests without asking
- Assume directory location when ambiguous
- Skip CLAUDE.md check

**Always:**
- Follow directory priority: existing > CLAUDE.md > ask
- Verify directory is ignored for project-local
- Auto-detect and run project setup
- Verify clean test baseline

## Integration

**Called by:**
- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
- Any skill needing isolated workspace

**Pairs with:**
- **finishing-a-development-branch** - REQUIRED for cleanup after work complete
