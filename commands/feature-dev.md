---
description: Feature development with multi-session harness for large features
argument-hint: Optional feature description
version: 1.0.0
---

# Feature Development (Harnessed)

You are helping a developer implement a new feature using a multi-session harness pattern. This enables completing large features that don't fit in a single context window.

## How This Works

This command detects state from the filesystem and executes the appropriate phase:
- **No `.feature-dev/active/feature_list.json`**: Run planning phases (1-4), then create artifacts
- **`feature_list.json` with pending items**: Run one implementation cycle
- **All items complete, not reviewed**: Run quality review phases (6-7)
- **Feature complete**: Archive and summarize

---

## State Detection

Before doing anything else, detect current state:

1. Check if `.feature-dev/active/feature_list.json` exists
2. If it exists, read it and check item statuses
3. If `.feature-dev/active/claude-progress.txt` exists, check if it contains "Feature complete"

**State determination:**
- If `.feature-dev/active/feature_list.json` does NOT exist → Execute **Planning Session**
- If `feature_list.json` exists AND has items with status "pending" → Execute **Implementation Session**
- If `feature_list.json` exists AND all items have status "pass" AND `claude-progress.txt` does NOT contain "Feature complete" → Execute **Review Session**
- If `claude-progress.txt` contains "Feature complete" → Execute **Complete State**

**Artifact location:** All feature development artifacts are stored in `.feature-dev/active/` during development. On completion, they are archived to `.feature-dev/completed/{version}-{feature}/`.

---

## Core Principles

- **Ask clarifying questions**: Identify all ambiguities, edge cases, and underspecified behaviors. Ask specific, concrete questions rather than making assumptions. Wait for user answers before proceeding with implementation. Ask questions early (after understanding the codebase, before designing architecture).
- **Understand before acting**: Read and comprehend existing code patterns first
- **Read files identified by agents**: When launching agents, ask them to return lists of the most important files to read. After agents complete, read those files to build detailed context before proceeding.
- **Simple and elegant**: Prioritize readable, maintainable, architecturally sound code
- **Use TodoWrite**: Track all progress throughout
- **NEVER delete or modify tests**: It is unacceptable to remove or edit existing tests as this could lead to missing or buggy functionality. Add new tests, never remove or weaken existing ones.

---

# Planning Session

Execute this section when `feature_list.json` does NOT exist.

## Phase 1: Discovery

**Goal**: Understand what needs to be built

Initial request: $ARGUMENTS

**Actions**:
1. Create todo list with all phases
2. If feature unclear, ask user for:
   - What problem are they solving?
   - What should the feature do?
   - Any constraints or requirements?
3. Summarize understanding and confirm with user

---

## Phase 2: Codebase Exploration

**Goal**: Understand relevant existing code and patterns at both high and low levels

**Actions**:
1. Launch 2-3 code-explorer agents in parallel. Each agent should:
   - Trace through the code comprehensively and focus on getting a comprehensive understanding of abstractions, architecture and flow of control
   - Target a different aspect of the codebase (eg. similar features, high level understanding, architectural understanding, user experience, etc)
   - Include a list of 5-10 key files to read

   **Example agent prompts**:
   - "Find features similar to [feature] and trace through their implementation comprehensively"
   - "Map the architecture and abstractions for [feature area], tracing through the code comprehensively"
   - "Analyze the current implementation of [existing feature/area], tracing through the code comprehensively"
   - "Identify UI patterns, testing approaches, or extension points relevant to [feature]"

2. Once the agents return, please read all files identified by agents to build deep understanding

3. **Detect project version** (for version tracking):
   - Search for version files: `pyproject.toml`, `package.json`, `VERSION`, `setup.py`, `Cargo.toml`, `.claude/VERSION`
   - Extract current version number (e.g., "3.1.1" or "1.0.0")
   - Note which files contain version info (may be multiple)
   - If no version files found, note "no version tracking detected"
   - Store findings for use in Phase 4

