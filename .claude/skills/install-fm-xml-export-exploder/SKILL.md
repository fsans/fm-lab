---
name: install-fm-xml-export-exploder
version: 0.8.5
description: "Clone and install the fm-xml-export-exploder repository for splitting FileMaker XML exports into individual components. Automatically checks for newer commits and prompts before updating. Triggers (English): \"install fm-xml-export-exploder\", \"clone the XML exploder\". Triggers (German): \"installiere fm-xml-export-exploder\", \"XML-Exploder klonen\"."
---

# fm-xml-export-exploder Repository Installation Skill

## When to Use This Skill

Use this skill when you need to:
- Perform initial setup of the fm-xml-export-exploder tool
- Update to the latest version of the fm-xml-export-exploder repository
- Reinstall after accidental deletion

The skill automates:
- Cloning the fm-xml-export-exploder repository from GitHub
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
4. **Clone/Update** - Clone the repository or pull latest changes to `docs/fm-xml-export-exploder/`
5. **Version Marker** - Store version information in `docs/fm-xml-export-exploder/.version`
6. **Report** - Provide clear success or error message

## Available Tools

This skill uses a bundled script that handles all operations:
- **Installation Script**: `scripts/install_fm_xml_export_exploder.sh`
  - Clones and updates the fm-xml-export-exploder repository
  - Usage: Execute with optional `--force` flag

## Working Process

### Step 1: Accept User Request
When the user asks to install or update fm-xml-export-exploder, determine if force installation is needed.

### Step 2: Execute Installation Script
Run the automation script:
```bash
bash .claude/skills/install-fm-xml-export-exploder/scripts/install_fm_xml_export_exploder.sh
```

Or with force flag:
```bash
bash .claude/skills/install-fm-xml-export-exploder/scripts/install_fm_xml_export_exploder.sh --force
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
- `SUCCESS: fm-xml-export-exploder repository installed successfully`
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
- Or manually resolve conflicts in `docs/fm-xml-export-exploder/`

## Output Format

Provide concise feedback:

**Success (Fresh Installation):**
```
No existing installation found. Cloning fm-xml-export-exploder repository...
Cloning from https://github.com/bc-m/fm-xml-export-exploder.git...
...

SUCCESS: fm-xml-export-exploder repository installed successfully
Version: abc1234 (2026-03-15 10:30:00 +0100)
Location: docs/fm-xml-export-exploder
Files: (8 Rust files, 1 TOML files)
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

SUCCESS: fm-xml-export-exploder repository installed successfully
```

**Failure:**
```
ERROR: [specific error message]
[suggestion for resolution]
```

## Notes

- The repository is cloned as a full git repo (not shallow) for complete history
- Updates use `git pull --ff-only` for safe fast-forward merges
- The version marker file (`docs/fm-xml-export-exploder/.version`) stores commit hash and date
- The `--force` flag removes the entire directory and re-clones
- The script is safe to run multiple times
- fm-xml-export-exploder is a Ruby tool that splits FileMaker XML exports into individual XML files per object for version control
