---
name: fm-graph-cluster
version: 0.8.5
description: "Segments the FileMaker object graph in `db/fm_catalog.duckdb` into modules/communities, finds the best resolution for this solution's size by sweeping candidate resolutions and scoring them (modularity Q + distribution guardrails), names the communities semantically (CommunityNames.Semantic_Name, optionally Semantic_Description in --deep-research), writes an analysis report to `output/`, and syncs the named partition to the Graph Explorer. Orchestrates the existing tools/graph-export tooling (cluster.sh, cluster_louvain.mjs/cluster_leiden.py, cluster_load.sql); it does not replace it. Triggers (English): \"/fm-graph-cluster\", \"cluster the graph\", \"detect communities/modules\", \"name the clusters\", \"segment the solution into modules\". Triggers (German): \"/fm-graph-cluster\", \"clustere den Graph\", \"finde die Module/Communities\", \"benenne die Cluster\", \"segmentiere die Lösung in Module\"."
---

# FileMaker Graph Clustering & Semantic Naming

Segment the FileMaker object graph into **modules (communities)**, pick the **resolution that
fits this solution's size**, give each community a **semantic name** (and, in deep-research mode,
a one-to-two-sentence description), then write a human-readable **analysis report** and sync the
named partition to the Graph Explorer.

This skill **orchestrates** the existing `tools/graph-export/` tooling — the Louvain/Leiden
engine, the heuristic names and the two-table model (`ObjectClusters` + `CommunityNames`) are
already built. What this skill adds: **automatic resolution selection** and **semantic naming**.

## Ground rules

- **DuckDB CLI**: `duckdb` in PATH; fallbacks (VS Code does not inherit the shell PATH, check in
  this order): `~/.duckdb/cli/latest/duckdb` → `/opt/homebrew/bin/duckdb` → `/usr/local/bin/duckdb`.
  **Never install DuckDB yourself** — if none is found, point the user to CLAUDE.md.
- **Database**: master `db/fm_catalog.duckdb` only. **Never** read from `rest-api/db/` (API-internal, may be stale).
  - **read-only** for all measuring (Preflight, sweep, naming-hint reads, report data).
  - **read-write only** for: the final `cluster.sh` run (Phase E) and the naming `UPDATE`s (F/G).
- **SQL identifiers stay English** (DuckDB DDL); never translate table/column names.
- **Response language** follows the user's prompt (like fm-analyze). The **generated**
  `Semantic_Name`/`Semantic_Description` are written in the prompt language (default German, since
  the solution is German). **FileMaker identifiers stay original** (script/field/layout/TO/variable names).
- **No silent caps**: every place the skill bounds work (deep-scan limit, name threshold) is
  **logged explicitly** — both in the chat and in the report.

## Parameters

Parse these from the user's invocation (e.g. `/fm-graph-cluster --deep-research --candidates=1,2,3`):

| Parameter | Default | Effect |
|---|---|---|
| `--resolution=<float>` | — | **Skips the multi-candidate sweep**, forces this resolution (see "Resolution mode" below) |
| `--candidates=0.5,1.0,…` | `0.5,0.75,1.0,1.5,2.0,3.0` | Candidate list for the sweep |
| `--engine=auto\|louvain\|leiden` | `auto` | Engine (auto → leiden if python3+igraph, else louvain) |
| `--seed=<int>` | `42` | PRNG seed (fixed ⇒ reproducible partition + colors) |
| `--deep-research` | off | Per-community member scan, additionally fills `Semantic_Description` |
| `--name-threshold=<N>` | `3` | Only communities with `Member_Count ≥ N` get a semantic name (smaller keep heuristic) |
| `--max-communities=<N>` | `60` | (deep-research only) Cap on deep-scanned communities (largest first); rest named from hints — **logged** |
| `--interactive` | off | Confirm the resolution choice by asking, instead of auto-pick |
| `--no-sync` | off | Skip the final rest-api sync/reload |

---

## Workflow (Phases A–H)

