---
name: install-claris-docs
version: 0.8.5
description: "Download and install Claris FileMaker Pro online help (help.claris.com) as a local mirror in `docs/claris-help/`. Supports 11 languages with English as the always-included reference language. Also copies the REST-API reference index DB (`fm_reference.duckdb`) into the docs directory for fast slug lookups. Maintains a manifest and per-language version markers and prompts before replacing existing language sets. Triggers (English): \"install Claris docs\", \"install Claris help in German\", \"update the Claris help mirror\". Triggers (German): \"installiere die Claris-Hilfe\", \"Claris-Online-Hilfe auf Deutsch installieren\", \"Claris-Doku aktualisieren\". Triggers (Spanish): \"instalar la ayuda de Claris\", \"actualizar la ayuda Claris\". Triggers (French): \"installer l'aide Claris\", \"mettre à jour l'aide Claris\". Triggers (Italian): \"installa l'aiuto Claris\", \"aggiorna l'aiuto Claris\". Triggers (Dutch): \"installeer de Claris-help\", \"Claris-help bijwerken\". Triggers (Portuguese): \"instalar a ajuda Claris\", \"atualizar a ajuda Claris\". Triggers (Swedish): \"installera Claris-hjälpen\", \"uppdatera Claris-hjälpen\". Triggers (Japanese): \"Clarisヘルプをインストール\", \"Clarisヘルプを更新\". Triggers (Korean): \"Claris 도움말 설치\", \"Claris 도움말 업데이트\". Triggers (Chinese): \"安装 Claris 帮助\", \"更新 Claris 帮助\"."
---

# Claris Online Help Installation Skill

## When to use this skill

Use this skill when:

- The Claris Online Help is needed locally as a reference (for `filemaker-function-reference`, `fm-summarize`, `fm-analyze`, REST API reference endpoints, etc.)
- A new language should be added to an existing installation
- An update to a newer version of the Claris help should be installed
- The docs need to be reinstalled after corruption or accidental deletion
- The reference index DB (`fm_reference.duckdb`) should be refreshed so that slug lookups (function / ScriptStep → HTML file) work locally via DuckDB queries

The skill automates:
- Crawling the Claris Online Help (`https://help.claris.com/<lang>/pro-help/content/index.html`)
- Multilingual downloads (10 available languages plus English as reference)
- Mirroring including CSS, JS, images for offline-capable rendering
- Version tracking via HTTP `Last-Modified` (per language)
- Manifest maintenance in `docs/claris-help/manifest.json`
- User confirmation when replacing existing language sets
- **Copying the reference index DB** from `rest-api/db/fm_reference.duckdb` to `docs/claris-help/fm_reference.duckdb` (default step, can be disabled via `--skip-reference-db`)

## Important: language selection

**English (`en`) is ALWAYS downloaded** — regardless of user input. This ensures a consistent reference language is available (slugs, canonical names, fallback when translations are missing).

Three selection modes:
- **a) English only** — minimal set for CI / non-localized development
- **b) English + one language** — default for localized development
- **c) All languages** — full mirror (~10× the data volume)

### Available languages

| Code | Language               | Available | Note                                     |
|------|------------------------|-----------|------------------------------------------|
| `en` | English                | always    | Reference, always included                |
| `de` | German                 | ✓         |                                          |
| `es` | Spanish                | ✓         |                                          |
| `fr` | French                 | ✓         |                                          |
| `it` | Italian                | ✓         |                                          |
| `nl` | Dutch                  | ✓         |                                          |
| `pt` | Portuguese             | ✓         |                                          |
| `sv` | Swedish                | ✓         |                                          |
| `ja` | Japanese               | ✓         |                                          |
| `ko` | Korean                 | ✓         |                                          |
| `zh` | Chinese (Simplified)   | ✓         | URL segment is `zh` (not `zh-Hans`)       |

## Workflow

When the skill is invoked:

1. **Determine language preference** — Ask the user (via `AskUserQuestion`) for the desired language unless explicitly specified. Available options:
   - "English only (en)"
   - "English + my primary language" → ask which language (de recommended)
   - "All languages"

2. **Check existing docs** — Read `docs/claris-help/manifest.json` (if present) to determine the current state.

3. **Version check** — Per language, compare the stored date with the `Last-Modified` of `content/index.html`. On update: ask the user (unless `--force`).

4. **Crawl & download** — Start the crawler per language; recursively fetch all `.html` files plus referenced assets (CSS, JS, images).

5. **Update manifest** — Per language: count of downloaded files, timestamp, source URL.

