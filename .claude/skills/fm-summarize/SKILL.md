---
name: fm-summarize
version: 0.8.5
description: "Generates a technical summary of a FileMaker object (Script, Field, Layout, CustomFunction, ValueList, BaseTable, TableOccurrence, Relationship, etc.) from the DuckDB database `db/fm_catalog.duckdb`. Uses ObjectCatalog/ObjectLinks for resolution and dependencies and produces a structured Markdown description. Supports two modes — Standard (complete with flow and dependencies) and Short (1-2 paragraphs of prose, via `--short` flag or trigger words). Triggers (English): \"describe script X\", \"summarize field X\", \"/fm-summarize\", \"explain layout X technically\". Triggers (German): \"beschreibe Script X\", \"fasse das Feld X zusammen\", \"erkläre mir das Layout X technisch\". Triggers (Spanish): \"describe el script X\", \"resume el campo X\". Triggers (French): \"décris le script X\", \"résume le champ X\". Triggers (Italian): \"descrivi lo script X\", \"riassumi il campo X\". Triggers (Dutch): \"beschrijf script X\", \"vat veld X samen\". Triggers (Portuguese): \"descreva o script X\", \"resuma o campo X\". Triggers (Swedish): \"beskriv skript X\", \"sammanfatta fältet X\". Triggers (Japanese): \"スクリプトXを説明して\", \"フィールドXを要約して\". Triggers (Korean): \"스크립트 X 설명해 줘\", \"필드 X 요약해 줘\". Triggers (Chinese): \"描述脚本 X\", \"总结字段 X\"."
---

# FileMaker Object Summary

Produce a structured, technical description of a FileMaker object based on the DuckDB database `db/fm_catalog.duckdb`.

## Ground rules

- **Database / SQL**: English — table and column names of `db/fm_catalog.duckdb` are English DDL identifiers; never translate them
- **Database**: `db/fm_catalog.duckdb` — the master catalog file, accessed read-only via the local DuckDB CLI binary (`duckdb` in PATH; fallbacks `~/.duckdb/cli/latest/duckdb`, `/opt/homebrew/bin/duckdb`, `/usr/local/bin/duckdb` — VS Code does not inherit the shell PATH). Never read from `rest-api/db/fm_catalog.duckdb` — that copy is API-internal and may be stale.
- **Call**: `duckdb db/fm_catalog.duckdb -c "<SQL>"` via Bash
- **File references**: Markdown links (e.g. `[Script_Name](db/fm_catalog.duckdb)`)
- **Before every DB query**: make sure the object is uniquely identified (see Step 1)
- **Response language**: follows the user's prompt language — see next section

## Response language

Reply in the language the user used for their prompt — that is the primary signal (e.g. an English question → English answer, a Spanish question → Spanish answer, even if the project default is German). Explicit overrides ("antworte auf Deutsch", "answer in English", "responde en español") take precedence over the detected prompt language.

**What gets translated to the response language**:
- Markdown section headers of the report (e.g. EN `### Purpose` ↔ DE `### Zweck` ↔ ES `### Propósito` ↔ FR `### Objectif` ↔ IT `### Scopo` ↔ NL `### Doel` ↔ PT `### Propósito` ↔ SV `### Syfte` ↔ JA `### 目的` ↔ KO `### 목적` ↔ ZH `### 目的`)
- Prose: Purpose, Notes, descriptive remarks, hedging vocabulary
- Generic table column headings (Field / Action / Comment, etc.)

**What stays original / English regardless of response language**:
- **FileMaker identifiers** (script, field, layout, table, TO, relationship names) — must match the actual FileMaker source 1:1
- **`Link_Role` values** (`calls_script`, `sets_field`, `displays_field`, `navigates_to_layout`, …) — technical labels of the data model
- **SQL queries, column names, table names of the DuckDB catalog** — always English (DDL identifiers)
- **CLI flags** (`--short`) and skill-call tokens (`/fm-summarize`)

## Output modes

There are two modes that produce outputs of differing scope:

### Standard mode (default)