```
A Preflight ─▶ B Init(note) ─▶ C/D Sweep + Ranking ─▶ E Final run (winner)
            ─▶ F Naming (default) ─▶ G deep-research (opt.) ─▶ H Report + chat summary + Sync
```

### A. Preflight — read-only, abort clearly on anything missing

Resolve the DuckDB binary first, then verify (all read-only):

```bash
DUCKDB="$(command -v duckdb || true)"; [ -z "$DUCKDB" ] && [ -x "$HOME/.duckdb/cli/latest/duckdb" ] && DUCKDB="$HOME/.duckdb/cli/latest/duckdb"
[ -z "$DUCKDB" ] && [ -x /opt/homebrew/bin/duckdb ] && DUCKDB=/opt/homebrew/bin/duckdb
[ -z "$DUCKDB" ] && [ -x /usr/local/bin/duckdb ] && DUCKDB=/usr/local/bin/duckdb
echo "duckdb: ${DUCKDB:-NOT FOUND}"

# Master DB + graph populated? + the P5 graph views present?
"$DUCKDB" db/fm_catalog.duckdb -readonly -c "
  SELECT (SELECT COUNT(*) FROM ObjectCatalog)                                  AS objects,
         (SELECT COUNT(*) FROM ObjectLinks WHERE Link_Type='operational')      AS op_links,
         (SELECT COUNT(*) FROM duckdb_views() WHERE view_name='ClusterEdges')   AS cluster_edges_view,
         (SELECT COUNT(*) FROM duckdb_views() WHERE view_name='LogicalLinks')   AS logical_links_view;"

# Tooling present?
ls tools/graph-export/cluster.sh tools/graph-export/graph_export_logical.sql \
   tools/graph-export/cluster_load.sql tools/graph-export/sync_db.sh

# Engine deps (at least one). graphology = Louvain baseline; igraph = Leiden.
node -e "require('graphology'); require('graphology-communities-louvain')" 2>&1 && echo "louvain OK"
python3 -c "import igraph" 2>/dev/null && echo "leiden available" || echo "leiden NOT available (auto → louvain)"
```

**Abort conditions** (each with a concrete next step):
- No DuckDB → point to CLAUDE.md install section (never install yourself).
- `objects = 0` **or** `op_links = 0` → "Graph not built — run `convert-xml` / Graphify first."
- `cluster_edges_view = 0` **or** `logical_links_view = 0` → the DB was built **before** the
  graph views existed. → "Re-run `convert-xml --batch` (P5 creates `LogicalLinks`/`ClusterEdges`)."
  Both the cleaned edge export and this skill's hub/degree analysis read `ClusterEdges`; without it
  the run would silently fall back to the raw `ObjectLinks` graph and report inflated god-nodes.
- Tooling files missing → name the missing file.
- Neither graphology nor igraph importable → "run `npm install` in the repo root (build-time dep)."

The cluster tables (`ObjectClusters`/`CommunityNames`) do **not** need to pre-exist — Phase E
creates them. The size numbers (`n_nodes`/`n_edges`/`n_files`) come from the sweep (Phase C/D),
so no separate size query is needed here.

### B. Table init — note only (no DDL)

If `ObjectClusters`/`CommunityNames` are absent or empty, **no separate DDL step is needed** —
Phase E's `cluster_load.sql` builds both via `CREATE OR REPLACE` (incl. the `Semantic_Name` /
`Semantic_Description` columns, which are fixed in the schema). Just **note** whether this is a
first run (tables new) or a re-run (tables replaced).

### C/D. Sweep + ranking — `scripts/sweep.mjs` (export once, engine N×)

The sweep driver exports the cleaned logical edges **once**, runs the engine for each candidate
(no DB write load, no sync), computes the raw metrics + score, and prints a ranking JSON:

```bash
node .claude/skills/fm-graph-cluster/scripts/sweep.mjs \
  --db="$PWD/db/fm_catalog.duckdb" \
  --tools="$PWD/tools/graph-export" \
  --candidates=0.5,0.75,1.0,1.5,2.0,3.0 \
  --seed=42 --engine=auto
```

