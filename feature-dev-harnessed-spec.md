# feature-dev-harnessed Specification

A fork of Anthropic's official [feature-dev plugin](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev) that integrates their [harness pattern for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents).

---

## Problem Statement

Anthropic's feature-dev plugin provides a 7-phase workflow for feature development:

1. Discovery
2. Codebase Exploration (2-3 code-explorer agents)
3. Clarifying Questions
4. Architecture Design (2-3 code-architect agents)
5. Implementation
6. Quality Review (3 code-reviewer agents)
7. Summary

This works well for features that fit in one context window. For larger features, the thorough exploration and architecture phases (1-4) consume significant context, leaving insufficient room for implementation (phase 5). The thoroughness is exactly what complex features need, but that same thoroughness prevents completion.

Separately, Anthropic published a harness pattern for long-running agents that solves multi-session execution using `feature_list.json` and `claude-progress.txt`. However, this pattern assumes a plan already exists - it provides no exploration or architecture phase.

**The gap:** There is no bridge between feature-dev's design phases and the harness pattern's execution mechanism.

---

## Solution

Fork feature-dev. Keep phases 1-4 and 6-7 exactly as Anthropic designed them. Replace phase 5 (Implementation) with the harness pattern for multi-session execution.

**Result:** One command that detects state and does the right thing across multiple sessions.

---

## Workflow

### Session 1: Planning

**User runs:** `/feature-dev-harnessed:feature-dev "feature description"`

**What executes:**

1. **Phase 1: Discovery** (from feature-dev, unchanged)
   - Create todo list with all phases
   - Clarify feature requirements with user
   - Summarize understanding and confirm

2. **Phase 2: Codebase Exploration** (from feature-dev, extended)
   - Launch 2-3 code-explorer agents in parallel
   - Each agent traces through code comprehensively
   - Each agent returns 5-10 key files to read
   - Read all identified files
   - **Detect project version**: Search for version files (pyproject.toml, package.json, VERSION, etc.)
   - Present comprehensive summary of findings **including current version**

3. **Phase 3: Clarifying Questions** (from feature-dev, unchanged)
   - Review codebase findings and original request
   - Identify underspecified aspects: edge cases, error handling, integration points, scope boundaries, design preferences, backward compatibility, performance needs
   - Present all questions to user in organized list
   - Wait for answers before proceeding

4. **Phase 4: Architecture Design** (from feature-dev, extended)
   - Launch 2-3 code-architect agents in parallel with different focuses:
     - Minimal changes (smallest change, maximum reuse)
     - Clean architecture (maintainability, elegant abstractions)
     - Pragmatic balance (speed + quality)
   - Review all approaches
   - Present trade-offs comparison with recommendation
   - Ask user which approach they prefer
   - **After architecture approval, suggest target version**:
     - Analyze feature scope (PATCH for bug fixes, MINOR for features, MAJOR for breaking changes)
     - Present suggestion: "Current version is X.Y.Z. This is a [scope], recommend [target]. Confirm?"
   - User approves architecture and version

5. **Artifact Creation** (new, replaces start of phase 5)
   - Create `.feature-dev/active/` directory
   - Convert approved architecture into `.feature-dev/active/feature_list.json`:
     - Include `current_version` and `target_version` from Phase 4
     - Include `version_files` array listing all detected version files
     - Break implementation into session-sized work items (aim for 10-20+ for complex features)
     - Each item has a verification command that returns pass/fail
     - All items start with status "pending"
   - Create initial `.feature-dev/active/claude-progress.txt` entry documenting:
     - Feature name, version info, and date
     - Architecture approach chosen
     - Number of items created
     - What the next session should work on
   - Git commit artifacts with message: "plan: create feature_list.json for [feature-name] (v[target_version])"

**Session ends.**

---

### Sessions 2 through N: Implementation

**User runs:** `/feature-dev-harnessed:feature-dev`

**State detection:** Command finds `feature_list.json` with pending items.

**What executes:**

1. **Read State**
   - Read `feature_list.json` to understand items and status
   - Read `claude-progress.txt` to understand context and previous work
   - Read recent git log to see commits since last session

2. **Smoke Tests**
   - For each item with status "pass", run its verification command
   - If any smoke test fails: STOP, report regression, do not proceed
   - If all smoke tests pass: continue

