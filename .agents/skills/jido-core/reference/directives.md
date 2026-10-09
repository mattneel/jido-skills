# Jido Directives and Outcomes Reference (v3)

Source guides: `directives-and-outcomes.md`, `directives.md`,
`turns-commit-and-effects.md`, `adopt`/ownership guides.

A Directive is a typed request for runtime work **after** an Agent state
commit. An Outcome is the terminal record for one admitted Turn.

## Built-In Directives

| Directive | Operation |
| --- | --- |
| `Emit` | Dispatch one Signal through a target |
| `EmitToParent` | Send a Signal to the logical parent |
| `EmitToChild` | Send a Signal to a tracked child |
| `SpawnProcess` | Start an untracked supervised OTP process |
| `SpawnChild` | Start and track a child Agent |
| `AdoptChild` | Attach an existing live child |
| `StopChild` | Stop and remove one tracked child |
| `Stop` | Stop the current Agent Server |
| `Error` | Send a structured error to Server policy |

```elixir
emit = Jido.Agent.Directive.emit(signal, dispatch_target)
to_child = Jido.Agent.Directive.emit_to_child(:worker, signal)
child = Jido.Agent.Directive.spawn_child(MyApp.Worker, :worker)
process = Jido.Agent.Directive.spawn_process({Task, fn -> do_work() end})
```

- Use `SpawnChild`/`StopChild` for owned Agents by relationship tag.
- Use `SpawnProcess` only for an untracked supervised OTP process.
- Prefer `{:error, reason}` when a Turn must fail before commit; the `Error`
  Directive reports an error to Server policy after commit.
- V2 names changed: `Spawn`→`SpawnProcess`, `SpawnAgent`→`SpawnChild`.
  Cron/Schedule/Sensor directives moved to their Plugins. `RunInstruction` has
  no post-commit replacement; send a later Signal instead.

## Validate Before Commit

- Jido validates all built-in and Plugin Directives before commit.
  Plugin-owned state contributions (`reduce/2`) also run before commit.
- Runtime dispatch starts only after the complete candidate is valid and any
  required checkpoint succeeds.
- A direct `Jido.Agent.cmd/3` call returns the Directive list and does not
  dispatch it.
- Custom Directive types are declared by a Plugin Agent facet, which owns
  `Jido.Agent.Directive.validate/1`; the paired Agent Server facet owns
  optional post-commit dispatch.

## Dispatch Behavior

- Directives run after commit, in list order.
- If one Directive fails, the committed state remains and later Directives in
  that batch do not run. Ordinary Directives have no crash-replay guarantee.
- A process or VM can stop after commit and before dispatch completes. Restore
  does not replay that batch.
- Ordinary Directive lists are **not** a durable outbox. For durable delivery,
  save explicit pending work (with stable operation IDs and acknowledgement)
  or use a durable Signal Bus subscription where that contract fits.

## Plugin-Defined Directives

A Plugin package can add Directive types. The Agent facet declares them and
validates them; the Agent Server facet handles optional post-commit dispatch
with a bounded `Jido.Plugin.DirectiveContext`. The Plugin's paired owned state
value is available to dispatch. The trust boundary is an API boundary, not a
sandbox for untrusted BEAM code.

## Turn Outcomes

`Jido.Agent.Turn.Outcome` records:

- Turn and Agent IDs
- source and effective Signals
- terminal status and stage
- state versions before and after the Turn
- whether the Turn committed
- Directive completion, failure, and skip counts
- timing and an optional error

Statuses: `:succeeded`, `:failed`, `:cancelled`, `:timed_out`,
`:indeterminate`. The first recovery question is always `committed?`. The
Outcome has no public authoring API and is not domain history.

## Child Relationships

- `SpawnChild` supports explicit remote placement through `node:`.
- `AdoptChild` attaches an existing live child; ownership data changed from v2
  (monitor, creation cause, remote spawn reference). Test local and remote
  adoption, orphans, and lost replies.
- `StopChild` identifies the child by relationship tag.
- `on_parent_death` (default `:stop`) selects child behavior when a logical
  parent ends.
- `Stop` commits state before the Server stops.

See `reference/runtime.md` for children views and lifecycle.
