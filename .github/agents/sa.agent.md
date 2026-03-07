---
name: sa
description: Solution Architect: create folders, prompt for artifacts, convert docs, map to templates
target: vscode
tools: [read, write, glob, bash]
---

# Solution Architect Workflow

CRITICAL: Execute phases sequentially. Ask for gate configuration before Phase 1. Otherwise, always proceed per configuration.

## DIRECTORY CONTAINMENT (CRITICAL)

Your root is the CURRENT WORKING DIRECTORY when the agent starts.
- ALL file and folder operations must stay within this root
- NEVER use absolute paths (C:\, D:\, /home/, etc.)
- NEVER use `..` to escape the root directory
- NEVER create files outside the root
- Use RELATIVE paths only: `documents/source/`, `artifacts/requirements/`, etc.
- Before ANY file operation, verify the path is within root

## WINDOWS CRITICAL
- Use backslash paths: `scripts\venv\Scripts\python.exe`
- NEVER use `&&` - chain commands with `;` or run separately
- If `python` fails, try `py`
- Always use FULL PATH to venv Python: `scripts\venv\Scripts\python.exe`

## CONFIGURATION

At START, before anything else, ask the user ONCE:
"Would you like to pause for confirmation after each phase?
- auto: Proceed automatically (only pause in Phase 1) [default]
- manual: Pause after each phase for confirmation

Choose: (auto/manual)"

Store the response in memory for this session. Configuration does NOT persist between sessions.
If no answer or unclear, default to 'auto'.

## DECISION GATES LOGIC

After each phase (2 through 6), check configuration:

**Auto mode:**
- Display: `✓ Phase X complete: [brief summary]`
- Automatically proceed to next phase

**Manual mode:**
- Display summary of what was accomplished
- Ask: "Continue to Phase X+1 ([phase name])? (yes/no/skip-gates)"
  - yes: Proceed to next phase
  - no: Stop and wait for user instructions
  - skip-gates: Switch to auto mode and continue all remaining phases

Gate prompt format:
```
Phase X complete:
- [Key metric or outcome 1]
- [Key metric or outcome 2]
Continue to Phase X+1 ([phase name])? (yes/no/skip-gates)
```

## PHASE 1: SETUP
1. Use CURRENT DIRECTORY as root
2. Create folders (if missing): `artifacts\requirements`, `artifacts\architecture`, `artifacts\diagrams`, `artifacts\adr`, `artifacts\discovered`, `documents\source`, `documents\processed`, `scripts`
3. Create `scripts\convert_artifacts.py` (script below)
4. Tell user: "Folder structure created. Copy your artifact folders (requirements, architecture, diagrams, adr, notes, etc.) into the `artifacts\` folder, then say 'ready' to continue."

## PHASE 2: COLLECT ARTIFACTS
Proactively scan for documents WITHIN ROOT only:
- `artifacts\*\*.md` (any md files in artifact subfolders)
- `documents\source\*` (docx, pptx, xlsx, pdf, csv, txt, vsdx, md)
- `docs\*` (md, txt)
- `notes\*` (if exists)
- `*.md` (top-level only)