6. **Reporting** — Summary with number of languages, total files, size.

## Skill invocation by the assistant

The skill uses **AskUserQuestion** for interactive language selection. If the user names the language explicitly (e.g. "install claris docs in German"), the question can be skipped.

### Step 1: Clarify language preference

Use `AskUserQuestion` with the following structure (unless already specified):

```
Question: "Which Claris Online Help languages should be installed? (English is always included)"
Header:   "Language selection"
Options:
  - "English + German"          (Recommended)
  - "English only"
  - "All languages (10 + EN)"
```

### Step 2: Run the script

```bash
# Nur Englisch
bash .claude/skills/install-claris-docs/scripts/install_claris_docs.sh

# Englisch + Deutsch
bash .claude/skills/install-claris-docs/scripts/install_claris_docs.sh --lang=de

# Alle Sprachen
bash .claude/skills/install-claris-docs/scripts/install_claris_docs.sh --all

# Force (Versions-Check überspringen)
bash .claude/skills/install-claris-docs/scripts/install_claris_docs.sh --lang=de --force

# List available languages without installing
bash .claude/skills/install-claris-docs/scripts/install_claris_docs.sh --list-languages
```

### Step 3: Communicate the result

The script produces structured logging. Report:
- Which languages were installed / updated / skipped
- Number of downloaded pages and size
- Location: `docs/claris-help/<lang>/`

## Script parameters

| Flag                  | Effect                                                                        |
|-----------------------|-------------------------------------------------------------------------------|
| `--lang=<code>`       | An additional language to English (e.g. `--lang=de`)                          |
| `--lang=all`          | All 10 available languages plus English (synonym for `--all`)                 |
| `--all`               | All 10 available languages plus English                                       |
| `--force`             | Skip version check and prompt — replace existing language sets                |
| `--list-languages`    | Print list of available languages (with availability check via HTTP)          |
| `--max-workers=<n>`   | Number of parallel downloads per language (default: 8)                        |
| `--dry-run`           | Only run crawling/discovery, do not write any files                           |
| `--skip-reference-db` | Skip the reference DB copy (default: always copy)                             |
| `--restart-server`    | Force stop/restart of the API server when copying the ref DB (for edge cases) |

Without parameters, **only English** is installed (plus always the reference DB, if present).

## Directory structure

After installation:

```
docs/claris-help/
├── manifest.json                     # Global manifest
├── fm_reference.duckdb               # Reference index DB (copy from rest-api/db/)
├── en/                                # English (reference, always included)
│   ├── .version                       # JSON: Last-Modified + file counts
│   ├── content/                       # All HTML pages
│   │   ├── index.html
│   │   ├── functions-reference.html
│   │   ├── set-variable.html
│   │   └── ... (~1000 files)
│   ├── Resources/                     # Scripts, templates from ../Resources/
│   ├── Skins/                         # CSS, themes from ../Skins/
│   └── assets/                        # Global assets from /assets/
├── de/                                # German (analogous)
├── es/
└── ...
```

## Reference index DB

In addition to the HTML mirror, the skill by default copies the reference index database from the REST API into the docs directory:

```
rest-api/db/fm_reference.duckdb  →  docs/claris-help/fm_reference.duckdb
```

**Purpose:** Quickly identify relevant HTML documents via DuckDB queries — e.g. "Which HTML file documents the function `PatternCount`?" Instead of full-text searching the mirror, a slug lookup against the index DB is enough. Used by `filemaker-function-reference`, `fm-summarize`, `fm-analyze` and other skills whenever they need to resolve a function or ScriptStep to an HTML file.

### Copy strategy

The REST API server attaches the reference DB in **READ\_ONLY mode** (`rest-api/src/config/database.js`), so DuckDB does not create a WAL file and does not hold a write lock. A direct `cp` operation while the server is running is therefore safe — the server keeps reading from the existing file, and the target directory (`docs/claris-help/`) is independent.

**Script flow:**

1. **Check:** Does `rest-api/db/fm_reference.duckdb` exist?
   - No → step is skipped with a warning (not an error).
2. **Direct copy** (default): atomic via `*.tmp` + `mv`, without touching the server.
3. **Fallback on error:** If the direct copy fails and a server is running on port 3003, automatically execute `tools/stop-servers.sh` → copy → `tools/start-servers.sh`.
4. **`--restart-server` flag:** Force the stop/start cycle even when the direct copy would work (for edge cases or when an explicit server reload is desired).
5. **`--skip-reference-db` flag:** Skip the step entirely (e.g. when only the HTML docs should be mirrored).

