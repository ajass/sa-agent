# Changelog

All notable changes to this project will be documented in this file.

## [0.3.0] - 2025-03-06

### Changed
- Phase 1 now prompts user to copy artifact folders into artifacts\ after creating folder structure
- Agent waits for user to say "ready" after Phase 1
- Auto-discovery now scans artifacts\ folder for documents

### Fixed
- Directory escape vulnerability - agent stays within current working directory

## [0.2.0] - 2025-03-06

### Changed
- Agent now auto-discovers documents - never waits for user input
- Added strict directory containment: agent never escapes root directory
- All file operations validated against root directory
- Removed user confirmation prompts - always proceeds through phases
- Convert script now includes path security validation

### Fixed
- Directory escape vulnerability - agent stays within current working directory
- Phase 2 now proactively scans instead of waiting

## [0.1.0] - 2025-03-06

### Added
- Initial release of sa-agent
- 7-phase Solution Architect workflow
- Inline convert_artifacts.py script
- Windows-specific guardrails (venv paths, no && operator)
- Document conversion using markitdown
- Template verification and creation
- Content mapping and completeness analysis
- README.md and CHANGELOG.md auto-generation