The JSON (`stdout`) carries: `engine`, `engine_reason`, `seed`, `q_available`, `size`
(`n_nodes`/`n_edges`/`n_files`), `band`, `weights`, `near_tie`, `timings_ms`, and
`candidates[]` (each with `resolution, K, Q, norm_Q, largest_share, singleton_share,
tiny_node_share, avg_size, median_size, band_penalty, score, rank`) plus `winner`/`runner_up`.

**Always show the full ranking table** in the chat (no silent culling). The scoring rubric:

```
score = w_Q·Q − w_big·max(0, largest_share−τ_big) − w_frag·max(0, singleton_share−τ_frag) − w_band·band_penalty
        w_Q=1.0  w_big=1.5  w_frag=1.0  w_band=0.5     τ_big=0.25  τ_frag=0.40
```
- **Q** (modularity, from engine stderr) is the primary signal. If `q_available=false` (un-patched
  engine), the Q term drops out and the distribution guardrails alone rank (conservative fallback).
- **band_penalty** encodes "fits the solution size": target mean module size 30–150 objects ⇒
  K band `[n_nodes/150, n_nodes/30]` with a soft floor at `n_files`. Out-of-band K is penalised by relative distance.

**Resolution mode (`--resolution=X`)**: skip the competition — run the sweep with `--candidates=X`
(single measurement). X is the winner by definition; you still get `size` + that resolution's
metrics for the report.

### Auto-pick vs. ask

**Default: auto-pick the top score** and always show the ranking table. Ask the user (via
`AskUserQuestion`) to confirm **only** when `near_tie=true` (top-2 score gap < ε) **or**
`--interactive` is set. Otherwise proceed with the winner.

### E. Final run with the winning resolution

Exactly **one** clean end-to-end run with the winner (`engine` and `seed` taken from the sweep
JSON so the partition matches what was measured). `NO_SYNC=1` — the sync happens once at the very
end (Phase H), so the Explorer never sees the un-named intermediate partition:

```bash
FMLAB_CLUSTER_RESOLUTION=<winner> FMLAB_CLUSTER_SEED=<seed> \
FMLAB_CLUSTER_ENGINE=<engine> FMLAB_CLUSTER_NO_SYNC=1 \
  bash tools/graph-export/cluster.sh
```

This (re-)creates `ObjectClusters` + `CommunityNames` with heuristic names; `Semantic_Name` /
`Semantic_Description` start NULL. Determinism: same seed+resolution ⇒ identical partition as the
sweep (the edge export is now sorted, so it is bit-identical across runs).

> ⚠ **Gotcha:** `cluster_load.sql` rebuilds `CommunityNames` via `CREATE OR REPLACE` — it
> **wipes any existing semantic names**. That is why naming (F/G) is the *last* write step, after E.
> There is no "name without reloading" path while E runs.

### F. Semantic naming (default mode)

> **Name cache:** `cluster.sh` restores `Semantic_Name`/
> `Semantic_Description` from `SemanticNameCache` via majority vote **before** this skill runs — so
> stable communities already carry their previous names. **Only name the dirty rest**
> (`Semantic_Name IS NULL`): genuinely new/split/merged modules. The cached names are the existing
> vocabulary — stay consistent with them. (To force a full re-name, clear names first:
> `UPDATE CommunityNames SET Semantic_Name=NULL, Semantic_Description=NULL;` — destroys cache reuse.)

Read the hint columns for **un-named** communities at/above the threshold (smaller ones keep their heuristic name):

```sql
SELECT Community, Engine, Member_Count, Dominant_Type, Dominant_File,
       Top_Member_Label, Sample_Labels, Heuristic_Name
FROM CommunityNames
WHERE Member_Count >= <name-threshold>
  AND Semantic_Name IS NULL          -- skip cache-restored communities (already named)
ORDER BY Member_Count DESC;
```

