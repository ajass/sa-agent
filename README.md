# sa-agent

GitHub Copilot agents for VS Code — Solution Requirements Design, Solution Architecture, and Enhancement Request workflows.

## Agents

### `srd` — Solution Requirements Designer

End-to-end requirements-to-backlog pipeline. Organizes raw business requirements, identifies gaps, generates stakeholder-ready clarifying questions, produces a technical assessment handoff for the engineering team, and finalizes a complete enhancement documentation package with Jira-ready story files.

This is a **multi-session agent**. Each session picks up where the last left off using file-based state detection. Offline stakeholder and technical team work happens between sessions — no continuous session required.

**Quick Start:**
1. Copy `.github\agents\srd.agent.md` to your project's `.github\agents\` folder
2. Open VS Code → Copilot Chat → select `srd` from the agent dropdown
3. Place BRDs, meeting notes, or transcripts in the generated `source\` folder, or type requirements directly
4. The agent determines which stage to run based on what files already exist

**Sessions & Stages:**

| Session | Stage | What Happens | Ends When |
|---------|-------|--------------|-----------|
| 1 (or more) | **A — Requirements Analysis** | Convert docs, consolidate reqs, identify gaps, generate follow-up questions, model processes, map system responsibilities | You go offline to gather stakeholder answers |
| 1 | **B — Technical Handoff** | Generate briefing + structured assessment form for the technical team | You give the document to the tech team |
| 1 | **C — Final Documentation** | Ingest tech feedback, produce solution design, integrations, governance, technical specs, Jira backlog | Complete enhancement package delivered |

**Stage A — Requirements Analysis (Phases 1–5):**
- **Phase 1** — Generate session ID (`SRD-YYYYMMDD-HHMM`), create folder structure, detect/create Python venv, convert source documents (docx, pptx, xlsx, pdf, txt, md) to Markdown via markitdown
- **Phase 2** — Read all converted documents, extract and deduplicate requirements (tagged `REQ-NNN`), run prioritized clarification loop: contradictions → ambiguities → gaps. Generates `clarification-questions.md` — a stakeholder-ready question document the user can forward directly. Loop exits when clean or user types `skip` (individual) or `defer-all` (exit loop)
- **Phase 3** — Categorize requirements into 7 categories, score completeness per category, compute overall percentage. Always proceeds regardless of score. Generates `completeness-report.md` with warning banner if < 80%. **Smart gate:** `continue` or `add-more`
- **Phase 4** — Model business processes As-Is and To-Be with Mermaid diagrams + text descriptions, highlight deltas. Auto-chains to Phase 5
- **Phase 5** — Map requirements to responsible systems/components, define boundaries, generate responsibility matrix. **Session ends** — go ask stakeholders follow-up questions

**Stage B — Technical Handoff (Phase 6):**
- **Phase 6** — Generate `tech-assessment.md`: Part 1 is a requirements briefing for context; Part 2 is a structured assessment form pre-populated with requirement IDs for the tech team to complete (feasibility table, risk/impact matrix, technical challenges checklist, LOE estimate table, recommended approach, open questions). **Session ends** — hand document to tech team

**Stage C — Final Documentation (Phases 7–13):**
- **Phase 7** — Ingest tech team's completed assessment (from edited markdown file or chat input — agent detects which). **Smart gate:** `continue` or `corrections`
- **Phase 8** — Functional solution design incorporating tech team's recommended approach. Auto-chains
- **Phase 9** — Integrations and data flows with Mermaid sequence diagrams + text descriptions. Auto-chains
- **Phase 10** — Stakeholder validation package consolidating completeness score, feasibility summary, key decisions, and open items. **Smart gate:** `continue` or `revise`
- **Phase 11** — Final document set generated. Auto-chains
- **Phase 12** — Jira-ready story files with `completeness_risk` and `technical_risk` flags per story. Auto-chains
- **Phase 13** — Final summary displayed. **Session ends — done**

**Outputs per session (`analysis\SRD-YYYYMMDD-HHMM\`):**

| File | Stage | Description |
|------|-------|-------------|
| `source\` | A | Raw BRDs, meeting notes, transcripts |
| `source\converted\` | A | Markdown versions of binary documents |
| `requirements\consolidated-requirements.md` | A | All requirements, deduplicated, source-tagged (`REQ-NNN`) |
| `requirements\categorized-requirements.md` | A | Requirements organized into 7 categories with scores |
| `requirements\completeness-report.md` | A | Completeness score, per-category breakdown, warning if < 80% |
| `requirements\clarification-questions.md` | A | Stakeholder-ready question document (resolved + open) |
| `processes\process-models.md` | A | As-Is / To-Be Mermaid diagrams + text descriptions |
| `design\system-responsibilities.md` | A | Responsibility matrix (system × capability) |
| `tech-assessment.md` | B | Briefing + structured assessment form for tech team |
| `design\technical-feedback-summary.md` | C | Structured digest of tech team's completed assessment |
| `design\functional-design.md` | C | Functional solution design |
| `design\integrations-data-flows.md` | C | Integration points, data flows, Mermaid sequence diagrams |
| `validation-package.md` | C | Executive summary for stakeholder review |
| `final\requirements.md` | C | Polished final requirements |
| `final\solution.md` | C | Solution design (business + technical approach merged) |
| `final\governance.md` | C | Merged risk, impact, and compliance assessment |
| `final\implementation-plan.md` | C | Phases, effort estimates, timeline |
| `final\technical-specifications.md` | C | Tech appendix: feasibility results, risks, architecture changes |
| `backlog\story-NNN.md` | C | Jira-ready story files with completeness and technical risk flags |
| `backlog\backlog-summary.md` | C | Story index with dependency map and suggested sprint groupings |
| `SUMMARY.md` | C | Full session summary across all stages |

**`analysis\README.md`** is created and maintained as a status dashboard across all SRD sessions.

**Completeness propagation:** The score and gap list from Phase 3 carry forward into the tech assessment (Part 1), validation package, story risk flags, and final SUMMARY — gaps remain visible throughout the entire workflow.

**Stage A re-entry:** Invoke the agent again after gathering stakeholder answers. It detects you are still in Stage A, accepts new input, re-runs the Phase 2 clarification loop, re-categorizes, and re-scores before prompting for Stage B.

**Gate model:** No auto/manual toggle. Three targeted smart gates + two session-ending stage boundaries. All other phase transitions auto-chain with a status line.

---

### `sa` — Solution Architect

Full-lifecycle solution architecture agent. Converts source documents, builds artifact templates, maps content, and generates a completeness report.

**Quick Start:**
1. Copy `.github\agents\sa.agent.md` to your project's `.github\agents\` folder
2. Open VS Code → Copilot Chat → select `sa` from the agent dropdown
3. Answer the gate configuration prompt (`auto` or `manual`)
4. Follow Phase 1 setup, then the agent proceeds through all phases

**Phases:**
- **Phase 1** — Create folder structure, prompt to copy artifacts
- **Phase 2** — Scan and collect source documents from `artifacts\` and `documents\source\`
- **Phase 3** — Pre-flight checks, venv setup, install dependencies, convert documents to Markdown
- **Phase 4** — Verify and create artifact templates
- **Phase 5** — Map content to templates, generate completeness report
- **Phase 6** — Update README.md and CHANGELOG.md
- **Phase 7** — Summary and next steps

**Supported file types:** `.txt`, `.csv`, `.xlsx`, `.docx`, `.pptx`, `.pdf`, `.md`, `.vsdx` (Visio — conditional)

**Dependencies installed automatically:**
- `markitdown[docx,xlsx,pptx,pdf]` — core document conversion
- `vsdx` — only installed if `.vsdx` files are detected in `documents\source\`

---

### `er` — Enhancement Request

Streamlined workflow for a specific, well-defined enhancement request. Captures business requirements, defines a solution path, assesses governance, and produces an actionable implementation plan with task files.

**Quick Start:**
1. Copy `.github\agents\er.agent.md` to your project's `.github\agents\` folder
2. Open VS Code → Copilot Chat → select `er` from the agent dropdown
3. Answer the gate configuration prompt (`auto` or `manual`)
4. Provide your enhancement description or place source documents in the generated folder

**Phases:**
- **Phase 1** — Auto-generate Enhancement ID (`ER-YYYYMMDD-HHMM`), create folder structure, prompt for input
- **Phase 2** — Collect and structure business requirements, cross-reference SA artifacts if present
- **Phase 3** — Define high-level solution, assess complexity, create ADR if needed
- **Phase 4** — Intelligent governance assessment (risk, impact, security, privacy, regulatory, accessibility)
- **Phase 5** — Generate implementation plan, task files with hour estimates, runbook, update index
- **Phase 6** — Final summary

**Outputs per enhancement (`enhancements\ER-YYYYMMDD-HHMM\`):**

| File | Description |
|------|-------------|
| `requirements.md` | Business need, user stories, success criteria |
| `solution.md` | High-level approach, decisions, dependencies |
| `governance.md` | Risk, impact, and compliance assessment |
| `implementation-plan.md` | Phases, effort estimates, timeline |
| `runbook.md` | Higher-level implementation guidance |
| `tasks\task-NNN.md` | Individual task files with hour estimates |
| `SUMMARY.md` | Auto-generated summary of all artifacts |

**`enhancements\README.md`** is created and maintained as a status dashboard across all enhancement requests.

**SA Agent Integration:** If `artifacts\` is present from an SA agent run, the ER agent will automatically cross-reference architecture docs, requirements, and ADRs. It never modifies SA agent files — read-only.

---

## When to Use Which Agent

| Situation | Use |
|-----------|-----|
| You have raw docs and need structured project scaffolding + architecture templates | `sa` |
| You have scattered BRDs and meeting notes, need to validate requirements, get a tech feasibility assessment, and produce a full enhancement package with backlog | `srd` |
| You have a specific, well-defined enhancement and need to capture it quickly with governance, tasks, and a runbook | `er` |

The `srd` agent is fully self-contained — it handles its own document conversion and does not require `sa` or `er` to have been run first.

---

## Decision Gates

**`sa` and `er`** share the same gate system. At startup you are asked once:
- **auto** — only pause in Phase 1, proceed automatically through remaining phases
- **manual** — pause after each phase with `yes / no / skip-gates` options

Configuration is per-session and does not persist.

**`srd`** uses a targeted smart gate model — no toggle required:
- **Smart gates** (always prompt with options): after completeness assessment (Phase 3), after tech feedback ingest (Phase 7), after stakeholder validation (Phase 10)
- **Stage boundaries** (session ends, new invocation required): end of Stage A (Phase 5), end of Stage B (Phase 6)
- **Auto-chain** (brief status line only): Phases 4→5, 8→9, 9→10, 11→12, 12→13

---

## Requirements

- Windows (PowerShell)
- VS Code with GitHub Copilot
- Python 3.8+

---

## Copy Agents to a New Project

```powershell
# Create agents folder if it doesn't exist
mkdir .github\agents

