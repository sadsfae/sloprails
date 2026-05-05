## Core Rules (Always Follow)

### Git & Version Control

- **Never** commit files directly.
- **Never** run `git add` (any form).
- **Never** suggest or perform any git commit operation.
- If changes are ready, only show the diff or the full file content and instruct the user to review and commit themselves.

### Code Style

- NEVER use emojis or unicode in the actual code, don't be cute.
- You can however use emoji in official TUI dialoges, logger output or where
  reasonable but do not go overboard.
- Use black formatting for Python
- Be sparing on using code comments, less is more

### Code Quality & DRY

- Always follow **DRY** (Don't Repeat Yourself) principles where feasible.
- Refactor duplicated code when reasonable and safe.
- Prefer reusable functions, classes, or modules over duplication.

### Package Management & Dependencies

- **Always** use a virtual environment (`venv`) for any `pip install`.
- Never run `pip install` outside of an activated virtual environment.
- **Never** install any package version that is newer than 10 days old.
  - Before installing, always check the release date of the latest version.
  - Only install versions released more than 10 days ago (preferably stable and well-established).
- When a new package is needed:
  1. Create/activate virtual environment first (`python -m venv venv` → `source venv/bin/activate` or equivalent).
  2. Check release date of the package.
  3. Install only if the version is at least 10+ days old.

### Testing & Coverage (Strict Enforcement)

- **Always** check current test coverage before and after any code changes.
- **Never** allow code coverage to decrease.
- If a change would reduce coverage, either:
  - Add the necessary tests to maintain or improve coverage, **or**
  - Refuse the change and explain why.
- When adding new functionality:
  - Always evaluate whether new tests are needed.
  - Add or update tests proactively to keep coverage stable or higher.
- Prefer running `pytest --cov` (or equivalent coverage command) to verify.

### General Behavior

- Be extremely cautious with any command that modifies the project state (especially git, pip outside venv, etc.).
- Prioritize code clarity, maintainability, and safety over speed.
- If unsure about a package's release date, check it explicitly before suggesting installation.
- When suggesting shell commands involving pip, always wrap them inside virtual environment activation.

## Preferred Workflow for Changes

1. Analyze the requested change.
2. Check existing tests and coverage.
3. Make changes while following DRY.
4. Write/update tests if coverage would otherwise drop.
5. Show the diff or the full file content clearly.
6. Instruct the user to run tests + coverage check.
7. Let the user handle git add/commit themselves.

## Plan Mode: Rules

- Do not use cute codenames for claude planning mode markdown files
- Use only legible, relative names that correspond to the feature/endeavor.
- Do not prompt to start work, planning mode is for planning.
- Always double-check the current repository and codebase thoroughly before devising plans

---

**You must strictly follow all rules above in every interaction.**

---

## Practical Rules (Always Follow)

### 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:

- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

### 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:

- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:

- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

### 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:

- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:

```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```
