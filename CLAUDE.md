# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Claude Code plugin that extends Anthropic's feature-dev plugin with multi-session execution for large features. Combines:
- Phases 1-4 and 6-7 from Anthropic's 7-phase feature-dev workflow (unchanged)
- A harness pattern for multi-session implementation (replaces phase 5)

## Plugin Structure

```
feature-dev-harnessed/
├── .claude-plugin/marketplace.json    # Plugin metadata
├── commands/feature-dev.md            # Modified command with harness logic
├── agents/                            # Symlinks to upstream agents
│   ├── code-explorer.md → vendor/...
│   ├── code-architect.md → vendor/...
│   └── code-reviewer.md → vendor/...
└── vendor/claude-code/                # Git submodule of anthropics/claude-code
```

## Key Architecture

**Agents via git submodule**: The three agent files are symlinked from `vendor/claude-code/plugins/feature-dev/agents/`. Updates come from upstream Anthropic repo.

**Command is our own copy**: `commands/feature-dev.md` is forked from Anthropic's version with harness logic added. This is where the plugin's value lives.

## Update Agents from Upstream

```bash
git submodule update --remote vendor/claude-code
git add vendor/claude-code
git commit -m "chore: update agents from upstream anthropics/claude-code"
```

## Session State Detection

The command auto-detects state from the filesystem:

| Condition | State | Action |
|-----------|-------|--------|
| No `feature_list.json` | New feature | Run phases 1-4, create artifacts |
| `feature_list.json` has pending items | Mid-implementation | Run one item |
| All items pass, not reviewed | Ready for review | Run phases 6-7 |
| `claude-progress.txt` contains "Feature complete" | Done | Show summary |

## Runtime Artifacts (gitignored)

- `feature_list.json`: Machine-readable work items with verification commands
- `claude-progress.txt`: Human-readable append-only session log

## Installation

```bash
# Clone with submodules
git clone --recurse-submodules https://github.com/chuggies510/feature-dev-harnessed.git

# Install plugin
claude plugin install /path/to/feature-dev-harnessed
```

## Usage

```bash
# Start new feature (planning session)
/feature-dev-harnessed:feature-dev "Add CSV export for cost reports"

# Continue implementation (auto-detects state)
/feature-dev-harnessed:feature-dev
```

## Reference Documents

- `feature-dev-harnessed-spec.md`: Complete specification with artifact schemas and behavioral decisions
- `README.md`: User-facing documentation
