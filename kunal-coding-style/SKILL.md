---
name: kunal-coding-style
description: Kunal Avghade's personal coding style — code-block shape, compactness, reusability, comments, resilience and per-language conventions (TypeScript/NestJS, Python, Go, plus defaults for others), distilled from production services Kunal has written. Apply whenever writing or modifying code for Kunal in any language, unless the target repo's own conventions clearly differ, in which case match the repo and keep the principles.
---

# Kunal's coding style

Distilled from production code Kunal wrote: TypeScript/NestJS workflow workers, Python
agent services and Go network services. When unsure how Kunal would shape something, look
for code they authored in the current repo (`git log --author`) and match it.

The whole style in one line: **compact, guard-first, reusable, production-hardened code with
few but pointed comments.** It must survive failures, be visible in logs, be tunable through
config, and still be easy to debug at 2 a.m. on a customer's machine.

---

## Part A — Principles for every language

### A1. Shape of a code block

- **Guard clauses first, happy path last, flat.** Validate and return/throw at the top; avoid
  `else` after a `return`. Nesting deeper than two levels is a signal to extract a helper.
  ```ts
  if (!output) return [];
  if (!excludePatterns.length) return false;
  ```
- **Single-statement bodies stay on one line (or the next line) without braces** where the
  language allows it. Multi-statement bodies always get braces.
  ```ts
  for (const pattern of excludePatterns)
    if (fullPathSplit.includes(pattern)) return true;
  ```
- **Dispatch with `switch`, not `if/else` ladders**, once there are three or more cases — including
  `switch (true)` for predicate dispatch in TS:
  ```ts
  switch (true) {
    case stats.isFile():   return FileType.FILE;
    case stats.isSocket(): return FileType.SOCKET;
    default:               return FileType.UNKNOWN;
  }
  ```
- **Lookup tables over branching** when a value maps to another value (`PERMISSION_MAP`,
  `_EFFORT_RUNG = {"low": 0, "mid": 0, "high": 1}`).
- **Build the result object up-front with defaults, then mutate it** as checks run. Keep
  `errors` (block the operation) and `warnings` (inform only) as separate lists.
- **Steps are visually separated** by one blank line; long multi-phase methods get
  `// Step 1: …`, `// Step 2: …` markers. Never more than one empty line in a row.

### A2. Compactness

- Prefer **one-expression functions** for small pure helpers:
  ```ts
  export const removePrefix = (str: string, prefix: string): string =>
    str.startsWith(prefix) ? str.slice(prefix.length) : str;
  export const isContentUpdate = (s: fs.Stats, d?: fs.Stats) =>
    !d || s.size !== d.size || s.mtime.toISOString() !== d.mtime.toISOString();
  ```
  ```go
  func (c *Config) Direct() bool { return c.Unrouted == UnroutedDirect }
  ```
- **Ternaries for two-way value choices**; never nest them more than once.
- **Tiny inner helpers** instead of repeating an expression three times
  (`const toRWX = (p: number) => \`${p & 4 ? 'r' : '-'}${p & 2 ? 'w' : '-'}${p & 1 ? 'x' : '-'}\``).
- **Destructure inputs** in the signature (`async ({ jobRunId, status }: UpdateStatusInput)`) and
  **spread to merge** outputs (`return { ...obj, ...checksums, sid }`).
- **Defensive normalisation inline**: `(message || '').trim().toLowerCase()`,
  `(history ?? []).slice(-maxTurns)`, `jobContext.jobConfig?.options?.flag`.
- Use the standard library's compact forms: `Map`/`Set`/`frozenset`/`seen := map[string]bool{}`
  for lookups and de-duplication; `some`/`every`/`any()`/`all()` for checks; `flatMap` for flattening.
- Compact ≠ cryptic. **No comma-operator tricks**, no clever one-liners that need a comment to
  decode, no `.map` used only for side effects (use `forEach`/`for…of`).

### A3. Reusability

- **Pure helpers live in a `utils` module next to the domain** (`utils/utils.ts`,
  `<domain>/<domain>.utils.ts`): parsers (`parseWindowsShares`, `parseProtocolVersions`), mappers
  (`getErrorCode(error, context)`), predicates (`shouldExclude`, `isContentUpdate`), factories
  (`generateDummyFileEntry`). Services stay thin and call them.
- **Parameterise instead of duplicating**: one function with a `context: 'TASK' | 'OPERATION'`
  argument instead of two near-copies.
- **Registries / plugin points for things that grow**: decorator-registered handlers sorted by
  `order` (`@build_assessor(tier=…, order=…)`), per-platform config maps
  (`nfs.win32 / nfs.linux / nfs.darwin`). Adding a capability must not require editing the loop.
- **Small focused primitives over dependencies** (a 30-line `Map`-based `LRUCache`).
- **Private helper classes next to their only consumer**; promote to a module when a second
  consumer appears.
