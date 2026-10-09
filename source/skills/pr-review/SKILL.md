---
name: pr-review
description: >-
  Review pull requests for correctness, tests, merge readiness, CI status,
  conflicts, and AgentJido review-state labels. Use when asked to review a PR,
  check if it is ready to merge, or provide PR feedback without editing code.
metadata:
  author: agentjido
  version: "0.3.0"
license: Apache-2.0
---

# PR Review

## Purpose

Review a pull request without changing code. This skill may update review-state
labels, but it must not commit, push, or fix the PR.

## When To Use

Use this skill when asked to:

- Review a PR
- Check if a PR is ready to merge
- Analyze code quality, test coverage, CI, or conflicts
- Apply `ready_to_merge` or `needs_work` based on review results

Use `pr-harden` instead when the user wants you to fix a `needs_work` PR.

## Requirements

- `git`
- `gh` CLI authenticated for PR metadata, checks, labels, and diffs
- Clean local worktree if checking out the PR locally

## Workflow

### 1. Identify the PR

Accept a PR URL, PR number, or current branch. Fetch:

```bash
gh pr view <number> --json title,body,author,baseRefName,headRefName,labels,reviewDecision,mergeable,statusCheckRollup
gh pr checks <number>
gh pr diff <number> --name-only
```

Record the title, author, base branch, changed files, labels, check status, and
mergeability.

### 2. Inspect the diff

Use `gh pr diff <number>` for small PRs. For large PRs, inspect file by file.
If local checkout is needed, prefer:

```bash
gh pr checkout <number> --detach
```

Do not use a writable branch unless the user explicitly asks for fixes.

### 3. Review in priority order

Prioritize findings in this order:

1. Correctness bugs and behavior regressions
2. Missing or weak regression coverage for changed behavior
3. Merge conflicts or dirty merge state against `main`
4. Failing or missing required GitHub CI
5. Public API, docs, or README drift caused by the PR
6. Security or secret-handling risks

Do not block on formatting that project tooling will handle. Do not nitpick
subjective preferences unless they create maintenance or correctness risk.

### 4. Check Elixir and Jido concerns

Inspect for:

- Pattern matching and function clauses where they simplify control flow
- Clear `{:ok, value}` / `{:error, reason}` boundaries
- No swallowed errors at system boundaries
- Public functions documented and specified when repo convention expects it
- Tests focused on behavior, not implementation details
- Jido actions and flows return the complete next state for agent turns and
  keep effects out of decision code unless the repo documents another pattern
- Runtime work delegated to directives, Plugin runtimes, or documented
  boundaries; no v2 Strategy, StateOp, or `signal_routes` patterns

### 5. Decide readiness label

Apply `ready_to_merge` only when all are true:

- No blocking correctness or regression findings
- Test coverage is adequate for the PR risk
- Required GitHub CI is green
- PR is merge-clean with the base branch
- No unresolved blocking review comments remain

Apply or keep `needs_work` when any blocker remains.

Use:

```bash
gh pr edit <number> --add-label ready_to_merge --remove-label needs_work
gh pr edit <number> --add-label needs_work --remove-label ready_to_merge
```

If labels are unavailable or permission is missing, report the intended label
change instead of failing the whole review.

## Output Expectations

Lead with findings, ordered by severity. Use file and line references whenever
possible.

Use this structure:

```markdown
## Findings

- [P1] path/to/file.ex:123 - Blocking issue and impact.
- [P2] path/to/file_test.exs:45 - Missing coverage or risky behavior.

## Readiness

Verdict: ready_to_merge | needs_work
CI: passing | failing | pending | unavailable
Merge state: clean | conflicted | unknown
Label update: applied | intended but not applied | not needed

## Notes

Short context, assumptions, or non-blocking suggestions.
```

If there are no findings, say that clearly and still report residual test or CI
risk.

## Guardrails

- Do not edit files.
- Do not commit or push.
- Do not merge the PR.
- Do not mark `ready_to_merge` when CI is failing, mergeability is conflicted,
  or blocking findings remain.
- If review requires a code change, stop at findings and recommend `pr-harden`.
