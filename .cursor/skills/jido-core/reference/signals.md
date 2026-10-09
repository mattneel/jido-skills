# Jido Signals Reference (v3)

Source guides: `signals-commands-and-routes.md` (jido),
`signals-and-dispatch.md`, `signal-router.md`, `event-bus.md` (jido_signal).

A Signal is the typed message boundary: Agent commands, emitted events, and
Bus records all use `Jido.Signal`.

## Signal Envelope

`Jido.Signal` uses CloudEvents 1.0 context attributes. Struct fields:
`id`, `source`, `type`, `specversion` (always `"1.0"`), `subject`, `time`,
`datacontenttype`, `dataschema`, `data`, `data_present?`, `data_base64?`,
`extensions`.

Required attributes: `specversion`, `id`, `source`, `type`. Jido generates
UUID7 IDs. A legacy `"1.0.2"` specversion is accepted on input and normalized.

```elixir
{:ok, signal} =
  Jido.Signal.new("user.created", %{user_id: "123"}, source: "/accounts")

signal =
  Jido.Signal.new!("counter.increment", %{amount: 2}, source: "/example")

{:ok, signal} = Jido.Signal.new(%{type: "order.created", source: "/orders", data: %{id: "o-1"}})
```

- `type` carries stable event meaning; `source` identifies the producing part
  of the application.
- Put domain fields in `data`; extension context attributes are optional flat
  metadata.
- A Signal ID identifies the message, not a unique business request. Use your
  own operation IDs for idempotency.
- Custom Signal modules declare a type and data schema:

  ```elixir
  defmodule MyApp.UserCreated do
    use Jido.Signal,
      type: "user.created",
      default_source: "/accounts",
      schema: Zoi.object(%{user_id: Zoi.string()})
  end
  ```

- Canonical maps keep data in `data`; non-UTF-8 binaries use CloudEvents
  `data_base64`. JSON serialization accepts only JSON values in `data`
  (string keys; atom-keyed maps fail with `:json_encode_failed`).
- `Jido.Signal.ID.generate!/0` generates UUID7 identifiers (the v2 `Jido.ID`
  module is removed).

### Wire maps and serialization

```elixir
map = Jido.Signal.to_map(signal)
{:ok, signal} = Jido.Signal.from_map(map)

{:ok, binary} = Jido.Signal.Serialization.serialize(signal)
{:ok, signal} = Jido.Signal.Serialization.deserialize(binary)
{:ok, binary} = Jido.Signal.Serialization.serialize(signal, format: :erlang_term)
```

- Use `to_map/1` and `from_map/1` at storage and transport boundaries; do not
  read the struct as a wire map.
- The v3 writer uses string keys, omits nil optional fields, and flattens valid
  CloudEvents context attributes. The reader accepts legacy
  `jido_schema_version` 1 and 2 maps.
- Serialization accepts Signals only: JSON by default, or the trusted Erlang
  Term Format. The v2 serializer framework, type providers, MessagePack, and
  runtime serializer configuration are removed.
- Small routing metadata uses flat CloudEvents context attributes
  (`Jido.Signal.put_context/3`); names are lower-case alphanumeric, start with
  a letter, ≤20 chars, and values must use CloudEvents context types.

## Routing

`Jido.Signal.Router` maps Signal type patterns to ordered targets.

- Patterns: exact paths, `*` (one segment), `**` (multiple segments).
- Precedence: exact → `*` → `**` → pattern complexity → higher explicit
  priority (range -100..100) → earlier registration.
- Predicates run only after a path match. Runtime routers accept a unary
  function; compiled `use Jido.Signal.Router` modules accept only
  `{module, function, args}` MFA values.

```elixir
router =
  Jido.Signal.Router.new!([
    {"user.created", :create_user},
    {"user.*", :user_event},
    {"audit.**", :audit}
  ])

{:ok, [:create_user, :user_event]} =
  Jido.Signal.Router.route(router, Jido.Signal.new!(type: "user.created", source: "/example"))
```

Agent definitions use the same pattern language in their `routes` block. A
command uses the **first matching target**; no match returns
`Jido.Error.RoutingError` before executable work starts. Exact routes take
priority over matching wildcard routes.

## Route Defaults

A route can pair a target with a defaults map:

```elixir
routes: [
  {"counter.add", {MyApp.Add, %{amount: 1}}}
]
```

The merge with map-shaped Signal data is shallow; Signal data wins and invalid
supplied values do not fall back to defaults. The Signal in execution context
keeps its original data.

## Dispatch

Dispatch is synchronous. A target is an `{adapter, options}` tuple; a list of
targets is delivered in registration order. The caller owns async work,
retries, concurrency limits, and circuit breaking.

