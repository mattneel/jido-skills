---
name: jido-ai
description: >-
  Build AI agents on the Jido v3 line with jido_ai: AI profiles, tools,
  structured results, streaming, sessions, controls, and deterministic tests.
  Use when adding LLM behavior, model profiles, tool calling, or prompt work to
  a Jido project.
metadata:
  author: agentjido
  version: "0.3.2"
license: Apache-2.0
---

# Jido AI

## Purpose

Add or maintain AI behavior in Jido v3 projects with explicit profiles,
bounded execution, validated results, and no provider assumptions baked into
Agent source.

## When To Use

Use this skill for:

- New `use Jido.AI.Agent` modules
- AI profiles: models, instructions, reasoning methods
- Tool calling with Jido actions and flows
- Structured results, schemas, and repair
- Streaming, steering, and cancellation
- Sessions, Threads, Context lanes, and checkpoint/resume
- Controls, tool policy, quotas, and observability
- Testing AI agents without live providers

Invoke `jido-core` first for shared ecosystem context.

## Requirements

- Inspect local `mix.exs`, config, and existing AI modules first.
- The v3 AI stack is **not published to Hex**. Hex `jido_ai 2.x` is the v2
  line; `{:jido_ai, "~> 2.3"}` pulls v2 packages, not v3 behavior.
- Install `jido_ai` from git `release/v3` and mirror the pinned git refs from
  that branch's `mix.exs`; the refs move together. Current set (`jido_ai`
  `release/v3` at `1f19ff2`; refresh before use):

  ```elixir
  {:jido_ai,     git: "https://github.com/agentjido/jido_ai.git",     ref: "1f19ff262081bc13eb30da7bff8bfe42685d63c8", override: true},
  {:jido,        git: "https://github.com/agentjido/jido.git",        ref: "8322de574c53d5c2096243d230c9de4f14803bda", override: true},
  {:jido_action, git: "https://github.com/agentjido/jido_action.git", ref: "19c77f94e5c2513b2e684a289857fcd568f7eb29", override: true},
  {:jido_signal, git: "https://github.com/agentjido/jido_signal.git", ref: "fd8d00555d6a64b4109619f4c26f1b75e8a91d41", override: true},
  {:zoi,         git: "https://github.com/mikehostetler/zoi.git",     ref: "2fff2a23e23e7ac0b26f62f49bbc1b12f7818ac9", override: true}
  ```
- Mixing Hex core pins (`jido ~> 3.0.0-beta.1` and friends) with git
  `jido_ai` fails `mix deps.get` with "dependency jido … is overriding a
  child dependency". Use the pinned git set above and treat it as
  **unreleased**: this `jido` ref is newer than the `3.0.0-beta.1` tag, so
  some core behavior differs from the beta packages.
- The pinned `jido_action` ref is also newer than the Hex betas: it keeps
  `to_json/0` and `run_async`/`await`/`cancel`, and has no `max_attempts`.
  The "Not In the Published Beta" list in `jido-core/reference/actions.md`
  describes Hex packages; verify against the pinned git ref when the AI
  stack is in use.
- Treat API keys and provider secrets as environment/runtime configuration,
  never source-controlled values.
- Keep runtime values (PIDs, clients, request handles) out of Agent state.
- Default tests must not depend on live providers.

## Workflow

### 1. Inspect the local AI setup

```bash
rg "Jido.AI|MockLLM|req_llm|model|ai :|ask_sync|ask_stream" mix.exs config lib test
```

Record:

- Dependencies and pinned refs (jido_ai release/v3 pins jido from git)
- Provider configuration and model alias conventions
- Existing AI agents and their profile style
- Tool registration and result-mapping patterns
- Test mocking strategy (typically `Jido.AI.Test.MockLLM`)

### 2. Start with one profile

An AI Agent is a Jido Agent plus an AI profile. The Agent owns durable domain
state; the profile states the model, reasoning, tools, controls, and result
destination.