3. **Select Next Item**
   - Find first item with status "pending"
   - Display: "Working on: [id] - [description]"

4. **Implement**
   - Implement the item following the architecture from feature_list.json
   - Follow codebase conventions
   - Write clean, well-documented code

5. **Verify**
   - Run the item's verification command
   - If fails: debug and fix until it passes
   - If passes: proceed

6. **Update State**
   - Update `feature_list.json`:
     - Set item status to "pass"
     - Set session_completed to current session number
   - Append to `claude-progress.txt`:
     - Session number and date
     - What was completed
     - Verification result
     - Smoke test results
     - What the next session should work on
   - Git commit with message: "feat: implement [item-id] - [brief description]"

7. **Clean State**
   - Ensure no broken code remains
   - Ensure all tests pass
   - Ready for next session or merge

**Session ends. Repeat until no pending items remain.**

---

### Final Session: Review

**User runs:** `/feature-dev-harnessed:feature-dev`

**State detection:** Command finds `feature_list.json` with all items status "pass".

**What executes:**

1. **Confirm Completion**
   - Verify all items in feature_list.json are "pass"
   - Run all verification commands one final time
   - If any fail: report and handle as implementation session

2. **Phase 6: Quality Review** (from feature-dev, unchanged)
   - Launch 3 code-reviewer agents in parallel with different focuses:
     - Simplicity/DRY/elegance
     - Bugs/functional correctness
     - Project conventions/abstractions
   - Consolidate findings
   - Identify highest severity issues
   - Present findings to user: "fix now", "fix later", or "proceed as-is"
   - Address issues based on user decision

3. **Phase 7: Summary and Finalization** (from feature-dev, extended)
   - Mark all todos complete
   - Summarize:
     - What was built
     - Key decisions made
     - Files modified
     - Suggested next steps
   - **Bump version in all version files**:
     - Read `target_version` and `version_files` from feature_list.json
     - Update each file (pyproject.toml, package.json, VERSION, etc.)
   - **Update CHANGELOG.md**:
     - Create if doesn't exist
     - Prepend new version section with feature summary

4. **Archive and Final State Update**
   - Append to `claude-progress.txt`:
     - "Version bumped: [current] → [target]"
     - "Feature complete"
     - Code review findings and resolutions
   - **Archive feature artifacts**:
     - Create `.feature-dev/completed/[target_version]-[feature]/`
     - Move feature_list.json and claude-progress.txt to archive
     - Remove `.feature-dev/active/` directory
   - Git commit with message: "feat: complete [feature-name] v[target_version]"
   - Suggest creating git tag: `git tag v[target_version]`

**Feature done.**

---

## State Detection Logic

The command determines which phase to execute based on filesystem state:

| Condition | State | Action |
|-----------|-------|--------|
| `.feature-dev/active/feature_list.json` does not exist | New feature | Run phases 1-4, create artifacts |
| `feature_list.json` exists AND has items with status "pending" | Mid-implementation | Run one implementation cycle |
| `feature_list.json` exists AND all items have status "pass" AND `claude-progress.txt` does not contain "Feature complete" | Ready for review | Run phases 6-7 |
| `claude-progress.txt` contains "Feature complete" | Complete | Archive to `.feature-dev/completed/`, display summary |

**Artifact locations:**
- During development: `.feature-dev/active/`
- After completion: `.feature-dev/completed/{version}-{feature}/`

---

## Artifact Specifications

### Directory Structure

```
.feature-dev/
├── active/                              # Current feature in progress
│   ├── feature_list.json
│   ├── claude-progress.txt
│   └── init.sh                          # Optional startup script
└── completed/                           # Archived completed features
    └── v3.2.0-csv-export/
        ├── feature_list.json
        └── claude-progress.txt
```

### feature_list.json

Location: `.feature-dev/active/feature_list.json` (during development)
Archived to: `.feature-dev/completed/{version}-{feature}/feature_list.json`