From `Dominant_File` (context), `Dominant_Type` (character), `Top_Member_Label` (anchor) and
`Sample_Labels` (word field), synthesise one concise `Semantic_Name` per community. **Name the
whole list in one pass** (you see all communities at once) so the vocabulary is coherent — no
duplicate/inconsistent module names. Example:
`Kunden · Eingangsdaten laden [Datum] (+141)` → **"Kunden-Import & Eingangsdaten-Aufbereitung"**.

Write all names in **one** bundled `duckdb` call via generated `UPDATE`s (never `CREATE OR
REPLACE` — that is destructive). **Escape single quotes** by doubling them (`'` → `''`):

```sql
UPDATE CommunityNames SET Semantic_Name = 'Kunden-Import & Eingangsdaten-Aufbereitung'
  WHERE Community = 7 AND Engine = 'louvain';
-- … one row per named community
```

Communities below the threshold stay `Semantic_Name = NULL` → the Explorer falls back via
`COALESCE(Semantic_Name, Heuristic_Name)`. **Log** how many were named vs. left heuristic.

### G. deep-research mode (optional, `--deep-research`)

Additionally look into the **real member objects** of each community to produce a `Semantic_Name`
**+ `Semantic_Description`** (1–2 sentences on what the module does).

**Bounds (mandatory — no silent caps):**
- **Which communities**: only the **largest `--max-communities`** (default 60) with
  `Member_Count ≥ --name-threshold` **and `Semantic_Name IS NULL`** (skip cache-restored, see F).
  **Log** how many were deep-scanned vs. cache-restored vs. only hint/heuristic-named.
- **Per community**: top-`K` members by **logical (cluster) degree** (default `K=12`); truncate
  plain-text steps/comments (≈ ≤40 steps / ≤2 KB text per community).

Top-members per community (degree-ranked). The degree is the **logical cluster degree** (from
`ClusterEdges` — the same cleaned graph the clustering ran on), **not** raw `ObjectLinks`: this
picks anchors by their weight in the actual partition, not by inflated multi-edge/un-hoisted counts:

The cluster node key is **(Object_UUID, File_Name)** (clones don't merge), so the degree and
both catalog joins are **file-aware** (`IS NOT DISTINCT FROM` keeps NULL-file synthetics):

```sql
WITH deg AS (
  SELECT id, file, COUNT(*) AS degree FROM (
    SELECT Source_UUID AS id, Source_File AS file FROM ClusterEdges
    UNION ALL SELECT Target_UUID, Target_File FROM ClusterEdges
  ) GROUP BY id, file
)
SELECT cl.Community, oc.Object_Type, oc.Object_Name, oc.File_Name,
       COALESCE(d.degree,0) AS degree
FROM ObjectClusters cl
JOIN ObjectCatalog oc
  ON oc.Object_UUID = cl.Object_UUID AND oc.File_Name IS NOT DISTINCT FROM cl.File_Name
LEFT JOIN deg d
  ON d.id = cl.Object_UUID AND d.file IS NOT DISTINCT FROM cl.File_Name
WHERE cl.Community = ? AND cl.Engine = ?
QUALIFY ROW_NUMBER() OVER (PARTITION BY cl.Community ORDER BY degree DESC) <= 12;
```

Further signals (all bounded) per community: object-type histogram, schema names (BaseTable/TO/
Field), layout names, script names **+ folder path** (`ScriptCatalog`), script comments, plain-text
steps (`DDR_ScriptSteps` when `Has_DDR_INFO='True'`, else `StepsForScripts.Step_Name`), variable
names (`VariablesCatalog`/`VariableUsages`). Write `Semantic_Name` + `Semantic_Description` bundled
via `UPDATE` exactly as in F.

> Parallelisation (large N): the per-community scan is embarrassingly parallel — you may fan it out
> over sub-agents in batches (each returns `{Community, Semantic_Name, Semantic_Description}`; the
> main run writes the `UPDATE`s bundled). Default is **sequential**; parallelise only on request / when budgeted.

### H. Report + chat summary + sync

#### H.1 Markdown report → `output/graph_cluster_report_<ts>.md`

Build the timestamp once: `TS=$(date +%Y-%m-%d_%H-%M-%S)`. Write an **analysis document** (not just
a run log) with the Write tool. Data source is the final, named state of
`CommunityNames`/`ObjectClusters` (read-only) plus the sweep JSON. Structure:

1. **Metadata & run protocol** — DB path, engine (+ `engine_reason`), seed, script versions
   (`@version` from `cluster.sh`/`cluster_load.sql`), solution size (`n_nodes`/`n_edges`/`n_files`),
   start/end/duration (total + per phase from `timings_ms`), **number of cluster attempts** (sweep
   candidates) + **full ranking table** + chosen resolution, **communities found**; semantically
   named vs. heuristic; **cache-restored vs. newly named** (node-reuse %, from `cluster.sh`'s
   cache-apply log); (deep-research: deep-scanned vs. skipped — no silent caps).
2. **Architecture findings** — from the cluster metrics: module count & size distribution (compact
   vs. fragmented), dominant files/types per module, **hubs/god-nodes** (highest **logical**-degree
   nodes — use the fixed query below, **not** raw `ObjectLinks`), **cross-file interweaving**,
   isolated vs. central clusters. → *What does this say about the solution's build?*
   **Do not list builtin primitives** (`and`/`or`/`Case`, `Object_Type='BuiltinFunction'`) as hubs —
   they are excluded from the clustered graph by design (`ClusterEdges`). If their raw prominence is
   worth a remark, mention them **separately** as "primitives excluded from clustering", never as
   geclusterte Hubs.
3. **Business-logic findings** — derive the **business domains** from the `Semantic_Name`/
   `Semantic_Description` (e.g. customer import, invoicing, rights/security, reporting). → *Which business functions, and how do they connect?*
4. **List of ALL communities with description** — table per community: `Semantic_Name` · Description
   · `Member_Count` · `Dominant_File` · `Dominant_Type`. Description = `Semantic_Description` where
   present (deep-research); otherwise a concise one-liner from the hints. The list is **complete in both modes**.
5. **Summary (at the file end)** — condensed key statements: size & modularity, main domains,
   notable hubs, architecture character (monolithic vs. modular, interwoven vs. cleanly separated), risks/hints.

**Hubs/god-nodes for section 2 — canonical logical-degree query (mandatory).** The degree is
counted on `ClusterEdges` (the exact graph the clustering ran on), so the hubs in the report are the
hubs in the partition — not the multi-edge/un-hoisted artefacts the raw `ObjectLinks` degree
produces (those over-report by up to ~77×). Bind `?` to the winning engine (`'louvain'`/`'leiden'`):

The degree and both joins are keyed **(uuid, file)** (clones are distinct nodes — same UUID in
two files counts twice, once per file copy), so a cloned hub does not inflate by summing across files:

```sql
WITH logical_degree AS (
  SELECT uuid, file, COUNT(*) AS degree
  FROM (SELECT Source_UUID AS uuid, Source_File AS file FROM ClusterEdges
        UNION ALL SELECT Target_UUID, Target_File FROM ClusterEdges)
  GROUP BY uuid, file
)
SELECT oc.Object_Type, oc.Object_Name, oc.File_Name, d.degree, cl.Community
FROM logical_degree d
JOIN ObjectCatalog oc
  ON oc.Object_UUID = d.uuid AND oc.File_Name IS NOT DISTINCT FROM d.file
LEFT JOIN ObjectClusters cl
  ON cl.Object_UUID = d.uuid AND cl.File_Name IS NOT DISTINCT FROM d.file AND cl.Engine = ?
ORDER BY d.degree DESC LIMIT 20;
```

Builtins never appear here (they are not in `ClusterEdges`). Note in the report protocol that
"degree = logical cluster degree (v2)" — older reports used the raw degree and are **not**
number-comparable.

> ⚠ **Same name ≠ same object — and same UUID ≠ same node.** The cluster node key is
> **(Object_UUID, File_Name)**: identical names are never merged across files, **and** a single UUID
> that is *cloned* into several files (e.g. a "Save a copy as…" template scaffold) is split into one
> distinct node per file — they never collapse into one cluster node. A name (or UUID) that recurs in
> the hub list or as `Top_Member_Label` of several communities is **separate per-file copies**, each
> clustered into its **own** file's community. This solution replicates utility/filter scripts per file
> (e.g. `Eingabe Filtern …` exists 34×, one per file, in 34 different communities, with **0**
> `calls_script` links between them). Do **not** read such a name as a single central hub that "clamps
> domains together" or where "changes propagate" — the opposite is true: it is **code duplication** (a
> maintenance/consistency risk). If the duplication is notable, report it as such — quantify with
> `COUNT(DISTINCT File_Name)` per name — never as a shared cross-file dependency.