Detailed Markdown summary with all sections — header, purpose, technical details, flow (numbered for scripts), Uses, Used by, notes. For the exact format see Step 4.

### Short mode (`--short`)

1-2 paragraphs of **prose**, **no** Markdown sections, **no** tables, **no** code blocks. Contains only the essentials: what the object is and what it does, optionally with a rough caller / callee hint.

**Activating short mode**:

1. **Explicit flag** in the skill call — position is free (before or after the object name):
   ```
   /fm-summarize Accounting_PrintInvoice --short
   /fm-summarize --short Accounting_PrintInvoice
   /fm-summarize --short <UUID>
   ```

2. **Natural language** — if the user request contains one of the following terms, automatically activate short mode, even without an explicit `--short`. Detection is **case-insensitive** and language-agnostic; the keyword may appear anywhere in the prompt:
   - **English**: short, brief, brief summary, short description, concise, 1-2 sentences, in a few sentences, rough, overview, TL;DR, TLDR
   - **German**: kurz, knapp, knappe Zusammenfassung, Kurzbeschreibung, 1-2 Sätze, in wenigen Sätzen, grob, überblicksartig
   - **Spanish**: breve, corto, resumen breve, descripción breve, conciso, en pocas frases, 1-2 frases
   - **French**: bref, court, résumé bref, description brève, concis, en quelques phrases, 1-2 phrases
   - **Italian**: breve, corto, riepilogo breve, descrizione breve, conciso, in poche frasi, 1-2 frasi
   - **Dutch**: kort, beknopt, korte samenvatting, korte beschrijving, in een paar zinnen, 1-2 zinnen
   - **Portuguese**: breve, curto, resumo breve, descrição breve, conciso, em poucas frases, 1-2 frases
   - **Swedish**: kort, kortfattat, kort sammanfattning, kort beskrivning, koncis, med några meningar, 1-2 meningar
   - **Japanese**: 短く, 簡潔に, 簡単に, 要約, 短い説明, 概要, 1-2文で
   - **Korean**: 짧게, 간단히, 간략히, 요약, 짧은 설명, 개요, 1-2문장으로
   - **Chinese**: 简短, 简要, 简单介绍, 简短说明, 概要, 1-2句话

**Mode differences**:

| Section | Standard | `--short` |
|---------|----------|-----------|
| Header (Name, Type, File, UUID) | ✓ | ✗ (only inline mention in prose) |
| Purpose | ✓ | ✓ (core of the short output) |
| Technical details | ✓ | ✗ |
| Flow (for scripts) | numbered, complete | ✗ (at most a half-sentence: "Calls 4 sub-scripts") |
| Uses | grouped by Link_Role | ✗ |
| Used by | grouped | ✗ (at most "called from 3 places") |
| Notes | ✓ | only critical (e.g. "DDR-Info missing") |

**Query efficiency in short mode**: In short mode only the header query and an aggregation count for callers / callees are executed. The expensive detail queries (all steps with DDR_ScriptSteps, all field usages with comments, all lookup sources) are skipped — this saves runtime and tokens when reading the tool results.

**Identification is the same in both modes** — in short mode too the object must first be unique (Step 1 is indispensable).

## Workflow

### Step 1 — Identify the object (BLOCKING)

Before any database query for the description runs, the object to be described MUST be unique.

**Input sources for identification**:
1. Explicitly passed UUID — directly usable
2. Explicitly passed name + (optional) type + (optional) file
3. Derivable from the previous conversation context (e.g. an object previously shown in a list)

**Process**:

1. **If a UUID is provided** → resolve via ObjectCatalog:
   ```sql
   SELECT Object_UUID, Object_Type, Object_Name, File_Name, Source_Table, Object_ID
   FROM ObjectCatalog
   WHERE Object_UUID = '<UUID>';
   ```
   On exactly 1 hit: continue with Step 2.
   On no hit: inform the user and ask back.
   **On >1 hit (clone/modular files share the same Object_UUID)**: the UUID is **not**
   unique. Do **NOT** silently take the first row. If a file is also known (passed as
   `--file <File>`, or carried in the conversation identity, see below), add
   `AND File_Name = '<File>'` to resolve. Otherwise list the matches (Type, Name,
   **File_Name**) and ask the user which file is meant.

