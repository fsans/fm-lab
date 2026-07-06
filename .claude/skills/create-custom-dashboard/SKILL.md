---
name: create-custom-dashboard
version: 0.8.5
description: "Interactively creates a new custom dashboard bundle for the fm-lab dashboard system. Asks the user about the desired dashboard content, drafts SQL queries, shows sample results, proposes a presentation format, asks for a name, and generates the full bundle directory under `rest-api/templates/dashboards-custom/<id>/`. Triggers (English): \"/create-custom-dashboard\", \"create a new dashboard for X\", \"new dashboard\", \"build a dashboard that shows X\". Triggers (German): \"erstelle ein neues Dashboard für X\", \"neues Dashboard\", \"baue ein Dashboard das X zeigt\"."
---

# Create a custom dashboard

Guide the user interactively through the creation of a new dashboard bundle. The result is a complete bundle directory under `rest-api/templates/dashboards-custom/<id>/` containing `manifest.json`, `layout.json`, and at least one SQL file under `data/`.

## System vs. Custom directory split

The dashboard system distinguishes between two bundle sources:

| Directory | Content | Who writes? |
|---|---|---|
| `rest-api/templates/dashboards/` | **System bundles** (`home`, `_generic`, `custom_queries`, `dashboards`) | core team only |
| `rest-api/templates/dashboards-custom/` | **Custom/plugin bundles** | this skill, user plugins |

The dashboard resolver searches both directories; in case of an ID collision the custom directory wins (override pattern for local extensions). This skill writes **exclusively** to `dashboards-custom/`.

## Ground rules

- **Language**:
  - **Dialogue** (steps 1–4): conduct in the user's input language — if the user writes in German, ask back in German; if in English, English; etc.
  - **SQL templates** (step 5.2): English only — filenames, parameter names, column aliases, comments. Language-agnostic, never translated.
  - **manifest.json / layout.json** (steps 5.3, 5.4): English defaults for `title`, `description`, card titles, KPI/column labels.
  - **Localization** (step 5.5): for every other supported language create a `locales/<lang>.json` file with translated `title`, `description` and (optionally) layout labels.
- **Database**: `db/fm_catalog.duckdb` (master — NOT `rest-api/db/`)
- **Read-only**: never UPDATE/INSERT/DELETE on the database
- **Interactive**: steps 3 and 4 wait for user confirmation before files are written
- **SQL style**: analogous to `rest-api/templates/dashboards/home/data/*.sql` (system reference) and `rest-api/templates/dashboards-custom/script_todos/data/*.sql` (custom reference)
- **DuckDB path**: if `duckdb` is not in PATH, check the known locations: `~/.duckdb/cli/latest/duckdb`, `/opt/homebrew/bin/duckdb`, `/usr/local/bin/duckdb`

---

## Workflow

### Step 1 — Clarify the dashboard goal

If the user already provided a topic when invoking the skill (e.g. `/create-custom-dashboard Variable analysis`), use it directly as the starting point: "For a variable dashboard I would show the following data: …" and proceed directly to step 2.

If no topic was provided, ask **one** short question:
> "What should the dashboard show? (e.g. variable overview, scripts without comment, lookup fields, relationship statistics, …)"

**BLOCKING** if no topic is available: wait for the answer.

---

### Step 2 — Draft SQL query, execute it and show the result

Based on the goal, draft a SQL query that returns the relevant data from `db/fm_catalog.duckdb`.

#### Query design rules

- Column names: short, lowercase, no spaces (`script_count`, `name`, `uuid`, `file`)
- For multi-row results, always include a `uuid` column if the object exists in the ObjectCatalog — this enables `openObject` navigation
- Parameterise via `getvariable('param_name')` for optional filters (e.g. `file`, `limit`)
- Parameter with default: `CAST(COALESCE(getvariable('limit'), '25') AS INTEGER)` or `(getvariable('file') IS NULL OR File_Name = getvariable('file'))`
- Prefer CTEs for intermediate results
- Orient yourself on `sql/sample_queries.sql` and the existing bundle SQL files

