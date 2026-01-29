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

## When to Use This Plugin

### Use feature-dev-harnessed for:

- **Large features** that require 3+ implementation sessions
- **Complex features** where thorough exploration consumes significant context
- **Multi-day features** where you'll return across multiple sessions
- **Features with many components** that benefit from incremental verification

### Use regular feature-dev for:

- Features completable in a single session
- Small to medium features with straightforward implementation
- Quick iterations where multi-session overhead isn't worth it

### Don't use either for:

- Single-line bug fixes
- Trivial changes
- Urgent hotfixes

## Workflow

### Planning Session (Phases 1-4)

#### Phase 1: Discovery

**Goal**: Understand what needs to be built

```
You: /feature-dev-harnessed:feature-dev Add CSV export for cost reports
Claude: Let me understand what you need...
        - What data should be included in the CSV?
        - Should users be able to select which columns to export?
        - Any specific formatting requirements?
```

#### Phase 2: Codebase Exploration

**Goal**: Understand relevant existing code and patterns

Launches 2-3 `code-explorer` agents in parallel to trace through:
- Similar features and their implementation patterns
- Architecture and abstractions in the relevant area
- UI patterns and extension points

```
Found similar features:
- PDF export (src/export/pdf.py): Uses ReportFormatter base class
- Data filtering (src/reports/filters.py): Column selection pattern

Key files to understand:
- src/export/base.py:23 - ReportFormatter interface
- src/reports/cost_report.py:45 - Cost report data structure
- src/ui/export_button.py:12 - Export button component
```

#### Phase 3: Clarifying Questions

**Goal**: Fill in gaps and resolve ambiguities before designing

```
Before designing the architecture, I need to clarify:

1. Column selection: Include all columns or let user choose?
2. Formatting: How should currency values be formatted?
3. File naming: Auto-generate filename or prompt user?
4. Large reports: Any pagination or size limits?
```

#### Phase 4: Architecture Design

**Goal**: Design multiple approaches with trade-offs

Launches 2-3 `code-architect` agents with different focuses:

```
I've designed 3 approaches:

Approach 1: Minimal Changes
- Add csv_export() method to existing CostReport class
- Reuse PDF export button with format parameter
Pros: Fast, minimal code
Cons: Couples CSV logic to report class

Approach 2: Clean Architecture
- New CsvFormatter implementing ReportFormatter
- Dedicated export service with format registry
Pros: Extensible, testable, follows existing patterns
Cons: More files, more abstraction

Approach 3: Pragmatic Balance
- New CsvFormatter class
- Wire into existing export button component
Pros: Clean separation, minimal refactoring
Cons: Some duplication with PDF formatter

Recommendation: Approach 3 - follows existing patterns without over-engineering.

Which approach would you like to use?
```

After user approves, creates `feature_list.json` with session-sized work items.

### Implementation Sessions (Phase 5, multi-session)

Each session:
1. Run smoke tests on previously completed items
2. Implement the next pending item
3. Run verification command until it passes
4. Update artifacts and commit

```
Session 3: Implementation

Smoke tests: 001 PASS, 002 PASS
Working on: 003 - Wire export button to CSV formatter
Verification: python3 -m pytest tests/test_csv_export.py -v

[Implementation happens here]

Verification PASSED
Ready to commit? Proceed?
```

### Review Session (Phases 6-7)

#### Phase 6: Quality Review

Launches 3 `code-reviewer` agents to check:
- Simplicity, DRY, elegance
- Bugs and functional correctness
- Project conventions and abstractions

```
Code Review Results:

High Priority:
1. Missing error handling for empty reports (src/export/csv.py:45)

Medium Priority:
1. Could extract common formatting logic from PDF and CSV exporters

All verification commands pass. Code follows project conventions.

What would you like to do? (fix now / fix later / proceed as-is)
```

#### Phase 7: Summary

```
Feature Complete: CSV Export for Cost Reports

What was built:
- CsvFormatter class implementing ReportFormatter interface
- Column selection UI integrated with export button
- Automated tests for CSV export functionality

Key decisions:
- Used pragmatic approach with dedicated formatter
- Reused existing export button component
- Added column selection based on user feedback

Files modified:
- src/export/csv.py (new)
- src/ui/export_button.py
- tests/test_csv_export.py (new)

Sessions: 5 (1 planning, 3 implementation, 1 review)
```