```elixir
defmodule MyApp.AnswerAgent do
  use Jido.AI.Agent, name: "answer_agent"

  agent do
    schema Zoi.object(%{answer: Zoi.string() |> Zoi.default("")})

    ai :assistant do
      model "openai:gpt-4o-mini"
      instructions "Answer in one short sentence."

      controls do
        timeout 10_000
        max_model_calls 1
      end

      result into: :answer
    end
  end

  routes do
    signal_source "/my_app"

    route "my_app.answer", ai: :assistant, as: :answer
  end
end
```

```elixir
{:ok, _jido} = Jido.start()
{:ok, server} = Jido.start_agent(MyApp.AnswerAgent)
{:ok, answer} = MyApp.AnswerAgent.ask_sync(server, "What is Jido AI?")
```

- A module definition does not start a process or call a model.
- A route that declares `as:` (or `define`) requires `signal_source` in the
  same `routes` block, or compilation fails with
  `signal_source is required for as:`.
- `ask_sync/3` waits; `ask/3` returns a request handle; `ask_stream/3` returns a
  handle plus an event stream. A core `as:` route also generates
  `answer_signal/1..2` (input map, optional opts).
- Keep one profile and one route until a second behavior is justified.
- Always set a timeout and a model-call limit so a provider cannot make the
  turn unbounded.

### 3. Choose models and instructions deliberately

- `model :fast` is an application-configured alias; keep provider choice out
  of Agent source when it is operational policy. A request-scoped model input
  changes one request, not the profile.
- Instructions state the task and output expectations. They are **not** a
  security or cost boundary: enforce tool access with policy, result shape
  with `result` validation, and cost with controls/quotas.
- Several profiles are for distinct work (for example `:answer` and
  `:review`), each with its own route.

### 4. Add tools

```elixir
reasoning :react

tools do
  action MyApp.Multiply, as: :multiply
  flow MyApp.BuildQuote, as: :quote
end

controls do
  timeout 30_000
  max_iterations 4
  max_model_calls 4
  max_tool_calls 3
end
```

- A tool is a Jido Action or Flow, never a bare Elixir call. Keep the `as:`
  name short and stable.
- The target must validate input before work starts and return structured,
  model-readable data. Flows must select their complete output explicitly.
- Give external I/O a timeout and decide retry safety. A tool can perform
  effects before the AI answer commits; a failed answer does not undo them.
- Pass trusted services through request `tool_context`, not Signal, Thread, or
  Agent state. A model's tool choice is not authorization.
- Narrow a request with `allowed_tools`; it can only reduce the declared set.
  Dynamic native tool sources are not part of the stable authoring contract.

### 5. Shape results

```elixir
result(Zoi.object(%{answer: Zoi.string() |> Zoi.min(1)}),
  into: :answer,
  max_repairs: 1
)
```

- `result into:` maps the validated answer to one domain field; unrelated
  state is preserved.
- `max_repairs` allows bounded retries with validation feedback. A repair is
  another model call, so `max_model_calls` must include it.
- Provider errors do not start schema repair. A failed request leaves the
  prior domain answer in place; request records may still show the attempt.
- Validation proves shape, not truth. Add business invariants after shape
  validation and before commit.

### 6. Pick reasoning and controls

- Default for one answer; `:react` when the model must choose a tool, read its
  output, and continue.
- Linear methods (Chain of Thought/Draft) shape one line of reasoning; tree
  and graph methods explore candidates and cost more calls. Adaptive selection
  chooses by policy. Compare methods on a measured task, not by name.
- Controls bound work: `timeout`, `max_iterations`, `max_model_calls`,
  `max_tool_calls`, plus stage controls (input, model, operation, output) and
  `steering true`.
- Put each rule at the earliest stage with enough information; reject input
  before model work when possible.
- Controls are not authorization, and a timeout does not make a
  non-idempotent tool safe to retry.

### 7. Run, observe, and control requests

```elixir
{:ok, request} = MyApp.Agent.ask(server, "Research the issue")
{:ok, answer} = Jido.AI.Request.await(request, timeout: 30_000)

{:ok, %{request: r, events: events}} = MyApp.Agent.ask_stream(server, "Review this change")
```

