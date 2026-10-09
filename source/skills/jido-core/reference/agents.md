# Jido Agents Reference (v3)

Source guides: `agent-definitions-and-instances.md`, `agent-dsl.livemd`,
`state-schemas.livemd`, `route-interfaces.livemd`, `signals-commands-and-routes.md`,
`builders-and-codecs.md`, `authoring-extensions.md`.

## The Agent Value

`%Jido.Agent{}` has two valid forms:

| Form | ID | State | Purpose |
| --- | --- | --- | --- |
| Definition | `nil` | `nil` | Reusable schema, routes, Plugins, metadata |
| Instance | nonempty binary | complete map | One identified Agent with validated state |

A value with only an ID or only state is invalid. `Jido.Agent.definition?/1`
and `Jido.Agent.instance?/1` test the forms.

## Declare an Agent

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

- `name` is a required `use Jido.Agent` option; `description` is optional but
  recommended.
- The `agent` block declares the data schema, metadata, and `plugin`
  declarations. A field may appear in only one source: either `use` options or
  the DSL block, never both.
- The `routes` block declares Signal routes and optional generated interfaces.
  Declaring `routes do` requires an `agent do` block in the same module
  (otherwise compilation fails with "An agent block is required when routes
  are declared").
- The DSL is declarative. It produces a definition and helper functions; it
  never starts a process.

### Declared and generated functions

| Function | Purpose |
| --- | --- |
| `name/0`, `description/0` | Declared identity |
| `metadata/0` | Portable application metadata map |
| `schema/0`, `domain_schema/0`, `complete_schema/0` | Static Zoi data schemas; `complete_schema/0` includes Plugin-owned fields |
| `plugins/0` | Canonical ordered Plugin declarations |
| `routes/0` | Declared routes |
| `definition/0` | Neutral Agent definition; `Jido.Agent.definition/1` strips identity and `Jido.Agent.definition?/1` validates a value |
| `new/1`, `new!/1` | Instantiate from module defaults |
| `route_action!/1` | Generated Action module for an inline route (tests, codecs) |

## Instantiate

```elixir
{:ok, first} = MyApp.Counter.new(id: "counter-1")
{:ok, second} = Jido.Agent.instantiate(definition, id: "counter-2", state: %{count: 5})
agent = MyApp.Counter.new!(id: "counter-3")
```

- Module `new/1` returns `{:ok, agent}` or `{:error, reason}`. Use `new!/1`
  only where an exception is the intended error path.
- Instance options are only `:id` and `:state`; `:initial_state` is a Server
  startup option, not an instance constructor option.
- Instantiation applies schema defaults and validates the complete state.
  Never build an Agent struct by hand and assume it is valid.
- `Jido.Agent.set/2` merges domain fields and validates the complete result.
  Plugin-owned keys are protected.
- `validate/1` checks a value; `validate_definition/1` and
  `validate_instance/1` check one boundary. The v2 `:strict` option is gone.

## Static State Schemas

State schemas are static Zoi objects. Defaults, enums, and refinements belong
in the schema.

```elixir
defmodule MyApp.CounterState do
  def schema do
    Zoi.object(%{
      count: Zoi.integer() |> Zoi.default(0),
      minimum: Zoi.integer() |> Zoi.default(0),
      maximum: Zoi.integer() |> Zoi.default(10)
    })
    |> Zoi.refine({__MODULE__, :within_bounds, []})
  end

  def within_bounds(%{count: count, minimum: minimum, maximum: maximum}, _opts) do
    if minimum <= count and count <= maximum, do: :ok, else: {:error, "count out of bounds"}
  end
end
```

- Keep schemas static and portable. Refinements must use named module
  functions, not closures.
- The same schema runs for direct commands, live candidates, checkpoints, and
  restore.
- Do not store PIDs, ports, references, functions, or task handles in Agent
  state. Put them in Plugin runtimes.

## Routes

A route selects exactly one Action, Flow module, or runtime `%Jido.Flow{}`
value for one Signal.

```elixir
routes do
  signal_source "/orders"

  route "order.created", MyApp.Actions.CreateOrder
  route "order.cancelled", MyApp.Actions.CancelOrder
  route "order.*", MyApp.Actions.LogOrder
  route "order.priority", {MyApp.Prioritize, %{level: :high}}
end
```

- Patterns use Jido Signal Router syntax and match predicates. Jido uses the
  **first matching target in Router order**; exact routes take priority over
  wildcards. No match returns `Jido.Error.RoutingError` before any executable
  work starts.
- A `{target, defaults}` route merges defaults with map-shaped Signal data.
  The merge is shallow and Signal data wins; invalid supplied values do not
  fall back to defaults. The Signal in execution context keeps its original
  data.
- Prefer the declarative `routes do` block for new code. Programmatic route
  lists (`routes: [...]`) are accepted for dynamic assembly.

### Generated interfaces from `define`

An exact route that declares `define` (or `as:`, used by the `jido_ai` route
DSL) generates helpers. The same `routes` block must set `signal_source`, or
compilation fails with `signal_source is required for define` (or
`... for as:`):

```elixir
routes do
  signal_source "/orders"

  route "order.status.set", SetStatus do
    define :set_status, args: [:status]
  end
end

{:ok, signal} = Order.set_status_signal(:closed)
{:ok, signal} = Order.set_status_signal(:closed, input: %{request_id: "r-1"}, signal: [subject: "order-7"])
{:ok, committed} = Order.set_status(server, :closed, context: %{actor: "reviewer"}, timeout: 5_000)
```

- `name_signal/n` builds a Signal; `name_signal!/n` raises on invalid input.
- The live helper `name/n` builds the same Signal and calls
  `Jido.AgentServer.call/3`.
- Do not provide the same input key both positionally and through `input:`.
- `args: [{:optional, :name}]` marks optional arguments.
- Wildcard routes and match predicates cannot generate interfaces.

## Programmatic Construction

| Module | Use |
| --- | --- |
| `Jido.Agent.Builder` | Ordered programmatic definition/instance steps |
| `Jido.Agent.Codec` | Versioned authoring documents (transport, not checkpoints) |
| `Jido.Codec.Registry` | Trusted stable IDs to modules/schemas/executables |

All authoring forms use the same construction validation. Use explicit
allowlists where v2 used Discovery or dynamic module names.

## Authoring Extensions

```elixir
use Jido.Agent, extensions: [MyExtension]
```

An extension implements `c:Jido.Agent.Extension.lower_agent/2` to lower extra
DSL entities into normal schema/routes/Plugins. Core processes extensions in
declaration order. Every foreign entity must be consumed; duplicates and
unclaimed entities fail. Extensions are static: no processes, Actions, or
service calls inside a lowerer.

## Identity Refs

```elixir
ref = Jido.Agent.Ref.new!(namespace: "my-app/primary", partition: nil, id: agent.id)
{:ok, ref} = MyApp.Jido.agent_ref(agent.id, partition: "north")
{:ok, server} = MyApp.Jido.start_agent_ref(ref, MyApp.Counter)
{:ok, ^server} = MyApp.Jido.resolve_agent(ref)
```

A Ref carries `{namespace, partition, id}`; it never contains the module, PID,
node, or revision. The instance `namespace` must be configured to use
Ref-first operations. Ids, PIDs, and partitions remain valid in the current
runtime APIs.

## Running an Agent Value

```elixir
{:ok, signal} = Jido.Signal.new(%{type: "counter.increment", source: "/counter", data: %{amount: 2}})
{:ok, candidate, directives} = MyApp.Counter.cmd(agent, signal)
```

- `cmd/2..3` routes one Signal and returns `{:ok, candidate_agent, directives}`
  or `{:error, reason}`. The original Agent value is unchanged.
- A direct command validates and returns Directives but does not dispatch them.
- Direct evaluation does not run live Plugin admission. Use a live Agent
  Server for admission, commit, and post-commit work.

See `reference/plugins.md` for Plugin declaration and owned state, and
`reference/runtime.md` for live execution.
