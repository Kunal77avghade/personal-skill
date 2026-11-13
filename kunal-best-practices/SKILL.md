---
name: kunal-best-practices
description: Kunal Avghade's engineering best practices — how to plan a change, design for failure, classify errors and bound retries, clean up async resources, keep secrets and shell/SQL/path boundaries safe, keep wire formats compatible, run durable workflows (Temporal-style) safely, test, and ship small reviewable PRs. Language-agnostic. Use together with `kunal-coding-style` (which covers code shape) whenever writing, modifying, reviewing or fixing code for Kunal.
---

# Kunal's engineering best practices

`kunal-coding-style` says **how code looks**. This skill says **how a change is made safely**:
what to check before writing, what every production path must handle, and what has to be true
before the change is handed over. Load both; when they overlap, the style skill wins on shape and
this one wins on behaviour.

The goal behind every rule: code that keeps working when the network blips, a file disappears,
a dependency is down, a job is retried or a worker restarts — and that tells you clearly what
happened when it doesn't.

---

## 1. Before writing code

1. **Read the repo's own rules first**: `CLAUDE.md`, `README`, lint/format config
   (`.eslintrc*`, `.prettierrc`, `pyproject.toml`, `.golangci.yml`) and two or three neighbouring
   files. Match them. If the repo is lax (e.g. `strictNullChecks: false`, `no-explicit-any` off),
   still write code that would pass the strict version.
2. **Search before adding.** Look for an existing helper in `utils`, `*.utils.ts`, shared
   libraries or internal packages. Reuse or extend it; never add a second near-copy.
3. **Find the root cause, not the symptom.** "Job stuck in RUNNING" is usually a heartbeat or
   status update that lies, not a slow job. Reproduce or reason it through, then state the cause
   in one sentence before fixing.
4. **Smallest change that fixes the cause.** No drive-by refactors in a bug-fix PR; propose them
   separately.
5. **Check who else consumes what you touch** — event/stream names, DTO/schema wire formats,
   workflow activity signatures, DB columns, env var names, public API routes. Renaming any of
   these is a breaking change for every consumer.

## 2. Design

- **Separate pure logic from IO.** Parsers, mappers, predicates and decision rules are pure,
  deterministic and unit-tested; services orchestrate IO and stay thin.
- **One reason to change per unit.** A service that fetches, decides, writes and reports is four
  units glued together — split the decision out.
- **Idempotent by default.** Any handler, activity or job step may run twice (retries,
  redelivery, restarts). Check current state first and skip if already done
  (`if (item.status === Status.COMPLETED) return result;`). Use deterministic ids, upserts and
  "create if not exists".
- **Bounded everything.** Every loop over external data has a limit, every queue a max size,
  every recursion a depth guard or an explicit stack, every batch a chunk size.
- **Config, not constants.** Timeouts, intervals, pool sizes, chunk sizes, concurrency, retry
  counts and thresholds come from env/config with a safe default, read once at construction.
  An inline `setInterval(fn, 10000)` becomes a named config value.
- **Platform-aware.** Code that shells out or touches the filesystem handles each OS through a
  per-platform map, not scattered `if (process.platform …)` checks.
- **Extension points for things that grow** (registries, strategy maps, factories) — adding a
  new type/protocol/handler must not mean editing a central `if` chain.

## 3. Errors, retries and failure handling

### Classify every failure
Decide which bucket each error belongs to and encode it in the type, never in the message text:

| Class | Meaning | What to do |
|---|---|---|
| **Recoverable** | likely to succeed on retry (network blip, lock, busy resource) | retry with backoff |
| **Exhausted** | retried the configured max times | escalate the error type, record it, move on |
| **Item-level** | one file/row/record is bad | collect into `errors[]`, continue the batch |
| **Fatal / structural** | bad config, invalid context, auth failure | fail fast, non-retryable, clear message |

- **Typed errors with stable codes** (`RetryExceededError`, `FatalError`, `ValidationError`);
  callers branch on `instanceof` / `.type` / `.code`.
- **Retry policies are explicit and bounded**: initial interval, backoff, max interval,
  **max attempts**, and the list of non-retryable types. Unbounded retries are how jobs get stuck
  for hours.
- **Escalate on the last attempt**: `const errorType = ++retryCount >= maxRetry ? TRANSIENT : RECOVERABLE;`
- **Catch at boundaries only** — request handler, job/activity entry, worker-thread message
  handler, plugin loop. Log with context once, then re-throw a typed error or return a
  structured error response. Never swallow (`catch {}`), and never log-and-rethrow at every layer.