- The AI Coordinator owns live request work; AgentServer owns committed state.
  A request can outlive the admitting Turn, and a second AI request to a busy
  profile can be rejected — it is not queued.
- `Jido.AI.Orchestration.steer/3` queues a correction for a running request
  (acceptance means queued, not consumed).
- `Jido.AI.Orchestration.cancel/2` asks the runtime to stop a request. A
  caller wait timeout is not cancellation.
- Use request/run IDs to join admission, events, and settlement. Inspect live
  state with `Jido.AI.Orchestration.snapshot/2` and committed state with core
  AgentServer functions.

### 8. Sessions, Threads, and Context

```elixir
agent do
  schema Zoi.object(%{
    reply: Zoi.string() |> Zoi.default(""),
    messages: Jido.AI.Thread.Projection.schema()
  })

  ai :assistant do
    model "openai:gpt-4o-mini"
    memory history: :messages
    result into: :reply
  end
end
```

- `Jido.Session` is a portable value holding a `Jido.Thread`; a Context is the
  selected conversation the next model request sees; the live request is
  runtime work, not a Session value.
- `memory history: :messages` tells the profile where to find retained
  conversation. Without it, retained request records do not create context.
- Raw Threads can hold settlement and application entries; not every entry
  becomes provider input. Use `Jido.AI.Thread.Projection.select/1` and
  `messages/1` instead of copying projection rules.
- Named Context lanes: `Jido.AI.Orchestration.modify_context/3` with
  `%{type: :replace | :switch}` and a stable `op_id`; repeated applied IDs are
  idempotent. A change during active work can defer until settlement.
- Continuation forms: import initial Context with
  `Jido.AI.Agent.from_initial_state/3` (validates Session, replays nothing);
  or use an execution checkpoint at a supported ReAct boundary (completed tool
  results are retained; partial tool batches and effect rollback are not
  supported).

### 9. Fail safely

- Classify failures before retrying at the application boundary: transient
  provider error vs schema/contract problem vs possibly-completed tool effect.
- Never replay a non-idempotent tool without a key or an application check.
- Keep new request IDs for independent attempts.
- Do not treat a caller timeout as proof that the server cancelled work.

### 10. Test deterministically

```elixir
{:ok, mock} = Jido.AI.Test.MockLLM.start_link(script: [%{reply: {:text, "Ready"}}])
context = %{ai: %{assistant: %{options: Jido.AI.Test.MockLLM.options(mock)}}}

{:ok, answer} = MyApp.Agent.ask_sync(server, "Help", context: context)
%{remaining: [], unexpected: []} = Jido.AI.Test.MockLLM.report(mock)
```

- Test through AgentServer and the generated `ask` helpers; keep real Flow,
  Action validation, transport, and state commit. Replace only the remote
  model with `Jido.AI.Test.MockLLM`.
- Scripts can return text, objects, tool calls, HTTP errors, or held
  responses. Use a mock barrier for steering/cancellation; never `sleep`.
- Assert the committed domain field, not only the returned text, and prove one
  important failure per agent.
- Run a small live-provider check only after deterministic tests pass.

## Output Expectations

Report:

- Profile/model/instruction behavior changed
- Tools exposed, their contracts, and effect policy
- Result schema and state destination
- Limits and quotas applied
- Tests run, which used mocks, and what they prove

## Guardrails

- Do not commit API keys, request logs with secrets, or provider responses
  containing sensitive data. Prompt text is not redaction.
- Do not put worker PIDs, clients, or request handles in portable Agent state.
- Do not rely on prompts for tool access, result shape, or cost bounds.
- Do not add live-network tests to the default suite.
- Do not change profile behavior without tests that capture the contract.
- Do not treat a model's tool selection as authorization.
- Keep content retention (`observability store_content`) off unless an audit
  or support case requires it.

## Reference Docs

- `jido-core/reference/actions.md` for Action/Flow tool targets
- `jido-core/reference/runtime.md` for AgentServer request APIs
