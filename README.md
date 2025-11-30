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

## Starting a New Feature

One feature at a time. After completing a feature, delete the artifacts to start fresh:

```bash
rm feature_list.json claude-progress.txt
```

Then run the command with a new feature description.

## Handling Failures

**Smoke test failure**: If a previously passing item fails, the command stops and reports the regression. Investigate and fix before continuing.

**Verification failure**: The command will debug and retry until the verification passes. If stuck, you can manually intervene.

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

Good examples:
```bash
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

# Manual verification - can't be automated
echo "Check if the button looks right"
```

### Sizing Work Items

Items should be completable in one context window. Signs an item is too large:
- Touches more than 3-4 files
- Requires multiple sub-tasks
- Has complex verification logic

Split large items into smaller ones during planning.

### Session Continuity

Between sessions, Claude recovers context by reading:
1. `feature_list.json` - what's done, what's next
2. `claude-progress.txt` - decisions made, context from previous sessions
3. Recent git log - what changed since last session

Write clear progress entries to help future sessions understand context.

### Tips

- **Be specific in your feature request**: More detail in phase 1 = fewer clarifying questions
- **Answer clarifying questions thoroughly**: Phase 3 prevents confusion during implementation
- **Trust the architecture recommendation**: It's based on actual codebase analysis
- **Review agent outputs**: Agents surface insights about your codebase you might not know
- **Don't skip phases**: Each phase builds context for the next

## Technical Details

See [feature-dev-harnessed-spec.md](feature-dev-harnessed-spec.md) for the complete specification.

## Credits

- [Anthropic's feature-dev plugin](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev) - Original 7-phase workflow
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) - Multi-session execution pattern