2. **If only a name is provided** → search in ObjectCatalog (case-insensitive, exact matches preferred):
   ```sql
   SELECT Object_UUID, Object_Type, Object_Name, File_Name, Source_Table
   FROM ObjectCatalog
   WHERE LOWER(Object_Name) = LOWER('<Name>')
   ORDER BY Object_Type, File_Name;
   ```
   - **0 matches**: LIKE fallback `LOWER(Object_Name) LIKE LOWER('%<Name>%')`. If still nothing → inform the user, suggest similar objects, DO NOT guess.
   - **Exactly 1 match**: continue with Step 2.
   - **>1 matches**: print a list of all matches (type, name, file) and ask the user to choose. Do **NOT** automatically take the first object.

3. **If the context is ambiguous** (e.g. the user says "describe the script" without a clear reference): ask which object is meant. Better to ask once too often than to describe the wrong object.

4. **If a type hint is provided** (e.g. "describe the layout 'Customers'"): add the filter `Object_Type = '<Type>'`.

**Important**: Only after unique identification may the type-specific description workflow start.

### Step 2 — Retrieve type-specific data

Based on `Object_Type` pick the matching workflow. All queries use `File_Name` AND the respective type UUID, because names are not unique across files.

#### Script

```sql
-- Header
SELECT * FROM ScriptCatalog WHERE Script_UUID = '<UUID>' AND File_Name = '<File>';

-- Steps (DDR_ScriptSteps provides readable text if available)
SELECT
    s.Step_Index,
    s.Step_Name,
    s.Is_Enabled,
    s.Variable_Name,
    s.Calculation_Text,
    ddr.Step_Text  -- preferred for display if NOT NULL
FROM StepsForScripts s
LEFT JOIN DDR_ScriptSteps ddr
    ON s.DDR_UUID = ddr.Step_UUID
   AND s.File_Name = ddr.File_Name
WHERE s.Script_UUID = '<UUID>' AND s.File_Name = '<File>'
ORDER BY s.Step_Index;
```

Then dependencies via ObjectLinks (see Step 3). Relevant Link_Roles for scripts:
- **Called from the script**: Source_UUID = Script-UUID, Link_Role IN (`calls_script`, `sets_field`, `navigates_to_field`, `navigates_to_layout`, `sets_variable`, `reads_variable`)
- **Who calls this script**: Target_UUID = Script-UUID, Link_Role IN (`calls_script`, `triggers_script`, `trigger_script`)

For scripts, the step-by-step flow is the centrepiece of the summary. Number each step with `Step_Index`. Mark disabled steps with `(disabled)`.

#### Field

```sql
SELECT *
FROM FieldsForTables
WHERE Field_UUID = '<UUID>' AND File_Name = '<File>';
```

Evaluating the columns:
- **Basics**: `Field_Name`, `Table_Name`, `Field_Type` (Normal/Calculated/Summary), `Data_Type`, `Field_Comment`, `Is_Global`, `Max_Repetitions`
- **Calculated Field**: `Calculation_Text` (plain text), `DDR_Hash` for JOIN to DDR_Calculations
- **AutoEnter**: `AutoEnter_Type` determines the detail columns to display
  - `Looked_up`: `Lookup_Field_Name`, `Lookup_TO_Name`, `Lookup_DontCopyIfEmpty`, `Lookup_NoMatchOption`
  - `Calculated`: `AE_Calc_Text`, `AE_Calc_Hash`, `AE_Calc_OverwriteExisting`, `AE_Calc_AlwaysEvaluate`
  - `ConstantData`: `AE_ConstantData`
  - `SerialNumber`, `CreationDate`, etc.: only report the type

