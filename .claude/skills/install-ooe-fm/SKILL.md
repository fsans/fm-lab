---
name: install-ooe-fm
version: 0.8.5
description: "Clone and install the ooe-fm (One Of Everything) FileMaker reference repository for XML test cases. Automatically checks for newer commits and prompts before updating. Triggers (English): \"install ooe-fm\", \"clone the ooe-fm test data\". Triggers (German): \"installiere ooe-fm\", \"ooe-fm Testdaten klonen\"."
---

# ooe-fm Repository Installation Skill

## When to Use This Skill

Use this skill when you need to:
- Perform initial setup of the ooe-fm XML test reference data
- Update to the latest version of the ooe-fm repository
- Reinstall after accidental deletion

The skill automates:
- Cloning the ooe-fm repository from GitHub
- Version checking via git commit comparison
- Updating via git pull
- User confirmation when updating existing installation

## Parameters

The skill accepts **optional parameters**:
- `--force` - Remove existing clone and re-clone from scratch

Without parameters, the skill will:
- Check for existing installation
- Compare local and remote commits, prompt if update is available
- Clone directly if no existing installation found

## Workflow

When invoked, the skill performs these steps:

1. **Check Existing Installation** - Verify if repository already cloned
2. **Version Check** - Compare local commit with remote HEAD
3. **User Prompt** - Ask for confirmation if newer commits are available
4. **Clone/Update** - Clone the repository or pull latest changes to `docs/ooe-fm/`
5. **Version Marker** - Store version information in `docs/ooe-fm/.version`
6. **Report** - Provide clear success or error message

## Available Tools

This skill uses a bundled script that handles all operations:
- **Installation Script**: `scripts/install_ooe_fm.sh`
  - Clones and updates the ooe-fm repository
  - Usage: Execute with optional `--force` flag

## Working Process

### Step 1: Accept User Request
When the user asks to install or update ooe-fm test data, determine if force installation is needed.

### Step 2: Execute Installation Script
Run the automation script:
```bash
bash .claude/skills/install-ooe-fm/scripts/install_ooe_fm.sh
```

Or with force flag:
```bash
bash .claude/skills/install-ooe-fm/scripts/install_ooe_fm.sh --force
```

### Step 3: Handle User Prompts
If the script finds newer commits, it will prompt:
```
Newer version available.
Current: abc1234 (2026-03-15 10:30:00 +0100)
Remote:  def5678
Update repository? (y/n):
```

Inform the user and let them decide.

### Step 4: Report Results
The script will output one of:
- `SUCCESS: ooe-fm repository installed successfully`
- `Repository is up to date (commit: abc1234)`
- `Update cancelled by user`
- `ERROR: Clone failed`

Report the result to the user with appropriate context.

## Error Handling

### Network Failures
If git clone/pull fails:
- Check internet connection
- Verify GitHub is accessible
- Try again later if server is temporarily unavailable

### Clone Conflicts
If pull fails due to local changes:
- Use `--force` to re-clone from scratch
- Or manually resolve conflicts in `docs/ooe-fm/`

## Output Format

Provide concise feedback:

**Success (Fresh Installation):**
```
No existing installation found. Cloning ooe-fm repository...
Cloning from https://github.com/mislavkos/ooe-fm.git...
...

SUCCESS: ooe-fm repository installed successfully
Version: abc1234 (2026-03-15 10:30:00 +0100)
Location: docs/ooe-fm
Files: (5 XML files, 3 fmp12 files)
```

**Success (Already Up to Date):**
```
Checking for updates...
Repository is up to date (commit: abc1234, date: 2026-03-15 10:30:00 +0100)
No action needed.
```

**Success (Update):**
```
Checking for updates...
Newer version available.
Current: abc1234 (2026-03-15 10:30:00 +0100)
Remote:  def5678
Update repository? (y/n): y
Pulling latest changes...

SUCCESS: ooe-fm repository installed successfully
```

**Failure:**
```
ERROR: [specific error message]
[suggestion for resolution]
```

## Notes

- The repository is cloned as a full git repo (not shallow) for complete history
- Updates use `git pull --ff-only` for safe fast-forward merges
- The version marker file (`docs/ooe-fm/.version`) stores commit hash and date
- The `--force` flag removes the entire directory and re-clones
- The script is safe to run multiple times
- ooe-fm is a FileMaker "One Of Everything" reference database with XML exports for testing