## Prerequisites

- [Claude Code](https://claude.ai/code) installed and configured
- The `feature-dev` plugin from Anthropic's claude-code-plugins marketplace (for agents)

## Installation

```bash
# Add Anthropic's marketplace (if not already added)
claude plugin marketplace add anthropics/claude-code

# Install feature-dev (provides the agents)
claude plugin install feature-dev@claude-code

# Add feature-dev-harnessed marketplace
claude plugin marketplace add chuggies510/feature-dev-harnessed

# Install this plugin (note: marketplace name is just the repo name)
claude plugin install feature-dev-harnessed@feature-dev-harnessed
```

Or install directly from GitHub:
```bash
claude plugin install github:chuggies510/feature-dev-harnessed
```

## Usage

```bash
# Start a new feature (planning session)
/feature-dev-harnessed:feature-dev "Add CSV export for cost reports"

# Continue implementation (subsequent sessions)
/feature-dev-harnessed:feature-dev
```

### State Detection

The command automatically detects what to do based on filesystem state:

| Condition | State | Action |
|-----------|-------|--------|
| No `dev/features/active/feature_list.json` | New feature | Run phases 1-4, create artifacts |
| `feature_list.json` exists with pending items | Mid-implementation | Run one item, verify, commit |
| All items pass, not reviewed | Ready for review | Run phases 6-7 |
| `claude-progress.txt` contains "Feature complete" | Complete | Archive and summarize |

## Artifacts

All artifacts are stored in `dev/features/`:

```
dev/features/
├── active/                              # Current feature (gitignored)
│   ├── feature_list.json
│   ├── claude-progress.txt
│   └── init.sh
└── completed/                           # Archived features (committed)
    └── v3.2.0-csv-export/
        ├── feature_list.json
        └── claude-progress.txt
```

### feature_list.json

Machine-readable work items with version tracking and verification commands.

```json
{
  "feature": "csv-export-for-cost-reports",
  "created": "2025-11-29",
  "current_version": "3.1.1",
  "target_version": "3.2.0",
  "version_files": ["pyproject.toml", ".claude/VERSION"],
  "architecture": "Pragmatic balance: Add CSV formatter utility...",
  "init_script": "./init.sh",
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
Version: 3.1.1 → 3.2.0
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

## Versioning

The plugin automatically detects and manages project versions:

1. **Detection** (Phase 2): Searches for `pyproject.toml`, `package.json`, `VERSION`, etc.
2. **Suggestion** (Phase 4): Recommends version bump based on feature scope:
   - PATCH: Bug fixes
   - MINOR: New features (default)
   - MAJOR: Breaking changes
3. **Bump** (Phase 7): Updates all version files and CHANGELOG.md
4. **Archive**: Completed features archived to `dev/features/completed/{version}-{feature}/`

## Starting a New Feature

Features are archived automatically on completion. To start a new feature, simply run the command with a description:

```bash
/feature-dev-harnessed:feature-dev "New feature description"
```

Previous features are preserved in `dev/features/completed/` for reference.

## Handling Failures

**Smoke test failure (regression)**: Claude stops all new work and fixes the regression until all smoke tests pass. No new items can be implemented until regressions are resolved.

**Verification failure**: Claude will debug and retry until the verification passes. If stuck, you can manually intervene.

**Init script failure**: If `init.sh` fails to start the development environment, Claude will debug and fix before proceeding.

**Mid-feature abandonment**: Delete `feature_list.json` and `claude-progress.txt` to start over, or leave them to resume later.

## Agents

Three specialized agents handle different aspects of feature development. They run in parallel during their respective phases.

### code-explorer

**Purpose**: Deeply analyzes existing codebase by tracing execution paths

**Focus areas**:
- Entry points and call chains
- Data flow and transformations
- Architecture layers and patterns
- Dependencies and integrations

**Output**: Entry points with file:line references, execution flow, key components, and list of essential files to read.

**When used**: Phase 2 (Codebase Exploration) - launches 2-3 in parallel with different focuses.

### code-architect

**Purpose**: Designs feature architectures and implementation blueprints

**Focus areas**:
- Codebase pattern analysis
- Architecture decisions with rationale
- Component design
- Implementation roadmap

**Output**: Patterns found, architecture decision, component design, implementation map with specific files, build sequence.

**When used**: Phase 4 (Architecture Design) - launches 2-3 in parallel with different approaches (minimal, clean, pragmatic).

### code-reviewer

**Purpose**: Reviews code for bugs, quality issues, and project conventions

**Focus areas**:
- Project guideline compliance
- Bug detection
- Code quality and simplicity
- Confidence-based filtering (only reports high-confidence issues)

**Output**: Critical and important issues with file:line references and specific fixes.

**When used**: Phase 6 (Quality Review) - launches 3 in parallel with different focuses (simplicity, correctness, conventions).

## Best Practices

### Writing Good Verification Commands

Verification commands determine if an item is complete. They should:

- **Return exit code 0 on success, non-zero on failure**
- **Be specific to the item's deliverable**
- **Be runnable from the project root**
- **For web apps: prefer browser automation over unit tests**

Good examples:
```bash
# Browser automation (preferred for web apps)
npx playwright test tests/csv-export.spec.ts
python -m pytest tests/test_e2e.py --headed

# Import check
python3 -c 'from utils.csv_formatter import format_csv'

# File existence
grep -q 'id="export-csv"' templates/cost-report.html

# Test suite
python3 -m pytest tests/test_csv_export.py -v

# Build check
npm run build && test -f dist/export.js
```

Bad examples:
```bash
# Too vague - doesn't verify the specific deliverable
python3 -c 'import utils'

# Unit test when E2E would be better (for web features)
python3 -c 'assert CsvFormatter().format([]) == ""'  # Doesn't verify UI works
```

### Manual Verification

Some items can't be verified by a computer (visual checks, interactive testing). Use `manual:` prefix:

```json
{
  "verification": "manual: Run the launcher, navigate all menus, verify theme looks like SMB3"
}
```

Claude will prompt: *"Manual verification required: Run the launcher, navigate all menus, verify theme looks like SMB3. Did it pass? (y/n)"*

Manual items are skipped during smoke tests since they can't be automated.

### Using init.sh

For projects that need environment setup, create an `init.sh` script:

```bash
#!/bin/bash
# Start development server
npm run dev &

# Wait for server to be ready
sleep 3

# Health check
curl -s http://localhost:3000/health || exit 1

echo "Development environment ready"
```

Reference it in `feature_list.json`:
```json
{
  "init_script": "./init.sh",
  ...
}
```

Claude will run this at the start of each implementation session.

### Sizing Work Items

**Aim for 10-20+ items for complex features.** Prefer too many small items over too few large ones. Small items reduce risk of half-implemented features breaking the next session.

Signs an item is too large:
- Touches more than 3-4 files
- Requires multiple sub-tasks
- Has complex verification logic
- Would take more than ~30 minutes to implement

Split large items into smaller ones during planning.

### Session Continuity

Between sessions, Claude recovers context by:
1. Reading `feature_list.json` - what's done, what's next
2. Reading `claude-progress.txt` - decisions made, context from previous sessions
3. Reading recent git log (20 commits) - what changed since last session
4. Running `init.sh` (if defined) - starting the development environment
5. Running smoke tests - verifying all previous work still passes

Write clear progress entries to help future sessions understand context.

### Tips

- **Be specific in your feature request**: More detail in phase 1 = fewer clarifying questions
- **Answer clarifying questions thoroughly**: Phase 3 prevents confusion during implementation
- **Trust the architecture recommendation**: It's based on actual codebase analysis
- **Review agent outputs**: Agents surface insights about your codebase you might not know
- **Don't skip phases**: Each phase builds context for the next
- **Never delete or modify existing tests**: Add new tests, never remove or weaken existing ones

## Credits

- [Anthropic's feature-dev plugin](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev) - Original 7-phase workflow
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) - Multi-session execution pattern
