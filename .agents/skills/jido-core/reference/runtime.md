# Jido Runtime Reference (v3)

Source guides: `jido-instances.livemd`, `start-and-address-agents.livemd`,
`runtime.md`, `agent-server-lifecycle.md`, `calls-casts-and-requests.livemd`,
`configuration.md`, `storage.md`, `compare-and-swap-hibernate-and-thaw.md`,
`child-agents.livemd`.

## Instances

A Jido instance owns the local Registry, task supervisor, Agent supervisor,
and runtime coordination store.

```elixir
defmodule MyApp.Jido do
  use Jido,
    otp_app: :my_app,
    namespace: "my-app/primary",
    persistence: {Jido.Persistence.ETS, table: :my_app_agents}
end

# Application supervision tree, before components that start Agents:
children = [MyApp.Jido]
```

- Add the instance module to the supervision tree; the installer/generators
  were removed in v3.
- `:namespace` enables stable Agent Ref operations; it must be a nonempty
  binary and unique among live local instances on one node.
- `:persistence` sets the default adapter for the instance. Jido does not start
  or supervise the storage process.
- Supported instance options: `:name`, `:otp_app`, `:namespace`, `:max_tasks`
  (default 1_000), `:persistence`, `:debug`. Unknown keys fail validation.
  Runtime options override application config:

  ```elixir
  config :my_app, MyApp.Jido, max_tasks: 2_000, debug: false
  ```

- The generated instance functions mirror the top-level `Jido` API with the
  instance name supplied (`MyApp.Jido.start_agent/2`, `list_agents/1`, ...).
- Use a fixed application module or another finite atom as the instance name.
  Never create atoms from external tenant IDs.

## Start, Find, Stop

Top-level defaults target the default instance; pass an instance first only
when the application runs more than one Jido supervisor.

```elixir
{:ok, _jido} = Jido.start()

{:ok, server} = Jido.start_agent(MyApp.Counter, id: "counter-1", initial_state: %{count: 0})
{:ok, agent} = Jido.start_agent(my_agent_value)          # an instance value
{:ok, server} = MyApp.Jido.start_agent(MyApp.Counter, id: "counter-1")

pid = Jido.whereis_agent("counter-1")                    # pid or nil
pid = Jido.whereis_agent("counter-1", partition: :north)
pid = MyApp.Jido.whereis_agent("counter-1")
servers = Jido.list_agents()
count = Jido.agent_count()
:ok = Jido.stop_agent(server)
```

- `whereis_agent/1..3` returns a `pid` or `nil` directly — not a tagged
  tuple. A Server appears only after Plugin readiness and any required
  initial persistence write.
- Use the same instance and partition for startup, lookup, stop, persistence,
  hibernate, and thaw.
- `initial_state` is a Server startup option; `state` is the instance
  constructor option.
- Ref-first facades exist for stable identity: `start_agent_ref/2..3`,
  `call/4`, `cast/3`, `stop`, `hibernate`, `resolve_agent/2`,
  `agent_ref/2..3`.

## Agent Server Queries

```elixir
agent    = Jido.AgentServer.agent(server)      # current Agent data
snapshot = Jido.AgentServer.snapshot(server)   # Agent + commit revision
status   = Jido.AgentServer.status(server)     # execution/queue status
children = Jido.AgentServer.children(server)   # public child view
state    = Jido.AgentServer.plugin_state(server, MyPlugin)
```

- The Server is a `:gen_statem`. The v2 GenServer State struct and its
  Strategy/snapshot shapes are gone; use the public queries above.
- `status/2` covers live phases: initialization, idle work, admission,
  executable work, Directive dispatch.
- `snapshot/2` is the revision view; `Jido.Agent.Turn.Outcome` is the terminal
  Turn view.

## Calls, Casts, Requests

```elixir
{:ok, committed} = Jido.AgentServer.call(server, signal, 5_000)
:ok = Jido.AgentServer.cast(server, signal)

request_id = Jido.AgentServer.send_request(server, signal, 5_000)
{:reply, {:ok, committed}} = Jido.AgentServer.receive_response(request_id, 5_000)
```

- Input is always a `Jido.Signal`. A successful call returns the committed
  Agent. `cast` is fire-and-forget.
- `send_request/3` + `receive_response/2` separate sending from waiting;
  `receive_response/2` returns the raw `:gen_statem` reply
  (`{:reply, {:ok, agent}}` or `{:reply, {:error, reason}}`).
- A caller timeout does not cancel active work. Use `cancel/2` or
  `cancel_turn/3` (stable Turn ID) explicitly. Cancellation affects eligible
  pre-commit work; it does not roll back a commit, an external effect, or
  Directive work.
- `await_ready/2` waits only for Plugin runtime readiness, never for domain
  completion.

## Lifetime Controls

| Function | Effect |
| --- | --- |
| `attach/2` | Monitor an owner and prevent idle shutdown; repeated attach is idempotent |
| `detach/2` | Remove an attachment; owner death also removes it |
| `touch/1` | Reset the idle timer |
| `hibernate/2` | Persist and stop an idle Server (needs persistence) |
| `thaw/2..4` | Restore and start by module + Agent identity |

