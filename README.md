# feature-dev-harnessed

Feature development with multi-session execution for large features.

A fork of Anthropic's [feature-dev plugin](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev) that integrates their [harness pattern for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents).

## The Problem

Anthropic's feature-dev plugin provides a thorough 7-phase workflow for feature development. This works well for features that fit in one context window. For larger features, the exploration and architecture phases (1-4) consume significant context, leaving insufficient room for implementation.

## The Solution

This fork keeps phases 1-4 (planning) and 6-7 (review) exactly as Anthropic designed them. It replaces phase 5 (implementation) with a multi-session harness pattern:

- **Session 1**: Planning - Run phases 1-4, create `feature_list.json` with work items
- **Sessions 2-N**: Implementation - Execute one item per session with verification
- **Final Session**: Review - Run phases 6-7, complete the feature

One command that detects state and does the right thing across multiple sessions.

## Installation

```bash
# Clone the repository
git clone --recurse-submodules https://github.com/chuggies510/feature-dev-harnessed.git

# Or if already cloned without submodules
git submodule update --init --recursive

# Install the plugin
claude plugin install /path/to/feature-dev-harnessed
```

## Usage

```bash
# Start a new feature (planning session)
/feature-dev-harnessed:feature-dev "Add CSV export for cost reports"

# Continue implementation (subsequent sessions)
/feature-dev-harnessed:feature-dev

# The command detects state automatically:
# - No feature_list.json → Planning session
# - Pending items → Implementation session
# - All items complete → Review session
# - Feature complete → Shows summary
```

## Artifacts

### feature_list.json

Machine-readable work items with verification commands. Created in planning session, updated during implementation.

```json
{
  "feature": "csv-export-for-cost-reports",
  "created": "2025-11-29",
  "architecture": "Pragmatic balance: Add CSV formatter utility...",
  "items": [
    {
      "id": "001",
      "description": "Create CSV formatter utility",
      "verification": "python3 -c 'from utils.csv_formatter import format_csv'",
      "status": "pass",
      "session_completed": 2
    }
  ]
}
```

### claude-progress.txt

Human-readable session log. Append-only. Helps Claude recover context between sessions.

```
Feature: csv-export-for-cost-reports
Created: 2025-11-29

Session 1 (2025-11-29):
Planning session. Ran phases 1-4.
Architecture chosen: Pragmatic balance...
Next: 001 (Create CSV formatter utility)

Session 2 (2025-11-29):
Completed: 001 (Create CSV formatter utility)
Verification: PASS
Next: 002 (Add export button)
```

## Updating Agents

The agents (code-explorer, code-architect, code-reviewer) are pulled from Anthropic's repository via git submodule:

```bash
# Update to latest upstream agents
git submodule update --remote vendor/claude-code
git add vendor/claude-code
git commit -m "chore: update agents from upstream anthropics/claude-code"
```

## Technical Details

See [feature-dev-harnessed-spec.md](feature-dev-harnessed-spec.md) for the complete specification.

## Credits

- [Anthropic's feature-dev plugin](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev) - Original 7-phase workflow
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) - Multi-session execution pattern
