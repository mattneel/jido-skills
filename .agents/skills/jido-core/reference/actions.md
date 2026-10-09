# Jido Actions, Flows, and Execution Reference (v3 beta)

Source guides: `actions-flows-and-instructions.md` (jido),
`actions.md`, `flows.md`, `execution.md`, `instructions.md`,
`configuration.md` (jido_action). Targets the **published** `jido_action`
betas (**beta.12+**; beta.11 drops Flow effects at the Agent boundary).

One Agent route selects one executable from Jido Action: an Action module, a
Flow module, or a runtime `%Jido.Flow{}` value.

## Define an Action

```elixir
defmodule MyApp.Actions.CreateGreeting do
  use Jido.Action,
    name: "create_greeting",
    description: "Creates one greeting",
    schema: Zoi.object(%{name: Zoi.string()}),
    output_schema: Zoi.object(%{message: Zoi.string()})

  @impl true
  def run(%{name: name}, context) do
    prefix = Map.get(context, :prefix, "Hello")
    {:ok, %{message: prefix <> ", " <> name <> "!"}}
  end
end
```

- `use` options: `name` (required), `description`, `schema`, `output_schema`;
  unknown options fail compilation.
- Generated: `name/0`, `description/0`, `schema/0`, `output_schema/0`,
  `validate_params/1`, `validate_output/1`, plus the `Jido.Executable`
  descriptor used by `Jido.Exec`. `run/2` is required.
- The published beta has no `to_json/0`.

### Callback results

```elixir
{:ok, result}                  # map or Jido.Action.Output
{:ok, result, effects}         # effects: proper list of opaque requests
{:error, reason}
{:error, reason, effects}      # third element is discarded
{:continue, input, target}     # run `target` next
```

- Effects are caller-owned requests. Exec treats them as opaque, does not
  dispatch them, and normalizes `[]` to the two-element form. `nil` or an
  improper list fails with `:invalid_effects`.
- Use `Jido.Action.Output.raw/1`, `stream/1`, or `batch/1` for intentional
  non-map success values.
- `{:continue, input, target}` is permitted from a **root Action** and from
  the expander of a `Jido.Flow.Dispatch` component at the end of a Flow.
  Other Flow positions reject it; the final target owns output validation.

### Validation boundaries

- A direct `run/2` call does **not** validate. Validate explicitly with
  `validate_params/1` / `validate_output/1`, or use `Jido.Exec.run/4`.
- `on_before_validate_params/1` may prepare raw input before Zoi parsing; it
  must return `{:ok, params}` or `{:error, reason}` and needs a final clause
  that passes other input through. Keep auth, secrets, and I/O out of it.
- Prefer Zoi coercion, defaults, enums, and refinements in the schema.

## Execution

```elixir
Jido.Exec.run(MyApp.Actions.CreateGreeting, %{name: "Ada"}, %{prefix: "Hi"}, timeout: 5_000)
```

- `Jido.Exec.run/4` validates the target and input, runs the callback,
  validates normal output, and converts raises, throws, exits, and invalid
  return shapes into structured errors.
- **Exec never retries.** No retry/backoff options exist; retryability in an
  error is information for a higher-level caller.

| Option | Default | Meaning |
| --- | --- | --- |
| `timeout` | `:infinity` | Complete-call limit |
| `task_supervisor` | `Jido.Exec.TaskSupervisor` | Local supervisor for workers |
| `max_continuations` | `256` | Continuations per complete call (0–10_000) |
| `max_concurrency` | `8` | Bounds concurrent ready Flow work |

Unknown options are errors (`max_attempts`, `max_retries`, and the old Flow
`async:` option are not accepted). `Jido.Exec.start/4` (paused step-wise
Flow execution) accepts only `task_supervisor` and `max_concurrency`.

### Errors

- Action: `Jido.Action.Error.InvalidInputError`, `ConfigurationError`,
  `ExecutionFailureError`, `TimeoutError`, `InternalError`.
- Flow: `Jido.Flow.Error.InvalidDefinitionError`, `InvalidExecutionError`,
  `ExecutionFailureError`, `TimeoutError`, `InternalError`.
- `Jido.Action.Error.to_map/1`, `Jido.Flow.Error.to_map/1`, and
  `Jido.Exec.Error.to_map/1` return the common public error map, including
  `retryable?`.

### Async calls

```elixir
handle = Jido.Exec.run_async(executable, input, context, opts)
{:ok, result} = Jido.Exec.await(handle)          # default wait: 5s
{:ok, result} = Jido.Exec.await(handle, 10_000)  # ms or :infinity
:ok = Jido.Exec.cancel(handle)
```

- The handle is owned by the calling process; only it can await, handle, or
  cancel. `Jido.Exec.handle_message/2` supports GenServer `handle_info/2`.
- A wait timeout cancels the work and returns
  `Jido.Exec.Error.AsyncTimeoutError`; the `timeout:` run option instead
  bounds target execution. `cancel/1` cannot undo completed side effects.

## Instructions

`Jido.Instruction` is data for one executable call.

```elixir
instruction =
  Jido.Instruction.new!(
    target: MyApp.Actions.SendEmail,
    params: %{to: "user@example.com"},
    context: %{tenant_id: "tenant-1"},
    metadata: %{request_id: "req-1"}
  )

Jido.Exec.run(instruction)
```

- Fields: `target`, `params`, `context`, `metadata`. Use `new/1` when
  construction can fail.