`idle_timeout` defaults to `:infinity`; the last attachment removal starts the
idle timer. Hibernation is not available during active Turn work; an active
record does not mean a Server is live.

## Actor Options

Pass actor options to `start_agent/2`:

```elixir
{:ok, server} =
  MyApp.Jido.start_agent(MyAgent,
    id: "agent-42",
    partition: :north,
    max_postponed_signals: 200,
    turn_timeout: 10_000,
    max_directives_per_turn: 20,
    directive_timeout: 3_000,
    idle_timeout: 60_000,
    restore: :if_found,
    error_policy: :stop_on_error,
    debug: false
  )
```

| Option | Default | Purpose |
| --- | --- | --- |
| `:max_postponed_signals` | `1_000` | Queued Signals while one Turn is active |
| `:turn_timeout` | `5_000` | Active pre-commit admission and candidate work |
| `:max_directives_per_turn` | `:infinity` | Directive work from one Turn |
| `:directive_timeout` | `5_000` | Post-commit Directive handling |
| `:idle_timeout` | `:infinity` | Stop a pool-owned idle actor |
| `:restore` | `:if_found` | Durable restore behavior |
| `:error_policy` | `:log_only` | Server behavior after a failed Turn |
| `:on_parent_death` | `:stop` | Child behavior when its logical parent ends |
| `:debug` / `:debug_max_events` | `false` / `500` | Local debug buffer |

Error policies: `:log_only`, `:stop_on_error`, `{:max_errors, count}`,
`{:emit_signal, dispatch}`, or a two-arity function. A required persistence
write error always stops that activation, even when the configured policy
would continue.

## Persistence

```elixir
defmodule MyApp.Jido do
  use Jido,
    otp_app: :my_app,
    persistence: {Jido.Persistence.Ecto, repo: MyApp.Repo}
end
```

- Adapter contract: binary `get/2` and atomic exact-byte
  `compare_and_swap/4`; `put/3` and `delete/2` are optional maintenance. A get
  followed by a put is **not** a valid CAS implementation.
- A new persistent activation writes revision zero before its start call
  succeeds. A successful commit is stored before the Server reports success.
  A stale revision fails; an indeterminate write result stops that writer.
- Restore validates identity, definition revision, complete state, and
  recursive portability.
- Normal durable delete writes a compact tombstone; it does not remove the
  storage key.
- Key formats: unnamed `jido:agent:v1:` (outer format 2); namespaced
  `jido:agent:v2:` (outer format 3). The reader also accepts v3
  outer-format-1 active records. V2 and Actor records need an explicit
  offline application conversion; do not point old and new writers at the same
  logical records.
- Without a persistence adapter, the instance RuntimeStore keeps checkpoints
  only for local abnormal restarts while the instance stays alive. A clean
  stop deletes them; instance or VM loss loses them.
- ETS is for development/tests; File requires one BEAM owner per directory;
  Redis TTL also limits tombstone retention and the stale-writer fence.
- The Bedrock adapter ships in the beta, but its real-service and MinIO
  snapshot tests are skipped pending an upstream fix — do not rely on it for
  proven durability.

## Restart, Upgrade, Hibernate

- An abnormal Agent Server restart can recover the instance runtime checkpoint
  while the instance stays alive. A normal stop removes it. A restarted Server
  keeps the Agent ID; it gets a new activation ID.
- `Jido.AgentServer.upgrade/2` runs a zero-arity installation operation after
  the Server becomes idle. `upgrade/3` migrates the committed Agent to a target
  module: same identity and Plugin declarations, validated and checkpointed
  before it becomes visible, then one state-version advance.
- `Jido.hibernate(instance, server)` saves and stops; `Jido.thaw(module, id)`
  restores. Cross-module durable migration requires a stable namespace key.

## Children

```elixir
child = Jido.Agent.Directive.spawn_child(MyApp.Worker, :worker)
Jido.AgentServer.children(server)   # public view; do not depend on private maps
```

- Children are tracked by relationship tag; `SpawnChild` supports remote
  placement through `node:`.
- `AdoptChild` attaches an existing live child; `StopChild` stops one tracked
  child.
- A remote disconnect does not prove a child is dead. Local duplicate
  registration checks do not establish cluster-exclusive ownership.

## Observability

```elixir
:ok = MyApp.Jido.debug(:on)
status = MyApp.Jido.debug_status()
Jido.AgentServer.set_debug(server, true)
events = Jido.AgentServer.recent_events(server)
```

```elixir
config :jido, :telemetry,
  semantic_log_mode: :interesting,      # :off | :errors | :interesting | :all
  semantic_slow_threshold_ms: 250
```

`Jido.Telemetry.metrics/0` returns the low-cardinality semantic metric set.
OpenTelemetry is optional; the host owns the SDK, and `config :jido,
:opentelemetry, enabled: false` disables the Jido mapping while an SDK stays
active. `Jido.Telemetry.open_telemetry?/0` checks for an active SDK tracer.