Optional (if `DDR_Hash` or `AE_Calc_Hash` is present): formula chunks from DDR_Calculations:
```sql
SELECT Chunk_Index, Chunk_Type, Chunk_Content
FROM DDR_Calculations
WHERE Calc_Hash = '<DDR_Hash or AE_Calc_Hash>'
  AND File_Name = '<File>'
ORDER BY Chunk_Index;
```

Usages via ObjectLinks: `Target_UUID = Field-UUID` shows where the field is used (`displays_field`, `sets_field`, `lookup_source`, `left_field`/`right_field` in Relationships, `source_field` in ValueLists, etc.).

#### Layout

```sql
-- Layout itself
SELECT L_ID, L_Name, L_TO_Name, File_Name FROM Layouts
WHERE L_UUID = '<UUID>' AND File_Name = '<File>';

-- Sections (Header/Body/Footer/...)
SELECT * FROM LayoutParts WHERE Layout_ID = <L_ID> AND File_Name = '<File>';

-- Object statistics (do not list every object individually — can be hundreds)
SELECT Object_Type, COUNT(*) AS Anzahl, MAX(Nesting_Level) AS Max_Tiefe
FROM LayoutObjects
WHERE Layout_ID = <L_ID> AND File_Name = '<File>'
GROUP BY Object_Type
ORDER BY Anzahl DESC;

-- Script triggers of the layout
SELECT * FROM ScriptTriggers
WHERE Object_UUID = '<L_UUID>' AND File_Name = '<File>';
```

Use ObjectLinks to determine which fields/scripts/value lists the layout references (Source_File = Layout file, Source_Type = `LayoutObject`, parent_layout points to the layout).

#### CustomFunction

```sql
SELECT * FROM CustomFunctionsCatalog
WHERE CF_UUID = '<UUID>' AND File_Name = '<File>';

SELECT Calculation_Code FROM CalcsForCustomFunctions
WHERE CF_UUID = '<UUID>' AND File_Name = '<File>';

-- If DDR_Hash present: chunks with resolved references
SELECT Chunk_Index, Chunk_Type, Chunk_Content
FROM DDR_Calculations
WHERE Calc_Hash = '<DDR_Hash>' AND File_Name = '<File>'
ORDER BY Chunk_Index;
```

Usages: ObjectLinks with Target_UUID = CF-UUID shows who calls the CF.

#### ValueList

```sql
SELECT vl.*, o.Source_Type, o.Custom_Values, o.Field_Name, o.TO_Name
FROM ValueListCatalog vl
LEFT JOIN OptionsForValueLists o
    ON vl.VL_UUID = o.VL_UUID AND vl.File_Name = o.File_Name
WHERE vl.VL_UUID = '<UUID>' AND vl.File_Name = '<File>';
```

Usages: ObjectLinks `Target_UUID = VL-UUID`, Link_Role `uses_valuelist` shows layout objects that use this value list.

#### BaseTable

```sql
SELECT * FROM BaseTableCatalog WHERE BT_UUID = '<UUID>' AND File_Name = '<File>';

-- Fields
SELECT Field_Name, Field_Type, Data_Type, Is_Global, Field_Comment
FROM FieldsForTables WHERE Table_UUID = '<UUID>' AND File_Name = '<File>'
ORDER BY Field_ID;

-- Table occurrences
SELECT TO_Name, TO_ID FROM TableOccurrenceCatalog
WHERE BT_UUID = '<UUID>' AND File_Name = '<File>';
```

#### TableOccurrence

```sql
SELECT * FROM TableOccurrenceCatalog WHERE TO_UUID = '<UUID>' AND File_Name = '<File>';

-- Relationships this TO participates in
SELECT * FROM RelationshipCatalog
WHERE Left_TO_UUID = '<UUID>' OR Right_TO_UUID = '<UUID>'
  AND File_Name = '<File>';
```

#### Relationship

```sql
SELECT * FROM RelationshipCatalog WHERE Rel_ID = <ID> AND File_Name = '<File>';
```

Relationship predicates are contained in the `Left_*` / `Right_*` columns, operator in `Operator`.

#### Generic fallback (all other Object_Types)

If no type-specific workflow is defined:

