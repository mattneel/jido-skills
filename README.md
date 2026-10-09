# Jido Skills

**The agent skills that make your AI coding assistant an expert in the Jido ecosystem.**

[![GitHub stars](https://img.shields.io/github/stars/agentjido/jido-skills?style=flat-square)](https://github.com/agentjido/jido-skills)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue?style=flat-square)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/spec-agentskills.io-purple?style=flat-square)](https://agentskills.io/specification)
[![Skills](https://img.shields.io/badge/skills-10-green?style=flat-square)](#commands)

---

A collection of [Agent Skills](https://agentskills.io) that teach AI coding agents how to build with [Jido](https://github.com/agentjido/jido) — the declarative actor and agent framework for Elixir. Instead of pasting docs into every prompt, install these skills once and your agent gains deep knowledge of Jido's patterns, APIs, and ecosystem.

These skills track the **Jido v3 beta line**: `jido ~> 3.0.0-beta.1`,
`jido_action ~> 3.0.0-beta.12` (Flow effects become Agent Directives), and
`jido_signal ~> 3.0.0-beta.4`. V3 is a breaking rewrite: agents are declared
as definitions, instantiated as values, and run as OTP actors; commands are
Signals routed to one Action or Flow that returns the complete next state.
The v3 AI stack (`jido_ai`) is not on Hex yet — see the `jido-ai` skill for
the pinned git set.

Works with **Codex** (primary), **Claude Code**, **Cursor**, **Gemini CLI**, and any tool supporting the [agentskills.io](https://agentskills.io/specification) standard.

## How It Works

Agent Skills use **progressive disclosure** — your coding agent sees skill names and descriptions at startup, then loads the full instructions only when relevant. Think of each skill as a slash command:

```
$jido-agent    ← Codex
/jido-agent    ← Claude Code, Cursor, Gemini
```

Ask your agent to build a Jido agent, and it automatically loads the `jido-agent` skill with step-by-step instructions, real API examples, and DO/DON'T guidance. No copy-pasting docs.

## Commands

### 🧭 Foundation

| Command | Description |
| --- | --- |
| `$jido-core` | **The hub skill.** Jido v3 model, ecosystem map, package versions, plus deep references for agents, actions/flows, directives, plugins, runtime, signals, topology, and the v2→v3 migration. |

### 🔨 Building

| Command | Description |
| --- | --- |
| `$jido-agent` | Build Jido v3 Agents — definitions vs instances, static Zoi state schemas, Signal routes and generated interfaces, plugin declarations, directives, and the Agent Server runtime. |
| `$jido-action` | Create composable v3 actions and flows — `use Jido.Action`, static input/output schemas, effects, `Jido.Exec`, flow graphs, inline steps, instructions, and AI-tool integration. |
| `$jido-ai` | AI/LLM integration — build AI agents with `jido_ai`, tool contracts, model profiles and instructions, streaming, sessions, and testing without live providers. |

### 🧪 Quality

| Command | Description |
| --- | --- |
| `$jido-testing` | Test Jido v3 behavior — direct candidate tests, live actor tests with isolated instances, plugin boundaries, and deterministic AI/adapter tests. |
| `$pr-review` | Review a Pull Request — code quality analysis, Elixir and Jido v3 checks, merge conflict detection, and structured readiness verdict. |
| `$pr-harden` | Harden a `needs_work` PR — fix blocking findings, strengthen regression tests, validate locally, push, and update review labels. |
| `$sync-docs-to-code` | Audit and reconcile repository docs with current code, examples, configuration, telemetry, and public APIs. |
| `$refresh-readme` | Refresh `README.md` so it matches package metadata, public APIs, examples, and validation commands. |

### 📦 Shipping

| Command | Description |
| --- | --- |
| `$hex-release` | Interactive Hex release workflow — pre-flight checks, automated (GitHub Actions) or manual publish, and post-release verification. |

> **Note:** Commands shown with `$` prefix (Codex). In Claude Code, Cursor, and Gemini CLI, use `/` instead.

## Quick Start

### Install for Codex (primary)

```bash
# Copy into your Jido project
cp -r .agents/skills/ your-project/.agents/skills/

# Or symlink for auto-updates
ln -s /path/to/jido-skills/.agents/skills your-project/.agents/skills
```

### Install for other agents

```bash
# Claude Code
cp -r .claude/skills/ your-project/.claude/skills/

# Cursor
cp -r .cursor/skills/ your-project/.cursor/skills/

# Gemini CLI
cp -r .gemini/skills/ your-project/.gemini/skills/
```

### Use it

Open your agent and ask:

```
Build me a Jido agent that manages a task queue with priorities
```

Your agent finds and loads `$jido-core` → `$jido-agent` automatically, then follows the skill instructions to build it correctly.

Or invoke directly:

```
$jido-action — create a new action that validates email addresses
```

## Architecture

```
jido-skills/
├── source/skills/          ← Edit here (single source of truth)
│   ├── jido-core/
│   │   ├── SKILL.md
│   │   └── reference/      ← Deep docs: agents, actions, directives,
│   │                          plugins, runtime, signals, topology, migration
│   ├── jido-agent/SKILL.md
│   ├── jido-action/SKILL.md
│   ├── jido-ai/SKILL.md
│   ├── jido-testing/SKILL.md
│   ├── pr-review/SKILL.md
│   ├── pr-harden/SKILL.md
│   ├── sync-docs-to-code/SKILL.md
│   ├── refresh-readme/SKILL.md
│   └── hex-release/SKILL.md
│
├── .agents/skills/         ← Generated: Codex
├── .claude/skills/         ← Generated: Claude Code
├── .cursor/skills/         ← Generated: Cursor
├── .gemini/skills/         ← Generated: Gemini CLI
│
├── scripts/                ← Zero-dep Node.js build system
└── skills-lock.json        ← SHA-256 integrity hashes
```

**One source, many targets.** Skills are authored once in `source/` with `{{command_prefix}}` placeholders, then built to each provider's directory with the correct prefix (`$` or `/`), paths, and config references.

```bash
bun run build       # Rebuild all providers
bun run validate    # Validate without writing
```

## Hub-and-Spoke Pattern

`jido-core` is the **hub skill** — it contains the v3 model, ecosystem map, package versions, and design principles that every other skill depends on. Each satellite skill starts with:

```markdown
Invoke $jido-core first for shared ecosystem context.
```

This ensures your agent always has foundational context before diving into a specific task. The hub also includes deep reference docs in `reference/` that the agent loads on demand:

- `reference/agents.md` — Definitions, instances, DSL, schemas, routes, interfaces
- `reference/actions.md` — Actions, Flows, Instructions, `Jido.Exec`
- `reference/directives.md` — Built-in directives, validation, outcomes
- `reference/plugins.md` — Package manifest, owner facets, owned state
- `reference/runtime.md` — Instances, Agent Server, persistence, controls
- `reference/signals.md` — Signal model, routing, dispatch, buses
- `reference/topology.md` — Static topology, controller, repair
- `reference/migration.md` — V2→v3 porting map and removed APIs

## The Jido Ecosystem

These skills cover the full Jido ecosystem:

| Package | Description | Skill |
| --- | --- | --- |
| [jido](https://github.com/agentjido/jido) | Declarative actor and agent framework — definitions, routes, plugins, runtime | `$jido-core` `$jido-agent` |
| [jido_action](https://github.com/agentjido/jido_action) | Composable actions, flows, instructions, and `Jido.Exec` | `$jido-action` |
| [jido_signal](https://github.com/agentjido/jido_signal) | CloudEvents signals, router, bus, dispatch | `$jido-core` |
| [jido_ai](https://github.com/agentjido/jido_ai) | AI agents, model profiles, tool calling | `$jido-ai` |
| [req_llm](https://github.com/agentjido/req_llm) | HTTP client for LLM APIs | `$jido-ai` |

🌐 [agentjido.xyz](https://agentjido.xyz) — Website, demos, and workbench examples

## Contributing

We welcome new skills! See [DEVELOP.md](DEVELOP.md) for:

- Skill authoring guidelines and the SKILL.md format
- Build system setup (`bun run build`)
- Provider compatibility matrix ([HARNESSES.md](HARNESSES.md))
- How to submit a PR

Skills follow the [agentskills.io specification](https://agentskills.io/specification). Keep `SKILL.md` under 500 lines; use a `reference/` subdirectory for deep documentation.

## License

[Apache-2.0](LICENSE) - Copyright 2026 Mike Hostetler <mike.hostetler@gmail.com>