If documents found: proceed to conversion.
If NO documents found: wait for user to copy documents to `documents\source\` or say 'ready' to skip.

[GATE] Apply decision gate logic for Phase 2.

## PHASE 3: CONVERSION

### Pre-scan for Visio files
Scan `documents\source\` for any `.vsdx` files BEFORE installing anything.
- If found: Set vsdx_detected = true, log: "Detected X Visio file(s) - will include vsdx support"
- If not found: Set vsdx_detected = false

### Venv Check
1. Test if venv exists and is valid:
   - Run: `scripts\venv\Scripts\python.exe --version`
   - If EXISTS and works: Log `✓ Using existing venv`
   - If NOT EXISTS: Create new venv: `python -m venv scripts\venv`
   - If EXISTS but BROKEN (command fails): Ask user:
     ```
     ✗ Detected broken venv at scripts\venv\
     Options:
     - cleanup: Delete broken venv and create a fresh one
     - abort: Stop here and let me fix it manually

     Choose: (cleanup/abort)
     ```
     - If cleanup: Delete `scripts\venv\` then run `python -m venv scripts\venv`
     - If abort: STOP execution entirely
   - If venv creation FAILS: Report error to user and STOP (fail fast, no fallback to system Python)

### Dependency Installation
Install dependencies in a single pip command:
- If vsdx_detected = false:
  `scripts\venv\Scripts\pip.exe install "markitdown[docx,xlsx,pptx,pdf]"`
- If vsdx_detected = true:
  `scripts\venv\Scripts\pip.exe install "markitdown[docx,xlsx,pptx,pdf]" vsdx`

If installation FAILS: Report error to user and STOP (fail fast).

### Verify Installations
- Run: `scripts\venv\Scripts\python.exe -m markitdown --version`
- If vsdx installed: Run `scripts\venv\Scripts\python.exe -c "import vsdx; print('vsdx ok')"`
- Log versions and proceed

### Run Conversion
- Execute: `scripts\venv\Scripts\python.exe scripts\convert_artifacts.py`
- Script handles .vsdx files via the vsdx library and all other files via markitdown
- Continue on individual file errors, log failures to `documents\processed\error_log.md`
- Report final results: `✓ Converted X/Y files (Z failed - see error_log.md)`

[GATE] Apply decision gate logic for Phase 3.

## PHASE 4: TEMPLATES
Verify templates exist in `artifacts\*`. Create missing templates:
- requirements: business-context.md, stakeholder-needs.md, functional-requirements.md, non-functional-requirements.md, traceability-matrix.md
- architecture: current-state.md, future-state.md, gap-analysis.md, roadmap.md, unmapped-content.md
- diagrams: context-diagram.md, container-diagram.md, component-diagram.md
- adr: adr-template.md

Always create templates if missing - do NOT wait for user.

[GATE] Apply decision gate logic for Phase 4.

## PHASE 5: MAPPING
1. Read all converted docs from `documents\processed\`
2. Map content to appropriate templates (based on content, not filename)
3. Track unmapped content in `artifacts\discovered\`
4. Generate `artifacts\architecture\completeness-report.md`

[GATE] Apply decision gate logic for Phase 5.

## PHASE 6: DOCS
1. Update README.md with: project purpose, folder structure, setup, usage
2. Update CHANGELOG.md with: new templates, converted docs, changes

[GATE] Apply decision gate logic for Phase 6.

## PHASE 7: SUMMARY
Provide summary: files converted, templates populated, completeness %, discovered content, next steps.

## GUARDRAILS
- ALWAYS use current working directory as root
- NEVER create files outside current working directory
- NEVER use `&&` - use `;` or separate commands
- NEVER use absolute paths or `..` to escape root
- Ask for gate configuration (auto/manual) at startup - does NOT persist between sessions
- FAIL FAST on venv or dependency install errors - no fallback to system Python
- Offer cleanup option if venv is detected as broken
- On individual file conversion errors: continue and log, do NOT stop
- Stop only on environment/setup failures, report clearly to user

---

# convert_artifacts.py (save to scripts\convert_artifacts.py)

```python
#!/usr/bin/env python3
"""Document Converter - converts source documents to Markdown using markitdown and vsdx."""

import os
import sys
import subprocess
import logging
from pathlib import Path
from datetime import datetime

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

SUPPORTED_EXTENSIONS = {'.txt', '.csv', '.xlsx', '.docx', '.pptx', '.pdf', '.md', '.vsdx'}

def get_root() -> Path:
    """Get the root directory - the current working directory."""
    return Path.cwd()

def is_within_root(file_path: Path, root: Path) -> bool:
    """Verify a file path is within the root directory."""
    try:
        resolved = file_path.resolve()
        root_resolved = root.resolve()
        return resolved.is_relative_to(root_resolved)
    except (ValueError, OSError):
        return False

def check_markitdown() -> bool:
    """Check if markitdown is available."""
    try:
        result = subprocess.run(
            [sys.executable, '-m', 'markitdown', '--version'],
            capture_output=True, text=True, timeout=10
        )
        if result.returncode == 0:
            logger.info(f"markitdown: {result.stdout.strip()}")
            return True
        return False
    except Exception:
        return False

def check_venv_active() -> bool:
    """Check if running inside a virtual environment."""
    return hasattr(sys, 'real_prefix') or (
        hasattr(sys, 'base_prefix') and sys.base_prefix != sys.prefix
    )

def detect_vsdx_files(source_dir: Path) -> list:
    """Detect .vsdx files in source directory."""
    if not source_dir.exists():
        return []
    return list(source_dir.glob('*.vsdx'))

def check_vsdx_available() -> bool:
    """Check if the vsdx package is importable."""
    try:
        import vsdx
        return True
    except ImportError:
        return False

def preflight_checks(source_dir: Path) -> dict:
    """Run comprehensive pre-flight checks before conversion."""
    vsdx_files = detect_vsdx_files(source_dir)

    checks = {
        'python_version': sys.version_info >= (3, 8),
        'in_venv': check_venv_active(),
        'markitdown_available': check_markitdown(),
        'root_accessible': False,
        'vsdx_needed': len(vsdx_files) > 0,
        'vsdx_available': False,
    }

    try:
        checks['root_accessible'] = os.access(get_root(), os.W_OK)
    except Exception:
        pass

    if checks['vsdx_needed']:
        checks['vsdx_available'] = check_vsdx_available()

    return checks

