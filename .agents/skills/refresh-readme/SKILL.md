---
name: refresh-readme
description: >-
  Refresh a repository README so it matches current package metadata, public
  APIs, examples, configuration, workflows, and validation commands. Use when
  README content is stale, incomplete, or inconsistent with code.
metadata:
  author: agentjido
  version: "0.2.0"
license: Apache-2.0
---

# Refresh README

## Purpose

Update `README.md` to reflect the current repository without turning it into a
marketing rewrite.

## When To Use

Use this skill when asked to:

- Refresh or update a README
- Sync README examples with current code
- Fix install, usage, config, badge, or docs links
- Improve README accuracy after API or workflow changes

Use `sync-docs-to-code` for broader documentation audits beyond README.

## Requirements

- Existing `README.md`
- Local access to `mix.exs`, `lib/`, `test/`, `examples/`, `docs/`, and
  workflow files when present
- No generated provider output or vendored skill changes in this workflow

## Workflow

### 1. Inspect source of truth

Read:

```bash
sed -n '1,240p' README.md
sed -n '1,220p' mix.exs
find lib examples docs .github/workflows -maxdepth 3 -type f 2>/dev/null
```

Search for public APIs and examples:

```bash
rg "@moduledoc|@doc|defmodule|def |use Jido|config :|Application.get_env" lib examples test config 2>/dev/null
```

### 2. Identify README drift

Check:

- Package name and one-line summary
- Installation instructions and dependency version placeholders
- Basic usage examples
- Configuration and required environment variables
- Public module/function names
- Links to Hex, HexDocs, GitHub Actions, docs, examples, and license
- Validation commands
- Badges that reference stale repo names or workflows

### 3. Update conservatively

Prefer accurate, concise edits:

- Keep existing voice and structure when it works
- Replace stale examples with runnable or clearly illustrative examples
- Remove claims not supported by code
- Add missing setup/config sections only when users need them
- Keep badges only when they point to real current targets

Avoid broad marketing rewrites unless the user explicitly asks.

### 4. Validate examples

For code snippets, either:

- Verify against current modules and function names
- Mark them as illustrative when not directly runnable
- Prefer examples copied from tested code when available

### 5. Validate README result

Run relevant checks for changed examples or docs. For Elixir package READMEs,
at minimum inspect formatting and run focused tests if examples map to tests.

## Output Expectations

Report:

- README sections changed
- Stale claims removed or corrected
- Examples verified or marked illustrative
- Commands run
- Remaining docs risks

## Guardrails

- Do not change public APIs to match README during this skill.
- Do not invent features, roadmap promises, or support status.
- Do not update generated docs or vendored files unless explicitly requested.
- Do not replace a good README structure just to standardize style.
- Do not claim a package is published unless local metadata or Hex confirms it.