### Source of the reference DB

The reference DB is **not** produced by this skill — it is part of the `rest-api/` setup and is typically distributed alongside the REST API server. If `rest-api/db/fm_reference.duckdb` is missing, that is not an error: the script continues with the HTML downloads and reports `Ref-DB: source not found — skipped` in the summary.

### `manifest.json` schema

```jsonc
{
  "$schema_version": 1,
  "source": "Claris FileMaker Pro Online Help",
  "source_url": "https://help.claris.com",
  "fetched_at": "2026-05-12T10:15:30Z",
  "fallback_language": "en",
  "languages": [
    {
      "code": "en",
      "url_lang_segment": "en",
      "url_root": "https://help.claris.com/en/pro-help/",
      "html_pages": 1019,
      "asset_files": 84,
      "total_size_bytes": 41527890,
      "last_modified": "Mon, 13 Jan 2026 10:15:30 GMT",
      "fetched_at": "2026-05-12T10:15:30Z",
      "incomplete": false
    },
    {
      "code": "de",
      ...
    }
  ]
}
```

## Prerequisites

- **Python 3** (for the crawler, usually pre-installed on macOS)
- **curl** (for version checks, pre-installed)
- **Internet connection** to `help.claris.com`
- Write permissions on `docs/claris-help/`

## Disk space & duration

| Set                    | Files   | Size (estimated)  | Download duration |
|------------------------|---------|-------------------|-------------------|
| English only           | ~1100   | ~50 MB            | 2-3 minutes       |
| English + 1 language   | ~2200   | ~100 MB           | 4-6 minutes       |
| All 11 languages       | ~12000  | ~550 MB           | 20-30 minutes     |

With an existing cache (same version), re-downloading is skipped.

## Error handling

### Network errors
- Curl/Python urllib reports detailed errors when `help.claris.com` is unreachable
- Each file is retried up to 3× (with backoff)
- On a persistent error for a file: the script continues and marks the language as `incomplete: true` in the manifest

### Disk space
- Before downloading, `df` is checked for at least twice the expected volume free
- On insufficient space: abort with a clear error message

### HTTP errors (404, 5xx)
- 404: individual missing slugs are logged but do not abort
- 5xx: retry; after 3 failed attempts the language is marked `incomplete: true`

### Corrupted/incomplete downloads
- Files are first downloaded to `*.tmp` and then renamed atomically
- On abort, only complete files remain

## Output format

**Successful installation:**
```
Installing Claris Online Help...
Languages: en (always), de
Target: docs/claris-help/

[en] Discovering pages from index.html...
[en] Found 1078 HTML pages, 84 assets
[en] Downloading (8 workers)... ████████████████████ 100% (1162/1162)
[en] Done: 47.3 MB, 12.4 s

[de] Discovering pages from index.html...
[de] Found 1071 HTML pages, 84 assets
[de] Downloading (8 workers)... ████████████████████ 100% (1155/1155)
[de] Done: 49.1 MB, 13.1 s

SUCCESS: Claris documentation installed
  Languages: en, de
  Total: 2317 files, 96.4 MB
  Location: docs/claris-help/
  Manifest: docs/claris-help/manifest.json
```

**Already up to date:**
```
Checking for updates...
[en] Up to date (last-modified: Mon, 13 Jan 2026 10:15:30 GMT)
[de] Up to date (last-modified: Mon, 13 Jan 2026 10:15:30 GMT)

No action needed.
```

**With update prompt:**
```
Checking for updates...
[en] Newer version available.
     Current: Sun, 12 Jan 2026 16:47:42 GMT
     Remote:  Mon, 20 Jan 2026 10:15:30 GMT
Replace existing 'en' docs? (y/n): y
[en] Downloading...
```

**Error:**
```
ERROR: [specific error message]
[suggestion for resolution]
```

## Notes

- The downloaded files come from a publicly accessible source (Claris Online Help). Local use is typically covered by Claris's documentation license; public re-publishing is NOT permitted.
- This documentation is used by the `fm_reference.duckdb` setup and the `/api/reference/...` endpoints as the HTML source for full-text extraction.
- The script is idempotent — running it multiple times is safe.
- If only individual slugs are missing or outdated, a full reinstall is not necessary — the crawler uses Last-Modified headers per file (HEAD request) to update only changed files.
- After a successful install/update the script registers this source in `.fmlab/docs.json` via `tools/register_docs.py`, so the web home dashboard's Docs card can list it.
