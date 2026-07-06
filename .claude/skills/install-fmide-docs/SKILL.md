---
name: install-fmide-docs
version: 0.8.5
description: "Download and install fmIDE documentation from the GitHub Wiki. Automatically checks for newer commits and prompts before replacing existing docs. Triggers (English): \"install fmIDE docs\", \"update fmIDE documentation\". Triggers (German): \"installiere die fmIDE-Doku\", \"fmIDE-Dokumentation aktualisieren\"."
---

# fmIDE Documentation Installation Skill

## When to Use This Skill

Use this skill when you need to:
- Perform initial setup of fmIDE documentation
- Update to the latest fmIDE documentation
- Reinstall documentation after corruption or accidental deletion

The skill automates:
- Cloning the fmIDE Wiki from GitHub
- Version checking via git commit hash to avoid unnecessary downloads
- Installation of Markdown pages to the correct location
- User confirmation when replacing existing documentation
- Cleanup of temporary files

## Parameters

The skill accepts **optional parameters**:
- `--force` - Skip version check and user prompts, force reinstallation

Without parameters, the skill will:
- Check for existing documentation
- Compare versions and prompt if update is available
- Install directly if no existing documentation found

## Workflow

When invoked, the skill performs these steps:

1. **Check Existing Docs** - Verify if documentation already exists
2. **Version Check** - Compare local commit hash with remote HEAD
3. **User Prompt** - Ask for confirmation if newer version is available
4. **Clone** - Clone the fmIDE Wiki repository to a temporary directory
5. **Install** - Copy Markdown files to `docs/fmIDE/` directory
6. **Version Marker** - Store commit hash and date for future comparisons
7. **Cleanup** - Remove all temporary files automatically
8. **Report** - Provide clear success or error message

## Available Tools

This skill uses a bundled script that handles all operations:
- **Installation Script**: `scripts/install_fmide_docs.sh`
  - Clones and installs fmIDE Wiki documentation
  - Usage: Execute with optional `--force` flag

## Working Process

### Step 1: Accept User Request
When the user asks to install or update fmIDE documentation, determine if force installation is needed.

### Step 2: Execute Installation Script
Run the automation script:
```bash
bash .claude/skills/install-fmide-docs/scripts/install_fmide_docs.sh
```

Or with force flag:
```bash
bash .claude/skills/install-fmide-docs/scripts/install_fmide_docs.sh --force
```

### Step 3: Handle User Prompts
If the script finds a newer version, it will prompt:
```
Newer version available.
Current: 862b9104 (2025-06-15 14:30:00 +0200)
Remote:  a1b2c3d4
Replace existing docs? (y/n):
```

Inform the user and let them decide.

### Step 4: Report Results
The script will output one of:
- `SUCCESS: fmIDE documentation installed successfully`
- `Docs are up to date (commit: [hash], date: [timestamp])`
- `Installation cancelled by user`
- `ERROR: Clone failed`
- `ERROR: No Markdown files found in wiki`
- `ERROR: Copy operation failed`

Report the result to the user with appropriate context.

## Error Handling

### Network Failures
If git clone fails:
- Check internet connection
- Verify the GitHub Wiki is accessible: https://github.com/fmIDE/fmIDE/wiki
- The wiki must be enabled on the repository
- Try again later if GitHub is temporarily unavailable

### No Wiki Pages
If the cloned wiki contains no Markdown files:
- The wiki may be empty or not yet initialized
- Check the wiki directly in a browser

### Copy Operation Failed
If copying files to `docs/fmIDE/` fails:
- Check file permissions on `docs/` directory
- Verify available disk space
- Ensure no other process is using the documentation files

### Disk Space
The installation requires minimal disk space:
- Wiki clone is typically < 10MB
- Final documentation is Markdown-only

## Output Format

Provide concise feedback:

**Success (Fresh Installation):**
```
No existing docs found. Installing fmIDE documentation...
Cloning fmIDE wiki from https://github.com/fmIDE/fmIDE.wiki.git...
Clone complete (25 Markdown pages)
Installing to docs/fmIDE/...

SUCCESS: fmIDE documentation installed successfully
Version: 862b9104 (2025-06-15 14:30:00 +0200)
Location: docs/fmIDE/
Files: (25 Markdown pages)
```

**Success (Already Up to Date):**
```
Checking for updates...
Docs are up to date (commit: 862b9104, date: 2025-06-15 14:30:00 +0200)
No action needed.
```

**Success (Update):**
```
Checking for updates...
Newer version available.
Current: 862b9104 (2025-06-15 14:30:00 +0200)
Remote:  a1b2c3d4
Replace existing docs? (y/n): y
Cloning fmIDE wiki from https://github.com/fmIDE/fmIDE.wiki.git...
...
SUCCESS: fmIDE documentation updated successfully
```

**Failure:**
```
ERROR: [specific error message]
[suggestion for resolution]
```

## Notes

- All temporary files are automatically cleaned up via trap mechanism
- Original documentation is only replaced after successful clone
- The version marker file (`.version`) stores git commit hash (line 1) and date (line 2)
- Multiple installations will overwrite the existing documentation
- The script is safe to run multiple times
- Source: https://github.com/fmIDE/fmIDE/wiki
- After a successful install/update the script registers this source in `.fmlab/docs.json` via `tools/register_docs.py`, so the web home dashboard's Docs card can list it.