```elixir
:ok = Jido.Signal.Dispatch.dispatch(signal, {:bus, target: MyBus})
:ok = Jido.Signal.Dispatch.dispatch(signal, [{:pid, target: worker}, {:logger, level: :info}])
```

Targets are validated before delivery. Adapters require the `target:` key and
reject unknown keys: `{:bus, bus: MyBus}` and `{:pid, pid: worker}` fail
validation with `is required, at target` / `unrecognized key`. Built-in
adapters: `:pid`, `:named` (through the pid adapter), `:pubsub`, `:bus`,
`:logger`, `:noop`, `:http`. A custom adapter implements
`Jido.Signal.Dispatch.Adapter` (with `options_schema/0` and `deliver/2`).
`validate_opts/1` validates targets without delivering.

A `:bus` target resolves the bus by name. Pass `jido: <instance>` in the
target options when the Bus was started with that instance scope
(`Jido.Signal.Bus.start_link(name: MyBus, jido: MyApp.Jido)`); an unscoped
Bus is found by name alone. Agent Server emit directives inherit the running
Jido instance scope automatically (see `reference/runtime.md`).

In Agent Directives, `Jido.Agent.Directive.emit(signal, dispatch_target)` uses
the same target format. Delivery happens after commit.

## Bus

`Jido.Signal.Bus` provides ordered local publish/subscribe delivery with
at-least-once semantics.

```elixir
children = [{Jido.Signal.Bus, name: MyBus}]

{:ok, subscription_id} = Jido.Signal.Bus.subscribe(MyBus, "user.*")
{:ok, records} = Jido.Signal.Bus.publish(MyBus, [signal])
{:ok, records} = Jido.Signal.Bus.replay(MyBus, "user.*", after: cursor, limit: 100)
```

- `start_link/1` options: `:name` (required), `:jido`, `:registry`, `:store`,
  `:store_opts`, `:max_log_size` (default 100_000).
- `subscribe/3`: the calling process is the default `:target`. A normal
  subscription receives `{:signal, signal}` messages.
- A durable subscription (`durable: "stable-id"`) receives one
  `{:signal, durable_id, recorded_signal}` at a time and requires
  `ack(bus, durable_id, cursor)` before the next record. If the target exits,
  the Bus keeps the cursor and redelivers to a new process attaching with the
  same durable ID. `start_from: :origin` or a retained cursor reads older
  records.
- `unsubscribe/3` detaches a target; `delete_subscription/2` removes a durable
  cursor permanently.
- `publish/2` returns after the Store accepts all records.
- The included `Jido.Signal.Bus.Store.Memory` does not survive a Bus or VM
  restart. Applications can implement `Jido.Signal.Bus.Store` for durable
  records. The Bus does not own retry timers, dead-letter queues, leases, or
  competing-consumer policy.

## Agent Integration

- `Jido.Plugin.Bus` is the namespace for Bus integrations:
  `Jido.Plugin.Bus.Manager` when an Agent owns a Bus process, and
  `Jido.Plugin.Bus.Client` when an Agent subscribes to a Bus path (durable
  subscriptions call the Agent and acknowledge after its Turn commits).
- `Jido.Plugin.Dispatch` owns explicit post-commit Signal delivery.
- `Jido.Plugin.Heartbeat` sends periodic input Signals.
- Emitted Signals follow the commit-then-dispatch rule: a dispatch failure
  does not roll back committed Agent state.

## V2 → V3 Changes

| V2 | V3 |
| --- | --- |
| NimbleOptions Signal schemas | Zoi schemas only; static data, MFA callbacks, no lazy schemas |
| Serializer framework, MessagePack, custom markers | `to_map/from_map`, JSON, trusted Erlang Term |
| `dispatch_async/2`, `dispatch_batch/2`, batch/concurrency/retry options, Fuse | Synchronous ordered dispatch; caller owns async, retry, rate limits |
| Custom adapters with `validate_opts/1` | Adapter returns `options_schema/0` (Zoi) |
| Bus journal, partitions, middleware, snapshots, DLQ, `persistent?` | Retained replay with cursors, durable string IDs, `ack/3`, Store seam |
| `Jido.Signal.Journal`, `Jido.Signal.Ext`, `TraceContext`, `BusSpy`, `Util`, `Instance`, `Names` | Removed; custom Signal modules, explicit `Jido.Signal.Trace`, Bus telemetry |
| Timestamp-based `replay/3` | Cursor-based `replay/3` (`after:`, `limit:`) |
| Inferred source/event time | `source` required (or typed `default_source`); no inferred time or content type |

## Testing

- Assert `type`, `source`, `subject`, and relevant `data` fields.
- Router tests: overlapping patterns, priority, predicates, and no-match.
- Bus tests: use real Bus processes with explicit subscriptions and
  acknowledgements; do not add sleep-based ordering.
- Property-test serialization round trips when a custom codec is involved.