4. Present comprehensive summary of findings and patterns discovered, including:
   - Codebase architecture and patterns
   - Current version: [version] (from [file])
   - Version files found: [list]

---

## Phase 3: Clarifying Questions

**Goal**: Fill in gaps and resolve all ambiguities before designing

**CRITICAL**: This is one of the most important phases. DO NOT SKIP.

**Actions**:
1. Review the codebase findings and original feature request
2. Identify underspecified aspects: edge cases, error handling, integration points, scope boundaries, design preferences, backward compatibility, performance needs
3. **Present all questions to the user in a clear, organized list**
4. **Wait for answers before proceeding to architecture design**

If the user says "whatever you think is best", provide your recommendation and get explicit confirmation.

---

## Phase 4: Architecture Design

**Goal**: Design multiple implementation approaches with different trade-offs

**Actions**:
1. Launch 2-3 code-architect agents in parallel with different focuses: minimal changes (smallest change, maximum reuse), clean architecture (maintainability, elegant abstractions), or pragmatic balance (speed + quality)
2. Review all approaches and form your opinion on which fits best for this specific task (consider: small fix vs large feature, urgency, complexity, team context)
3. Present to user: brief summary of each approach, trade-offs comparison, **your recommendation with reasoning**, concrete implementation differences
4. **Ask user which approach they prefer**
5. **After architecture approval, suggest target version**:
   - Use version detected in Phase 2
   - Analyze feature scope to determine bump type:
     - **PATCH** (x.y.Z → x.y.Z+1): Bug fixes, documentation, minor tweaks
     - **MINOR** (x.Y.z → x.Y+1.0): New features, enhancements, backward-compatible changes
     - **MAJOR** (X.y.z → X+1.0.0): Breaking changes, major rewrites, API changes
   - Present suggestion: "Current version is [version]. This is a [scope] change (new feature/bug fix/major rewrite). Recommend [target version]. Confirm?"
   - If no version files detected: "No version tracking detected. Skip versioning, or create VERSION file starting at 1.0.0?"
   - Store user's confirmed target version for Phase 5

---

## Phase 5: Artifact Creation

**Goal**: Convert approved architecture into multi-session execution artifacts

**DO NOT START WITHOUT USER APPROVAL OF ARCHITECTURE AND VERSION**

**Actions**:
1. Wait for explicit user approval of architecture approach and target version
2. Create `.feature-dev/active/` directory if it doesn't exist
3. Create `.feature-dev/active/feature_list.json` with the following structure:
   ```json
   {
     "feature": "descriptive name of the feature",
     "created": "YYYY-MM-DD",
     "current_version": "version detected in Phase 2 (e.g., 3.1.1)",
     "target_version": "user-approved version from Phase 4 (e.g., 3.2.0)",
     "version_files": [
       "pyproject.toml",
       ".claude/VERSION"
     ],
     "architecture": "brief description of chosen approach from phase 4",
     "init_script": "optional - path to startup script (e.g., ./init.sh)",
     "items": [
       {
         "id": "001",
         "description": "what this item accomplishes, session-sized scope",
         "verification": "shell command that returns exit code 0 on success",
         "status": "pending",
         "session_completed": null
       }
     ]
   }
   ```
4. Break implementation into **many small, granular items** (aim for 10-20+ items for complex features)
   - Each item should be completable in one context window
   - Prefer too many small items over too few large ones
   - Small items reduce risk of half-implemented features breaking the next session
5. Each item must have a verification command that returns exit code 0 for success
   - For web apps: prefer browser automation (Playwright, Puppeteer) over unit tests
   - Verify end-to-end user actions, not just code-level tests
   - Example: `npx playwright test tests/csv-export.spec.ts` or `python -m pytest tests/test_e2e.py`