```sql
-- Basic info
SELECT * FROM ObjectCatalog WHERE Object_UUID = '<UUID>';

-- Incoming links (what uses the object)
SELECT Source_Type, Source_File, Link_Role,
       (SELECT Object_Name FROM ObjectCatalog WHERE Object_UUID = ol.Source_UUID) AS Source_Name
FROM ObjectLinks ol
WHERE Target_UUID = '<UUID>'
ORDER BY Source_Type;

-- Outgoing links (what the object uses)
SELECT Target_Type, Target_File, Link_Role,
       (SELECT Object_Name FROM ObjectCatalog WHERE Object_UUID = ol.Target_UUID) AS Target_Name
FROM ObjectLinks ol
WHERE Source_UUID = '<UUID>'
ORDER BY Target_Type;
```

### Step 3 — Dependencies (for all types)

Standard query for incoming and outgoing links. **Important**: filter only on `Link_Type = 'operational'` to hide structural hierarchy noise (parent_object, parent_layout, parent_script). Include structural links only when they are substantively relevant for the object type.

```sql
-- What this object uses (outgoing)
SELECT
    ol.Link_Role,
    ol.Target_Type,
    oc.Object_Name AS Target_Name,
    ol.Target_File,
    ol.Is_Cross_File
FROM ObjectLinks ol
LEFT JOIN ObjectCatalog oc ON ol.Target_UUID = oc.Object_UUID
WHERE ol.Source_UUID = '<UUID>'
  AND ol.Link_Type = 'operational'
ORDER BY ol.Link_Role, oc.Object_Name;

-- Who uses this object (incoming)
SELECT
    ol.Link_Role,
    ol.Source_Type,
    oc.Object_Name AS Source_Name,
    ol.Source_File,
    ol.Is_Cross_File
FROM ObjectLinks ol
LEFT JOIN ObjectCatalog oc ON ol.Source_UUID = oc.Object_UUID
WHERE ol.Target_UUID = '<UUID>'
  AND ol.Link_Type = 'operational'
ORDER BY ol.Link_Role, oc.Object_Name;
```

### Step 4 — Produce the Markdown summary

**In short mode** (`--short` or trigger word): jump directly to the "Short-mode output" section at the end of this step. The detailed section structure below only applies in standard mode.

The standard output follows this schema. Omit sections without content.

```markdown
## <Object_Type>: <Name>

**File**: <File_Name>
**UUID**: `<Object_UUID>`
**Internal ID**: <Object_ID> (if relevant)

<!-- Conversation identity = the pair (UUID, File_Name). Always emit BOTH lines:
     clone/modular files share Object_UUID, so downstream skills (fmide-show,
     fm-analyze) must read the File_Name back to resolve the object unambiguously. -->


### Purpose
<From Field_Comment / other comments, or a brief derivation from name + context.
If no comment is present: "No comment stored in the object." and a note
that the purpose was derived from the behaviour.>

### Technical details
<Type-specific — see below>

### Flow  *(scripts only)*
1. **<Step_Name>** — <DDR_Step_Text if present, otherwise Step_Name + Calculation_Text>
2. ...
   *(mark disabled steps with `(disabled)`)*

### Uses
<List of objects this object calls / references, grouped by Link_Role>
- **calls_script**: ScriptA, ScriptB
- **sets_field**: Table::Field
- ...

### Used by
<List of objects that call / reference this object>
- **LayoutObject** (displays_field): Layout "Customers"
- ...

### Notes
<Optional: notable items, e.g. cross-file links, very many usages, missing comments,
disabled steps, unusual constructions>
```

**Format rules (standard mode)**:
- Markdown tables only when they really add value (>5 rows, multiple columns)
- For very long lists (>20 entries): summarise ("12 more fields ...") and provide details on request
- Keep all FileMaker identifiers in the original language of the solution (script/field/table names are not translated)
- **Section headers and prose are produced in the response language** (see the "Response language" section near the top); the English headers in the template above are illustrative
- For scripts: when `DDR_ScriptSteps.Step_Text` is present, ALWAYS prefer the readable text — it contains resolved field names, variable contents and parameters

