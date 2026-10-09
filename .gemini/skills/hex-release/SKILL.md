---
name: hex-release
description: >-
  Guide human-in-the-loop Hex package releases for AgentJido repositories.
  Use when preparing, validating, dry-running, publishing, tagging, or
  dispatching a Hex release workflow.
metadata:
  author: agentjido
  version: "0.2.0"
license: Apache-2.0
---

# Hex Release

## Purpose

Release an AgentJido Hex package with explicit verification before any push,
tag, GitHub release, or Hex publish.

## When To Use

Use this skill for:

- Preparing a new Hex release
- Bumping package version
- Running release dry-runs
- Dispatching `.github/workflows/release.yml`
- Publishing to Hex.pm
- Creating or verifying release tags and GitHub releases

Do not use this skill for release-status reporting only. For reporting, inspect
the workspace release-status tooling instead.

## Requirements

- `mix`, `git`, and `gh` when using the automated workflow
- Hex credentials locally for actual local publish
- `HEX_API_KEY` configured in GitHub Actions for workflow publish
- Clean working tree before release actions

## Workflow

### 1. Identify the package

Read `mix.exs` and record:

- Mix app/package name
- Current version
- Whether `git_ops` is configured
- Release workflow path, normally `.github/workflows/release.yml`

Prefer structured inspection of `mix.exs` over hardcoded package assumptions.

### 2. Check dependency blockers

Inspect `mix.exs` for `github:`, `git:`, and `path:` dependencies.

- Dev/test-only git or path deps do not block Hex publish; note them.
- Runtime git or path deps block Hex publish; stop and report which deps must
  be published to Hex or switched to Hex requirements first.

### 3. Verify repository state

Run:

```bash
git status --porcelain
git branch --show-current
git fetch --tags origin
```

Stop if the working tree is dirty. Warn if the branch is not `main`. Do not
switch branches or stash work unless the user explicitly asks.

### 4. Check releasable commits

Run:

```bash
git log --oneline $(git describe --tags --abbrev=0 2>/dev/null || echo "")..HEAD
```

If there are no commits since the last tag, stop and report that there is
nothing to release.

Review commit messages for conventional commit prefixes because `git_ops`
release notes and version bumps depend on them.

### 5. Validate locally

Run the project-standard checks before release:

```bash
mix deps.get
mix test
```

If the repo documents `mix quality`, run it. If not, run at least:

```bash
mix format --check-formatted
mix compile --warnings-as-errors
```

Stop on failures and report the failing command and key error lines.

### 6. Dry-run Hex publish

Run:

```bash
mix hex.publish --dry-run
```

Review package metadata and file list for missing files, secrets, build
artifacts, stale docs, or unexpected package names.

If the dry-run fails only because the local machine is not authenticated to
Hex, treat that as a local environment limitation. Show the useful metadata
or file-list output and remind the user that CI still needs `HEX_API_KEY`.
Stop on any other dry-run failure.

### 7. Choose release path

After pre-flight checks pass, show a short summary and ask the user to choose:

- Automated release through GitHub Actions
- Manual local release

Prefer the automated workflow when `.github/workflows/release.yml` delegates to
the shared AgentJido release workflow and uses `secrets: inherit`.

## Automated Path

### Confirm workflow

Inspect `.github/workflows/release.yml`. Treat the repo as configured if it
delegates to an AgentJido shared release workflow and passes compatible inputs.
Do not rewrite a working wrapper only for cosmetic consistency.

Verify support for these inputs before mentioning them as available:

- `dry_run`
- `hex_dry_run`
- `skip_tests`
- `version_override`

### Run dry-run first

Use GitHub CLI when available:

```bash
gh workflow run release.yml -f dry_run=true
gh run list --workflow=release.yml --limit=1
gh run watch
```

If the workflow supports an explicit version:

```bash
gh workflow run release.yml -f dry_run=true -f version_override=1.2.3
```

Review the workflow summary for version bump, changelog, tests, Hex dry-run
output, tag that would be created, and any publish blockers.

### Run real release

Only after the user confirms the dry-run result, dispatch the real release:

```bash
gh workflow run release.yml
```

Use `skip_tests=true` only when CI has already passed and the user explicitly
accepts that shortcut.

If supported and requested:

```bash
gh workflow run release.yml -f version_override=1.2.3
```

Watch the run and verify:

- Tag exists
- GitHub release exists
- Hex package version exists
- `origin/main` contains the release commit when the workflow creates one

## Manual Path

Use the manual path only when the workflow is unavailable, inappropriate for
the repo, or explicitly requested.

### Preview version

Run:

```bash
mix git_ops.release --dry-run
```

Ask the user to confirm the proposed version or provide an override.

### Create release commit and tag

After confirmation:

```bash
mix git_ops.release --yes
```

For an explicit version, use the repo-supported `git_ops` override flag after
checking `mix help git_ops.release`.

Review the release commit, version change, changelog, and tag locally.

### Push and publish

Before pushing, ask for user confirmation. Then:

```bash
git push origin main
git push origin --tags
```

Before publishing, ask for user confirmation again. Then:

```bash
mix hex.publish --dry-run
mix hex.publish --yes
```

Create the GitHub release from the tag using generated notes or the changelog,
depending on the repo convention.

## Output Expectations

Report:

- Package name and current/proposed version
- Pre-flight result summary
- Release path used
- Commands dispatched
- Final verification status for tag, GitHub release, and Hex package
- Any blockers that stopped the release

## Guardrails

- Never publish to Hex without explicit user confirmation.
- Never push release commits or tags without explicit user confirmation.
- Never ignore runtime git/path dependency blockers.
- Do not assume `mix quality` exists; inspect the repo first.
- Do not assume every repo uses `git_ops`; detect it.
- Hex packages cannot be unpublished after the Hex grace period; prefer dry-runs
  and workflow verification over speed.