6. Order items by dependency (items that depend on others come later)
7. If the project needs a startup script, create `init.sh` with:
   - Application startup commands
   - Basic health checks
   - Any environment setup
   - Make it executable: `chmod +x init.sh`
8. Create `.feature-dev/active/claude-progress.txt` with initial entry:
   ```
   Feature: [feature name]
   Version: [current_version] → [target_version]
   Created: [date]

   Session 1 ([date]):
   Planning session. Ran phases 1-4.
   Architecture chosen: [approach description]
   Created feature_list.json with [N] items.
   Next: [first item id] ([first item description])
   ```
9. Ask user: "Ready to commit artifacts? Proceed?"
10. If approved, git commit with message: `plan: create feature_list.json for [feature-name] (v[target_version])`

**Planning session ends. Run this command again to start implementation.**

---

# Implementation Session

Execute this section when `.feature-dev/active/feature_list.json` exists AND has items with status "pending".

## Step 1: Read State and Initialize

**Actions**:
1. Read `.feature-dev/active/feature_list.json` to understand items and their status
2. Read `.feature-dev/active/claude-progress.txt` to understand context and previous work
3. Read recent git log (last 20 commits) to see changes since last session
4. Determine current session number by counting "Session N" entries in `claude-progress.txt`
5. Display: "Session [N]: Implementation"
6. If `init_script` is defined in `feature_list.json`, run it to start the development environment
   - If init script fails, debug and fix before proceeding

---

## Step 2: Smoke Tests

**Goal**: Verify previously completed work still passes, fix any regressions

**Prerequisite**: Init script (if defined) must have completed successfully in Step 1.