- **Preserve the cause**: `new XError(msg, { cause: err })`, `fmt.Errorf("…: %w", err)`,
  `raise X(...) from exc`, and keep the stack in the log call.
- **Pre-flight before expensive work**: check the input exists and is readable, the mount or
  connection is alive, the disk has space, the dependency is reachable — then start.
- **Fail loudly at startup** on missing required config; fail softly at runtime on optional
  dependencies (degrade + `warn`).

### Async and resource safety
- **Everything that can hang has a timeout** (`Promise.race` with a timer that is cleared,
  `AbortSignal.timeout`, `context.WithTimeout`, `asyncio.wait_for`).
- **Settle once.** When several event sources can reject a promise (read stream, write stream,
  timer), guard with a `settled` flag so `reject` fires exactly once.
- **Everything started is stopped in `finally`** — intervals, timers, streams
  (`if (s && !s.destroyed) s.destroy()`), child processes, file handles, locks. Initialise the
  handle to `null` before `try` and guard the cleanup.
- **No floating promises.** Every promise is awaited, returned, or explicitly `void`ed with a
  `.catch` that logs. Async callbacks inside `setInterval` / event handlers wrap their body in
  `try/catch`.
- **Batch fan-out uses `Promise.allSettled`** (or per-item try/catch inside `Promise.all`) so one
  rejection doesn't drop the rest; cap concurrency with chunks or a limiter — never
  `items.map(async …)` over an unbounded list.
- **Process-level safety nets** in worker threads/child processes: `uncaughtException` →
  log + `process.exit(1)`; `unhandledRejection` → log; message-channel errors → log.

## 4. Durable workflows (Temporal and similar)

- **Workflow code is deterministic**: no wall-clock reads, random numbers, IO, env reads or
  non-deterministic libraries inside workflows — use the SDK's sleep/uuid/time helpers and do IO
  in activities. Only pure rule modules are shared into workflow code.
- **Activities are idempotent** (section 2) and **heartbeat honestly**: a heartbeat means "I am
  making progress", so check what you depend on before beating, and stop beating when it's
  broken so the engine can reschedule.
- **Heartbeat timeout ≥ 3× the heartbeat interval**; start-to-close sized for the worst real
  case; both from config where they vary per deployment.
- **Every retry policy has max attempts and non-retryable error types**; activities raise
  application failures with a `type` the workflow can branch on, instead of matching messages.
- **Deterministic child workflow ids** (`` `${WorkflowName}-${traceId}-${workerId}` ``) so restarts
  dedupe instead of duplicating.
- **Long loops use continue-as-new** before history grows large; iteration caps are constants.
- **Changing a workflow/activity signature with runs in flight is a migration** — use the SDK's
  versioning/patching or keep accepting the old shape.

## 5. Security

- **No secrets in code, logs, commits or test fixtures.** Templates (`.env.template`,
  `.env.example`) hold placeholders (`API_SECRET=<set-me>`), never working values. Real values
  come from env or a secret store.
- **Mask sensitive values in logs**: passwords, tokens, keys, connection strings, personal data.
  Log ids, not payloads; truncate large bodies.
- **Never build shell commands by string-concatenating external input.** Prefer
  `execFile`/`spawn`/`exec.Command`/`subprocess.run([...])` with an argument array and no shell.
  When a shell template is unavoidable, escape every interpolated value for that specific shell,
  and keep credentials off the command line (use a credential file, stdin or env).
- **Parameterised queries only** — ORM methods or placeholders, never string-built SQL.
- **Validate at the edge**: DTOs with `class-validator` / Pydantic / struct validation; reject
  unknown fields on external input; treat payloads as `unknown` until narrowed.
- **Paths from outside are hostile**: normalise, resolve and confirm they stay under the intended
  base (no `..` or symlink escapes) before reading, writing or deleting.
- **Least privilege**: services request only the scopes/permissions they use; elevated
  operations are isolated and logged.
- **Dependencies**: lockfile committed, CVE fixes applied promptly in their own PR, no new
  dependency for something a 30-line helper does.

## 6. Data, schemas and compatibility

- **Wire formats are contracts.** Adding an optional field is safe; renaming, removing or
  retyping is breaking — update every consumer in the same change, or keep the old field during
  a transition.
- **DB changes go through migrations** (Liquibase, Flyway, Alembic, goose), stay compatible with
  the currently deployed code (expand → migrate → contract), and never edit an applied migration.
- **Event, stream and queue names are constants** in one module, shared or mirrored by consumers.
- **Use the right id.** Before querying, confirm which entity the id belongs to, and name
  variables after the entity they hold (`configId`, not `id`).
