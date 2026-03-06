# sa-agent

Solution Architect workflow agent for GitHub Copilot in VS Code.

## Quick Start

1. Copy `.github\agents\sa.agent.md` to your project's `.github\agents\` folder
2. Open VS Code → Copilot Chat
3. Select "sa" from the agent dropdown
4. Agent auto-discovers documents and proceeds through all 7 phases

## What It Does

- **Phase 1**: Setup project folder structure and Python converter script
- **Phase 2**: Prompt to copy artifact folders into artifacts\
- **Phase 3**: Convert documents to Markdown using markitdown
- **Phase 4**: Verify/create artifact templates
- **Phase 5**: Map content to templates, generate completeness report
- **Phase 6**: Update README.md and CHANGELOG.md
- **Phase 7**: Summary and next steps

## Key Features

- **Folder setup**: Creates standard SA folder structure
- **Artifact collection**: Prompts to copy artifact folders into artifacts\
- **Directory containment**: Agent stays within current working directory
- **Windows-optimized**: PowerShell paths and commands

## Requirements

- Windows (PowerShell)
- VS Code with GitHub Copilot
- Python 3.11+

## Usage

The agent creates folder structure, prompts you to copy artifacts, then proceeds through remaining phases automatically.

## Copy to New Project

```powershell
# From any project folder
mkdir .github\agents
curl -o .github\agents\sa.agent.md https://raw.githubusercontent.com/ajass/sa-agent/main/.github/agents/sa.agent.md
```

Or download the file directly from [GitHub](https://github.com/ajass/sa-agent).