Schema:
```json
{
  "feature": "string - descriptive name of the feature",
  "created": "string - ISO date (YYYY-MM-DD)",
  "current_version": "string - version detected at start (e.g., 3.1.1)",
  "target_version": "string - user-approved target version (e.g., 3.2.0)",
  "version_files": ["array - files containing version (e.g., pyproject.toml, .claude/VERSION)"],
  "architecture": "string - brief description of chosen approach from phase 4",
  "init_script": "string (optional) - path to startup script",
  "items": [
    {
      "id": "string - sequential identifier (001, 002, etc.)",
      "description": "string - what this item accomplishes, session-sized scope",
      "verification": "string - shell command that returns exit code 0 on success",
      "status": "string - either 'pending' or 'pass'",
      "session_completed": "number or null - session number when completed, null if pending"
    }
  ]
}
```

Rules:
- `id`, `description`, `verification`, `current_version`, `target_version`, `version_files` are immutable after creation
- Only `status` and `session_completed` fields may be modified
- Verification commands must be executable in the project directory
- Verification commands must return exit code 0 for success, non-zero for failure
- Items must be session-sized: completable within one context window
- Items should be ordered by dependency (items that depend on others come later)
- Aim for 10-20+ items for complex features

Example:
```json
{
  "feature": "csv-export-for-cost-reports",
  "created": "2025-11-29",
  "current_version": "3.1.1",
  "target_version": "3.2.0",
  "version_files": ["pyproject.toml", ".claude/VERSION"],
  "architecture": "Pragmatic balance: Add CSV formatter utility, wire to existing report UI with minimal changes to current architecture",
  "items": [
    {
      "id": "001",
      "description": "Create CSV formatter utility that converts cost report data structure to CSV string",
      "verification": "python3 -c 'from utils.csv_formatter import format_csv; assert format_csv([{\"item\": \"test\", \"cost\": 100}])'",
      "status": "pass",
      "session_completed": 2
    },
    {
      "id": "002",
      "description": "Add export button to cost report UI template",
      "verification": "grep -q 'id=\"export-csv\"' templates/cost-report.html",
      "status": "pass",
      "session_completed": 3
    },
    {
      "id": "003",
      "description": "Wire export button click handler to CSV formatter and trigger download",
      "verification": "python3 -m pytest tests/test_csv_export.py -v",
      "status": "pending",
      "session_completed": null
    }
  ]
}
```

### claude-progress.txt

Location: `.feature-dev/active/claude-progress.txt` (during development)
Archived to: `.feature-dev/completed/{version}-{feature}/claude-progress.txt`

Format: Plain text, human-readable, append-only

Structure:
```
Feature: [feature name from feature_list.json]
Version: [current_version] → [target_version]
Created: [date]

Session [N] ([date]):
[What happened in this session]
[Verification results if applicable]
[Smoke test results if applicable]
Next: [What the next session should work on]

Session [N+1] ([date]):
...
```

The phrase "Feature complete" in this file triggers archival to `.feature-dev/completed/`.

Rules:
- Append-only: never edit or delete previous session entries
- Each session entry includes: session number, date, work done, verification results, smoke test results, next steps
- Final entry when feature complete includes "Feature complete" phrase (used for state detection)
- Human-readable: no JSON, no structured format requirements

Example:
```
Feature: csv-export-for-cost-reports
Created: 2025-11-29

Session 1 (2025-11-29):
Planning session. Ran phases 1-4.
Architecture chosen: Pragmatic balance - add CSV formatter utility, wire to existing report UI.
Created feature_list.json with 3 items.
Next: 001 (Create CSV formatter utility)

Session 2 (2025-11-29):
Completed: 001 (Create CSV formatter utility)
Verification: PASS - format_csv function works correctly
Smoke tests: N/A (first implementation session)
Next: 002 (Add export button to UI)

Session 3 (2025-11-30):
Completed: 002 (Add export button to UI)
Verification: PASS - button element exists in template
Smoke tests: 001 PASS
Next: 003 (Wire button to formatter)

Session 4 (2025-11-30):
Completed: 003 (Wire button click handler)
Verification: PASS - integration test passes
Smoke tests: 001 PASS, 002 PASS
All items complete. Ready for code review.

Session 5 (2025-11-30):
Code review completed.
Issues found: 2
- Missing error handling in format_csv for empty input (fixed)
- Button styling inconsistent with other UI elements (fixed)
All issues resolved.
Feature complete.
```

---

## Plugin File Structure

