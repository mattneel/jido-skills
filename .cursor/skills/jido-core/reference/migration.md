# Jido v2 → v3 Migration Reference

Source guides: `migration.md` and `api-migration-map.md` (jido v3.0.0-beta.1).

There is no v2 compatibility mode, automatic code rewrite, or automatic stored
data conversion. Port by boundaries, not by renames.

## Upgrade Order

1. Keep a working v2 revision and its lock file.
2. Inventory affected APIs and stored state.
3. Update dependencies and port one small Agent with a direct command.
4. Port state changes and command error handling.
5. Port Plugins, directives, and live process ownership.
6. Rehearse stored-data conversion separately from the code port.
7. Test application behavior, then review deployment and rollback.

Useful source searches:

```sh
rg -n 'Jido\.(Agent|AgentServer|Instruction|Plugin|Storage|Sensor|Pod|Memory|Discovery)' lib test config
rg -n 'signal_routes|on_before_cmd|on_after_cmd|StateOp|DirectiveExec|directive_handler|InstanceManager|WorkerPool' lib test config
rg -n 'mount|prepare_signal|prepare_action|transform_result|on_checkpoint|on_restore' lib test
```

## Main Boundary Changes

| V2 boundary | V3 boundary |
| --- | --- |
| Agent module creates a ready instance | Definition, then validated instance |
| `cmd` accepts Actions, Instructions, or lists | `cmd` accepts one Signal; a route selects one Action or Flow |
| Action result can be a state patch | Executable returns the complete next domain state |
| Strategy owns command policy and progress | Actions/Flows own domain work; Plugins own admission and runtime |
| Plugin options declare metadata, routes, schedules, state | Agent declares Plugin options and routes; each Plugin declares state and Directive contracts |
| Server state/status structs expose runtime detail | Public queries: agent, status, children, plugin-state field, snapshot |
| Pods/WorkerPool manage mutable groups | Static Topology plus application-owned capacity policy |
| Storage combines Agent checkpoints and Thread storage | Persistence stores versioned Agent checkpoints; Threads become application state |

## Required Ports by Area

### Agents

- `new/1` returns `{:ok, agent}`; use `new!/1` for raising paths.
- Schemas must become static Zoi data schemas.
- Replace patch returns with complete state from `context.agent_state`.
- Replace `signal_routes` with `routes/0` or the `routes` DSL block.
- Move Action sequences into one Flow when they must commit once.
- Replace `on_before_cmd/2` / `on_after_cmd/3` with Actions, Flows, routes,
  and Plugin callbacks.
- Remove Strategy modules and `:strict` validation options.

### Agent Server

- `call/3` and `cast/2` take a Signal; a successful call returns the committed
  Agent.
- Replace `state/1` reads with `agent/2`, `status/2`, `snapshot/2`,
  `children/2`, `plugin_state/3`.
- Replace `Jido.Await` with `call/3`, `send_request/3`,
  `receive_response/2`, and application coordination; `await_ready/2` only
  waits for Plugin readiness.
- Replace cancellation through Await with `cancel/2` or `cancel_turn/3`.
- `whereis/3` and `via_tuple/3` take an Agent Registry; prefer
  `Jido.whereis_agent/1..3`.
- InstanceManager, WorkerPool, lifecycle modules, and the v2 Strategy
  snapshot/status structs are removed.

### Directives

- Rebuild with v3 constructors: `Spawn`→`SpawnProcess`,
  `SpawnAgent`→`SpawnChild`; `Emit`, `AdoptChild`, `StopChild`, `Stop`,
  `Error` retained with changed contracts.
- `Cron`/`CronCancel`/`Schedule`→`Jido.Plugin.Scheduler.*`.
- `StartSensor`/`StopSensor`→`Jido.Plugin.SensorManager.*`.
- `RunInstruction` has no post-commit replacement; send a later Signal.
- `DirectiveExec` and custom `directive_handler` are replaced by Plugin
  Directive validation and `dispatch/4`.

### Built-In Actions

All `Jido.Actions.*` modules (Control, Lifecycle, Scheduling, Status) were
removed. Write application Actions that return complete state and supported
Directives.

### Plugins

- `use Jido.Plugin` is a callback-free manifest with owner facets.
- `mount/2`→static defaults in `state_spec/1`; runtime setup in the Agent
  Server facet's `child_spec/1` with `Jido.Plugin.Init`.
- `handle_signal/2`, `prepare_signal/2`, `prepare_action/3`→Action/Flow input
  handling plus `admit/3` for live checks.
- `transform_result/3`→Action/Flow domain change plus `reduce/2` for the owned
  field.
- `on_checkpoint/2`/`on_restore/2`→Persistence facet `dump/3`/`load/3`.
- Plugin Config/Instance/Requirements/Routes/Schedules modules are removed;
  Spec is private.

### Systems, Await, Scheduler, Sensors

- `Jido.Pod`→`Jido.Topology` + `Jido.Topology.Controller`; mutations are removed.
- `Jido.Await`→Server calls/requests, `children/2`, `cancel/2`.
- `Jido.Scheduler`→`Jido.Plugin.Scheduler` directives and routes.
- `Jido.Sensor` behavior→OTP child accepting `Jido.Plugin.SensorManager.Init`;
  Heartbeat→`Jido.Plugin.Heartbeat`.

### Storage and Application-Owned Data

- `Jido.Storage`→`Jido.Persistence` (binary get + exact-byte
  compare-and-swap). Adapters: ETS, File, Redis, Ecto, Bedrock. Bedrock ships
  in the beta but its service tests are skipped; do not rely on it for proven
  durability.
- `Jido.Thread`, `Jido.Memory`, identity profiles, and Discovery are removed;
  keep history/memory/profile policy in application state and use explicit
  modules plus `Jido.Codec.Registry`.
- V2 records need an explicit offline conversion. Store a V3 instance through
  the V3 API, keep V2 backups separate, and rehearse rollback before the first
  namespaced write. Never run old and new writers against the same logical
  records.

### Observation and Tooling

- `Jido.Observe.*` removed; use the v3 semantic Telemetry catalog and optional
  OpenTelemetry (`config :jido, :opentelemetry`).
- Old telemetry events (`span_agent_cmd/3`, `span_strategy/4`, legacy metrics)
  are removed.
- Removed Mix tasks: `jido.install`, `jido.gen.agent`, `jido.gen.plugin`,
  `jido.gen.sensor`. Add the instance module to the supervision tree instead.

## Completion Checks

- Constructors validate state and return the expected result shape.
- Direct and live commands preserve unrelated fields and protect Plugin-owned
  fields.
- Invalid input and failed work leave committed state unchanged.
- Effect retries obey application idempotency rules.
- Restart and restore rebuild owned runtimes without losing pending work.
- Remote failure and cancellation do not leak workers or resources.

For the complete module-level status table, use the upstream
[V2 to V3 API map](https://github.com/agentjido/jido/blob/v3.0.0-beta.1/guides/api-migration-map.md)
and [Upgrade guide](https://github.com/agentjido/jido/blob/v3.0.0-beta.1/guides/migration.md).