> **⚠️ Multi-file JOIN rule (critical).** The database holds **many FileMaker files** (one solution = dozens of files). Internal FileMaker IDs like `L_ID`, `Script_ID`, `BT_ID`, `Layout_ID`, `Table_ID` are **only unique within a single file**, not across the database. JOINing on a bare ID fans out across every file that happens to reuse the same ID and **silently inflates counts** (e.g. `LayoutObjects.Layout_ID = Layouts.L_ID` summed the objects of 50+ files into one row).
>
> Always join on either:
> - the **global UUID** (`L_UUID`, `Script_UUID`, `Object_UUID`, …) — preferred when available, **or**
> - the **ID *plus* `File_Name`** (`lo.Layout_ID = l.L_ID AND lo.File_Name = l.File_Name`) when no shared UUID column exists.
>
> The same rule applies inside aggregating subqueries: `GROUP BY Script_ID` across all files double-counts — `GROUP BY Script_UUID` instead. When validating a new query, sanity-check that the top counts look plausible; identical or suspiciously round top values are the classic fan-out symptom.

#### Execute the query (LIMIT 10 for preview)

```bash
~/.duckdb/cli/latest/duckdb db/fm_catalog.duckdb -c "<SQL with LIMIT 10>"
```

On query error: correct it once and re-run. After two failures, involve the user.

#### Prepare the result

| Situation | Output |
|-----------|---------|
| ≤ 10 rows, ≤ 8 columns | full result as a Markdown table |
| > 10 rows | "Query returns many results. First 10 as preview:" + table |
| > 8 columns | show the most important columns, mention the rest |
| 0 rows | inform the user, suggest an alternative query or different database content |

Finally provide a short assessment: "That's N rows with X columns — the result is well suited for [primitive]."

---

### Step 3 — Propose a presentation format (BLOCKING)

Based on the query result, recommend a presentation format:

#### Decision logic

| Condition | Recommendation |
|-----------|------------|
| 1 row, 1–6 numeric/aggregated columns | **KPIStrip** |
| 1 column `content` (free text) | **Markdown** |
| Multiple rows, contains `uuid` + `name`, primarily for navigation | **List** (clickable via `openObject`) |
| Multiple rows, 3–8 mixed columns, analysis focus | **Table** |
| Many rows (>50), uuid present | **Table** with `onRowClick` |
| Many objects for navigation (queries, dashboards) | **TileGrid** |

If List and Table both fit, suggest both:
> "**List**: compact, good for navigation (click → detail view). **Table**: shows all columns, better for data analysis."

Output:
1. Recommendation with a short reason
2. Optional alternative with description
3. Question: "Should I use [recommendation], or do you prefer a different presentation?"

**BLOCKING**: wait for the user's answer.

---

### Step 4 — Propose a dashboard name (BLOCKING)

Suggest an ID + an English title. Conduct the dialogue in the user's input language (per the Language ground rule), but the ID and the manifest title themselves stay language-agnostic / English.

**Rules for the ID** (= directory name, language-agnostic):
- Lowercase ASCII only (a–z, 0–9, `_`), max. 30 characters
- Avoid reserved system IDs: `home`, `_generic`, `custom_queries`, `dashboards` (those would override the system bundle in `dashboards/`)
- Good: `variable_hotspots`, `unused_scripts`, `lookup_fields`, `relation_overview`

**Rules for the title** (= default for `manifest.json`):
- **English**, max. 50 characters, human-readable
- Good: "Variable hotspots", "Scripts without callers", "Lookup fields"
- The title in the user's input language is **not** asked separately — it gets generated automatically in step 5.5 as part of the matching `locales/<lang>.json` file (alongside the other 9 languages).