```
feature-dev-harnessed/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   └── feature-dev.md          ← Our modified version (copied from Anthropic, then edited)
├── agents/
│   └── (symlinks to git submodule)
├── vendor/
│   └── claude-code/            ← git submodule of anthropics/claude-code
├── feature-dev-harnessed-spec.md
└── README.md
```

### Approach: Submodule for Agents, Own Command File

**Agents (via submodule):** The three agent files are pulled from Anthropic's official repo via git submodule. When Anthropic improves their agents, we get those updates.

**Command (our own copy):** The `feature-dev.md` command file is copied from Anthropic's original, then modified to add harness logic. We own this file and maintain it ourselves. If Anthropic improves phases 1-4 or 6-7, we can manually merge those changes.

**Rationale:** The command file is where our value-add lives. Agents are generic and benefit from auto-updates. The command phases are stable and unlikely to change frequently.

### Git Submodule Setup

```bash
# Add submodule
git submodule add https://github.com/anthropics/claude-code.git vendor/claude-code

# Create agents directory with symlinks
mkdir -p agents
ln -s ../vendor/claude-code/plugins/feature-dev/agents/code-explorer.md agents/code-explorer.md
ln -s ../vendor/claude-code/plugins/feature-dev/agents/code-architect.md agents/code-architect.md
ln -s ../vendor/claude-code/plugins/feature-dev/agents/code-reviewer.md agents/code-reviewer.md

# Copy command file as starting point (then modify)
cp vendor/claude-code/plugins/feature-dev/commands/feature-dev.md commands/feature-dev.md
```

**To update agents from upstream:**
```bash
git submodule update --remote vendor/claude-code
git add vendor/claude-code
git commit -m "chore: update agents from upstream anthropics/claude-code"
```

### .claude-plugin/plugin.json

```json
{
  "name": "feature-dev-harnessed",
  "version": "1.0.0",
  "description": "Feature development with integrated harness for multi-session execution. Fork of Anthropic's feature-dev with their long-running agent pattern.",
  "author": "chuggies510",
  "repository": "https://github.com/chuggies510/feature-dev-harnessed"
}
```

### commands/feature-dev.md

Starts as a copy of Anthropic's original, then modified to add:
- State detection logic at the top
- Artifact creation section after phase 4
- Implementation session instructions (new phase 5)
- Conditional flow based on detected state

The original phases 1-4 and 6-7 text remains largely unchanged.

### agents/ Directory

Symlinks to the submodule agents. This keeps agents in sync with Anthropic's upstream while maintaining a clean plugin structure.

### README.md

User-facing documentation:
- What the plugin does
- Installation instructions (including submodule init)
- Usage examples
- Link to this spec for technical details

---

## What Is Unchanged from feature-dev

The following are used exactly from Anthropic's feature-dev plugin without modification:

- Phase 1: Discovery - all instructions and behavior
- Phase 2: Codebase Exploration - all instructions, agent launching pattern, file reading pattern
- Phase 3: Clarifying Questions - all instructions, emphasis on not skipping
- Phase 4: Architecture Design - all instructions, three-approach pattern, trade-off presentation
- Phase 6: Quality Review - all instructions, three-reviewer pattern, issue presentation
- Phase 7: Summary - all instructions, summary format
- code-explorer agent definition (via git submodule)
- code-architect agent definition (via git submodule)
- code-reviewer agent definition (via git submodule)
- Core principles: ask clarifying questions, understand before acting, read files identified by agents, simple and elegant, use TodoWrite

---

## Behavioral Decisions

These clarify edge cases and behaviors not specified in the original feature-dev:

### Session Numbering
Session numbers are determined by parsing `claude-progress.txt` and counting "Session N" entries. Session 1 is always the planning session.

### Git Commit Behavior
The command asks user permission before committing. It does not auto-commit. After completing work, it says "Ready to commit. Proceed?" and waits for confirmation.

### Multiple Features
One feature at a time. If `feature_list.json` exists, the command continues that feature. To start a new feature, user must delete or rename the existing `feature_list.json` and `claude-progress.txt` manually.

### Smoke Test Failures
If a smoke test fails, the command reports the regression and exits. It does not attempt to debug or fix. The user decides what to do next.

---

## What Is New

The following are additions that do not exist in Anthropic's feature-dev:

