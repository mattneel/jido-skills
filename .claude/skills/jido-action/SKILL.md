---
name: jido-action
description: >-
  Build and maintain composable Jido actions and flows with static Zoi
  schemas, explicit outputs and effects, predictable errors, and AI-tool
  compatibility. Use when defining actions, flows, inline steps, instructions,
  or action execution behavior.
metadata:
  author: agentjido
  version: "0.3.3"
license: Apache-2.0
---

# Jido Action

## Purpose

Create focused Jido actions, flows, and instructions that are easy to
validate, compose, execute, test, and expose to agents or tools. This skill
targets the **published** `jido_action` betas, not the moving `release/v3`
branch.

## When To Use

Use this skill for:

- New `use Jido.Action` modules
- Input/output schema changes
- `run/2` behavior and error handling
- `Jido.Exec` execution, timeouts, concurrency, and async calls
- `use Jido.Flow` graphs: steps, choices, map/reduce, iterate, dispatch
- Inline actions and steps inside flows or agent routes
- `Jido.Instruction` construction and execution
- Converting actions into AI-callable tools
- Debugging validation or execution failures

Invoke `jido-core` first for shared ecosystem context.

## Requirements

- Inspect local action and flow modules before writing new patterns.
- Confirm the repo resolves `jido_action ~> 3.0.0-beta.12`. Add the explicit
  dependency pin (or update the lock): jido beta.1 accepts beta.11, which
  silently drops Flow effects. With beta.12+ those effects are preserved and,
  at an Agent boundary, become the command's Directive list.
- Pin `zoi` to `0.18.7` on a fresh resolve if compilation fails with
  `Zoi.Types.Default is undefined` (newer zoi releases break the published
  beta).
- Use static Zoi schemas for input and output. There is no schema adapter.
- Pass `timeout:` explicitly on `Jido.Exec.run/4` calls that need a limit.
- Do not configure retries: the beta `Jido.Exec` never retries, and unknown
  options (for example `max_attempts:`) are rejected.

## Workflow

### 1. Inspect action conventions

Search before editing:

```bash
rg "use Jido.Action|use Jido.Flow|Jido.Exec.run|Jido.Instruction" lib test
```

Record:

- Schema style and output schemas in use
- Return shapes and error conventions
- Context keys read by `run/2`
- Whether actions are invoked directly, through flows, or by agents
- Tests for validation and execution

### 2. Define the action contract

Before coding, identify:

- Action name (required, non-blank, ≤256 bytes) and description
- Input schema and expected output schema
- Error cases and their structured reasons
- Effects that must defer to the caller
- Context fields read by `run/2` (caller-owned)
- Whether the action is safe to expose as an AI tool
- Idempotency key when an outer runtime may repeat work

Keep actions narrow. Use a Flow for multi-step work.

### 3. Implement the module

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

`use Jido.Action` options: `name` (required), `description`, `schema`,
`output_schema`; unknown options fail compilation. Generated functions:
`name/0`, `description/0`, `schema/0`, `output_schema/0`,
`validate_params/1`, `validate_output/1`, plus the `Jido.Executable`
descriptor used by `Jido.Exec`. The `run/2` callback is required. There is
no `to_json/0` in the published beta.

### 4. Return supported shapes

```elixir
{:ok, result}                     # map or Jido.Action.Output
{:ok, result, effects}            # effects: proper list of opaque requests
{:error, reason}
{:error, reason, effects}         # third element is discarded
{:continue, input, target}        # run `target` next (see below)
```

- Effects are caller-owned requests. `Jido.Exec` treats each as an opaque
  value, does not validate or dispatch them, and normalizes `[]` to the
  two-element success form. `nil` or an improper list fails with
  `:invalid_effects`.
- Use `Jido.Action.Output.raw/1`, `stream/1`, or `batch/1` when a success
  value is intentionally non-map.
- `{:continue, input, target}` is allowed from a **root Action** and from the
  expander of a `Jido.Flow.Dispatch` component at the end of a Flow. Other
  Flow positions reject it. The target is an Action or Flow and owns output
  validation and the final result.
- `on_before_validate_params/1` may coerce raw input before Zoi parsing. It
  must return `{:ok, params}` or `{:error, reason}` and include a final clause
  that passes other input through. Keep auth, secrets, I/O, retries, and
  compensation out of it.