- **Extract a helper the second time a block repeats** — e.g. the "get token → call service →
  log success/failure" sequence belongs in one `callService(method, url, payload)` helper, not
  copied into every method.
- **Injectable seams for tests**: constructor DI in TS, function-valued fields in Go
  (`DialDirect func(ctx, addr) (net.Conn, error)` "tests replace it"), plain arguments in Python.

### A4. Comments

Kunal comments sparingly and only where the code can't speak for itself.

- **Why, not what.** A comment explains a domain rule, a constraint or a trap:
  ```ts
  // Only consider AccessAllowed (0) and AccessDenied (1) ACEs in comparison.
  // Audit/object ACEs are not stamped, so comparing them gives false errors.
  ```
  ```go
  // details goes as text cast to jsonb: behind pgbouncer's transaction pooling a
  // []byte argument is sent as bytea, which jsonb rejects.
  ```
- **Short tags on edge cases**: `// File or directory does not exist [edge case]`,
  `// Too many open files [rare edge case]`, `process.exit(1); // mandatory (Node.js docs)`.
- **Unit hints on literals**: `> 300000) { // 5 minutes`.
- **Two-to-four-word comments are fine** when they mark intent (`// move up`, `// keep the last good one`).
- **Doc comments only on public/exported API**, and short. Module-level docstrings explain the
  module's purpose and extension point, not each line.
- **Never** leave commented-out code, restate the code in English, or write JSDoc walls.

### A5. Errors, resilience, config, observability

- **Domain error types with a stable code**; callers branch on type/code, never on message text.
- **Catch at the boundary, log with context, re-raise a typed error.** Status-returning
  operations return a structured error response instead of throwing.
- **Collect per-item failures and continue**; abort only on structural errors (bad config,
  invalid context). Go/Python validators accumulate `errs` and return them together.
- **One bad plugin/item never breaks the batch** (`try … except Exception: log and skip`).
- Kunal's resilience toolkit — reach for these whenever code touches processes or networks:
  pools with round-robin / least-loaded selection, bounded queues with backpressure,
  per-operation timeouts (longer minimums for known-slow classes, always cleared), health checks
  with failure counters and self-healing recreation, bounded retries with small delays, unique
  markers / `traceId` on every operation, reject-all-pending on shutdown, TTL or LRU caches in
  front of slow lookups (keeping the last good value on read failure), fixed-size batches run
  concurrently, escaping anything that crosses a shell/SQL boundary, platform guards.
- **All tunables from config/env with safe defaults** — read once, stored in fields/constants,
  with an inline comment when the default is a hard limit (`# gpt-4o's hard ceiling — do not raise`).
- **Log the lifecycle**: start, success with duration, failure with duration and reason; prefix
  with component and id (`[PersistentShell:${id}]`, `[${traceId}]`); truncate big payloads.
  Levels: `debug` routine, `info/log` business events, `warn` degraded, `error` real failures only.
- **Expose counters / rolling stats** (`getStats()`) for anything long-running.
- **Everything started is stopped** (intervals, child processes, connections) in a shutdown hook.

---

## Part B — Per-language conventions

### B1. TypeScript / NestJS

- Formatting: Prettier `{ "singleQuote": true, "trailingComma": "all" }`, 2-space indent.
- Imports: `@nestjs/*` → shared libs (`@org/*`) → project modules via absolute `src/...`.
- **Utilities as exported `const` arrow functions** in `*.utils.ts`; services as
  `@Injectable()` classes with constructor DI and a per-class logger:
  ```ts
  private readonly logger: LoggerService;
  constructor(
    @Inject(LoggerFactory) loggerFactory: LoggerFactory,
    private readonly redisService: RedisService,
  ) {
    this.logger = loggerFactory.create(PathResolverService.name);
  }
  ```
- Types: `interface`/`type` declarations at the top of the file or in a sibling `*.types.ts`;
  `enum` for every closed set (statuses, error codes, command keys, workflow names); module
  constants in `SCREAMING_SNAKE_CASE`; `Record<string, T>` lookup tables.
- Errors: `class XError extends Error { constructor(msg, public readonly code = 'X_ERROR') { super(msg); this.name = 'X_ERROR'; } }`.
- Async: `Promise.all` over batches; chunk loops `for (let i = 0; i < items.length; i += 50)`;
  `for await` for streams; wrap callback APIs in `new Promise((resolve, reject) => …)` with an
  `errored` flag so reject fires once.
- Temporal: parent workflow fans out `executeChild` per worker with deterministic ids
  (`` `${WorkFlows.VALIDATE}-${traceId}-${workerId}` ``) and per-worker task queues, then merges.
- NestJS lifecycle: `onModuleInit` creates pools/monitors, `onModuleDestroy` tears them down.
- Prefer `unknown` + narrowing over `any` for external payloads; use the shared logger, not `console.*`
  (except inside worker-thread scripts that have no DI).