- **State detection:** Command reads filesystem to determine which phase to execute
- **Artifact creation:** After phase 4, create feature_list.json and claude-progress.txt
- **feature_list.json:** Machine-readable work items with verification commands
- **claude-progress.txt:** Human-readable session log
- **Multi-session implementation:** Phase 5 spread across sessions, one item per session
- **Verification commands:** Automated pass/fail check for each item
- **Smoke tests:** Run previous items' verification before starting new work
- **Session state updates:** Update artifacts and git commit after each session

---

## Sources

- Anthropic's feature-dev plugin: https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev
- Effective harnesses for long-running agents: https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- Claude 4 best practices - multi-context-window workflows: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-4-best-practices#multi-context-window-workflows
- Anthropic autonomous-coding quickstart: https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding

---

## Implementation Steps

For a fresh session implementing this plugin:

### Step 1: Set up plugin structure
```bash
mkdir -p .claude-plugin commands agents vendor
```

### Step 2: Add git submodule
```bash
git submodule add https://github.com/anthropics/claude-code.git vendor/claude-code
```

### Step 3: Create agent symlinks
```bash
ln -s ../vendor/claude-code/plugins/feature-dev/agents/code-explorer.md agents/code-explorer.md
ln -s ../vendor/claude-code/plugins/feature-dev/agents/code-architect.md agents/code-architect.md
ln -s ../vendor/claude-code/plugins/feature-dev/agents/code-reviewer.md agents/code-reviewer.md
```

### Step 4: Create plugin.json
Create `.claude-plugin/plugin.json` with the content specified in this doc.

### Step 5: Copy and modify command file
```bash
cp vendor/claude-code/plugins/feature-dev/commands/feature-dev.md commands/feature-dev.md
```
Then modify `commands/feature-dev.md` to add:
1. State detection logic at the very top (before Phase 1)
2. Conditional routing based on detected state
3. Artifact creation section after Phase 4
4. Implementation session section (new Phase 5 content)
5. Keep Phases 1-4 and 6-7 text largely unchanged

### Step 6: Create README.md
User-facing documentation with installation and usage instructions.

### Step 7: Test the plugin
```bash
claude plugin install github:chuggies510/feature-dev-harnessed
```

---

## Session Handoff

**Planning session completed:** 2025-11-29

**What was done:**
- Read and understood the spec
- Fetched all source materials (Anthropic harness blog, Claude 4 best practices, autonomous-coding quickstart, feature-dev plugin files)
- Confirmed implementation location: `github:chuggies510/feature-dev-harnessed`
- Confirmed: build everything with git commits, then test

**Key files fetched from feature-dev plugin:**
- `commands/feature-dev.md` - 7-phase workflow, use as base for modified command
- `agents/code-explorer.md` - YAML frontmatter: name, description, tools, model (sonnet), color (yellow)
- `agents/code-architect.md` - YAML frontmatter: name, description, tools, model (sonnet), color (green)
- `agents/code-reviewer.md` - YAML frontmatter: name, description, tools, model (sonnet), color (red)
- `.claude-plugin/plugin.json` - name, version, description, author object

**Key insights from harness pattern:**
- Two-agent pattern: initializer (planning) + coding agent (implementation)
- `feature_list.json`: only edit the `passes`/`status` field, never modify other fields
- `claude-progress.txt`: append-only log for context recovery
- Each session: verify state → read progress → work on one feature → update state
- Claude 4.5 excels at discovering state from filesystem

**Next session should:**
1. Initialize git repo if needed
2. Create directory structure: `.claude-plugin/`, `commands/`, `agents/`, `vendor/`
3. Add git submodule: `git submodule add https://github.com/anthropics/claude-code.git vendor/claude-code`
4. Create symlinks from `agents/` to `vendor/claude-code/plugins/feature-dev/agents/`
5. Create `.claude-plugin/plugin.json`
6. Copy `vendor/claude-code/plugins/feature-dev/commands/feature-dev.md` to `commands/feature-dev.md`
7. Modify `commands/feature-dev.md` to add harness logic (main work)
8. Create `README.md`
9. Git commit
10. Test: `claude plugin install github:chuggies510/feature-dev-harnessed`

---

**Document version:** 1.2
**Created:** 2025-11-29
**Updated:** 2025-11-29
**Status:** Planning complete, ready for implementation in next session