### 5. Execute through Jido.Exec

```elixir
Jido.Exec.run(MyApp.Actions.CreateGreeting, %{name: "Ada"}, %{prefix: "Hi"}, timeout: 5_000)
```

- `Jido.Exec.run/4` validates the target and input, runs the callback in the
  execution process (or a supervised worker when a timeout is set), validates
  normal output, and converts raises, throws, exits, and invalid return
  shapes into structured errors.
- **Exec does not retry work.** Retryability in an error is information for a
  higher-level caller.

| Option | Default | Meaning |
| --- | --- | --- |
| `timeout` | `:infinity` | Complete-call limit; terminates the execution process and active work |
| `task_supervisor` | `Jido.Exec.TaskSupervisor` | Local Task.Supervisor for workers and async control |
| `max_continuations` | `256` | Continuations per complete call (0–10_000) |
| `max_concurrency` | `8` | Bounds concurrent ready Flow work |

- Unknown options are errors (`max_attempts:`, `max_retries:`, and the old
  Flow `async:` option are not accepted).
- `Jido.Exec.start/4` accepts only `task_supervisor` and `max_concurrency`; it
  starts a paused step-wise Flow execution.

Errors:

- Action boundary: `Jido.Action.Error.InvalidInputError`,
  `ConfigurationError`, `ExecutionFailureError`, `TimeoutError`,
  `InternalError`.
- Flow boundary: `Jido.Flow.Error.InvalidDefinitionError`,
  `InvalidExecutionError`, `ExecutionFailureError`, `TimeoutError`,
  `InternalError`.
- Use `Jido.Action.Error.to_map/1`, `Jido.Flow.Error.to_map/1`, or
  `Jido.Exec.Error.to_map/1` for the common public error map. It includes
  `retryable?` information for callers.

### 6. Run asynchronously when the caller must not block

```elixir
handle = Jido.Exec.run_async(executable, input, context, opts)
{:ok, result} = Jido.Exec.await(handle)        # default wait: 5s
{:ok, result} = Jido.Exec.await(handle, 10_000)
:ok = Jido.Exec.cancel(handle)
```

- `run_async/4` accepts the same targets as `run/4` and returns a handle
  owned by the calling process. Only that process can await, handle, or
  cancel it.
- Use `Jido.Exec.handle_message/2` in a GenServer `handle_info/2`; it returns
  `{:done, result}` for the exact completion message and `:ignore` otherwise.
- `await/2` accepts milliseconds or `:infinity`; if the **wait** expires,
  Exec cancels the work and returns `Jido.Exec.Error.AsyncTimeoutError`. The
  `timeout:` option on `run_async/4` is different: it bounds target
  execution.
- `cancel/1` stops active work; it cannot undo side effects that already
  completed. A second wait on the same handle returns
  `Jido.Exec.Error.InvalidHandleError`.

### 7. Compose with Flows

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

- Components: `step` (an Action, a child Flow module, or an inline body —
  a child Flow becomes a `Jido.Flow.Subflow`), `choice`, `map`, `reduce`,
  `iterate`, `dispatch`. Dependencies come from references and `needs:`;
  source order alone never creates one.
- Values: literals, `input(:x)`, `context(:key)`, `result("step", :field)`,
  and `Jido.Expr` operations (Boolean, comparison, arithmetic, concatenation).
  Expressions cannot call functions.
- `schema:` validates input; `output_schema:` validates output. A normal Flow
  output must be a map; use a `Jido.Action.Output` envelope for intentional
  raw, stream, batch, or opaque values.
- Dispatch: at most one, must be last, output is the complete Dispatch
  result. Dispatch is not available in step-wise execution or as a Subflow.
