# Jido Topology Reference (v3)

Source guides: `topology-definitions.md`, `topology-dsl.livemd`,
`topology-builders-codecs-and-composition.md`, `activate-and-repair-a-topology.livemd`,
`core-scope.md`.

A Topology is a declarative description of Agents, groups, resources, and
logical ownership. Declaration, validation, and planning start no processes.

## Main Values

| Value | Purpose |
| --- | --- |
| Topology definition | Static components, references, startup policy |
| Topology instance | Definition plus validated input and instance ID |
| Topology plan | Expanded members, exact nodes, dependency layers |
| Controller | Live activation, placement, and repair process |

Topology input is not Agent state; it supplies values used to build initial
component configuration.

## Declare a Topology

```elixir
defmodule MyApp.System do
  use Jido.Topology, name: "my_system"

  topology do
    agents do
      agent :leader, MyApp.Cell do
        initial_state %{label: "leader"}
      end

      agent :worker, MyApp.Cell do
        initial_state %{label: "worker"}
      end
    end

    relationships do
      owns :leader, :worker do
        on_parent_exit :stop
      end
    end

    startup do
      concurrency 2
      max_agents 10
      task_timeout 5_000
    end
  end
end
```

- Root blocks: an optional `agent do` (control Agent) plus `routes do` using
  normal Agent route syntax, and `topology do` for all topology-specific
  sections. The control Agent and topology are independent values.
- Use singleton Agents for fixed roles; counted groups for repeated identical
  roles; keyed groups when input records provide stable member IDs.
- `resources` is not Bus-specific, but a Bus is the first and only current
  core resource type. A Bus resource is one owned Jido Signal Bus. A
  subscription connects an Agent or group to one Bus path. An ownership
  relationship gives one Agent logical parent responsibility for another
  Agent or group.
- Use a finite `max_agents` and startup concurrency. The controller starts
  only the expanded plan.

## Instantiate, Plan, Activate

```elixir
topology = MyApp.System.new!(id: "system-1", input: %{})
# topology.plan.agents, topology.plan.layers

{:ok, _jido} = Jido.start()

{:ok, controller} =
  Jido.Topology.Controller.start_link(
    jido: Jido.default_instance(),
    topology: topology
  )

:ok = Jido.Topology.Controller.await_ready(controller)

leader = Jido.Topology.Controller.whereis_agent(controller, :leader)
state = Jido.AgentServer.agent(leader).state
```

- `Controller.start_link/1` `:jido` takes a Jido instance **name** (for
  example `Jido.default_instance()` or `MyApp.Jido`), not the PID returned by
  `Jido.start/1`; a PID fails validation.
- Instantiation validates input, expands groups, resolves references, assigns
  stable IDs, resolves exact nodes, checks startup bounds, and builds
  dependency layers. It does not check external service readiness.
- `Controller.await_ready/2` waits for activation; `status/2` inspects errors.
- Programmatic construction: `Jido.Topology.Builder`; authoring documents:
  `Jido.Topology.Codec` with the shared trusted `Jido.Codec.Registry`.

## Repair and Change

```elixir
{:ok, controller} =
  Jido.Topology.Controller.start_link(
    jido: MyApp.Jido,
    topology: topology,
    repair: :manual
  )

:ok = Jido.Topology.Controller.reconcile(controller)
```

- Default `repair: :automatic` repeats passes at the definition's
  `startup.retry_interval`. Manual mode performs initial startup once and then
  waits for explicit `reconcile/2` requests.
- Requests during an active pass produce one follow-up pass and share the
  startup concurrency limit. Healthy Agents retain their PIDs and committed
  state.
- `Controller.update/3` adds Agent entries to a ready controller. Topology ID,
  resources, and all existing Agent specifications must stay identical.
  Removal, changed definitions, resource changes, and ownership transfer
  require Controller replacement.
- `Controller.place_agent/4` moves one planned Agent to an exact Erlang node
  after a control Agent or Plugin selects it. A target node must run
  compatible code and the same named Jido instance; core does not fall back to
  a local node. The new Agent restores through shared persistence, or starts
  from its declared initial state without it.

## Lifecycle Signals

Pass a control Agent PID or `Jido.Agent.Ref` as the Controller `:lifecycle`
option. The Controller sends best-effort `jido.topology.lifecycle.**` Signals
for operations, component results, and status changes; each event is a custom
`Jido.Signal` module under `Jido.Topology.Signal`. These use normal Agent
routes. An observation failure cannot change a Controller result.

## Static Plugin Contributions

An Agent can declare a Plugin package with a `Jido.Topology.Plugin` facet.
When Jido builds an instance plan, the facet can contribute Bus resources,
ownership relationships, and Bus subscriptions for the Agent declaration.

- Jido applies Agent contributions first, then group contributions, keeping
  source and Plugin declaration order. Included Topologies expand in their own
  scope.
- The validated source definition stays unchanged; the complete expanded graph
  is validated before activation. Duplicate keys, unknown endpoints, duplicate
  subscriptions, or cycles fail before the Controller starts.
- The facet is pure static planning. It cannot add Agents or new resource
  types, start processes, persist state, or control live activation.

## Scope Limits

Supported: readiness, normal Bus input, logical ownership, automatic or manual
repair, additive Agent updates, lifecycle Signals, exact known-node placement.

Not provided: membership discovery, placement selection, automatic rebalance,
live removal, distributed authority, work ownership, or cluster-exclusive
ownership. Use child Directives when a live Agent decides to create or stop
workers from changing domain state; use a Topology for a system whose desired
members are known from one static definition and instance input.
