---
name: jido-agent
description: >-
  Build and maintain Jido v3 agent modules: definitions, instances, static
  state schemas, Signal routes, generated interfaces, plugins, and Agent Server
  runtime wiring. Use when creating agents, changing agent behavior, or working
  with live Jido actors.
metadata:
  author: agentjido
  version: "0.3.1"
license: Apache-2.0
---

# Jido Agent

## Purpose

Create or modify Jido v3 agents while preserving local repo conventions, the
definition/instance split, the complete-state contract, and the separation
between pure agent behavior and runtime side effects.

## When To Use

Use this skill for:

- New `use Jido.Agent` modules
- Definition and instance construction (`definition/0`, `new/1`,
  `instantiate/2`)
- Static state schema changes
- Route declarations, route defaults, and generated interfaces
- Plugin declarations and Plugin-owned state
- Live execution: `Jido.start_agent`, `AgentServer` queries, calls, casts,
  cancellation
- Directives, children, persistence, hibernation, and upgrades
- Debugging route selection, candidate rejections, commits, or directives

Invoke `jido-core` first for shared ecosystem context.

## Requirements

- Read local `AGENTS.md`, `mix.exs`, existing agent modules, and tests before
  writing examples or code.
- Confirm the repo runs Jido v3 (`jido ~> 3.0.0-beta.1` or the local checkout).
  Do not port v2 agent code by renaming alone; follow
  `jido-core/reference/migration.md`.
- Use static Zoi data schemas. Do not build schemas dynamically.
- Keep PIDs, references, functions, and timers out of Agent state; Plugin
  runtimes own them.
- Prefer the repo's current agent macro options and route style.

## Workflow

### 1. Inspect existing patterns

Search before designing:

```bash
rg "use Jido.Agent|routes do|plugin |Jido.start_agent|AgentServer" lib test
```

Record:

- Current schema style and metadata conventions
- Route declaration style and generated interfaces
- Plugin declarations and owned state fields
- Runtime module names and supervision pattern
- Test style for direct candidates and live actors

### 2. Define the agent boundary

State clearly:

- What state the agent owns (domain fields; one Plugin owns each Plugin field)
- Which Signals it handles and which one executable each route selects
- Which Actions or Flows do the domain work
- Which effects must return as Directives or move to a Plugin runtime
- What identity and partition scheme applies

Do not hide external I/O in pure command logic unless the repo already
documents that pattern. Remember that Actions may perform I/O before commit and
a later failure cannot undo it.

### 3. Declare the definition

```elixir
defmodule MyApp.Counter do
  use Jido.Agent, name: "counter", description: "Counts things"

  agent do
    schema Zoi.object(%{count: Zoi.integer() |> Zoi.default(0)})
    metadata %{domain: "counters"}
  end

  routes do
    signal_source "/counters"

    route "counter.increment", MyApp.Increment do
      defaults %{amount: 1}
      define :increment, args: [{:optional, :amount}]
    end
  end
end
```

- Declare each field in exactly one place: `use` options or the DSL block.
- Add `plugin MyApp.Plugin, config: [...]` in the `agent` block for
  capabilities.
- Keep metadata portable data; do not put runtime values in it.

### 4. Implement state and command behavior

- Executables start from `context.agent_state` and return the complete next
  state. A partial map is not a state patch.
- Preserve every Plugin-owned key; only the owning facet's `reduce/2` changes
  it.
- Route selection uses the first matching target in Router order. Exact routes
  beat wildcards. No match returns `Jido.Error.RoutingError`.
- Route defaults merge with Signal data (shallow; Signal wins).
- Use `Jido.Agent.set/2` for field merges outside a Turn; it validates the
  complete result and protects Plugin keys.
- Validate with `validate/1`, `validate_definition/1`, `validate_instance/1`.
  The v2 `:strict` option is gone.

### 5. Handle directives and runtime

Return Directives as the third success element for effects that belong to the
runtime:

```elixir
{:ok, next_state, [Jido.Agent.Directive.emit(signal, {:bus, target: MyBus})]}
```

Dispatch targets use `target:` (see `jido-core/reference/signals.md`); an
invalid target rejects the Turn before commit. For buses, the Agent Server
inherits the Jido instance scope, so start the Bus with the same scope
(`[{Jido.Signal.Bus, name: MyBus, jido: MyApp.Jido}]`) or pass `jido: nil`
for an unscoped bus.

Use `emit`, `emit_to_parent`, `emit_to_child`, `emit_to_pid`, `spawn_child`,
`spawn_process`, `adopt_child`, `stop_child`, and `stop`. Directives validate
before commit and dispatch after commit in list order; a dispatch failure does
not roll back the commit, and ordinary directives are not a durable outbox.

Direct evaluation returns the Directive list without dispatching it. Use a live
Server when admission, commit, or effects matter. AgentServer tests should come
after direct candidate tests unless the bug only exists at runtime.

### 6. Wire live execution

```elixir
{:ok, _jido} = Jido.start()
{:ok, server} = Jido.start_agent(MyApp.Counter, id: "counter-1")

{:ok, committed} = Jido.AgentServer.call(server, signal)
{:ok, committed} = MyApp.Counter.increment(server, 2)   # generated live helper

snapshot = Jido.AgentServer.snapshot(server)
status = Jido.AgentServer.status(server)
children = Jido.AgentServer.children(server)
```

- `start_agent` waits for Plugin readiness and required initial persistence.
- Use `send_request/3` + `receive_response/2` when sending and waiting must be
  separate; `cancel/2` or `cancel_turn/3` for cancellation.
- `attach/2`, `detach/2`, and `touch/1` control idle shutdown.
- `Jido.AgentServer.upgrade/3` migrates a committed Agent to a target module
  with the same identity and Plugin declarations.
- Add the instance module to the supervision tree; v3 has no installer
  generators.

### 7. Validate

Run the narrowest meaningful tests first, then repo-standard validation:

```bash
mix test path/to/agent_test.exs
mix test
```

If the repo documents a quality alias, run it after tests. For live tests use a
unique named Jido instance per test and explicit monitors/replies instead of
sleeps.

## Output Expectations

Report:

- Agent modules changed or added
- Definition, schema, and route behavior affected
- Plugin and owned-state changes
- Runtime/directive behavior added or changed
- Tests run and what they prove

## Guardrails

- Do not use v2 APIs: `signal_routes`, Strategy modules, `StateOp`/`StateOps`,
  `cmd` with Actions or lists, partial state patches, `Jido.Await`,
  `Jido.Scheduler`, `Jido.Sensor`, or `Jido.Storage`.
- Do not build `%Jido.Agent{}` by hand and assume it is valid; use
  `new/1`, `new!/1`, or `instantiate/2`.
- Do not create a GenServer when an Agent plus Agent Server already fits.
- Do not add side effects to pure state-transition code without explicit repo
  precedent, and never inside schema refinements or Plugin `prepare/2`.
- Do not change public agent behavior without regression tests.
- Do not broaden plugin or route scope beyond the task.

## Reference Docs

- `jido-core/reference/agents.md` for the full authoring contract
- `jido-core/reference/plugins.md` for plugin facets and owned state
- `jido-core/reference/runtime.md` for instances, Server, persistence
- `jido-core/reference/directives.md` for directive and outcome semantics
- `jido-core/reference/topology.md` for static systems
