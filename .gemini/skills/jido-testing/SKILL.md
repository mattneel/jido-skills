---
name: jido-testing
description: >-
  Test Jido v3 actions, flows, plugins, routes, directives, and live actors
  with focused deterministic coverage. Use when adding tests, fixing test
  gaps, or validating Jido behavior at the right boundary.
metadata:
  author: agentjido
  version: "0.3.1"
license: Apache-2.0
---

# Jido Testing

## Purpose

Build focused tests that prove Jido v3 behavior at the cheapest reliable
layer: action/flow contracts first, direct Agent candidates next, live actor
commits and effects last.

## When To Use

Use this skill for:

- Tests for Jido actions, flows, or instructions
- Route selection and generated interface tests
- Direct candidate tests for agents
- Live `Jido.AgentServer` tests: commits, queries, cancellation, children
- Plugin tests at pure and runtime boundaries
- Directive and effect assertions
- AI/provider tests without live credentials
- Improving weak or flaky test coverage

Invoke `jido-core` first for shared ecosystem context.

## Requirements

- Inspect existing tests and helpers before adding patterns.
- Prefer repo-local test helpers and aliases.
- Avoid live network/provider dependencies in default tests.
- Keep tests behavior-focused; do not read private runtime state.
- Use a unique named Jido instance for each live test.

## Workflow

### 1. Inspect test conventions

```bash
rg "use ExUnit.Case|Jido.start|Jido.start_agent|Jido.Exec.run|use Jido.Action|use Jido.Agent" test lib
sed -n '1,180p' mix.exs
```

Record:

- Test helper modules and async conventions
- Repo aliases such as `mix quality`
- Mocking/bypass libraries and provider test strategy
- Existing runtime test patterns and cleanup style

### 2. Test actions directly

```elixir
{:ok, params} = Add.validate_params(%{left: 1, right: 2})
assert {:ok, %{sum: 3}} = Add.run(params, %{})
```

- A direct `run/2` call skips validation; validate params first when the
  callback relies on schema defaults.
- Cover: valid input, invalid input, domain failures, output shape, effects
  ordering, and caller-visible error structs.
- Add one `Jido.Exec.run/4` contract test: invalid input is rejected before
  `run/2`, output is validated, and failures become structured errors such as
  `Jido.Action.Error.InvalidInputError` (assert on the struct and its
  `details`, not message text). Exec never retries, so do not test retry
  counts.
- Async calls (`Jido.Exec.run_async/4` + `await/1,2`) and Flow step-wise
  execution (`start` → `continue` → `result`) have their own contracts; test
  them directly instead of sleeping or polling.

### 3. Test flows without running them

- `Jido.Flow.validate/1` checks structure, references, and cycles;
  `validate_executable/1` additionally checks Action and child Flow targets.
- `Jido.Flow.dependencies/1` and `explain/1` show the graph;
  `Jido.Flow.compile/2` derives the compiled value. None runs work.
- For stored flows, test a `Jido.Flow.Codec` round trip and expected errors for
  unknown identifiers.
- Run behavior through `Jido.Exec.run/4` and assert the complete result and
  effect order. Results and effects keep canonical order under any
  `max_concurrency`; assert that order.
- To prove an action ran or did not run, send a message and use
  `assert_received` / `refute_received`.

### 4. Test agents as data first

```elixir
agent = Counter.new!(id: "direct-test")
{:ok, signal} = Counter.add_signal(3)

{:ok, candidate, []} = Counter.cmd(agent, signal)
0 = agent.state.count        # original unchanged
3 = candidate.state.count    # complete next state

{:ok, invalid} = Counter.add_signal("three")
{:error, %Jido.Action.Error.InvalidInputError{}} = Counter.cmd(agent, invalid)
```

For agents, test:

- Initial state defaults and instance validation failures
- Route selection: exact vs wildcard, no-match `Jido.Error.RoutingError`,
  route defaults merging