Output (phrased in the user's input language; ID and English title verbatim):
> "Suggested ID: `<id>` / Title (EN, manifest default): "<title>""
> "If the user's input language ≠ English, additionally show the localized title for transparency:"
> "Title (<user-lang>): "<title in user language>" — will be written to `locales/<user-lang>.json`"
> "Does that work, or do you want a different name?"

**BLOCKING**: wait for the user's answer.

**If the user supplies a name in their own language**: derive both
1. the **English title** for `manifest.json` (translate),
2. the **ASCII-conform ID** (lowercase, spaces → `_`, ä→ae, ö→oe, ü→ue, ß→ss, é→e, ñ→n, etc., strip everything outside [a-z0-9_]).

The user-language title is preserved and reused in step 5.5 for the corresponding locale file.

---

### Step 5 — Generate the bundle

Only when content, presentation **and** name are confirmed, write the files.

#### 5.1 Check the directory

```bash
ls "rest-api/templates/dashboards-custom/"
ls "rest-api/templates/dashboards/"
```

List both directories so that ID collisions with system bundles are ruled out (system IDs are reserved, see step 4). If a bundle with the chosen ID already exists in `dashboards-custom/`: ask the user whether to overwrite or to rename the ID.

#### 5.2 Create the SQL file

Path: `rest-api/templates/dashboards-custom/<id>/data/<dataset_name>.sql`

```sql
-- @template_type: report
-- @description: <short description of the dataset>
-- @params: <param_name> (optional, default <value>), ...

<final SQL query — without the preview LIMIT, but with a parameterised LIMIT>
```

If several logically separate datasets make sense (e.g. an aggregate KPI plus a detail list), create separate SQL files.

#### 5.3 Create manifest.json

Path: `rest-api/templates/dashboards-custom/<id>/manifest.json`

> **Schema constraints — DO NOT VIOLATE.** The bundle loader validates the manifest with Joi (`rest-api/src/services/dashboard-schemas.js`). Violations land in `console.warn` and the bundle silently disappears from `/api/dashboards`. Memorise these rules:
>
> - `id` must match `^[a-zA-Z0-9_-]+$` AND be **identical** to the directory name. No dots, spaces, Unicode.
> - `title` is required and non-empty.
> - `params[].type` ∈ `{ "string", "number", "boolean" }` — **never** `"integer"`, `"int"`, `"float"`, `"bool"`. Integer-valued params use `"number"`.
> - `datasets[].source` must start with one of `bundle:`, `custom:`, `report:`, `builtin:` followed by at least one character.
> - Every `dataset` name referenced in `layout.json` (`data.dataset`) must be declared in `manifest.datasets[].id`. Loader doesn't catch this — shows up as an empty card at runtime.

```json
{
  "id": "<id>",
  "version": "1.0.0",
  "title": "<title>",
  "description": "<1-2 sentences describing what the dashboard shows>",
  "author": "fm-lab custom",
  "icon": "<Lucide icon name>",
  "tags": ["custom", "<thematic tag>"],
  "entry": "layout.json",
  "datasets": [
    { "id": "<dataset_name>", "source": "bundle:data/<dataset_name>.sql" }
  ],
  "params": [
    { "name": "file",  "type": "string",  "required": false,
      "description": "Optional file filter." },
    { "name": "limit", "type": "number",  "required": false,
      "description": "Row limit (integer-valued — but the JSON type stays \"number\")." }
  ],
  "permissions": { "read_only": true, "allow_navigation": true }
}
```

**Icon selection** (Lucide React icons — suitable examples):
`database`, `code`, `layout-list`, `git-fork`, `table-2`, `search`, `variable`, `function-square`, `tag`, `alert-triangle`, `link`, `box`, `layers`, `filter`
When in doubt: `database` as a safe default.

**Dataset sources**:
- `bundle:data/<name>.sql` — SQL in the same bundle (standard, **preferred** for self-contained custom bundles)
- `custom:<template-name>` — template resolver with fallback search in the following order:
  1. `rest-api/templates/sql-custom/` (standalone custom queries)
  2. `rest-api/templates/sql-custom-details/**/` (detail-view templates for UI hooks)
  3. `rest-api/templates/dashboards-custom/<bundle>/queries/` (bundle-owned drilldowns)
  4. `rest-api/templates/dashboards/<bundle>/queries/` (system-bundle drilldowns)
- `report:<template-name>` — existing template from `rest-api/templates/sql/`
- `builtin:list_dashboards` / `builtin:list_custom_queries` / `builtin:files` / `builtin:query_meta` — server builtins

#### 5.4 Create layout.json

Path: `rest-api/templates/dashboards-custom/<id>/layout.json`

Always `Grid(columns:12)` as root. Wrap every content unit in a `Card`.

**Base structure**:
```json
{
  "schemaVersion": 1,
  "root": {
    "type": "Grid",
    "props": { "columns": 12, "gap": 16 },
    "children": [
      {
        "type": "Card",
        "props": { "span": 12, "variant": "hero" },
        "data": { "dataset": "<overview_dataset>" },
        "children": [
          { "type": "KPIStrip", "props": { ... } }
        ]
      },
      {
        "type": "Card",
        "props": { "span": 12, "title": "<section title>" },
        "data": { "dataset": "<dataset_name>" },
        "children": [
          { "type": "<Primitive>", "props": { ... } }
        ]
      }
    ]
  }
}
```

> **Title convention — single source of truth.** The **entry title comes from
> `manifest.title`**: `DashboardHost` renders it as a framed title box (together
> with the manifest `description`) above the grid. Therefore:
>
> - **The first card carries NO `title`.** Leave it off (`{ "span": 12 }`, or
>   `+ "variant": "hero"`) so the KPI/overview sits directly under the title box.
>   Never write `"title": "<entry name>"` or a filler `"Overview"` there — that is
>   the duplicate/redundant second heading we want to avoid.
> - **Only follow-up cards get a `title`**, and only a *genuine, distinct section*
>   label (e.g. `"Findings"`, `"API families"`, `"URL details"`). If a card title
>   merely repeats the dashboard name or says "Overview", drop it.
> - `variant: "hero"` is **purely cosmetic** (a gradient background — see
>   [Card.tsx](../../../apps/web/src/dashboard/primitives/Card.tsx),
>   [dashboard.css](../../../apps/web/src/dashboard/dashboard.css)). It does **not**
>   mark a "title card"; use it at most to emphasise the KPI-summary card, never to
>   host the entry title.
>
> Rule of thumb: *"Does this card title repeat the dashboard name or just say
> 'Overview'? → drop it. Is it a distinct section label? → keep it."*

**Primitive props templates**:

*KPIStrip* (single-row aggregate):
```json
{ "type": "KPIStrip", "props": { "items": [
  { "label": "Scripts",       "field": "scripts",      "format": "number" },
  { "label": "Without calls", "field": "unused_count", "format": "number" }
]}}
```

*List* (clickable, requires a `uuid` column in the query):
```json
{ "type": "List", "props": {
  "rowTemplate": {
    "primary":   "{{name}}",
    "secondary": "{{file}} · {{step_count}} steps",
    "onClick":   { "action": "openObject",
                   "args": { "uuid": "{{uuid}}", "type": "Script" } }
  },
  "empty": { "message": "No entries found." }
}}
```

*Table* (analysis-focused):
```json
{ "type": "Table", "props": {
  "rowKey": "<unique column>",
  "density": "comfortable",
  "columns": [
    { "field": "name",  "label": "Name" },
    { "field": "count", "label": "Count", "align": "right" },
    { "field": "file",  "label": "File" }
  ],
  "onRowClick": { "action": "openObject",
                  "args": { "uuid": "{{uuid}}", "type": "{{type}}" } }
}}
```

*TileGrid* (navigation):
```json
{ "type": "TileGrid", "props": { "tile": {
  "title":    "{{name}}",
  "subtitle": "{{description}}",
  "icon":     "{{icon}}",
  "onClick":  { "action": "runQuery",
                "args": { "query": "{{name}}" } }
}}}
```

**Span values**: `12` = full width, `6` = half width, `4` = one-third, `8` + `4` = two-thirds + one-third. Several equally wide cards next to each other: `Stack` or direct children inside the grid.

#### 5.5 Create localization files

The dashboard system resolves UI labels at request time. The bundle files (`manifest.json`, `layout.json`) hold the English defaults; every other supported language lives in `locales/<lang>.json`.

**Languages to create** (10 total): `de`, `es`, `fr`, `it`, `nl`, `pt`, `sv`, `ja`, `ko`, `zh-Hans`. Do **not** create `en.json` — English is the bundle default and resolves directly from `manifest.json` / `layout.json`.

Path: `rest-api/templates/dashboards-custom/<id>/locales/<lang>.json`

**Structure** — two sections:

- `manifest`: translates the user-visible manifest fields (`title`, `description`).
- `layout` (optional but recommended): dot-path overrides into `layout.json`. The key is the JSON path to a string value; array indices use `[0]`, `[1]`, …

```json
{
  "manifest": {
    "title": "<title in target language>",
    "description": "<description in target language>"
  },
  "layout": {
    "root.children[0].children[0].props.items[0].label": "<KPI label>",
    "root.children[1].props.title": "<section card title>",
    "root.children[1].children[0].props.columns[0].label": "<column label>"
  }
}
```

**Only override user-visible literals**: card titles, KPI labels, column headers, list `primary`/`secondary` text, empty-state messages. **Never translate**: dataset IDs, field names, parameter names, `format` values, icon names, action arguments.

Reference: `rest-api/templates/dashboards-custom/script_todos/locales/de.json` (compact) and `rest-api/templates/dashboards-custom/external_apis/locales/fr.json` (with `layout` overrides).

Generate all 10 files in one pass. Keep the JSON-pointer structure of the `layout` section identical across languages — only the values change.

---

#### 5.6 Post-write verification (BLOCKING — must succeed before step 5.7)

The bundle loader fails silently on schema violations. Never claim the dashboard is "immediately available" without running this gate.

##### 5.6.1 Trigger the cache reload

```bash
curl -s -X POST http://localhost:3003/api/admin/reload >/dev/null
```

If the REST API is not running on `:3003`: skip the reload, but warn the user that the dashboard will only become visible after `bash tools/start-servers.sh` and a browser reload. Do **not** continue with 5.6.2 — the verification can't run without the server. Jump to 5.7 with a clearly-labelled warning.

##### 5.6.2 Verify the bundle loads

```bash
HTTP=$(curl -s -o /tmp/fmlab_dashboard_check.json \
            -w "%{http_code}" \
            http://localhost:3003/api/dashboards/<id>)
echo "HTTP $HTTP"
```

| Result | Action |
|---|---|
| HTTP 200 with `"success":true` and `data.manifest.id == "<id>"` | Verification OK — continue to 5.7 |
| HTTP 404 with `TEMPLATE_NOT_FOUND` | Bundle was rejected during load. Go to 5.6.3 (read the log) |
| HTTP 5xx or curl error | Server problem unrelated to the bundle — show error to user, do not auto-fix |

##### 5.6.3 Read the server log for the real cause

```bash
tail -100 logs/rest-api.log | grep -F "[dashboard:<id>]" | tail -5
```

The Joi message is human-readable and points at the offending field. Use it to drive 5.6.4.

##### 5.6.4 Auto-heal the top-3 root causes

For each of the following patterns, patch the bundle on disk, re-run 5.6.1 + 5.6.2 once. If the second attempt also fails, stop and surface the log line to the user — don't loop further.

| Log message contains | Auto-fix |
|---|---|
| `params[N].type must be one of [string, number, boolean]` | In `manifest.json` rewrite the offending `type` value: `integer`/`int`/`float` → `number`, `bool` → `boolean`. |
| `manifest.id="X" differs from directory name` | In `manifest.json` set `id` to the directory name (the truth). Never rename the directory to match — IDs were chosen in step 4. |
| `datasets[N].source ... pattern` or `"source" with value "..." fails to match the required pattern` | Source URI missing its scheme. Prepend `bundle:` if the value looks like a path (`data/foo.sql`); otherwise ask the user. |

For anything else (layout schema errors, dataset name mismatch, JSON parse error) do **not** auto-fix — show the log line to the user and ask.

##### 5.6.5 Probe each dataset (optional — only if the project's DuckDB is accessible)

If a local DuckDB binary is reachable (per the resolver in CLAUDE.md), do a minimal smoke-run of each declared dataset with default param values:

```bash
# Replace <sql_file> and any required params with the manifest defaults
duckdb db/fm_catalog.duckdb -csv -noheader \
  -c "$(cat rest-api/templates/dashboards-custom/<id>/data/<sql_file>) LIMIT 1"
```

A DuckDB error here means the SQL is broken even though the manifest validates. Report it before 5.7; do not auto-fix SQL.

If DuckDB is not reachable, skip silently — the runtime will surface SQL errors when the user opens the dashboard.

---

#### 5.7 Emit the closing message

```
Dashboard bundle created: rest-api/templates/dashboards-custom/<id>/
  ├── manifest.json
  ├── layout.json
  ├── data/
  │   └── <dataset_name>.sql
  └── locales/
      ├── de.json
      ├── es.json
      ├── fr.json
      ├── it.json
      ├── nl.json
      ├── pt.json
      ├── sv.json
      ├── ja.json
      ├── ko.json
      └── zh-Hans.json

Dashboard ID:   <id>
Title:          <title> (English default; 10 translations available)
Presentation:   <Primitive>
Dataset:        <dataset_name> (Preview: N rows)
Verified:       GET /api/dashboards/<id> → HTTP 200 (loader accepted manifest + layout)

The dashboard is live at /api/dashboards/<id> — browser reload (Ctrl+R) picks it up.
Bundles, SQL templates and locale files are hot-reloaded by mtime; no server restart needed.
```

If 5.6 ran in **server-unavailable mode** (REST API not on :3003), replace the `Verified:` line with:

```
Verified:       SKIPPED (REST API not running on :3003 — start with bash tools/start-servers.sh, then reload the browser)
```

If 5.6 auto-healed a manifest issue, mention it briefly:

```
Auto-heal:      manifest.params[].type "integer" → "number" (re-verified OK)
```

---

## Conventions at a glance

### Primitive registry (v1)

| Primitive  | When to use                                  | Important props              |
|------------|----------------------------------------------|------------------------------|
| Grid       | Root container (always)                      | `columns`, `gap`             |
| Card       | Frame with title around each piece of content | `span`, `title`, `variant`   |
| Stack      | Stack multiple cards vertically              | `gap`, `align`               |
| Row        | Horizontal group                             | `gap`, `align`               |
| KPI        | A single key figure                          | `label`, `field`, `format`   |
| KPIStrip   | 2–8 key figures next to each other           | `items[]`                    |
| List       | Clickable list (uuid required in the query)  | `rowTemplate`, `empty`       |
| TileGrid   | Tile-based navigation                        | `tile`, `minTileWidth`       |
| Table      | Tabular display, multiple columns            | `columns[]`, `rowKey`, `density` |
| Markdown   | Help text, static content                    | `content`                    |
| NavButton  | Navigation button                            | `label`, `icon`, `onClick`   |
| Spacer     | Whitespace                                   | `size`                       |
| Image      | Image from the bundle (`asset:`)             | `src`, `alt`, `width`        |
| Empty      | Placeholder for an empty dataset             | `message`                    |

### Click actions (whitelist)

| Action            | Args                             | Effect                                    |
|-------------------|----------------------------------|-------------------------------------------|
| `openObject`      | `uuid`, `type`, `file?`          | Open the object's detail view             |
| `openDashboard`   | `id`, `params?`                  | Load another dashboard                    |
| `applyFilter`     | `q?`, `type?`, `file?`           | Set the search filter in the header       |
| `runQuery`        | `query`, `params?`               | Open a custom template in the `_generic` bundle |
| `openUrl`         | `url` (https:// only)            | External URL with confirmation            |
| `copyToClipboard` | `value`                          | Copy a value to the clipboard             |

### Token substitution

`{{field}}` is replaced at render time with the field value of the current data row.
Optional filters: `{{ field | upper }}`, `{{ field | number:2 }}`, `{{ field | date:relative }}`, `{{ field | truncate:50 }}`, `{{ field | default:"-" }}`

### Format values for KPI/Table columns

`number`, `badge`, `date:relative`, `date:short`, `boolean`

---

## Example run

**User**: `/create-custom-dashboard Scripts without comment`

**Step 1** — topic is clear, continue directly.

**Step 2** — draft the query:

```sql
-- @template_type: report
-- @description: Scripts without a Comment step at the first position.
-- @params: file (optional)

SELECT
    s.Script_UUID                 AS uuid,
    s.Script_Name                 AS name,
    s.File_Name                   AS file,
    COUNT(st.Step_UUID)           AS step_count
FROM ScriptCatalog s
LEFT JOIN StepsForScripts st ON st.Script_UUID = s.Script_UUID
WHERE (s.Folder_Type IS NULL)
  AND NOT s.Is_Separator
  AND (getvariable('file') IS NULL OR s.File_Name = getvariable('file'))
  AND s.Script_UUID NOT IN (
      SELECT Script_UUID FROM StepsForScripts
      WHERE Step_Name = 'Comment' AND Step_Index = 1
  )
GROUP BY ALL
ORDER BY step_count DESC
LIMIT CAST(COALESCE(getvariable('limit'), '50') AS INTEGER);
```

Run the preview → show the table → "47 scripts found. Recommend **List** (clickable)."

**Step 3** — "List recommended, alternative Table. Please confirm."
→ User: "List works."

**Step 4** — "Suggested name: `scripts_without_comment` / Title: "Scripts without comment""
→ User: "Yes."

**Step 5** — create the bundle files:
- `rest-api/templates/dashboards-custom/scripts_without_comment/data/scripts_without_comment.sql`
- `rest-api/templates/dashboards-custom/scripts_without_comment/manifest.json`
- `rest-api/templates/dashboards-custom/scripts_without_comment/layout.json`
- `rest-api/templates/dashboards-custom/scripts_without_comment/locales/{de,es,fr,it,nl,pt,sv,ja,ko,zh-Hans}.json`

---

## Important notes

- **Preview first, then save**: files are written only in step 5, after confirmation of the presentation (step 3) and the name (step 4).
- **No free-form SQL in the bundle**: queries exclusively as `.sql` files under `data/`. No SQL code in `manifest.json` or `layout.json`.
- **Multiple datasets**: if the user wants more than one data perspective (e.g. overview KPI + detail list), create multiple SQL files and datasets. In `layout.json` place each dataset in its own `Card`.
- **Cross-file**: if the database contains multiple FileMaker files, always plan for an optional `file` parameter in the query.
- **Keep the step order**: the interactive dialogue in steps 3 and 4 is not optional — no direct bundle writing without confirmation.
- **Never skip step 5.6**: writing the bundle and announcing success without verifying `GET /api/dashboards/<id>` has hit us repeatedly — Joi rejects silently, the bundle vanishes from the list, and the API only says "not found or invalid". The real reason is always in `logs/rest-api.log` with prefix `[dashboard:<id>]`. Step 5.6 is the only safe finish line.