Final, named community data for the report:

```sql
SELECT Community, COALESCE(Semantic_Name, Heuristic_Name) AS Name, Semantic_Description,
       Member_Count, Dominant_File, Dominant_Type
FROM CommunityNames ORDER BY Member_Count DESC;
```

#### H.2 Chat summary

Print the **summary** (= H.1 section 5) directly in chat and link the report as a clickable path:
`→ full report: [output/graph_cluster_report_<ts>.md](output/graph_cluster_report_<ts>.md)`.

#### H.3 Sync (final, once)

Unless `--no-sync`: publish master → rest-api copy + reload, exactly once, **after** naming, so the
Explorer sees the named partition in a single reload:

```bash
bash tools/graph-export/sync_db.sh "$PWD/db/fm_catalog.duckdb"
```

The report file lives in `output/` and is **not** part of the DB sync.

#### H.4 Persist the winning resolution → `.fmlab/cluster.json` (R1)

Write the sweep winner so the **Auto-P7 pipeline phase** (after a fresh/force rebuild) and the
**Rebuild-Communities button** (`POST /api/graph/recluster`) re-cluster at the **same** granularity —
otherwise a from-scratch build or a button click would silently fall back to the `cluster.sh`
defaults (`auto`/1.0/42) and invalidate every semantic/user name via a different partition. The file
freezes the granularity decision here, in the skill (the only place that explores it via the sweep):

```bash
# Values = the winning engine/resolution/seed (the same ones passed to Phase E) +
# the winner's modularity Q from the sweep JSON. updated_at = ISO-8601.
cat > "$PWD/.fmlab/cluster.json" <<JSON
{ "engine": "<engine>", "resolution": <winner>, "seed": <seed>, "modularity_q": <Q>, "updated_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)" }
JSON
```

This is plain config (no DB sync). Auto-P7 downgrades a stored `leiden` engine to `auto` when the
host lacks python3+igraph, so writing the real winning engine is safe even across hosts.

---

## Definition of done (verify)

- Preflight catches missing DB/tools/deps with a concrete next step, **and** aborts clearly if the
  `ClusterEdges`/`LogicalLinks` views are missing (DB built before the views → re-run `convert-xml --batch`).
- Sweep exports edges **once**, evaluates N candidates **without** N DB writes, prints a full ranking table.
- A justified resolution is chosen; `--resolution=` skips the competition correctly.
- Final run reproduces the same partition for the same seed+resolution.
- Report hubs use the **logical cluster degree** (`ClusterEdges`), so the top hubs are schema anchors /
  central scripts — **not** the inflated sort/portal artefacts; builtins (`and`/`or`/`Case`) never listed as hubs.
- deep-research top-members are ranked by logical cluster degree (not raw `ObjectLinks`).
- `Semantic_Name` filled for communities ≥ threshold via `UPDATE`; `COALESCE` display shows them, small ones keep heuristic.
- deep-research additionally fills `Semantic_Description`; deep-scanned vs. skipped counts are logged.
- Report exists at `output/graph_cluster_report_<ts>.md` with metadata, architecture **and**
  business-logic findings, **all** communities with description, and a closing summary.
- Chat shows the summary with a clickable link to the report.
- Sync publishes the **named** partition exactly once (unless `--no-sync`).
