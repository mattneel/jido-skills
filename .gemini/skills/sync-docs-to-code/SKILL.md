---
name: sync-docs-to-code
description: >-
  Audit and reconcile repository docs with current code, examples, module docs,
  configuration, Mix aliases, telemetry, workflows, and public APIs. Use when
  docs may be stale after code changes or before release.
metadata:
  author: agentjido
  version: "0.2.0"
license: Apache-2.0
---

# Sync Docs To Code

## Purpose

Make documentation match the current codebase and clearly flag conflicts where
product intent is unclear.

## When To Use

Use this skill for:

- Docs/code consistency audits
- Guide or example refreshes
- Module docs and README consistency
- Release-prep documentation checks
- Fixing stale config, telemetry, workflow, or Mix command references

Use `refresh-readme` when the task is limited to `README.md`.

## Requirements

- Access to docs, README, guides, examples, `lib/`, `test/`, config, and
  workflow files
- Ability to run focused tests or compile checks when docs include executable
  examples

## Workflow

### 1. Inventory docs and code surfaces

List documentation and source files:

```bash
find README.md docs guides examples lib test config .github/workflows -maxdepth 4 -type f 2>/dev/null
```

Search for names and commands likely to drift:

```bash
rg "mix |config :|Application\\.get_env|telemetry|defmodule|@doc|@moduledoc|use Jido|workflow|release|Hex" README.md docs guides examples lib test config .github/workflows 2>/dev/null
```

### 2. Compare docs against code

Check documentation references to:

- Public modules and functions
- Options, config keys, env vars, and defaults
- Mix aliases and setup commands
- Telemetry event names
- Workflows and release commands
- Examples and expected output
- Package names, links, and support status

### 3. Decide correction direction

Prefer code as source of truth when docs are stale. Prefer docs as intent only
when code clearly violates documented behavior and the user asked for code
alignment.

If intent is ambiguous, do not silently choose. Flag the conflict with concrete
file references and ask or leave a focused note.

### 4. Update docs

Make docs accurate and minimal:

- Fix stale names, commands, links, and examples
- Remove or qualify unsupported claims
- Mark non-runnable examples as illustrative
- Keep tested examples close to real code
- Preserve docs structure unless it blocks clarity

### 5. Validate

Run the narrowest relevant validation:

```bash
mix test
mix compile --warnings-as-errors
```

Use repo-specific aliases when documented. For docs-only changes, run checks
that prove referenced commands, modules, or examples still exist.

## Output Expectations

Report:

- Docs surfaces audited
- Code/docs drift found
- Changes made
- Conflicts left for user decision
- Validation commands run

## Guardrails

- Do not change code during this skill unless the user explicitly asks to make
  code match docs.
- Do not rewrite docs for style only.
- Do not leave runnable-looking examples that are known not to run.
- Do not update generated docs unless the repo documents that workflow.
- Do not hide unresolved code/docs contradictions.