- Complete next state; unrelated fields survive
- Plugin-owned keys: executables preserve them; tampering rejects the
  candidate
- Directives returned without dispatch
- Generated interfaces (`name_signal/n`, `name/n`) for exact routes

### 5. Test live actors explicitly

```elixir
jido = :"test_#{System.unique_integer([:positive])}"
{:ok, jido_pid} = Jido.start_link(name: jido)
{:ok, server} = Jido.start_agent(jido, TestGuide.Counter, id: "live-test")

before = Jido.AgentServer.snapshot(server)
{:ok, committed} = TestGuide.Counter.add(server, 4)
after_commit = Jido.AgentServer.snapshot(server)

0 = before.state_version
1 = after_commit.state_version
^committed = after_commit.agent

monitor = Process.monitor(server)
:ok = Jido.stop_agent(jido, server)
assert_receive {:DOWN, ^monitor, :process, ^server, _reason}, 5_000
:ok = Supervisor.stop(jido_pid)
```

- Start a unique Jido instance per asynchronous test to prevent registry and
  runtime state crossing cases.
- Cover: commit and state-version advance, `status/2`, `children/2`,
  `send_request/3` + `receive_response/2`, cancellation (`cancel/2`,
  `cancel_turn/3`), attach/detach idle behavior, and cleanup.
- Use `Jido.Exec`/Server timeouts and explicit options instead of sleeps.
- Test `hibernate`/`thaw` through configured persistence when that contract is
  in scope.
- Ordering/failure/replacement/cleanup checks use barriers, process monitors,
  replies, or explicit state barriers — **never** `Process.sleep/1` or a
  finite `refute_receive` as a completion signal.
- Capture logs only when log or redaction behavior is the contract
  (`capture_log: true` or `ExUnit.CaptureLog`).

### 6. Test plugins at both boundaries

- State-only Plugin: test option normalization, `state_spec/1` defaults,
  Directive validation, and `reduce/2` as ordinary functions. Then run one
  Agent Turn and verify Plugin-owned and domain fields commit together.
- Plugin runtime: start under a unique Jido instance and verify its child
  specification, startup failure behavior, Directive handling, cleanup, and
  restart behavior through messages or public inspection functions.
- Direct command tests run pure preparation but not live admission; test
  admission through a Server.

### 7. Test AI integrations without live providers

- Prefer request/response fixtures, deterministic model adapters, or local
  HTTP/SSE stubs.
- Test tool schemas, prompt/instruction rendering, and result parsing as pure
  units.
- Run live-provider smoke tests only when explicitly requested and credentials
  exist; keep them out of the default suite.

### 8. Validate

Run the smallest proof first:

```bash
mix test path/to/test_file.exs
```

Then run repo-standard validation. If no alias is documented:

```bash
mix test
mix compile --warnings-as-errors
```

The core jido repo separates larger profiles (`mix test.authoring`,
`test.system`, `test.services`, `test.examples`, `test.bench`). Run the
profile that covers the change instead of assuming the default suite proves it.

## Output Expectations

Report:

- Behavior covered at each layer
- Regression risk addressed
- Runtime or provider dependencies avoided or required
- Commands run and their results
- Remaining coverage gaps

## Guardrails

- Do not add sleeps for async behavior; use replies, monitors, or barriers.
- Do not make default tests depend on live APIs or credentials.
- Do not inspect private Server state, `:sys.get_state` shapes, or internal
  structs; use public queries.
- Do not test private implementation details when public behavior is enough.
- Do not weaken existing assertions to pass.
- Do not skip failing tests without explaining the product risk.
- Do not leak subscriptions, telemetry handlers, or instances between tests;
  clean up in `on_exit/1`.

## Reference Docs

- `jido-core/reference/signals.md` for routing and dispatch assertions
- `jido-core/reference/directives.md` for commit and directive semantics
- `jido-core/reference/runtime.md` for live actor APIs
