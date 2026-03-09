# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Planned
- **ER agent — Requirements Discrepancy Report:** Compare enhancement request requirements against SA agent artifacts and templates to identify gaps, conflicts, and misalignments

---

## [0.7.0] - 2026-03-09

### Fixed
- **VS Code Windows compatibility** — all three agents (`srd`, `sa`, `er`)
  - Replaced `tools:` frontmatter values with correct VS Code built-in tool names:
    `readFile`, `createFile`, `editFiles`, `fileSearch`, `listDirectory`, `runInTerminal`, `getTerminalOutput`
    (previously used generic names `read`, `write`, `glob`, `bash` which are not valid VS Code tool identifiers)
  - Fixed `DIRECTORY CONTAINMENT` rule: blanket "NEVER use absolute paths" replaced with a qualified rule that permits workspace-rooted absolute paths when required by tooling (resolves VS Code state detection deadlock where `fileSearch`/`listDirectory` require absolute paths)
  - Fixed `GUARDRAILS` section with the same qualified wording

### Changed
- **`sa.agent.md`** — removed inline `convert_artifacts.py` Python script (345 lines)
  - Document conversion now uses `markitdown` CLI directly: `python -m markitdown [input] -o [output]`
  - Visio `.vsdx` conversion uses a slim inline Python one-liner via terminal instead of the full script
  - Phase 1 no longer creates `scripts\convert_artifacts.py`
  - File reduced from 516 lines to 179 lines

---

## [0.6.0] - 2026-03-09