- Effects: a Flow collects requests from **all** successful executed
  components in canonical dependency order (depth, then component name) and
  returns them with the final output. Failed, timed-out, or cancelled
  executions return no effect batch. (At an Agent boundary under beta.12+,
  this batch becomes the command's Directive list — see step 9.)
- Inspect without running: `Jido.Flow.validate/1`,
  `validate_executable/1`, `dependencies/1`, `explain/1`,
  `semantic_identity/1`, `Jido.Flow.compile/2` (`%Jido.Flow.Compiled{}`).
- Run: `Jido.Exec.run(flow_or_module, input, context, max_concurrency: 4)`
  (`max_concurrency: 1` for serial scheduling).
- Step-wise (Flows only):

  ```elixir
  {:ok, execution} = Jido.Exec.start(flow_or_instruction, input)
  {:ok, execution} = Jido.Exec.continue(execution)
  {:ok, output, effects} = Jido.Exec.result(execution)   # or {:ok, output}
  ```

- Storage: use `Jido.Flow.Codec.encode/2` and `decode/2` with an
  application-owned `Jido.Flow.Registry`. `Jido.Flow.to_map/1` is for
  comparison, not the stored format.

### 8. Use Instructions for deferred calls

```elixir
instruction =
  Jido.Instruction.new!(target: MyApp.Actions.SendEmail, params: %{to: "a@b.com"}, context: %{tenant_id: "t-1"})

Jido.Exec.run(instruction)
```

- Fields: `target`, `params`, `context`, `metadata`. Targets can be Action
  modules, Flow modules, or runtime Flow values.
- Call-site params/context override equal keys (shallow merge); metadata is
  never execution policy. Instructions accept the Exec options of their
  target.
- Only a Flow Instruction supports step-wise execution.
- Instructions are not a storage format; store flows with `Jido.Flow.Codec`.

### 9. Integrate with an Agent

- The normal Agent result is the complete next domain state based on
  `context.agent_state`; a partial map is not a patch.
- Preserve Plugin-owned keys; only the owning Plugin facet's `reduce/2`
  changes them.
- Return Directives as the third element; the Server dispatches after commit.
- One Flow inside one Turn produces at most one commit. Do not split one
  logical command across live calls unless every intermediate state is valid
  and useful.
- Inline route actions use the agent DSL; the route block needs a
  `signal_source` when it declares `define`:

  ```elixir
  routes do
    signal_source "/counters"

    route "counter.add" do
      action %{amount: amount}, name: "counter_add", schema: Zoi.object(%{amount: Zoi.integer()}), context: context do
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
  before commit (`ValidationError` "Agent executable returned an invalid
  Directive"). With beta.11 the Agent boundary silently drops Flow effects.
  For agent-routed Flows, return Directives (or a plain success) and let a
  Plugin own capability effects.

### 10. Tool compatibility

- A tool is an Action or Flow with a stable name, validated input, and a
  model-readable result. `name/0`, `description/0`, `schema/0`, and
  `output_schema/0` are the model-facing contract (for example in `jido_ai`
  tool declarations).
- Keep `as:`/tool names short and stable; avoid internal-only structures at
  the boundary.

### 11. Validate

Run focused action/flow tests first:

```bash
mix test path/to/action_test.exs
```

Then run the repo-standard test or quality command. Tests should cover valid
input, invalid input, domain failures, output shape, effect ordering, and
caller-visible errors.

## Output Expectations

Report:

- Action/flow contract changed
- Validation behavior changed
- Error cases covered
- Effects and idempotency implications
- Tool-facing behavior, if any
- Tests run

## Guardrails

- Do not use v2 APIs: `Jido.Plan`, action chains,
  `Jido.Action.TaskSupervisor`, built-in retry/compensation policy, or
  NimbleOptions schemas.
- Do not use `release/v3`-only APIs with published betas: `to_json/0`,
  `max_attempts`, `Jido.Exec.start/6` with a Runic.Runner, or
  `Jido.Instruction.validate/1`. The beta async API is
  `run_async`/`await`/`cancel`.
- Do not configure retries in Exec options; own retry policy in the caller
  and only for idempotent work.
- Do not use actions as unstructured service objects; one action, one unit of
  work.
- Do not swallow errors or return ambiguous values.
- Do not use the process dictionary or global state.
- Do not put external effects on the caller's behalf: return them as effect
  requests or perform them inside the action, never in a Flow expression.
- Do not expose unsafe filesystem, network, or code-evaluation behavior as a
  model tool.
- Do not change return shapes without updating callers and tests.

## Reference Docs

- `jido-core/reference/actions.md` for the condensed execution contract