- Tests: Jest + `Test.createTestingModule`, every dependency a `Partial<T>` of `jest.fn()`s,
  `*.spec.ts` next to the source, error classes tested too.

### B2. Python

- Tooling: Python 3.12, `uv`, ruff (`E,F,I,UP`) + black, **line length 100**, double quotes.
- Every module starts with a short **module docstring** (purpose + how to extend), then
  `from __future__ import annotations`, then a module logger `logger = logging.getLogger(__name__)`.
- **Section dividers** inside longer modules:
  `# ── Assessors ──────────────────────────────────────────────`.
- **Private module constants** with a leading underscore, immutable where possible:
  `_CANCEL_WORDS = {...}`, `_APPROVE_PHRASES = frozenset({...})`, `_EFFORT_RUNG: dict[str, int] = {...}`.
- Modern typing: `list[dict]`, `str | None`, `collections.abc.Callable/Awaitable`, type aliases
  for callable shapes (`BuildAssessor = Callable[[dict, dict], Awaitable[list[dict]]]`).
- `@dataclass(frozen=True)` for value objects; plain `@dataclass` for mutable context threaded
  through handlers.
- **Keyword-only arguments** (`def fast_intent(*, action, message, has_plan)`) for anything with
  more than ~3 parameters or boolean flags.
- **Decorator registries** for extensibility (`def build_action(*, field=None, order=0)` returning
  `deco`, list kept sorted by `(order, name)`).
- Pure, deterministic logic separated from IO (`intent_rules.py` — "no IO; safe in workflow code").
- Guards return early; small helpers return `""`/`None`/a default instead of raising.
- Env config at module level: `int(os.getenv("X", "16384"))  # why this default`.
- Broad excepts only at isolation boundaries, always with a reason:
  `except Exception as exc:  # noqa: BLE001 - one action must never break the loop`.
- Logging with `%` args, not f-strings: `logger.warning("build action '%s' skipped: %s", name, exc)`.
- Docstrings: one line for simple functions; a short paragraph only when behaviour has a
  non-obvious guarantee. Inline code in docstrings uses double backticks.
- Tests: pytest, flat `tests/test_*.py`, run with `uv run pytest -q`.

### B3. Go

- `gofmt`, standard layout (`cmd/<binary>/main.go`, `internal/<pkg>/`), `log/slog` with
  key-value pairs: `s.log.Info("egress proxy ready", "gateways", gws, "routes", len(routes))`.
- **Doc comments on every exported identifier, starting with its name**, one or two sentences:
  `// CredentialFileFor returns the credential file used for gateway.`
- **One-line methods** for trivial accessors/predicates; named constants in a `const ( … )` block
  with a heading comment (`// Unrouted modes.`).
- Config structs with YAML tags and a field comment explaining behaviour and defaults; a
  `setDefaults`-style block (`if c.Unrouted == "" { c.Unrouted = UnroutedDeny }`) and a
  `Validate()` that **accumulates `errs []error`** with actionable messages
  (`"gateway %q has no credential: set credentials.%s, credential_file or EGRESS_CREDENTIAL"`).
- Errors wrapped with context and `%w`: `fmt.Errorf("reading credential for gateway %s: %w", gw, err)`.
- De-duplicate with `seen := map[string]bool{}` while preserving order.
- Small private structs for cached values (`type cachedCred struct { value string; read time.Time }`)
  guarded by a `sync.Mutex`; TTL re-read with "keep the last good one" on failure.
- Function-valued fields as test seams; always set timeouts on dialers
  (`&net.Dialer{Timeout: 20 * time.Second}`).
- Table-driven `_test.go` files next to the code.

### B4. Java / other languages

No Kunal-authored Java was found, so apply Part A with the language's mainstream conventions
(Spring Boot: constructor injection, `@ConfigurationProperties` for tunables, custom exceptions
with an error code, SLF4J with `{}` placeholders, JUnit 5 + Mockito tests next to the package).

---

## Part C — Commits

- Ticket style: `[TICKET-ID] Short description (fix)` — e.g. `[ABC-123] Slow copy on large directories (fix)`.
- Conventional Commits: `feat: …`, `fix(store): …`, `chore: …`.
- Match whichever the repo already uses.

## Checklist before handing code to Kunal

1. Guards first, flat happy path, no nesting deeper than two levels?
2. Could any block be a one-expression helper, a lookup table or a `switch`?
3. Is any block copied a second time? Extract it.
4. Every external call: timeout, typed error, log line with a context id?
5. Tunables in config/env with sane defaults? Everything started is stopped?
6. Per-item failures collected instead of aborting the batch?
7. Comments only where they explain *why* — and no commented-out code?
8. Tests next to the source with mocked dependencies?