- **Time and size comparisons are exact**: compare timestamps at the precision the source gives,
  sizes as integers, never via formatted strings.

## 7. Performance

- **Stream large data** with buffers sized to the payload; never read whole files into memory.
- **Batch IO** (pipelines, bulk inserts, chunked reads) and **avoid N+1** calls in loops — fetch
  once, index into a `Map`.
- **Bounded concurrency**, tuned by config.
- **Cache slow, stable lookups** (TTL/LRU), keep the last good value on refresh failure, and
  expose hit/miss stats.
- **Measure before optimising**: log durations on lifecycle lines, profile, and put the
  before/after numbers in the PR.

## 8. Observability

(The shape of log lines is in `kunal-coding-style`; these are the behavioural rules.)
- **Every log line in a request/job path carries a correlation id** (`[${jobId}]`, `traceId`,
  workflow id).
- **Use the injected/structured logger in services**; `console.*`/`print` only where no logger
  exists (worker-thread scripts, workflow sandboxes), and then prefixed with the component.
- **Log each failure once**, at the boundary that handles it, with message + stack + ids.
- **Metrics for long-running things**: counts, durations, queue depth, pool health, so a stuck
  job shows up on a dashboard before a user reports it.

## 9. Testing

- **Every bug fix ships with a regression test** that fails before the fix and passes after, in
  the same PR. Every new branch of logic gets a test, including the error branches.
- **Tests sit next to the source** (`*.spec.ts`, `*_test.go`, `tests/test_*.py`), one spec per
  unit, named for behaviour: `it('should skip if already completed')`.
- **Mock at module boundaries** (filesystem, cache, HTTP, workflow context, child processes), not
  internal helpers of the unit under test. Use `requireActual`-style partial mocks and override
  only what you need.
- **Shared mocks live in a test-helper module** (`test/mocks/logger.mock.ts`), never imported
  from another spec file.
- **Typed mocks over `as any`**: `Partial<T>` / `jest.Mocked<T>` / factories like
  `createMockItem(status)`; `as any` only for deliberately malformed input.
- **Build fixtures per test** (or with a factory) instead of mutating a shared object; reset
  mocks in `beforeEach`.
- **Deterministic tests**: fake timers for intervals/timeouts, no real network, no sleeps, no
  order dependence. Workflow code uses the SDK's test environment.
- **Assert on outcomes and the calls that matter** (status, error type, "did not call X"), not on
  log text.
- Run the repo's commands before handing over, e.g. `npm run lint && npm test`,
  `uv run ruff check . && uv run pytest -q`, `go vet ./... && go test ./...`.

## 10. Git and PRs

- **One concern per PR**, small enough to review in one sitting. Bug fix, refactor, dependency
  bump and feature are separate PRs.
- **Commit messages follow the repo**: ticket style `[TICKET-123] Short description (fix)` or
  Conventional Commits (`fix(store): …`). Imperative, specific, ticket id when one exists.
- **The PR description says**: root cause, the fix, how it was verified, and any
  compatibility/config/migration impact (new env vars listed with defaults).
- **New env vars** are added to the env template, deployment templates and the config loader in
  the same PR.
- **When a merge breaks main, revert first, fix forward second.**
- Never commit generated output, local env files, credentials or debug logging.

## 11. Things to never leave behind

- Commented-out code — delete it; git remembers.
- `TODO` without a ticket id (`// TODO(ABC-123): handle FatalError`).
- Debug `console.log` / `print` lines in services.
- Typos in file or identifier names — fix in new code; rename old ones in a dedicated PR.
- Magic numbers for timing, size or concurrency.
- Silent `catch` blocks, unbounded retries, uncleared intervals, unawaited promises.
- Unused imports, dead exports, stale feature flags.

---

## Pre-handover checklist

Run through this (along with the `kunal-coding-style` checklist) before saying a change is done:

1. Root cause stated in one sentence; the change is the smallest fix for it?
2. Consumers of any touched contract (events, schemas, signatures, env vars, DB) checked?
3. Each failure classified (recoverable / exhausted / item / fatal) with a typed error?
4. Every external call bounded by a timeout; every retry bounded by max attempts?
5. Every timer/stream/process/handle cleaned up in `finally` or a shutdown hook?
6. Idempotent on re-run? Loops, queues and recursion bounded?
7. No secrets in code/logs; shell/SQL/path inputs escaped or validated?
8. Correlation id on every log line; each failure logged once?
9. Regression test for the bug, tests for the error paths, lint + tests run and passing?
10. New config documented in the env template; PR text covers cause, fix, verification and impact?