#### Short-mode output (`--short`)

In short mode the section structure above is dropped entirely. Instead: **1-2 paragraphs of prose** that compactly answer the following questions:

1. **What is the object?** — type, name, file (inline, e.g. "The script **Accounting_PrintInvoice** in the file `Invoices`")
2. **What does it do?** — 1-3 sentences, core function in your own words
3. **(Optional) How is it embedded?** — at most one half-sentence about callers or sub-calls, ONLY if it materially illuminates the context

**Prohibitions in short mode**:
- No Markdown headers (`##`, `###`)
- No lists, no bullet points
- No tables
- No code blocks
- No UUID display (technical detail information belongs in standard mode)
- No "Flow" (not even shortened)

**Reduced query list in short mode**:

| Object_Type | Standard-mode queries | Short-mode queries |
|-------------|------------------------|---------------------|
| Script | ScriptCatalog + StepsForScripts + DDR_ScriptSteps + all ObjectLinks | Only ScriptCatalog + COUNT(*) callers + COUNT(*) sub-calls |
| Field | FieldsForTables + DDR_Calculations + all usages | Only FieldsForTables (Field_Comment + Field_Type + AutoEnter_Type) |
| Layout | Layouts + LayoutParts + LayoutObjects aggregation + triggers | Only Layouts + COUNT(*) of LayoutObjects |
| CustomFunction | CustomFunctionsCatalog + CalcsForCustomFunctions + DDR + callers | Only CustomFunctionsCatalog + COUNT(*) callers |
| Other | Type-specific queries + ObjectLinks bidirectional | Only ObjectCatalog entry + COUNT(*) incoming/outgoing links |

**Example output (short mode, script)**:

> The script **Accounting_PrintInvoice** in the file `Invoices` produces a PDF output of a single invoice at the supplied storage location. It is called from 2 places (manually from invoice editing and from the batch processing) and uses 2 helper scripts.

**Example output (short mode, field)**:

> The field **Email** in the table `Customers` is a text field with AutoEnter Calculated `Lower(Self)`. According to the field comment the normalisation serves unique comparability for email dispatch.

**If short mode delivers too little information**: append a single hint sentence at the end of the prose, such as *"For the full steps and dependencies call `/fm-summarize <Name>` without `--short`."*

### Step 5 — Output

Print the Markdown summary in chat. Do NOT write a file (except as part of the planned extension described below).

## Important notes

- **Check DDR availability**: `SELECT Has_DDR_INFO FROM XMLMetadata WHERE Filename = '<File>';` If `False`, `DDR_ScriptSteps` and `DDR_Calculations` are empty and deliver no plain-text texts. In that case fall back to `Step_Name`, `Calculation_Text` and `Variable_Name`.
- **Multi-file**: If the object has cross-file dependencies (`Is_Cross_File = TRUE`), call them out explicitly.
- **Performance**: For layouts with hundreds of objects do NOT list all LayoutObjects individually — always aggregate.
- **No speculation**: If the data is incomplete, note that honestly instead of guessing.
- **Read vs. write**: This skill only reads. Never execute UPDATE/INSERT/DELETE on the database.
- **Order of actions**: identification → confirmation (if ambiguous) → type-specific queries → dependencies → output. Do not cut corners.

## Examples

### Example 1: Unambiguous script name

**User (English, primary)**: "Describe the script 'Kunde anlegen'"
**User (German, equivalent)**: "Beschreibe das Script 'Kunde anlegen'"
**User (French, equivalent)**: "Décris le script 'Kunde anlegen'"

Note: the object name `Kunde anlegen` stays as-is in any language — it is the FileMaker source identifier.

1. Search in ObjectCatalog → 1 match (Object_Type = `Script`, File_Name = `KundenDB`)
2. Query ScriptCatalog + StepsForScripts + DDR_ScriptSteps
3. Query ObjectLinks for incoming/outgoing links
4. Produce the Markdown summary with numbered flow, headers and prose in the response language (English/German/French depending on the prompt)

### Example 2: Ambiguous name

