---
name: sa
description: Solution Architect workflow: setup folders, convert docs, map to templates, generate artifacts
target: vscode
tools: [read, write, glob, bash]
---

# Solution Architect Workflow

Execute phases sequentially. Wait for user confirmation before each phase.

## WINDOWS CRITICAL
- Use backslash paths: `scripts\venv\Scripts\python.exe`
- NEVER use `&&` - chain commands with `;` or run separately
- If `python` fails, try `py`
- Always use FULL PATH to venv Python: `scripts\venv\Scripts\python.exe`

## PHASE 1: SETUP
Use the CURRENT DIRECTORY as the working directory. Create folders: `artifacts/{requirements,architecture,diagrams,adr,discovered}`, `documents/{source,processed}`, `scripts`. Create convert_artifacts.py script below. Confirm completion.

## PHASE 2: SOURCE COLLECTION
Tell user: "Place source documents in /documents/source. Say 'ready' when done."

## PHASE 3: CONVERSION
1. Create venv: `python -m venv scripts\venv`
2. Install: `scripts\venv\Scripts\pip.exe install "markitdown[all]"`
3. Run: `scripts\venv\Scripts\python.exe scripts\convert_artifacts.py`
4. Report results.

## PHASE 4: TEMPLATES
Verify templates exist in /artifacts/*, create missing: requirements (business-context, stakeholder-needs, functional-requirements, non-functional-requirements, traceability-matrix), architecture (current-state, future-state, gap-analysis, roadmap, unmapped-content), diagrams (context-diagram, container-diagram, component-diagram), adr (adr-template).

## PHASE 5: MAPPING
Read converted docs from /documents/processed/. Map content to appropriate templates based on actual content (not filename). Track unmapped content in /artifacts/discovered/. Generate completeness-report.md in /artifacts/architecture/.

## PHASE 6: DOCS
Update README.md (project purpose, folder structure, setup, usage). Update CHANGELOG.md (new templates, converted docs, changes).

## PHASE 7: SUMMARY
Summarize: files converted, templates populated, completeness %, discovered content, next steps.

## GUARDRAILS
- ALWAYS use current working directory as root
- NEVER create files outside current working directory
- NEVER use `&&` - use `;` or separate commands
- Stop on any error, report to user

---

# convert_artifacts.py (save to scripts/convert_artifacts.py)

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

def find_repo_root() -> Path:
    return Path.cwd()

def check_markitdown() -> bool:
    try:
        result = subprocess.run([sys.executable, '-m', 'markitdown', '--version'], capture_output=True, text=True, timeout=10)
        if result.returncode == 0:
            logger.info(f"markitdown: {result.stdout.strip()}")
            return True
        return False
    except Exception:
        return False

def is_within_repo(file_path: Path, repo_root: Path) -> bool:
    try:
        return file_path.resolve().is_relative_to(repo_root.resolve())
    except (ValueError, OSError):
        return False

def convert_file(input_file: Path, output_file: Path, timeout: int = 30) -> tuple:
    try:
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
    if not check_markitdown():
        logger.error("markitdown not found. Run: pip install markitdown[all]")
        sys.exit(1)
    repo_root = find_repo_root()
    logger.info(f"Repository: {repo_root}")
    source_dir = repo_root / 'documents' / 'source'
    output_dir = repo_root / 'documents' / 'processed'
    if not source_dir.exists():
        logger.error(f"Source not found: {source_dir}")
        sys.exit(1)
    output_dir.mkdir(parents=True, exist_ok=True)
    files = [f for ext in SUPPORTED_EXTENSIONS for f in source_dir.rglob(f'*{ext}')]
    if not files:
        logger.info("No files to convert.")
        return
    logger.info(f"Found {len(files)} files")
    results, errors = [], []
    for f in sorted(files):
        if not is_within_repo(f, repo_root):
            continue
        out = output_dir / (f.relative_to(source_dir).with_suffix('.md'))
        logger.info(f"Converting: {f.name}")
        ok, err = convert_file(f, out)
        if ok:
            results.append(f"- {f.name} -> {out.name}")
        else:
            errors.append(f"| {f.name} | {err} |")
    (output_dir / 'error_log.md').write_text('# Error Log\n\n| File | Error |\n|------|-------|\n' + '\n'.join(errors))
    (output_dir / 'conversion_results.md').write_text(f'# Results\n\nTotal: {len(files)} | Success: {len(results)} | Failed: {len(errors)}\n\n' + '\n'.join(results))
    logger.info(f"Done: {len(results)} ok, {len(errors)} failed")

if __name__ == '__main__':
    main()
```
