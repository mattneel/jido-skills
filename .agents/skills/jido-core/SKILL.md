---
name: jido-core
description: >-
  Provide Jido v3 ecosystem context, package roles, core contracts, and
  conventions. Use before Jido agent, action, AI, testing, runtime, plugin, or
  package work when the local repo needs shared Jido guidance.
metadata:
  author: agentjido
  version: "0.3.3"
license: Apache-2.0
---

# Jido Core

## Purpose

Load shared Jido v3 ecosystem context before making package-specific decisions.
This skill is the hub for Jido framework work. Jido v3 is a declarative actor
and agent framework: you declare an Agent definition, instantiate it as an
immutable Agent value, and run that value as an OTP actor when you need a live
process.

## When To Use

Use this skill when working with:

- Jido agents, actions, flows, directives, plugins, signals, or schedules
- Jido package docs or examples
- Jido AI integration
- Tests for Jido actions, agents, or live actors
- Cross-package conventions in AgentJido repos

## Requirements

- Inspect the local repo before applying examples.
- Read local `AGENTS.md`, `mix.exs`, and nearby modules first.
- Confirm the Jido version in `mix.exs` before using v3 APIs. V3 is a beta-line
  rewrite; v2 code does not work with v3 and vice versa.
- Pin `zoi` to `0.18.7` on a fresh resolve: newer zoi releases (for example
  0.18.11) break the published `jido_action` beta compile with
  `Zoi.Types.Default is undefined`. jido beta.1's lockfile pins 0.18.7.
- APIs documented on the satellites' `release/v3` branches after the
  published betas are not in Hex packages; check the installed package first.
- Prefer local APIs and conventions over examples in this skill when they
  differ. If the repo still runs Jido v2, stop and confirm the intent before
  porting.

### V3 package line

| Package | V3 requirement | Notes |
| --- | --- | --- |
| `jido` | `~> 3.0.0-beta.1` | Declares Elixir `~> 1.18` / OTP 27, but the beta gate fails there; beta.1 is verified on Elixir 1.20.3 / OTP 29.0.5 |
| `jido_action` | `~> 3.0.0-beta.12` | Pin this explicitly: jido beta.1's own constraint and lockfile can resolve beta.11, which silently drops Flow effects at the Agent boundary instead of turning them into Directives |
| `jido_signal` | `~> 3.0.0-beta.4` | Signal model, Router, Bus, Dispatch |
| `jido_ai` | unreleased (git `release/v3`) | Hex `jido_ai` 2.x is the v2 package; the v3 stack needs pinned git refs — see the `jido-ai` skill |

The v3 guide set lives at
`https://github.com/agentjido/jido/tree/v3.0.0-beta.1/guides`. Prefer the local
repo's docs when they conflict.

## The V3 Model

1. Declare an Agent definition: data schema, routes, Plugins, metadata.
2. Instantiate the definition with an identity and validated initial state.
3. Route one Signal to exactly one Action or Flow.
4. The executable returns the complete next domain state, plus optional
   Directives.
5. Jido validates and commits the complete Agent value.
6. The Agent Server dispatches runtime effects after commit.

Direct `Jido.Agent.cmd/3` returns a candidate Agent and Directives; a live
Agent Server commits state. Actions and Flows can perform I/O before commit; a
later failure preserves committed state but does not undo completed I/O.
Applications own external idempotency and recovery.

## V2 Habit → V3 Contract

Do not write these v2 patterns. They were removed or redefined:

| V2 pattern | V3 replacement |
| --- | --- |
| `new/1` returns an Agent directly | `new/1` returns `{:ok, agent}`; `new!/1` raises |
| Agent instance created in one step | Neutral definition, then `instantiate/2` |
| `cmd(agent, {Action, params})` or Action lists | `cmd(agent, signal)`; one Signal selects one Action or Flow |
| Action returns a partial state patch | Action/Flow returns the complete next domain state |
| `signal_routes do on ..., do: act(...) end` | Declarative `routes do route "type", Target do ... end end` |
| `Jido.Agent.Strategy`, FSM strategy, command hooks | Actions/Flows own domain work; Plugins own admission and runtime |
| `Jido.Agent.StateOp`, `StateOps` | Return the complete next state from `context.agent_state` |
| `use Jido.Plugin` with schema/routes/mount options | Callback-free `Jido.Plugin` manifest selecting owner facets |
| `Jido.Await`, `Jido.Scheduler`, `Jido.Sensor` | Server call/request APIs, Scheduler/SensorManager Plugins |
| `Jido.Storage` | `Jido.Persistence` with binary CAS adapters |
| `Jido.Discovery`, `Jido.Memory`, `Jido.Thread` | Explicit modules, application-owned state/history |
| `Jido.Pod`, WorkerPool, InstanceManager | `Jido.Topology`, owned children, instance helpers |
| NimbleOptions agent state schemas | Static Zoi object schemas |

