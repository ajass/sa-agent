# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Changed (`er.agent.md`)
- **Phase 1 rewrite** — replaced "prompt user to specify document locations" with SRD-style intake: agent creates `source\` folder upfront, tells user to place documents there and say "ready" (or type requirements directly in chat), then waits for input
- **Dynamic dependency detection** — scans `source\` for file extensions after user says "ready", builds `pip install` command with only the markitdown extras actually needed (e.g. `markitdown[docx,pdf]` if only `.docx` and `.pdf` files are present)
- **Conditional venv** — skips venv creation entirely when only text-native files (`.md`, `.txt`) are present; installs base `markitdown` (no extras) for formats like `.html`, `.csv`, `.json`, `.xml` that don't require extras
- **Folder rename** — `input\` → `source\`, `input\converted\` → `source\converted\` to match SRD agent convention
- **Expanded supported formats** — added `.html`, `.htm`, `.json`, `.xml`, `.xls` to the supported file list
- **Unsupported format handling** — images (`.png`, `.jpg`, `.jpeg`, `.gif`), audio (`.mp3`, `.wav`), archives (`.zip`), and `.epub` are skipped with a warning (images require OCR plugin + external API key; audio requires speech recognition)
- **Venv handling** — added broken-venv detection with cleanup/abort prompt, fail-fast on venv creation or markitdown install failure (adapted from SRD agent)
- **Conversion approach** — replaced inline Python `MarkItDown` class usage with CLI `python -m markitdown` one-file-at-a-time conversion; per-file errors logged to `source\converted\conversion-errors.md` without stopping
- **Removed** `/er <path-to-documents>` invocation — source folder is now created by the agent; no external path needed
- Version bumped to v1.1

### Changed (`README.md`)
- Updated ER agent Quick Start to describe new source-folder workflow
- Updated Phase 1 description: source folder prompt, dynamic dependency detection, conditional venv
- Added "Supported file types" line listing all accepted formats
- Added `source\converted\` to outputs table
- Updated folder structure diagram: `input\` → `source\`

---

## [0.9.0] - 2026-03-12

### Changed (`er.agent.md`)
- Reverted planned 4-phase business-sign-off rebuild — retained the original **3-phase workflow**: Document Intake & Conversion → Clarification & Assessment → Technical Assessment & Final Documentation
- ER agent remains focused on requirements analysis and technical feasibility rather than business sign-off

### Changed (`README.md`)
- Updated `er` agent section to accurately document the 3-phase workflow and actual output files
- Updated "When to Use Which Agent" table description for `er` to reflect technical feasibility focus
- Updated Decision Gates section: `er` uses a completeness gate after Phase 2 (3 options) and optional tech review in Phase 3
- Updated Folder Structure diagram: merged ER sessions into `analysis\` tree alongside SRD sessions

---

## [0.8.0] - 2026-03-09

### Added
- **`srd.agent.md` — `defer-assessment` path:** New option at the Stage A→B transition prompt that allows the user to walk through the technical assessment interactively and continue all the way to final output in a single session, without waiting for a tech team to complete the form offline
  - **Phase 6B — Interactive Assessment Loop:** Agent presents each of the 8 Part 2 sections (2.1–2.8) one at a time. User can respond with `answer [text]` to provide input, `skip` to defer an individual section, or `defer-all` to defer all remaining sections and proceed immediately. Answered sections are written to `tech-assessment.md` in real time; deferred sections are annotated `**Status: Deferred**`. Auto-chains to Phase 7 with no session break.
  - **Deferred section tracking:** Agent maintains a `deferred_sections[]` list and a `session_mode` flag (`defer-assessment`) carried through all subsequent phases
  - **Phase 7.1 update:** Sections marked `**Status: Deferred**` are treated as intentionally skipped — no `file-ready / chat-input` prompt is triggered for those sections
  - **Phase 7.2 update:** `technical-feedback-summary.md` includes a **Deferred Assessment Sections** block when any sections were skipped, listing each deferred section and what it captures
  - **Functional Architect Handoff (Phase 13.1):** `SUMMARY.md` includes a new `## Functional Architect Handoff` section (only on `defer-assessment` path) with four actionable to-do checklists: Business Clarification Required, Technical Feasibility Required, Assessment Sections to Complete, and High-Risk Stories Requiring Attention — framed as a functional architect handing off to a solution architect
  - **Phase 13.3 update:** Final terminal display appends a `Functional Architect Handoff` status block with counts of open items and a pointer to the handoff to-do list in `SUMMARY.md`
  - **Status fields:** `SUMMARY.md` status = `Complete - Pending SA Review` and `analysis\README.md` dashboard status = `Pending SA Review` when `defer-assessment` path was used (vs `Complete` on the standard path)
  - **GUARDRAILS updated:** Two new rules enforce that `defer-assessment` never ends at Stage B and that the Handoff sections are only generated on the `defer-assessment` path

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