### Added
- New `srd.agent.md` specification — Solution Requirements Designer agent
  - End-to-end requirements-to-backlog pipeline for organizing raw business requirements through to a Jira-ready implementation backlog
  - Multi-session architecture: file-based state detection determines which stage to run on each invocation — no user coordination required
  - **Stage A (Phases 1–5):** Document conversion via markitdown, prioritized requirements clarification loop, completeness scoring, As-Is/To-Be process models with Mermaid diagrams, system responsibility matrix
  - **Stage B (Phase 6):** Generates `tech-assessment.md` — a briefing + structured assessment form (feasibility table, risk/impact matrix, challenges checklist, LOE estimate, recommended approach) pre-populated with requirement IDs for the technical team to complete
  - **Stage C (Phases 7–13):** Ingests tech team feedback (edited markdown or chat input), functional solution design, integrations and data flows, stakeholder validation package, final document set, Jira-ready backlog
  - Prioritized clarification loop in Phase 2: contradictions first, then critical ambiguities, then gaps. Generates `clarification-questions.md` — a stakeholder-ready document the user can forward directly. User can `skip` individual questions or `defer-all` to exit the loop
  - Completeness scoring in Phase 3 across 7 categories (Functional, NFR, Data, Integration, Business Rules, Regulatory, UX). Always proceeds regardless of score; generates `completeness-report.md` with warning banner if < 80%
  - Completeness propagation: score and gap list carry forward into tech assessment, validation package, story risk flags, and SUMMARY — gaps remain visible throughout the entire workflow
  - Jira story files include `completeness_risk` and `technical_risk` frontmatter flags; stories linked to incomplete or risky requirements get inline warning notes
  - Smart gate model: 3 targeted smart gates (after completeness assessment, after tech feedback ingest, after stakeholder validation) + 2 session-ending stage boundaries. No auto/manual toggle needed
  - Stage A re-entry: invoking the agent again while in Stage A re-runs the clarification loop with new input and re-scores completeness before proceeding to Stage B
  - Self-contained: handles its own document conversion, reuses SA agent venv if present, no dependency on `sa` or `er` agents
  - All output under `analysis\SRD-YYYYMMDD-HHMM\`; maintains `analysis\README.md` status dashboard

### Changed
- `README.md` updated to document all three agents (`srd`, `sa`, `er`)
  - Added full `srd` agent section with stage/phase descriptions, outputs table, and gate model
  - Added "When to Use Which Agent" decision table
  - Updated Decision Gates section to cover both the `sa`/`er` toggle model and `srd` smart gate model
  - Updated Copy Agents section to include `srd` curl command
  - Updated folder structure diagram to include `analysis\` tree
  - Removed Roadmap section (SRD agent supersedes the planned requirements discrepancy report)

---

## [0.5.0] - 2026-03-06

### Added
- New `er.agent.md` — Enhancement Request agent (846 lines)
  - 6-phase workflow: setup, requirements, solution, governance, implementation, summary
  - Auto-generated Enhancement IDs in `ER-YYYYMMDD-HHMM` format
  - SA agent artifact detection at startup — cross-references `artifacts\` if present, standalone if not
  - Intelligent governance assessment: evaluates 6 sections independently based on content keywords (risk, impact, security, privacy, regulatory, accessibility)
  - Phase 2: processes typed input or source documents; reuses SA agent venv if available
  - Phase 3: complexity assessment (simple/moderate/complex) drives solution depth; conditional ADR creation
  - Phase 5: generates `implementation-plan.md`, individual task files with hour estimates, higher-level `runbook.md`
  - Auto-creates and maintains `enhancements\README.md` status dashboard
  - Generates `SUMMARY.md` at end of each enhancement
  - Owner fields default to TBD; never prompts user for owner during workflow
  - Task estimates in hours based on complexity (1–4h simple, 4–8h moderate, 8–16h complex)
  - Read-only cross-referencing of SA agent artifacts — never modifies `artifacts\`
  - Same auto/manual decision gate system as SA agent

---

## [0.4.0] - 2026-03-06

### Changed (`sa.agent.md`)
- Replaced `markitdown[all]` with targeted install flags: `markitdown[docx,xlsx,pptx,pdf]`
- Added conditional Visio support: scans for `.vsdx` files before installation; installs `vsdx` package in the same pip command only if Visio files are detected
- Added venv existence check — reuses existing `scripts\venv\` instead of recreating it every run
- Added broken venv detection with cleanup/abort prompt
- Fail fast on venv and dependency install errors — no fallback to system Python
- Added `CONFIGURATION` section: asks auto/manual gate preference once per session
- Added `DECISION GATES LOGIC` section with `[GATE]` checkpoints after phases 2–6
- Added `yes / no / skip-gates` options in manual gate prompts
- Added `convert_artifacts.py` pre-flight checks: Python version, venv detection, markitdown availability, root write access
- Added `convert_vsdx_file()`: extracts metadata, shapes with text, and connections/relationships with deduplication
- Updated `SUPPORTED_EXTENSIONS` to `{.txt, .csv, .xlsx, .docx, .pptx, .pdf, .md, .vsdx}` — removed image formats
- Split conversion pipeline: vsdx files via `vsdx` library, all other files via markitdown
- Individual file errors continue and log — only environment/setup failures stop execution
- Added progress indicators `[X/Y]` and ✓/✗ symbols to conversion output
- Updated GUARDRAILS to reflect new fail-fast and gate behaviors

### Removed
- Image file support (`.png`, `.jpg`, `.jpeg`, `.gif`) — removed from `SUPPORTED_EXTENSIONS`

---

## [0.3.0] - 2026-03-06

### Changed
- Phase 1 now prompts user to copy artifact folders into `artifacts\` after creating folder structure
- Agent waits for user to say "ready" after Phase 1
- Auto-discovery now scans `artifacts\` folder for documents

### Fixed
- Directory escape vulnerability — agent stays within current working directory

---

## [0.2.0] - 2026-03-06

### Changed
- Agent now auto-discovers documents — never waits for user input
- Added strict directory containment: agent never escapes root directory
- All file operations validated against root directory
- Removed user confirmation prompts — always proceeds through phases
- Convert script now includes path security validation

### Fixed
- Directory escape vulnerability — agent stays within current working directory
- Phase 2 now proactively scans instead of waiting

---

## [0.1.0] - 2026-03-06

### Added
- Initial release of sa-agent
- 7-phase Solution Architect workflow
- Inline `convert_artifacts.py` script
- Windows-specific guardrails (venv paths, no `&&` operator)
- Document conversion using markitdown
- Template verification and creation
- Content mapping and completeness analysis
- README.md and CHANGELOG.md auto-generation