Full map: `reference/migration.md` and the upstream
[V2 to V3 API map](https://github.com/agentjido/jido/blob/v3.0.0-beta.1/guides/api-migration-map.md).

## Ecosystem Map

| Package | Role |
| --- | --- |
| `jido` | Agent values, routes, plugins, Agent Server, persistence, topology |
| `jido_action` | Actions, Flows, Instructions, `Jido.Exec` execution |
| `jido_signal` | Signal structs, Router patterns, Bus, Dispatch targets |
| `jido_ai` | AI agents, model profiles, tool calling on jido_action |
| `req_llm` | LLM provider HTTP client used by `jido_ai` |
| `jido_*` packages | Domain adapters, runtimes, tools, or examples |

Do not assume every repo depends on every package. Confirm with `mix.exs`.

## Core Principles

- Definition and instance are different values. A definition has `id: nil` and
  `state: nil`. An instance has a nonempty binary ID and complete validated
  state. A value with only one of the two is invalid.
- Agent state is one complete map. Executables start from `context.agent_state`
  and return the full next map. Unknown keys and invalid values fail validation.
- Plugin-owned state keys are protected. Only the owning Agent Plugin facet
  changes its key through `reduce/2`.
- Schemas are static Zoi data schemas. Do not build schemas dynamically and do
  not store runtime-only values (PIDs, refs, functions) in Agent state.
- Signals are the command boundary. Routing picks the first matching target;
  route defaults merge with Signal data (shallow, Signal wins).
- Directives are validated before commit and dispatched after commit in list
  order. A dispatch failure does not roll back the commit.
- Plugins package capabilities as a callback-free manifest plus up to four
  owner facets: Agent, Agent Server, Persistence, Topology.
- Tests prove behavior at the cheapest reliable layer: Action/Flow, then direct
  Agent candidate, then live actor, then effects.

## Workflow

1. Read the repo-local guidance:

   ```bash
   sed -n '1,220p' AGENTS.md 2>/dev/null || true
   sed -n '1,220p' mix.exs
   ```

2. Identify package role from `mix.exs`, modules under `lib/`, and existing
   tests. Note the Jido version (`jido 3.x` vs `2.x`) before choosing APIs.

3. Choose the narrower skill:

   - `jido-agent` for Agent definitions, routes, plugins, and live actors
   - `jido-action` for Actions, Flows, Instructions, and execution
   - `jido-ai` for LLM agents, tools, and prompts
   - `jido-testing` for test strategy and test helpers

4. Follow existing repo patterns unless they conflict with current user intent
   or documented project standards.

5. Validate with repo-local commands. Prefer documented aliases; otherwise use
   `mix test` and focused compile checks.

## Output Expectations

When using this skill to guide work, report:

- Which local repo conventions were found
- Which Jido package role and version applies
- Which narrower skill was used next
- Which validation command proves the change

## Guardrails

- Do not paste large API references into responses; load reference files only
  when needed.
- Do not port v2 code by rename alone. Each removed module needs a recorded
  replacement or intentional deletion.
- Do not assume a v2 `jido_ai`, `jido_browser`, or custom integration works
  with v3 without checking its Plugin, Strategy, and Server API use.
- Do not introduce new runtime patterns without checking existing modules.
- Do not assume release, docs, or CI commands exist across all repos.

## Reference Docs

Load these only when needed:

- `reference/agents.md` — definitions, DSL, schemas, routes, interfaces
- `reference/actions.md` — Actions, Flows, Instructions, `Jido.Exec`
- `reference/directives.md` — built-in Directives, outcomes, ordering
- `reference/plugins.md` — package manifest and four owner facets
- `reference/runtime.md` — instances, Agent Server, persistence, controls
- `reference/signals.md` — signal model, routing, dispatch, buses
- `reference/topology.md` — static topology, controller, repair
- `reference/migration.md` — v2 to v3 porting and removed APIs