- Call-site params/context override equal keys (shallow merge); metadata is
  never execution policy. Instructions accept the Exec options of their
  target.
- Only Flow Instructions support step-wise execution.
- Instructions are not a storage format; store flows with
  `Jido.Flow.Codec`.

## Flows

A Flow is the canonical authoring value for a local workflow. It is data
until `Jido.Exec` runs it. A Flow selected by an Agent route is **one Turn**
and produces at most one commit; do not split one logical command into
separate live calls unless every intermediate state is valid and useful.

```elixir
defmodule MyApp.Flows.Quote do
  use Jido.Flow, name: "quote", schema: Zoi.object(%{quantity: Zoi.integer(), unit_price: Zoi.integer()})

  flow do
    step "price", action: MyApp.Actions.Price,
      params: %{quantity: input(:quantity), unit_price: input(:unit_price)}

    step "discount", total <- result("price", :total) do
      {:ok, %{total: total, discounted: div(total * 9, 10)}}
    end

    output result("discount")
  end
end
```

| Component | Does |
| --- | --- |
| `step` | Calls one Action, a child Flow (a `Jido.Flow.Subflow`), or an inline body |
| `choice` | Runs the first true option or the required fallback |
| `map` | Calls one Action per item; ordered fan-out/fan-in |
| `reduce` | Serial left fold through one Action |
| `iterate` | Bounded local loop with explicit state |
| `dispatch` | One end-of-Flow choice; may continue into the next executable |

- References create dependencies: `input(:x)`, `context(:key)`,
  `result("step", :field)`, `needs: [...]`. Source order alone never creates
  a dependency.
- Values support literals, references, and `Jido.Expr` operations (Boolean,
  comparison, arithmetic, concatenation). Expressions cannot call functions.
- A normal Flow output must be a map; use a `Jido.Action.Output` envelope for
  intentional raw/stream/batch/opaque values.
- At most one Dispatch, last in the Flow; it is unavailable in step-wise
  execution and Subflows.
- Effects: the Flow collects requests from all successful executed
  components in canonical dependency order (depth, then component name) and
  returns them with the final output. Failed, timed-out, or cancelled
  executions return no batch. (Use beta.12+; beta.11 drops Flow node
  effects.)
- Inspect: `Jido.Flow.validate/1`, `validate_executable/1`,
  `dependencies/1`, `explain/1`, `semantic_identity/1`, `Jido.Flow.compile/2`.
  None runs work.
- Run: `Jido.Exec.run(flow_or_module, input, context, max_concurrency: 4)`.
- Step-wise (Flows only): `Jido.Exec.start/1..4` → `continue/1` →
  `result/1` (plus advanced `ready/1`, `step/1..2`, `wave/1`).
- Storage: `Jido.Flow.Codec.encode/2` + `decode/2` with an application-owned
  `Jido.Flow.Registry`; `Jido.Flow.to_map/1` is not the stored format.

## Actions Inside an Agent

- The normal Agent result is the **complete next domain state**, based on
  `context.agent_state`. A partial map is not a state patch.
- Preserve Plugin-owned keys. Only the owning Agent Plugin facet changes them
  through `reduce/2`.
- Return Directives as the third element (`{:ok, state, directives}`) for the
  Server to dispatch after commit.
- Direct evaluation (`Jido.Agent.cmd/3`) returns the Directive list without
  dispatching it.
- Inline route actions compile to an Action; the route block requires a
  `signal_source` when it declares `define`:

  ```elixir
  routes do
    signal_source "/counters"

    route "counter.add" do
      action %{amount: amount},
        name: "counter_add",
        schema: Zoi.object(%{amount: Zoi.integer()}),
        context: context do
        {:ok, %{context.agent_state | count: context.agent_state.count + amount}}
      end

      define :add, args: [:amount]
    end
  end
  ```

- Under `jido_action` **beta.12+**, the Flow's collected effects become the
  Agent command's Directive list: direct `cmd` returns them, and a live Server
  validates, commits, and dispatches them in Flow effect order. Every entry
  must be a valid Directive — an ordinary (opaque) effect rejects the Turn
  before commit with `ValidationError` "Agent executable returned an invalid
  Directive". With beta.11 the Agent boundary silently drops Flow effects.
  For agent-routed Flows, return Directives (or a plain success), and let a
  Plugin own capability effects.
- Actions and Flows may perform I/O before commit. A later validation or
  storage failure cannot undo completed I/O. Use stable operation IDs and
  application idempotency for retried work.

## Reserved Context Keys (live execution)

The Server adds: `:agent_id`, `:agent_state`, `:plugin_inputs`, `:signal`,
`:jido`, `:partition`. Plugin inputs arrive as
`context.plugin_inputs[Package].prepared` and `...runtime`. The Jido Action
execution layer reserves `context.__jido_exec__`; read time budgets through
`Jido.Exec.remaining_time/1`.

## Not In the Published Beta

These exist only on the satellites' `release/v3` branches after the published
betas; do not use them with Hex packages. (The `jido_ai` pinned git refs are
newer than the Hex betas — see the `jido-ai` skill.)

- `Jido.Action.to_json/0`
- `Jido.Exec.compile/2`, `Jido.Exec.start/6` + `Runic.Runner` managed
  execution
- `Jido.Instruction.validate/1`, `resolve/3`, `template/3`, `bind/4`
- retry options such as `max_attempts`

See `reference/directives.md` for Directive contracts and outcomes.
