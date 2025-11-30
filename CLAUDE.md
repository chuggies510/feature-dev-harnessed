# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Claude Code plugin that extends Anthropic's feature-dev plugin with multi-session execution for large features. Combines:
- Phases 1-4 and 6-7 from Anthropic's 7-phase feature-dev workflow (unchanged)
- A harness pattern for multi-session implementation (replaces phase 5)
- Automatic version detection, suggestion, and bumping
- Feature archival for documentation

## Plugin Structure

```
feature-dev-harnessed/
├── .claude-plugin/marketplace.json    # Plugin metadata
├── commands/feature-dev.md            # Command with harness + versioning logic
├── agents/                            # Symlinks to feature-dev agents
└── README.md
```

**Agents**: This plugin depends on `feature-dev` from anthropics/claude-code marketplace for agents (code-explorer, code-architect, code-reviewer).

## Feature Development Directory

When using this plugin in a project, artifacts are stored in `.feature-dev/`:

```
project/
└── .feature-dev/
    ├── active/                        # Current feature (gitignored)
    │   ├── feature_list.json
    │   ├── claude-progress.txt
    │   └── init.sh
    └── completed/                     # Archived features (committed)
        └── v3.2.0-csv-export/
            ├── feature_list.json
            └── claude-progress.txt
```

## Session State Detection

The command auto-detects state from the filesystem:

| Condition | State | Action |
|-----------|-------|--------|
| No `.feature-dev/active/feature_list.json` | New feature | Run phases 1-4, create artifacts |
| `feature_list.json` has pending items | Mid-implementation | Run one item |
| All items pass, not reviewed | Ready for review | Run phases 6-7 |
| `claude-progress.txt` contains "Feature complete" | Done | Archive and summarize |

## Versioning

The plugin automatically manages project versions:

1. **Detection** (Phase 2): Finds version files (pyproject.toml, package.json, VERSION, etc.)
2. **Suggestion** (Phase 4): Recommends version bump based on feature scope
3. **Bump** (Phase 7): Updates all version files and CHANGELOG.md
4. **Archive**: Moves artifacts to `.feature-dev/completed/{version}-{feature}/`

## Key Behaviors

- **Smoke tests**: Each implementation session runs all previous verifications before new work
- **Auto-fix regressions**: If a smoke test fails, fix the regression before proceeding
- **Never delete or modify tests**: Only add new tests, never remove or weaken existing ones
- **Granular items**: Aim for 10-20+ items for complex features
- **Browser verification**: For web apps, prefer Playwright/Puppeteer over unit tests
- **Suggest then confirm**: Always suggest and ask for confirmation, never ask open-ended questions

## Installation

```bash
# Install feature-dev (provides agents)
claude plugin install feature-dev@anthropics/claude-code

# Install this plugin
claude plugin install github:chuggies510/feature-dev-harnessed
```

## Usage

```bash
# Start new feature (planning session)
/feature-dev-harnessed:feature-dev "Add CSV export for cost reports"

# Continue implementation (auto-detects state)
/feature-dev-harnessed:feature-dev
```

## Reference Documents

- `feature-dev-harnessed-spec.md`: Complete specification with artifact schemas
- `README.md`: User-facing documentation