# SRD Agent
curl -o .github\agents\srd.agent.md https://raw.githubusercontent.com/ajass/sa-agent/master/.github/agents/srd.agent.md

# SA Agent
curl -o .github\agents\sa.agent.md https://raw.githubusercontent.com/ajass/sa-agent/master/.github/agents/sa.agent.md

# ER Agent
curl -o .github\agents\er.agent.md https://raw.githubusercontent.com/ajass/sa-agent/master/.github/agents/er.agent.md
```

Or download directly from [GitHub](https://github.com/ajass/sa-agent).

---

## Folder Structure

```
project-root\
├── .github\
│   └── agents\
│       ├── srd.agent.md         # Solution Requirements Designer agent
│       ├── sa.agent.md          # Solution Architect agent
│       └── er.agent.md          # Enhancement Request agent
├── analysis\                    # SRD agent — requirements analysis sessions
│   ├── README.md                # Status dashboard
│   └── SRD-YYYYMMDD-HHMM\
│       ├── source\              # Raw input documents
│       │   └── converted\       # Markitdown-converted markdown
│       ├── requirements\        # Consolidated, categorized, completeness report, clarification questions
│       ├── processes\           # As-Is / To-Be process models
│       ├── design\              # System responsibilities, functional design, integrations, tech feedback
│       ├── tech-assessment.md   # Technical team handoff document
│       ├── validation-package.md
│       ├── final\               # Final enhancement document set
│       ├── backlog\             # Jira-ready story files
│       └── SUMMARY.md
├── artifacts\                   # SA agent — strategic architecture artifacts
│   ├── requirements\
│   ├── architecture\
│   ├── diagrams\
│   ├── adr\
│   └── discovered\
├── documents\                   # SA agent — source and converted documents
│   ├── source\
│   └── processed\
├── enhancements\                # ER agent — enhancement requests
│   ├── README.md                # Status dashboard
│   └── ER-YYYYMMDD-HHMM\
│       ├── requirements.md
│       ├── solution.md
│       ├── governance.md
│       ├── implementation-plan.md
│       ├── runbook.md
│       ├── SUMMARY.md
│       ├── tasks\
│       └── source\
└── scripts\                     # Shared — Python venv
    └── venv\
```