**User**: "What does 'Suchen' do?"

1. Search in ObjectCatalog → 4 matches (1× Script in `KundenDB`, 1× Script in `RechnungenDB`, 1× CustomFunction in `KundenDB`, 1× Layout in `KundenDB`)
2. **Output to the user**:
   ```
   There are several objects named 'Suchen'. Which one do you mean?
   1. Script "Suchen" in KundenDB
   2. Script "Suchen" in RechnungenDB
   3. CustomFunction "Suchen" in KundenDB
   4. Layout "Suchen" in KundenDB
   ```
3. Only continue with Step 2 after the user's answer.

### Example 3: Context derivation

**Conversation context**: a previous query listed five fields; the last one was `Kunden::Telefon` (UUID `abc-123`).

**User**: "Describe the last of those fields"

1. Derive UUID `abc-123` from the context
2. Start directly with the FieldsForTables query (no follow-up needed)
3. Evaluate AutoEnter columns, include DDR_Calculations if applicable
4. Usages via ObjectLinks
5. Print summary

### Example 4: Short mode

**User (flag form)**: "/fm-summarize Accounting_PrintInvoice --short"
**User (English, natural)**: "Describe Accounting_PrintInvoice briefly"
**User (German, natural)**: "Beschreibe Accounting_PrintInvoice kurz"
**User (French, natural)**: "Décris brièvement Accounting_PrintInvoice"

All four prompts activate short mode; the response is produced in the prompt's language.

1. Identification as usual → 1 match (Script in `Invoices`)
2. **Reduced queries**: only ScriptCatalog header + two COUNT(*) aggregations over ObjectLinks (callers / sub-calls). NO steps, NO DDR_ScriptSteps, NO field usages.
3. **Output** (1-2 paragraphs of prose, no sections):

   > The script **Accounting_PrintInvoice** in the file `Invoices` produces a PDF output of a single invoice. It is called from 2 places and calls 2 helper scripts.
   >
   > For the full steps and dependencies call `/fm-summarize Accounting_PrintInvoice` without `--short`.

### Example 5: Not found

**User**: "Describe the script 'KundeAnlegenV2'"

1. Search in ObjectCatalog → 0 matches
2. LIKE fallback `%Kunde%anlegen%` → 1 match: "KundeAnlegen"
3. **Output**: "A script named 'KundeAnlegenV2' does not exist. Did you perhaps mean 'KundeAnlegen'? Should I describe that one?"

## Planned extensions (future stage)

> **Status**: Documentation only — not implemented. Activation happens once the Obsidian vault is set up.

After producing the summary, the skill should ask the user whether the description should be stored as a note for the FileMaker object. Specification:

- **Target location**: Obsidian vault containing all project notes for the FileMaker solution. Path still to be configured (presumably in a project-local configuration file or environment variable, e.g. `FM_OBSIDIAN_VAULT`).
- **Storage structure**: one sub-folder per object type (`Scripts/`, `Fields/`, `Layouts/`, `CustomFunctions/`, …).
- **File names**: must contain the object UUID so the object remains uniquely referenceable even if the FileMaker name changes. Proposal: `<sanitized-name>__<uuid-short>.md`.
- **Update behaviour**: existing notes are NOT overwritten. Instead the new summary is appended to the existing file (append) — typically with a separator section `## Update <date>`. Rationale: content manually added by the user (e.g. design decisions, ToDos) must not be lost. Compare memory `feedback_obsidian_updates`.
- **Frontmatter**: when creating a note for the first time, set YAML frontmatter with `object_uuid`, `object_type`, `file_name`, `created_at`.
- **User interaction**: after printing the summary in chat, ask: "Should I store this description as a note in the Obsidian vault for the object? (y/n)". On yes: check whether a note already exists for the UUID; `create` or `append` accordingly.

**TODOs before activation**:
1. Decide on a configuration mechanism for the vault path
2. Implement append logic (detection of existing file + separator section)
3. Sanitising function for file names derived from FileMaker names (special characters, spaces)
4. Align frontmatter schema with the user
