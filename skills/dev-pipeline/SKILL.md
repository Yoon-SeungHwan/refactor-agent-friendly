---
name: dev-pipeline
description: Autonomous development pipeline — architect → builder → reviewer
argument-hint: [requirement or ticket description]
---

Autonomous development pipeline. Orchestrates 3 agents sequentially:
architect (design) → builder (implement) → reviewer (verify).

## Input

$ARGUMENTS

## Pipeline

### Phase 1: Architecture

Spawn the `architect` agent with this prompt:

```
Requirement: $ARGUMENTS

IMPORTANT — Context First:
1. Analyze the requirement first. If logs, errors, or warnings are provided, parse them to identify exact files, APIs, and line numbers.
2. Only then read the specific files identified from the context.
3. Do NOT do a blind full codebase scan.

Design an implementation plan for this requirement.
Read apps/ooc-app/CLAUDE.md for conventions.
Explore the codebase to find existing patterns to reuse.
Output your plan in the structured format specified in your instructions.
```

Wait for the architect's plan. Present the plan summary to the user.
If the user approves (or no user input within 30 seconds in auto mode), proceed.

Record the current HEAD commit: `git rev-parse HEAD` → $PRE_BUILD_SHA

### Phase 2: Build

Spawn the `builder` agent with the architect's plan:

```
Implement this plan:

<architect's plan output here>

Follow the plan exactly. Run yarn ooc:lint and yarn typescript after all changes.
Output your completion report in the structured format specified in your instructions.
```

Wait for the builder's completion report.
If lint or typecheck FAILED, re-spawn builder with the errors. Max 2 retries.

Record the post-build commit: `git rev-parse HEAD` → $POST_BUILD_SHA

### Phase 3: Review

Spawn the `reviewer` agent with the explicit diff range:

```
Review the changes from commit $PRE_BUILD_SHA to $POST_BUILD_SHA.

Run: git diff $PRE_BUILD_SHA..$POST_BUILD_SHA

Check all changes against the CLAUDE.md conventions and .claude/rules/ patterns.
Output your verdict in the structured format specified in your instructions.
```

### Phase 4: Resolution

**If APPROVE:**
- Report success to user with summary of changes
- Done

**If REQUEST_CHANGES:**
- Spawn a fresh `builder` agent with the reviewer's feedback:
  ```
  Fix these review issues:

  <reviewer's issues here>

  The original plan was:
  <architect's plan>

  Fix only the issues listed. Run lint + typecheck after.
  ```
- After builder completes, spawn `reviewer` again with new diff range
- Max 3 review loops total. If still failing after 3, escalate to user:
  ```
  Pipeline stuck after 3 review iterations.
  Remaining issues:
  <latest reviewer feedback>
  Please resolve manually or adjust the requirement.
  ```

## Context Management

- Each agent runs in its own context window (subagent isolation)
- Only structured outputs cross the boundary (plan / completion / verdict)
- If orchestrator context exceeds 60%, run /compact before next phase
- Fresh builder per review loop (no accumulated confusion)

## When NOT to Use This Pipeline

- Trivial changes (typo, single-line fix) → just do it directly
- Large multi-domain features → use `/team-feature` instead (parallel workers)
- PR review → use `/team-review` instead (Security/Performance/Quality)
- Debugging → use `/team-debug` instead