**Actions**:
1. For each item in `.feature-dev/active/feature_list.json` with status "pass":
   - If verification starts with "manual:" → skip (manual items don't get smoke tested)
   - Otherwise → run verification command and record result (PASS or FAIL)
2. If ANY smoke test fails:
   - Report: "REGRESSION DETECTED: Item [id] verification failed"
   - Display the failed command and output
   - **FIX THE REGRESSION BEFORE PROCEEDING**:
     - Analyze what broke and why
     - Fix the issue while preserving all existing functionality
     - Re-run the failed verification until it passes
     - Re-run ALL smoke tests to ensure fix didn't break something else
   - Once all smoke tests pass, continue to next step
3. If all smoke tests pass (or no items have status "pass" yet):
   - Report: "Smoke tests: [N] passed" (or "Smoke tests: N/A (first implementation session)")
   - Continue to next step

---

## Step 3: Select Next Item

**Actions**:
1. Find the first item in `feature_list.json` with status "pending"
2. Display: "Working on: [id] - [description]"
3. Display: "Verification: [verification command]"
4. Create todo list for this item's implementation

---

## Step 4: Implement

**Goal**: Implement the selected item

**Actions**:
1. Read `feature_list.json` to recall the architecture approach
2. Read relevant source files needed for this item
3. Implement the item following the documented architecture
4. Follow codebase conventions strictly
5. Write clean, well-documented code
6. Update todos as you progress

---

## Step 5: Verify

**Goal**: Confirm the implementation works

**Actions**:
1. Check if verification starts with "manual:"
   - If yes: This is a manual verification
     - Display: "Manual verification required:"
     - Display the text after "manual:" (e.g., "Run the launcher, check the menu looks correct")
     - Ask user: "Did it pass? (y/n)"
     - If user says yes → treat as passed
     - If user says no → ask what's wrong, fix it, ask again
   - If no: This is an automated verification
     - Run the verification command
     - If it fails:
       - Analyze the failure
       - Debug and fix
       - Re-run verification
       - Repeat until it passes
2. When verification passes:
   - Display: "Verification PASSED"
   - Proceed to state update

---

## Step 6: Update State

**Goal**: Record progress in artifacts

**Actions**:
1. Update `.feature-dev/active/feature_list.json`:
   - Set item status to "pass"
   - Set session_completed to current session number
   - **Do not modify any other fields** (id, description, verification are immutable)
2. Append to `.feature-dev/active/claude-progress.txt`:
   ```
   Session [N] ([date]):
   Completed: [id] ([description])
   Verification: PASS - [brief result description]
   Smoke tests: [previous items' results, or "N/A" for first implementation session]
   Next: [next pending item id and description, or "All items complete. Ready for code review."]
   ```
3. Ask user: "Ready to commit? Proceed?"
4. If approved, git commit with message: `feat: implement [id] - [brief description]`

---

## Step 7: Clean State

**Goal**: Ensure codebase is in good state

**Actions**:
1. Ensure no broken code remains
2. Ensure the project builds/runs without errors
3. Display: "Session complete. Run this command again to continue."

**Implementation session ends.**

---

# Review Session

Execute this section when `.feature-dev/active/feature_list.json` exists AND all items have status "pass" AND `.feature-dev/active/claude-progress.txt` does NOT contain "Feature complete".

## Step 1: Confirm Completion

**Actions**:
1. Verify all items in `.feature-dev/active/feature_list.json` have status "pass"
2. Run all verification commands one final time
3. If any fail:
   - Report: "Item [id] verification failed"
   - Treat as implementation session - fix the item
4. If all pass:
   - Display: "All [N] items verified. Proceeding to code review."

---

## Phase 6: Quality Review

**Goal**: Ensure code is simple, DRY, elegant, easy to read, and functionally correct

**Actions**:
1. Launch 3 code-reviewer agents in parallel with different focuses: simplicity/DRY/elegance, bugs/functional correctness, project conventions/abstractions
2. Consolidate findings and identify highest severity issues that you recommend fixing
3. **Present findings to user and ask what they want to do** (fix now, fix later, or proceed as-is)
4. Address issues based on user decision

---

## Phase 7: Summary and Finalization

**Goal**: Document what was accomplished, bump version, and archive artifacts

**Actions**:
1. Mark all todos complete
2. Summarize:
   - What was built
   - Key decisions made
   - Files modified
   - Suggested next steps

3. **Bump version in all version files**:
   - Read `target_version` and `version_files` from `.feature-dev/active/feature_list.json`
   - For each file in `version_files`, update version string to `target_version`:
     - `pyproject.toml`: Update `version = "X.Y.Z"` line
     - `package.json`: Update `"version": "X.Y.Z"` field
     - `VERSION` or `.claude/VERSION`: Replace entire content with version
     - `setup.py`: Update `version="X.Y.Z"` in setup() call
   - If a version file doesn't exist, skip it (warn user)

4. **Update CHANGELOG.md**:
   - If CHANGELOG.md doesn't exist, create it with Keep a Changelog header
   - Prepend new version section at top:
     ```markdown
     ## [target_version] - YYYY-MM-DD

     ### Added/Changed/Fixed
     - [feature description]
     - [list key items completed]

     Architecture: [approach from feature_list.json]
     ```

5. Append to `.feature-dev/active/claude-progress.txt`:
   ```
   Session [N] ([date]):
   Code review completed.
   Issues found: [count]
   [List issues and resolutions if any]
   Version bumped: [current_version] → [target_version]
   Feature complete.
   ```

6. **Archive feature artifacts**:
   - Create `.feature-dev/completed/[target_version]-[feature-slug]/` directory
   - Move `.feature-dev/active/feature_list.json` to archive
   - Move `.feature-dev/active/claude-progress.txt` to archive
   - Move `.feature-dev/active/init.sh` to archive (if exists)
   - Remove `.feature-dev/active/` directory (now empty)

7. Ask user: "Ready to commit final state? Proceed?"
8. If approved:
   - Stage all changes: version files, CHANGELOG.md, `.feature-dev/completed/`
   - Git commit with message: `feat: complete [feature-name] v[target_version]`
   - Suggest: "Consider creating a git tag: `git tag v[target_version]`"

**Feature development complete.**

---

# Complete State

Execute this section when `.feature-dev/active/claude-progress.txt` contains "Feature complete" OR when `.feature-dev/active/` is empty but `.feature-dev/completed/` has entries.

**Actions**:
1. Check for archived features in `.feature-dev/completed/`
2. Display summary of most recent completed feature:
   ```
   Most Recent Feature: [name]
   Version: [target_version]
   Status: ARCHIVED
   Location: .feature-dev/completed/[version]-[feature]/
   ```
3. Ask user: "Would you like to:"
   - Start a new feature (run this command with a feature description)
   - Review archived features (list `.feature-dev/completed/` contents)
   - Exit

---

# Artifact Reference

## Directory Structure

```
.feature-dev/
├── active/                              # Current feature in progress
│   ├── feature_list.json                # Machine-readable work items
│   ├── claude-progress.txt              # Human-readable session log
│   └── init.sh                          # Optional startup script
└── completed/                           # Archived completed features
    ├── v3.2.0-csv-export/
    │   ├── feature_list.json
    │   └── claude-progress.txt
    └── v4.0.0-launcher-rewrite/
        ├── feature_list.json
        └── claude-progress.txt
```

- `.feature-dev/active/` is gitignored (work in progress)
- `.feature-dev/completed/` is committed (documentation of completed features)

## feature_list.json Schema

Location: `.feature-dev/active/feature_list.json`

```json
{
  "feature": "string - descriptive name of the feature",
  "created": "string - ISO date (YYYY-MM-DD)",
  "current_version": "string - version detected at start (e.g., 3.1.1)",
  "target_version": "string - user-approved target version (e.g., 3.2.0)",
  "version_files": ["array - files containing version (e.g., pyproject.toml, .claude/VERSION)"],
  "architecture": "string - brief description of chosen approach",
  "init_script": "string (optional) - path to startup script (e.g., ./init.sh)",
  "items": [
    {
      "id": "string - sequential identifier (001, 002, etc.)",
      "description": "string - what this item accomplishes",
      "verification": "string - shell command returning exit code 0 on success",
      "status": "string - 'pending' or 'pass'",
      "session_completed": "number or null"
    }
  ]
}
```

**Rules**:
- `id`, `description`, `verification`, `current_version`, `target_version`, `version_files` are IMMUTABLE after creation
- Only `status` and `session_completed` may be modified during implementation
- Aim for 10-20+ items for complex features; prefer too many small items over too few large ones
- Items must be ordered by dependency
- For web apps, prefer browser automation (Playwright, Puppeteer) verification over unit tests

**Verification types**:
- **Automated**: A shell command that returns exit code 0 on success (e.g., `python3 -m pytest tests/`)
- **Manual**: Starts with `manual:` followed by instructions (e.g., `manual: Run the app, verify the menu looks correct`)
  - Manual items prompt the user to confirm pass/fail
  - Manual items are skipped during smoke tests (can't be automated)

## Version Files Supported

| File | Pattern | Example |
|------|---------|---------|
| `pyproject.toml` | `version = "X.Y.Z"` | Python (Poetry, uv) |
| `package.json` | `"version": "X.Y.Z"` | JavaScript/Node |
| `VERSION` | `X.Y.Z` or `vX.Y.Z` | Language-agnostic |
| `.claude/VERSION` | `vX.Y.Z` | Claude Code projects |
| `setup.py` | `version="X.Y.Z"` | Python (setuptools) |
| `Cargo.toml` | `version = "X.Y.Z"` | Rust |

## claude-progress.txt Format

Location: `.feature-dev/active/claude-progress.txt`

Plain text, human-readable, append-only. Each session entry includes:
- Session number and date
- What was completed
- Verification results
- Smoke test results (for implementation sessions)
- What the next session should work on
- Version info (current → target)

The phrase "Feature complete" in this file indicates the feature is done and triggers archival.

---