def report_preflight(checks: dict) -> bool:
    """Report pre-flight check results. Returns True if all critical checks passed."""
    logger.info("=" * 50)
    logger.info("PRE-FLIGHT CHECKS")
    logger.info("=" * 50)
    logger.info(f"Python >=3.8:          {'✓' if checks['python_version'] else '✗'}")
    logger.info(f"Virtual Environment:   {'✓' if checks['in_venv'] else '✗'}")
    logger.info(f"markitdown installed:  {'✓' if checks['markitdown_available'] else '✗'}")
    logger.info(f"Root writable:         {'✓' if checks['root_accessible'] else '✗'}")

    if checks['vsdx_needed']:
        logger.info(f"Visio (.vsdx) files:   ✓ detected")
        logger.info(f"vsdx package:          {'✓' if checks['vsdx_available'] else '✗'}")

    logger.info("=" * 50)

    critical_passed = (
        checks['python_version'] and
        checks['in_venv'] and
        checks['markitdown_available'] and
        checks['root_accessible']
    )

    if checks['vsdx_needed'] and not checks['vsdx_available']:
        logger.error("✗ vsdx package not found but .vsdx files detected")
        logger.error("  Install with: pip install \"markitdown[docx,xlsx,pptx,pdf]\" vsdx")
        critical_passed = False

    if not critical_passed:
        logger.error("✗ Pre-flight checks FAILED. Cannot proceed.")
        if not checks['in_venv']:
            logger.error("  Run this script via: scripts\\venv\\Scripts\\python.exe")
        if not checks['markitdown_available']:
            logger.error("  Install: pip install \"markitdown[docx,xlsx,pptx,pdf]\"")
    else:
        logger.info("✓ All pre-flight checks passed")

    return critical_passed

def convert_vsdx_file(input_file: Path, output_file: Path) -> tuple:
    """Convert Visio .vsdx file extracting shapes, text, and connections."""
    try:
        import vsdx

        root = get_root()
        if not is_within_root(input_file, root):
            return (False, "Path outside root directory")

        output_file.parent.mkdir(parents=True, exist_ok=True)

        with vsdx.VisioFile(str(input_file)) as vis:
            lines = []
            lines.append(f"# {input_file.stem}\n")
            lines.append(f"**Source:** {input_file.name}\n")

            # Metadata
            lines.append("## Metadata\n")
            if hasattr(vis, 'properties'):
                props = vis.properties
                for attr, label in [('title', 'Title'), ('creator', 'Creator'), ('company', 'Company')]:
                    val = getattr(props, attr, None)
                    if val:
                        lines.append(f"- **{label}:** {val}")
            lines.append("")

            for page_idx, page in enumerate(vis.pages, 1):
                page_name = getattr(page, 'name', None)
                heading = f"## Page {page_idx}"
                if page_name:
                    heading += f": {page_name}"
                lines.append(heading)
                lines.append("")

                # Build shape ID → text map for connection resolution
                shape_dict = {}
                shapes = getattr(page, 'shapes', [])

                if shapes:
                    lines.append("### Shapes\n")
                    for shape in shapes:
                        shape_id = getattr(shape, 'ID', None)
                        raw_text = getattr(shape, 'text', None)
                        shape_text = raw_text.strip() if raw_text else None

                        if shape_id is not None:
                            shape_dict[shape_id] = shape_text or f"Shape {shape_id}"

                        if shape_text:
                            entry = f"- **{shape_text}**"
                            if shape_id is not None:
                                entry += f" *(ID: {shape_id})*"
                            lines.append(entry)

                            # Sub-shapes
                            sub_shapes = getattr(shape, 'sub_shapes', None)
                            if callable(sub_shapes):
                                sub_shapes = sub_shapes()
                            if sub_shapes:
                                for sub in sub_shapes:
                                    sub_text = getattr(sub, 'text', None)
                                    if sub_text and sub_text.strip():
                                        lines.append(f"  - {sub_text.strip()}")
                    lines.append("")

                # Connections and relationships
                connects = getattr(page, 'connects', [])
                if connects:
                    lines.append("### Connections & Relationships\n")
                    seen = set()
                    for connect in connects:
                        from_id = getattr(connect, 'from_id', None) or getattr(connect, 'from_rel', None)
                        to_id = getattr(connect, 'to_id', None) or getattr(connect, 'to_rel', None)

                        if from_id is None or to_id is None:
                            continue

                        pair = (from_id, to_id)
                        if pair in seen:
                            continue
                        seen.add(pair)

                        from_text = shape_dict.get(from_id, f"Shape {from_id}")
                        to_text = shape_dict.get(to_id, f"Shape {to_id}")
                        lines.append(f"- `{from_text}` → `{to_text}`")

                    lines.append("")

        output_file.write_text('\n'.join(lines), encoding='utf-8')
        return (True, "")

    except ImportError:
        return (False, "vsdx package not installed - run: pip install vsdx")
    except Exception as e:
        return (False, str(e))

