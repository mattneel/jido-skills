# Jido Plugins Reference (v3)

Source guides: `plugins.md`, `plugin-contract-and-lifecycle.md`,
`plugin-state.md`, `plugin-runtimes.livemd`, `core-scope.md`.

A Plugin package adds one declared capability to an Agent. The package module is
a **callback-free manifest** that selects up to four owner facets. Each facet
has one bounded authority.

## Declare a Package

```elixir
defmodule MyApp.RateLimit do
  use Jido.Plugin,
    agent: MyApp.RateLimit.Agent,
    agent_server: MyApp.RateLimit.Server,
    persistence: MyApp.RateLimit.Persistence,
    topology: MyApp.RateLimit.Topology,
    vsn: 1,
    option_keys: [agent: [:limit], agent_server: [:endpoint]]
end
```

Declare it in the Agent's `agent` block:

```elixir
agent do
  schema MyApp.State.schema()
  plugin MyApp.RateLimit, config: [limit: 100, endpoint: "local"]
end
```

- Select only the facets the package needs. At most one facet per owner type.
- `vsn` is a positive integer.
- `option_keys`: if absent, each selected facet receives all common options; if
  present, it must assign every declared option to a selected facet.
- A package module can appear only once in one Agent definition. Aliased
  duplicate Plugin instances are not supported.
- The Agent declaration supplies common package options; facets validate their
  own options.

## Facet Responsibilities

| Facet | Purpose |
| --- | --- |
| `Jido.Agent.Plugin` | Pure input preparation, Plugin-owned state, state reduction |
| `Jido.AgentServer.Plugin` | Live admission, runtime root, readiness, outbound preparation, post-commit dispatch |
| `Jido.Persistence.Plugin` | Pure dump/load of one paired owned-state value |
| `Jido.Topology.Plugin` | Pure static Topology contribution (Bus resources, ownership, subscriptions) |

## Agent Facet

```elixir
defmodule MyApp.TurnCount.Agent do
  use Jido.Agent.Plugin

  @impl Jido.Agent.Plugin
  def state_spec(_opts) do
    {:turn_count, Zoi.integer() |> Zoi.min(0) |> Zoi.default(0)}
  end

  @impl Jido.Agent.Plugin
  def reduce(%Jido.Agent.Plugin.Reduction{} = reduction, _opts) do
    {:ok, reduction.plugin_state + 1}
  end
end
```

Callbacks: `prepare/2`, `state_spec/1`, `directives/1`, `reduce/2`.

- `prepare/2` runs in package declaration order **before route selection**. It
  receives `%Jido.Agent.Plugin.Preparation{}` (source Signal, Agent identity,
  Agent module, complete current state, owned state) plus static options. It
  returns `{:ok, portable_input}` or `{:error, reason}`. The value is stored at
  `context.plugin_inputs[Package].prepared`. It cannot replace the Signal,
  caller context, route, or Agent state.
- `state_spec/1` declares the single owned state key and its static schema.
  The complete Agent schema combines domain fields and all Plugin-owned field
  schemas.
- `reduce/2` runs after executable success and Directive validation, in package
  declaration order. It receives a read-only `%Jido.Agent.Plugin.Reduction{}`
  (prior complete state, current complete candidate state, owned value, pure
  prepared input, all validated Directives) and returns **only** the next value
  for its owned field. Jido validates it with the owned schema and the
  portable-value rule.
- `directives/1` declares custom Directive modules; each owns its
  `Jido.Agent.Directive.validate/1` callback.

## Owned State

- One Plugin owns one key in the complete `agent.state` map. There is no
  separate Plugin state map or Agent struct field.
- Executables may read the key but must preserve it. If an Action deletes or
  replaces a Plugin-owned key, finalization rejects the candidate; the live
  Agent keeps prior state and dispatches no Directives.
- Only the owning Agent facet's `reduce/2` changes the value, before commit.
- `plugin_state` in callback contexts is a bounded view, not a second store.
- If the owned value is an object that can be absent, put the default on the
  outer object; a nested default does not create the outer object.

## Agent Server Facet

```elixir
defmodule MyApp.RateLimit.Server do
  use Jido.AgentServer.Plugin

  @impl Jido.AgentServer.Plugin
  def admit(runtime_ref, %Jido.AgentServer.Plugin.Admission{} = admission, opts) do
    if admission.state_version >= 0, do: {:ok, %{checked_at: System.monotonic_time()}}, else: {:error, :rejected}
  end

  # One optional permanent runtime root:
  # def child_spec(%Jido.Plugin.Init{} = init), do: %{id: __MODULE__, start: {__MODULE__, :start_link, [init]}, restart: :permanent}
end
```

Callbacks: `admit/3`, `prepare_dispatch/4`, `dispatch/4`, `await_ready/2`,
plus OTP `child_spec/1`.

- `admit/3` (`runtime_ref`, Admission, opts) runs in declaration order after
  pure preparation and before Turn evaluation. It receives a read-only
  `%Jido.AgentServer.Plugin.Admission{}` (Signal, caller context, Agent
  identity, owned state, pure prepared input, current state version). It
  returns `{:ok, runtime_input}` or rejects the command. The value is stored at
  `context.plugin_inputs[Package].runtime` and may contain PIDs, references, or
  functions because it never enters Agent state or a checkpoint.
- Direct `Jido.Agent.cmd/3` runs pure preparation but **not** live admission.
- `prepare_dispatch/4` runs in reverse declaration order to prepare outbound
  Signals.
- `dispatch/4` handles owned Directives after commit and receives a bounded
  `Jido.Plugin.DirectiveContext` with the committed owned state value.
  Dispatch failure does not roll back the commit. A Plugin without a runtime
  child receives `nil` as `runtime_ref`.
- `await_ready/2` gates startup; readiness failure stops the owner.
- `child_spec/1` must return `restart: :permanent`. The Server wraps each root
  generation in a temporary owner so a replacement root gets a fresh committed
  state/version pair. Do not do this work in the facet module itself.

Each runtime receives `%Jido.Plugin.Init{}` with `agent_server`, `agent_id`,
`module`, `plugin_state`, `state_version`, `jido`, `partition`, and `options`.
`plugin_state` is the immutable committed view for that runtime generation.

Read later committed state from a running resource with:

```elixir
Jido.Plugin.state(init, 5_000)
```

## Persistence and Topology Facets

- `Jido.Persistence.Plugin` implements `dump/3` and `load/3` for **one paired
  owned value**. It gets a `Jido.Persistence.Plugin.Context` (package, record
  versions, direction, reason) and mapped static options. It never receives
  the adapter, record key, expected revision, complete Agent, or commit result.
  Dump output must be portable; load output must match the Agent facet schema.
  A complete custom Agent checkpoint bypasses this conversion.
- `Jido.Topology.Plugin` implements `contribute/2` with a
  `Jido.Topology.Plugin.Context` (package version, static Agent identity). It
  returns bounded canonical entries: Bus resources, ownership relationships,
  and Bus subscriptions. It cannot start processes, persist data, or control
  live activation.

## Trust Boundary

The callback API prevents accidental cross-owner changes; it is not a sandbox
for untrusted BEAM code. Runtime handles never enter Agent state or a
checkpoint.
