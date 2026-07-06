# Components
 
The repo is organized into separate sections for different parts and tasks within the overall workflow.

```
fm-lab/
├── .claude/                    Claude Code configuration (skills, settings)
├── .fmlab/                     FM-Lab configuration (plugins, settings)
├── .git/                       Git repository metadata
├── apps/                       Frontend / application code
├── db/                         DuckDB databases (master: fm_catalog.duckdb)
├── docs/                       Project documentation
├── logs/                       Log files
├── packages/                   Shared packages / modules
├── rest-api/                   REST API server with its own read-only DB copy
├── scripts/                    Reserved for generation of new scripts (output)
├── sql/                        SQL templates (convert_xml.sql, …)
├── tools/                      Developer tools / CLI utilities
├── xml/                        FileMaker XML exports (input data)
│
├── .gitignore                  Git ignore rules
├── Banner.jpg                  Repo banner
├── CHANGELOG.md                Version history
├── CLAUDE.md                   Project instructions for Claude
├── LICENSE                     License
├── README.md                   Project overview
└── package.json                Node.js workspace configuration
```


### XML (Input)
`xml/` — FileMaker XML exports (SaXML) prepared for conversion from your solution.

The folder can include multiple files for one solution. Currently, only one FileMaker solution can be processed at a time.

### SQL Templates
`sql/` — Conversion templates and parser templates for universal catalogs.

This is the main ingestion logic and is executed by the DuckDB CLI, which must be installed beforehand.

### DuckDB Catalog
`db/fm_catalog.duckdb` — The generated DuckDB database containing the extracted FileMaker objects and their relationships.

It is populated during XML conversion.

### REST API
- `rest-api/` — Express server for HTTP access to the analysis database.
- `rest-api/db/fm_catalog.duckdb` — DuckDB database for exclusive REST API access.
- `rest-api/db/fm_reference.duckdb` — DuckDB database with reference information about FileMaker script steps and functions.
- `rest-api/templates/sql/` — SQL templates for standard queries on API endpoints.
- `rest-api/templates/sql-custom/` — additional SQL templates for your custom use cases.

### Web Client
`apps/web/` — React/Vite frontend

### Tools
- `tools/` — Utility scripts for various tasks.
- `tools/init.sh` — Initializes the project on first run by installing npm packages, paths and default settings. Includes a preflight check for dependencies and expected versions.
- `tools/convert_fm_xml.sh` — Runs the XML batch conversion and accepts CLI options.
- `tools/start-servers.sh` — Start the included HTTP servers.
- `tools/stop-servers.sh` — Stop the included HTTP servers.

### Docs
- `docs/` — Documentation files for FileMaker Pro and MBS plugin functions, installable via Claude Skills.
- `docs/fm-lab/` — Location of this documentation.
- `docs/claris-help/` — Official documentation files for FileMaker Pro, installable on demand in one or more local languages.
- `docs/mbs/` — Official documentation files for MBS plugin functions, installable on demand.
- `docs/.../` — Optional documentation files, installable on demand.

It is highly recommended to install the basic documentation set. It provides inline help for the Web Client and grounded reference material for agentic workflows. Some documentation packages include their own database for fast index queries.

### Claude Skills
`.claude/skills/` contains Claude Code skills and slash commands for installation, conversion, lookup and analysis.

**Setup**
- `.claude/skills/install-claris-docs` — Install Claris FileMaker documentation.
- `.claude/skills/install-mbs-docs` — Install MBS plugin documentation.

**Optional tools**
- `.claude/skills/install-ooe-fm` — Installs OOE references as a test suite for the XML converter. This is completely optional and not used elsewhere in the project.
- `.claude/skills/install-fm-xml-export-exploder` — Installs XML Export Exploder for reference and local tests. This is completely optional and not used elsewhere in the project.
- `.claude/skills/skill-creator` — Helps you build your own skills to extend the agentic workflow.

**XML conversion**
- `.claude/skills/convert-fm-xml` — Runs the XML conversion with checks and options.
- `.claude/skills/test-convert-xml` — Runs a test conversion against OOE references.

**Agentic analysis**
- `.claude/skills/fm-summarize` — Creates a concise technical briefing for a given object.
- `.claude/skills/fm-analyze` — Runs a deeper object analysis using semantic signals and recursive graph traversal up to five levels deep. It gathers context about dependencies, structure, logic, technical rules and semantic meaning, helping the agent explain functionality and business rules within the solution.

**Lookup documentation and explain features**
- `.claude/skills/filemaker-function-reference` — Looks up Claris FileMaker documentation.
- `.claude/skills/mbs-function-reference` — Looks up MBS plugin documentation.

### Plugin registry
`.fmlab/` — Registry and preferences for FM-Lab plugins.

### Scripts (Output)
`scripts/` — Reserved for generated FileMaker scripts produced by agentic coding workflows.



---

## Local servers

### REST API
Provides a local HTTP server at `http://localhost:3003`

Manual start:
For custom setups — e.g. running the REST API as a standalone service.
```bash
cd rest-api
cp .env.example .env   # adjust ports if needed
npm run dev
```

### Web Client
Provides a local HTTP server at `http://localhost:5173`

Manual start:
```bash
cd apps/web
cp .env.example .env   # adjust VITE_API_URL if API runs on a different port
npm run dev
```


**Automatic startup**
Both servers are started and stopped with the corresponding scripts:
`tools/start-servers.sh`
`tools/stop-servers.sh`

**Important**
npm dependencies and shared packages must be set up in advance by the init script:
`tools/init.sh`
