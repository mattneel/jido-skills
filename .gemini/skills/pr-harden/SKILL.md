---
name: pr-harden
description: >-
  Harden a pull request by fixing blocking review findings, strengthening
  regression tests, validating locally, pushing to the PR branch, and updating
  AgentJido review-state labels. Use only when explicitly asked to fix a PR.
metadata:
  author: agentjido
  version: "0.2.0"
license: Apache-2.0
---

# PR Harden

## Purpose

Turn a `needs_work` pull request into a merge-ready PR by editing the PR branch
directly.

## When To Use

Use this skill only when:

- The user explicitly asks to harden, fix, repair, or finish a PR
- The PR is labeled `needs_work`
- The user explicitly asks you to address blocking review findings

Use `pr-review` instead for read-only PR review.

## Requirements

- `git`
- `gh` CLI with permission to push to the PR branch
- Clean local worktree before checking out the PR
- Clear target PR URL or number

## Workflow

### 1. Start from a clean base

Run:

```bash
git status --short --branch
git fetch --prune origin
```

Stop if the worktree is dirty unless the dirty files are known to be yours and
part of this hardening task.

### 2. Inspect before editing

Fetch PR metadata, checks, labels, reviews, comments, and changed files:

```bash
gh pr view <number> --json title,body,labels,reviewDecision,mergeable,statusCheckRollup,headRefName,baseRefName
gh pr checks <number>
gh pr diff <number> --name-only
```

Identify blocking issues first:

- Correctness bugs
- Missing regression tests
- Failing CI
- Merge conflicts
- Unresolved blocking review comments

### 3. Check out the writable PR branch

Use:

```bash
gh pr checkout <number>
```

Confirm the checkout is writable before editing. If the PR comes from a fork
and is not writable, stop and report the limitation.

### 4. Fix blockers only

Keep changes scoped to the PR goal. For each blocking issue:

- Fix the behavior
- Add or strengthen focused regression tests
- Preserve unrelated user changes
- Avoid broad refactors unless required to make the fix safe

### 5. Validate

Run the focused tests that prove each fix. Then run repo-standard validation.
If no alias is documented, run at least:

```bash
mix test
mix compile --warnings-as-errors
```

If GitHub CI was failing, make sure local validation covers the failing area
before pushing.

### 6. Push and relabel

Commit with a conventional commit message and push to the PR branch only.
Never push to `main`.

After push, update labels:

```bash
gh pr edit <number> --add-label needs_work --remove-label ready_to_merge
gh pr edit <number> --add-label ready_to_merge --remove-label needs_work
```

Apply `ready_to_merge` only when all blockers are resolved, focused validation
passes, required CI is green, and the PR is merge-clean. If CI is pending after
the push, keep or apply `needs_work` and report that the PR should be relabeled
after CI passes.

## Output Expectations

Report:

- Blocking issues fixed
- Tests added or strengthened
- Validation commands run
- Commit and push status
- Final label state
- Remaining blockers, if any

## Guardrails

- Do not merge the PR.
- Do not rewrite unrelated PR history unless the user explicitly asks.
- Do not push to `main`.
- Do not mark `ready_to_merge` with known failing CI, unresolved blockers, or
  conflicts.
- Do not expand scope beyond blockers needed for merge readiness.