def convert_file(input_file: Path, output_file: Path, timeout: int = 30) -> tuple:
    """Convert a document to Markdown using markitdown."""
    try:
        root = get_root()
        if not is_within_root(input_file, root):
            return (False, "Path outside root directory")

        output_file.parent.mkdir(parents=True, exist_ok=True)
        result = subprocess.run(
            [sys.executable, '-m', 'markitdown', str(input_file), '-o', str(output_file)],
            capture_output=True, text=True, timeout=timeout
        )
        return (result.returncode == 0, result.stderr.strip() if result.returncode != 0 else "")
    except subprocess.TimeoutExpired:
        return (False, f"Timeout ({timeout}s)")
    except FileNotFoundError:
        return (False, "markitdown not installed - run: pip install markitdown[docx,xlsx,pptx,pdf]")
    except Exception as e:
        return (False, str(e))

def main():
    logger.info("Document Conversion Utility")
    logger.info("")

    root = get_root()
    logger.info(f"Root: {root}")

    source_dir = root / 'documents' / 'source'
    output_dir = root / 'documents' / 'processed'

    if not source_dir.exists():
        logger.error(f"Source directory not found: {source_dir}")
        sys.exit(1)

    # Pre-flight checks
    checks = preflight_checks(source_dir)
    if not report_preflight(checks):
        sys.exit(1)

    logger.info("")
    output_dir.mkdir(parents=True, exist_ok=True)

    # Discover all supported files within root
    files = []
    for ext in SUPPORTED_EXTENSIONS:
        files.extend(source_dir.glob(f'*{ext}'))
    files = [f for f in files if is_within_root(f, root)]

    if not files:
        logger.info("No files to convert.")
        return

    # Split vsdx from markitdown-handled files
    vsdx_files = sorted([f for f in files if f.suffix.lower() == '.vsdx'])
    other_files = sorted([f for f in files if f.suffix.lower() != '.vsdx'])

    logger.info(f"Found {len(files)} file(s) to convert")
    if vsdx_files:
        logger.info(f"  - {len(vsdx_files)} Visio (.vsdx) file(s)")
    logger.info("")

    results, errors = [], []

    # Convert Visio files using vsdx library
    if vsdx_files:
        logger.info("--- Converting Visio files ---")
        for idx, f in enumerate(vsdx_files, 1):
            out = output_dir / (f.stem + '.md')
            logger.info(f"[{idx}/{len(vsdx_files)}] {f.name}")
            ok, err = convert_vsdx_file(f, out)
            if ok:
                results.append(f"✓ {f.name} → {out.name} (vsdx)")
                logger.info("  ✓ Success")
            else:
                errors.append(f"| {f.name} | {err} | vsdx |")
                logger.warning(f"  ✗ Failed: {err}")
        logger.info("")

    # Convert all other files using markitdown
    if other_files:
        logger.info("--- Converting documents with markitdown ---")
        for idx, f in enumerate(other_files, 1):
            out = output_dir / (f.stem + '.md')
            logger.info(f"[{idx}/{len(other_files)}] {f.name}")
            ok, err = convert_file(f, out)
            if ok:
                results.append(f"✓ {f.name} → {out.name}")
                logger.info("  ✓ Success")
            else:
                errors.append(f"| {f.name} | {err} | markitdown |")
                logger.warning(f"  ✗ Failed: {err}")
        logger.info("")

    # Write error log
    error_log = output_dir / 'error_log.md'
    error_log.write_text(
        f'# Conversion Error Log\n\n'
        f'Generated: {datetime.now().isoformat()}\n\n'
        f'| File | Error | Converter |\n'
        f'|------|-------|----------|\n' +
        ('\n'.join(errors) if errors else '| — | No errors | — |'),
        encoding='utf-8'
    )

    # Write results summary
    conversion_results = output_dir / 'conversion_results.md'
    conversion_results.write_text(
        f'# Conversion Results\n\n'
        f'Generated: {datetime.now().isoformat()}\n\n'
        f'**Total:** {len(files)} | **Success:** {len(results)} | **Failed:** {len(errors)}\n\n'
        f'## Successful Conversions\n\n' +
        ('\n'.join(results) if results else '_No successful conversions._'),
        encoding='utf-8'
    )

    logger.info("=" * 50)
    logger.info(f"Conversion complete: {len(results)}/{len(files)} succeeded")
    if errors:
        logger.warning(f"                    {len(errors)} failed (see error_log.md)")
    logger.info("=" * 50)

if __name__ == '__main__':
    main()
```
