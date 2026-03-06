---
name: sa
description: Solution Architect: create folders, prompt for artifacts, convert docs, map to templates
target: vscode
tools: [read, write, glob, bash]
---

# Solution Architect Workflow

CRITICAL: Execute phases sequentially. Wait for user confirmation in Phase 1. Otherwise, always proceed.

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

## PHASE 1: SETUP
1. Use CURRENT DIRECTORY as root
2. Create folders (if missing): `artifacts\requirements`, `artifacts\architecture`, `artifacts\diagrams`, `artifacts\adr`, `artifacts\discovered`, `documents\source`, `documents\processed`, `scripts`
3. Create `scripts\convert_artifacts.py` (script below)
4. Tell user: "Folder structure created. Copy your artifact folders (requirements, architecture, diagrams, adr, notes, etc.) into the `artifacts\` folder, then say 'ready' to continue."

## PHASE 2: COLLECT ARTIFACTS
Proactively scan for documents WITHIN ROOT only:
- `artifacts\*\*.md` (any md files in artifact subfolders)
- `documents\source\*` (docx, pptx, xlsx, pdf, md, txt)
- `docs\*` (md, txt)
- `notes\*` (if exists)
- `*.md` (top-level only)

If documents found: proceed to conversion.
If NO documents found: wait for user to copy documents to `documents\source\` or say 'ready' to skip.

## PHASE 3: CONVERSION
1. Create venv: `python -m venv scripts\venv`
2. Install: `scripts\venv\Scripts\pip.exe install "markitdown[all]"`
3. Run: `scripts\venv\Scripts\python.exe scripts\convert_artifacts.py`
4. Report results (continue even if some conversions fail)

## PHASE 4: TEMPLATES
Verify templates exist in `artifacts\*`. Create missing templates:
- requirements: business-context.md, stakeholder-needs.md, functional-requirements.md, non-functional-requirements.md, traceability-matrix.md
- architecture: current-state.md, future-state.md, gap-analysis.md, roadmap.md, unmapped-content.md
- diagrams: context-diagram.md, container-diagram.md, component-diagram.md
- adr: adr-template.md

Always create templates if missing - do NOT wait for user.

## PHASE 5: MAPPING
1. Read all converted docs from `documents\processed\`
2. Map content to appropriate templates (based on content, not filename)
3. Track unmapped content in `artifacts\discovered\`
4. Generate `artifacts\architecture\completeness-report.md`
5. Always proceed - do NOT wait for user

## PHASE 6: DOCS
1. Update README.md with: project purpose, folder structure, setup, usage
2. Update CHANGELOG.md with: new templates, converted docs, changes
3. Always proceed - do NOT wait for user

## PHASE 7: SUMMARY
Provide summary: files converted, templates populated, completeness %, discovered content, next steps.

## GUARDRAILS
- ALWAYS use current working directory as root
- NEVER create files outside current working directory
- NEVER use `&&` - use `;` or separate commands
- NEVER use absolute paths or `..` to escape root
- Wait for user in Phase 1, then proceed automatically through remaining phases
- Stop only on critical errors, report to user and continue

---

# convert_artifacts.py (save to scripts\convert_artifacts.py)

```python
#!/usr/bin/env python3
"""Document Converter - converts source documents to Markdown using markitdown."""

import sys
import subprocess
import logging
from pathlib import Path
from datetime import datetime

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

SUPPORTED_EXTENSIONS = {'.docx', '.pptx', '.xlsx', '.pdf', '.md', '.txt', '.png', '.jpg', '.jpeg', '.gif'}

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
    try:
        result = subprocess.run([sys.executable, '-m', 'markitdown', '--version'], capture_output=True, text=True, timeout=10)
        if result.returncode == 0:
            logger.info(f"markitdown: {result.stdout.strip()}")
            return True
        return False
    except Exception:
        return False

def convert_file(input_file: Path, output_file: Path, timeout: int = 30) -> tuple:
    try:
        # SECURITY: Verify input is within root
        root = get_root()
        if not is_within_root(input_file, root):
            return (False, "Path outside root directory")
        
        output_file.parent.mkdir(parents=True, exist_ok=True)
        result = subprocess.run([sys.executable, '-m', 'markitdown', str(input_file), '-o', str(output_file)], capture_output=True, text=True, timeout=timeout)
        return (result.returncode == 0, "")
    except subprocess.TimeoutExpired:
        return (False, f"Timeout ({timeout}s)")
    except FileNotFoundError:
        return (False, "markitdown not installed - run: pip install markitdown[all]")
    except Exception as e:
        return (False, str(e))

def main():
    logger.info("Starting document conversion...")
    
    root = get_root()
    logger.info(f"Root directory: {root}")
    
    if not check_markitdown():
        logger.error("markitdown not found. Run: pip install markitdown[all]")
        sys.exit(1)
    
    source_dir = root / 'documents' / 'source'
    output_dir = root / 'documents' / 'processed'
    
    if not source_dir.exists():
        logger.error(f"Source not found: {source_dir}")
        sys.exit(1)
    
    output_dir.mkdir(parents=True, exist_ok=True)
    
    # SECURITY: Only find files within root, no recursion outside
    files = []
    for ext in SUPPORTED_EXTENSIONS:
        files.extend(source_dir.glob(f'*{ext}'))
    
    if not files:
        logger.info("No files to convert.")
        return
    
    # SECURITY: Filter to ensure all files are within root
    files = [f for f in files if is_within_root(f, root)]
    
    logger.info(f"Found {len(files)} files")
    results, errors = [], []
    
    for f in sorted(files):
        out = output_dir / (f.stem + '.md')
        logger.info(f"Converting: {f.name}")
        ok, err = convert_file(f, out)
        if ok:
            results.append(f"- {f.name} -> {out.name}")
        else:
            errors.append(f"| {f.name} | {err} |")
    
    # Write results (within root)
    error_log = output_dir / 'error_log.md'
    error_log.write_text('# Error Log\n\n| File | Error |\n|------|-------|\n' + '\n'.join(errors))
    
    conversion_results = output_dir / 'conversion_results.md'
    conversion_results.write_text(f'# Results\n\nTotal: {len(files)} | Success: {len(results)} | Failed: {len(errors)}\n\n' + '\n'.join(results))
    
    logger.info(f"Done: {len(results)} ok, {len(errors)} failed")

if __name__ == '__main__':
    main()
```
