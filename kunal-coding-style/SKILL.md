---
name: kunal-coding-style
description: Kunal Avghade's coding style — senior-level, industry-standard code that is readable, reusable, pattern-based and optimised. Covers naming, function and module shape, SOLID and design patterns (when to use them and when not), performance, error handling, comments, testability and per-language conventions (TypeScript/NestJS, Python, Go, Java). Apply whenever writing, refactoring or reviewing code for Kunal in any language; if the target repo's conventions clearly differ, match the repo and keep the principles. Pair with `kunal-best-practices` for how a change is planned, hardened and shipped.
---

# Kunal's coding style

Write code the way a senior engineer would want to inherit it: **meaningful names, small focused
units, guard-first flat flow, one obvious place for each responsibility, patterns only where they
remove real duplication or coupling, and no wasted work.** It must be easy to read in review, easy
to change next month, and easy to debug in production.

Industry baselines this skill builds on: SOLID, DRY / KISS / YAGNI, *Clean Code* naming and
function rules, *Refactoring* smells, the Gang-of-Four patterns, Effective Go, PEP 8 / PEP 484 /
PEP 20, Google TypeScript and Java style guides, typescript-eslint recommended, Effective Java.

`kunal-best-practices` owns behaviour (failure handling, retries, security, testing strategy,
PRs). This skill owns **how the code itself is shaped**.

---

## Part A — Principles for every language

### A1. Readability first

Code is read far more than it is written. Optimise for the next reader.

- **Names reveal intent.** A reader should know what a thing is or does without opening it.
  - Variables and fields: nouns (`pendingInvoices`, `retryDelayMs`, `activeSessionCount`).
  - Functions and methods: verb + object (`calculateTax`, `fetchUserProfile`, `buildReport`).
  - Booleans: `is` / `has` / `can` / `should` prefix (`isExpired`, `hasPermission`).
  - Collections are plural; maps say what they map (`priceBySku`, `usersById`).
  - **Units in the name** when the type doesn't carry them: `timeoutMs`, `maxSizeBytes`, `ttlSeconds`.
  - No vague names: `data`, `info`, `obj`, `tmp`, `res2`, `handle()`, `process()`, `doStuff()`,
    `Manager`/`Helper` classes with no clear job. No cryptic abbreviations (`usrCnt`, `cfgMgr`);
    standard ones (`id`, `url`, `ctx`, `err`, `i` in a short loop) are fine.
  - Name length tracks scope: one-letter only inside a three-line loop; exported names are
    fully descriptive.
- **One level of abstraction per function.** A function either orchestrates steps or does one
  low-level thing — not both. If you need a comment to label a block, that block wants to be a
  named function.
- **Positive, explicit conditions.** `if (isValid)` over `if (!isInvalid)`; extract complex
  conditions into a named boolean or predicate (`const canRetry = attempt < maxAttempts && isTransient(err);`).
- **Consistent vocabulary.** Pick one word per concept (`fetch` vs `get` vs `load`) and use it
  across the codebase.
- **Least surprise.** A function named `getX` does not mutate state or perform IO beyond reading X.

### A2. Shape of a code block

- **Guard clauses first, happy path last, flat.** Validate and return/throw at the top; no `else`
  after `return`. More than two levels of nesting means extract a function.
  ```ts
  export function applyDiscount(order: Order, code?: string): Order {
    if (!code) return order;
    const discount = DISCOUNTS[code];
    if (!discount) throw new InvalidDiscountError(code);
    return { ...order, total: discount.apply(order.total) };
  }
  ```
- **Small functions, few parameters.** Aim for a function that fits on one screen and does one
  thing. More than three parameters, or any boolean flag parameter → pass an options object /
  keyword-only arguments / a config struct, or split the function.
- **Dispatch with a map or `switch`, not `if/else` ladders**, once there are three or more cases.
  Prefer a lookup table when each case maps a value to a value or a handler:
  ```ts
  const SHIPPING_RATE: Record<ShippingTier, number> = { standard: 0, express: 9.99, overnight: 24.99 };
  ```
- **Single-statement bodies may sit on one line** where the language and formatter allow it;
  multi-statement bodies always get braces.
- **Build results up-front, then fill them.** Create the result object with defaults, then
  populate it as steps run. Keep blocking `errors` separate from informational `warnings`.
- **Visually separate steps** with a single blank line; never more than one blank line in a row.
- **Declare variables close to first use**, with the narrowest scope and `const`/`final`/
  immutable by default.

### A3. Compact, not clever

- **Expression-bodied helpers** for small pure functions:
  ```ts
  export const isExpired = (expiresAt: Date, now = new Date()): boolean => expiresAt <= now;
  ```
  ```go
  func (o *Order) IsPaid() bool { return o.PaidAt != nil }
  ```
- **Ternaries for a two-way value choice only**; never nest them.
- **Destructure inputs, spread to compose outputs**:
  `async function createUser({ email, role = Role.MEMBER }: CreateUserInput)`,
  `return { ...defaults, ...overrides }`.
- **Normalise defensively at the edge**: `(input ?? '').trim().toLowerCase()`,
  `(items ?? []).slice(0, limit)`, optional chaining for optional data.
- **Use the standard library's idioms**: `Map`/`Set`, `some`/`every`, `flatMap`,
  comprehensions, `any()`/`all()`, `slices`/`maps` packages, `Optional`/streams in Java.
- **Compact ≠ cryptic.** No comma-operator tricks, no chained ternaries, no `.map` for side
  effects, no one-liners that need a comment to decode. If the clever version is shorter but
  slower to understand, write the plain one.

### A4. SOLID and structure

- **Single Responsibility** — a class/module has one reason to change. Split "fetch + decide +
  persist + notify" into units and compose them.
- **Open/Closed** — new behaviour is added by adding a strategy/handler/registration, not by
  editing a growing `switch` in core code.
- **Liskov** — subtypes honour the contract of their base; no "not supported" overrides.
  If a subtype can't, the abstraction is wrong.
- **Interface Segregation** — small, role-focused interfaces (`Reader`, `Notifier`) rather than
  one fat interface every consumer half-uses.
- **Dependency Inversion** — business logic depends on interfaces/ports; concrete IO
  (database, HTTP, queue, filesystem) is injected. This is also the test seam.
- **Composition over inheritance.** Inherit only for a true *is-a* with shared invariants;
  otherwise inject collaborators.
- **Pure core, imperative shell.** Keep business rules as pure, deterministic functions; push IO
  to the edges (controllers, repositories, adapters). Pure code is trivially testable and reusable.
- **Layering**: transport (controller/handler) → application service (orchestration) → domain
  (rules, entities, value objects) → infrastructure (repositories, clients). Dependencies point
  inward; the domain imports nothing from transport or infrastructure.
- **Cohesive modules**: group by feature/domain (`billing/`, `users/`), not by technical type
  across the whole app. Inside a feature: `*.service`, `*.repository`, `*.types`, `*.utils`,
  `*.errors`, tests next to them.

### A5. Reusability

- **Rule of three, applied early for logic.** Extract on the second copy of non-trivial logic;
  tolerate duplication of trivial lines rather than inventing a premature abstraction.
- **Parameterise instead of near-copying.** One function with a well-typed parameter
  (`kind: 'invoice' | 'credit-note'`) beats two functions that differ by a line.
- **Generic, typed building blocks** — `Result<T, E>`, `Paginated<T>`, `retry<T>(fn, policy)`,
  `withTimeout<T>(promise, ms)`, `chunk<T>(items, size)` — live in a shared module and are
  reused everywhere instead of re-implemented inline.
- **Wrap the repeated sequence.** When "authenticate → call → log → map errors" appears in every
  client method, it becomes one private `request<T>(method, path, body)` helper.
- **Small focused primitives over heavy dependencies** — a 30-line bounded cache beats pulling
  in a framework for one feature; a well-maintained library beats a hand-rolled crypto/date/parsing
  implementation.
- **Keep helpers next to their only consumer**; promote to a shared module when a second
  consumer appears.
- **Stable, minimal public surface.** Export only what consumers need; everything else is
  private/unexported so it can change freely.

### A6. Design patterns — use them to remove duplication or coupling, never for show

| Pattern | Reach for it when | Typical shape |
|---|---|---|
| **Strategy** | behaviour varies by type/config and the set grows | `Record<Kind, Handler>` / interface + map lookup |
| **Factory** | construction depends on input/config or hides concrete classes | `createNotifier(channel): Notifier` |
| **Registry / plugin** | new variants should be added without touching the core loop | decorator/`register()` adds handlers, loop iterates sorted registry |
| **Adapter** | wrapping a third-party SDK or legacy API behind your own interface | `PaymentGateway` interface, `StripeGatewayAdapter` implements it |
| **Decorator / middleware** | cross-cutting concerns: retry, timeout, caching, logging, metrics | `withRetry(withTimeout(fetchRates))`, framework interceptors |
| **Repository** | isolating persistence from business logic | `OrderRepository.findPendingOlderThan(date)` |
| **Builder** | objects with many optional parts or validation at build time | `ReportBuilder().withRange(r).withFilters(f).build()` |
| **Template method / pipeline** | fixed sequence of steps with pluggable steps | ordered list of `Step` objects run by a runner |
| **Observer / pub-sub** | decoupling producers from many consumers | domain events + handlers, message queue |
| **Object pool** | expensive resources (connections, processes, workers) | bounded pool with health checks and least-busy selection |
| **Circuit breaker** | protecting callers from a failing dependency | open/half-open/closed wrapper around a client |
| **Value object** | a concept with rules (money, email, date range) | immutable type validated on construction |
| **Null object / defaults** | removing repeated `if (x == null)` checks | no-op logger, empty config with safe defaults |

Rules:
- **Start simple.** A function and a map are often the whole Strategy pattern. Introduce class
  hierarchies only when state or multiple methods per variant justify them.
- **YAGNI.** No interface with exactly one implementation unless it is a test seam or an IO
  boundary. No factory for a class that is only ever `new`-ed one way.
- **Name the pattern in the type, not in comments** (`TaxStrategy`, `OrderRepository`,
  `RetryingHttpClient`), so the design is visible from the file tree.

### A7. Performance by default (without premature micro-optimisation)

- **Right data structure first.** `Set`/`Map`/dict for membership and lookup (O(1)) instead of
  `array.includes`/`find` inside a loop (O(n²)). Index once (`const usersById = new Map(users.map(u => [u.id, u]))`), then look up.
- **Don't repeat work.** Hoist invariant computations out of loops; compile regexes once; cache
  pure, expensive results (memoise) with a bound; avoid recomputing derived values on every call.
- **Batch and stream IO.** Bulk queries over per-item queries (no N+1); stream large files and
  result sets; paginate APIs.
- **Concurrency with a limit.** Run independent IO in parallel (`Promise.all`, goroutines,
  `asyncio.gather`) but bounded by a configured limit; never unbounded fan-out.
- **Lazy where it pays.** Compute or load only when needed; short-circuit `some`/`any` instead of
  building full lists to check one condition.
- **Avoid needless allocation in hot paths**: string building via join/builder, reuse buffers,
  preallocate slices/arrays when the size is known.
- **Measure before tuning.** Profile, then optimise the hot spot; keep the readable version
  everywhere else. Record the before/after numbers in the PR.

### A8. Types and data

- **Make illegal states unrepresentable**: enums / union literals for closed sets, discriminated
  unions for variants, value objects for validated concepts, required vs optional fields chosen
  deliberately.
- **Strict typing on**: no implicit `any`; external input is `unknown` until validated and
  narrowed; no unchecked casts.
- **Immutability by default**: `readonly`, `const`, `final`, frozen dataclasses; return new
  objects instead of mutating arguments.
- **No magic values**: named constants (`MAX_UPLOAD_SIZE_BYTES`) and enums; tunables from config.
- **DTOs at boundaries, domain types inside.** Map between them explicitly; never leak ORM
  entities or raw HTTP payloads through the domain.

### A9. Errors (shape; behaviour lives in `kunal-best-practices`)

- **Domain error types with a stable `code`**, extending the language's base error; callers
  branch on type/code, never on message text.
  ```ts
  export class PaymentDeclinedError extends Error {
    readonly code = 'PAYMENT_DECLINED';
    constructor(readonly orderId: string, cause?: unknown) {
      super(`Payment declined for order ${orderId}`, { cause });
      this.name = 'PaymentDeclinedError';
    }
  }
  ```
- **Messages are actionable**: what failed, for which id, and what to do
  (`"missing API key: set PAYMENTS_API_KEY or payments.apiKey in config"`).
- **Return `Result`/`(value, err)` for expected outcomes** (validation, not-found); throw/raise for
  unexpected failures. Be consistent within a module.

### A10. Comments and documentation

Comment sparingly and only where the code can't speak for itself.

- **Why, not what.** Explain business rules, constraints, trade-offs and traps:
  ```ts
  // Card networks reject amounts with more than 2 decimals, so round before charging.
  ```
- **Short tags for edge cases** (`// empty cart [edge case]`) and **unit hints on literals**
  (`300_000 // 5 minutes`) when a named constant would be overkill.
- **Doc comments on public API only** — exported functions, classes, interfaces — short, stating
  purpose, non-obvious parameters, return value and errors. Go: start with the identifier name.
  Python: one-line docstring; a short paragraph only for non-obvious guarantees.
- **Module header** (one or two lines) for modules with an extension point, saying how to extend.
- **Never**: commented-out code, comments that restate the code, banner-comment walls, stale
  comments, `TODO` without an owner or ticket.

### A11. Logging and configuration (shape)

- **Structured logs** with a component prefix and a correlation id
  (`[OrderService] [${requestId}] payment captured in ${durationMs}ms`). Levels: `debug` routine,
  `info` business events, `warn` degraded, `error` real failures.
- **One logger per class/module**, injected or created from the class name; no `console.*`/
  `print` in application code.
- **All tunables from config/env with safe defaults**, read once at startup into a typed,
  validated config object — never `process.env`/`os.getenv` scattered through business logic.
- **Everything started is stopped** (timers, pools, connections) in lifecycle/shutdown hooks.

### A12. Testability

- **Inject dependencies** (constructor injection, function parameters, interface fields) so tests
  replace IO without monkey-patching.
- **Pure functions for rules** — tested with plain inputs/outputs, table-driven where there are
  many cases.
- **Tests next to source**, named for behaviour (`returns 404 when the order does not exist`),
  arranged as Arrange / Act / Assert, one behaviour per test.

---

## Part B — Per-language conventions

### B1. TypeScript / NestJS

- **Tooling**: `strict: true` in `tsconfig`; ESLint with `typescript-eslint` recommended +
  `eslint-config-prettier`; Prettier `{ "singleQuote": true, "trailingComma": "all" }`, 2-space
  indent, 100-char lines.
- **Naming**: `camelCase` variables/functions, `PascalCase` types/classes/enums, `UPPER_SNAKE_CASE`
  module constants, kebab-case file names with role suffix (`order.service.ts`,
  `order.repository.ts`, `order.types.ts`, `order.utils.ts`, `order.errors.ts`).
- **Imports** grouped: framework (`@nestjs/*`) → third-party → internal shared libs → relative
  feature modules; no deep relative chains (`../../../`) — use path aliases.
- **Utilities as exported `const` arrow functions** in `*.utils.ts`; **services as `@Injectable()`
  classes** with constructor injection, `private readonly` dependencies and a per-class logger:
  ```ts
  @Injectable()
  export class InvoiceService {
    private readonly logger = new Logger(InvoiceService.name);

    constructor(
      private readonly invoices: InvoiceRepository,
      private readonly payments: PaymentGateway,
      private readonly config: AppConfig,
    ) {}
  }
  ```
- **Types**: `interface` for object contracts that may be implemented/extended, `type` for unions
  and compositions; `enum` or `as const` objects for closed sets; `Record<K, V>` for lookup tables;
  `unknown` + type guards (or zod/class-validator) for external data; avoid `any`.
- **NestJS**: DTOs with `class-validator`, global `ValidationPipe({ whitelist: true })`; modules
  per feature; providers for IO behind interfaces/tokens; config via `@nestjs/config` with a
  validated schema; exception filters map domain errors to HTTP; `onModuleInit` /
  `onModuleDestroy` own resource lifecycle.
- **Async**: always `await` or return promises; `Promise.all` for independent work in bounded
  chunks; `for await` for streams; wrap callback APIs once in a promise helper that settles exactly once.
- **Tests**: Jest with `Test.createTestingModule`, dependencies as typed mocks (`jest.Mocked<T>`),
  `*.spec.ts` next to the source.

### B2. Python

- **Tooling**: Python 3.12+, `uv`, ruff (`E,F,I,UP,B,SIM`) + black (or `ruff format`), mypy/pyright
  in strict mode, line length 100, double quotes. Follow PEP 8 naming: `snake_case` functions and
  variables, `PascalCase` classes, `UPPER_SNAKE_CASE` constants, `_leading_underscore` private.
- **Module layout**: module docstring (purpose + how to extend) → `from __future__ import annotations`
  → imports (stdlib / third-party / local, sorted) → `logger = logging.getLogger(__name__)` →
  constants → types → functions/classes.
- **Section dividers** in long modules: `# ── Handlers ─────────────────────────────`.
- **Typing**: built-in generics (`list[str]`, `dict[str, int]`), `X | None`,
  `collections.abc` (`Callable`, `Iterable`, `Awaitable`), `Protocol` for structural interfaces,
  `TypeAlias` for callable shapes, `Literal`/`Enum` for closed sets.
- **Data**: `@dataclass(frozen=True, slots=True)` for value objects; Pydantic models at IO
  boundaries; plain dataclasses for mutable context.
- **Keyword-only arguments** (`def create_invoice(*, customer_id, items, due_in_days=30)`) for
  more than ~3 parameters or any boolean flag.
- **Patterns the Pythonic way**: registries via decorators, strategies as dicts of callables,
  context managers for resources (`with`/`async with`), generators for streaming.
- **Errors**: custom exception hierarchy per package; `raise X(...) from exc`; broad `except`
  only at isolation boundaries with a reason (`# noqa: BLE001 - one plugin must not break the loop`).
- **Logging** with `%`-style args, not f-strings: `logger.warning("plugin %s skipped: %s", name, exc)`.
- **Docstrings**: one line for simple functions; Google style for public functions with
  non-obvious parameters or raised errors.
- **Tests**: pytest, `tests/test_<module>.py`, fixtures and `parametrize`, `uv run pytest -q`.

### B3. Go

- **Tooling**: `gofmt`/`goimports`, `go vet`, `golangci-lint` (errcheck, staticcheck, revive,
  gosec); follow Effective Go and Go Code Review Comments.
- **Layout**: `cmd/<binary>/main.go`, `internal/<domain>/`, small packages named by what they
  provide (`billing`, `storage`), no `util`/`common` dumping grounds.
- **Naming**: `MixedCaps`, short receiver names, initialisms kept upper-case (`userID`, `HTTPClient`),
  getters without `Get` (`Name()`), interfaces named by behaviour (`Reader`, `Notifier`) and
  defined where they are consumed.
- **Doc comments on every exported identifier, starting with its name**:
  `// Charge captures the payment for the given order.`
- **Errors**: return `error` last; wrap with context `fmt.Errorf("charging order %s: %w", id, err)`;
  sentinel errors (`var ErrNotFound = errors.New("not found")`) and typed errors checked with
  `errors.Is`/`errors.As`; never panic for expected failures.
- **Config structs** with tags and field comments, an `applyDefaults()` step and a `Validate()`
  that accumulates errors (`errors.Join`) with actionable messages.
- **Concurrency**: `context.Context` first parameter for anything that blocks; timeouts on every
  client/dialer; `errgroup` with `SetLimit` for bounded fan-out; mutex-guarded small structs for
  shared state; close channels from the sender only.
- **Test seams**: small interfaces or function-valued fields; table-driven `_test.go` files with
  `t.Run` subtests next to the code.

### B4. Java / Spring Boot

- Google Java Style / Effective Java; `final` by default, records for value objects, `Optional`
  only as a return type.
- Constructor injection only; `@ConfigurationProperties` (validated) for tunables; custom
  unchecked exceptions with an error code mapped by `@ControllerAdvice`; SLF4J with `{}`
  placeholders; JUnit 5 + Mockito + AssertJ, tests mirroring the package structure.

### B5. Any other language

Apply Part A with the language's mainstream style guide, formatter and linter, and match the
existing repo first.

---

## Part C — Commits

- Ticket style: `[TICKET-ID] Short description (fix)` — e.g. `[ABC-123] Slow export on large reports (fix)`.
- Conventional Commits: `feat: …`, `fix(billing): …`, `refactor: …`, `chore: …`.
- Imperative mood, one concern per commit. Match whichever style the repo already uses.

---

## Checklist before handing code to Kunal

1. Could a new teammate understand every name without opening its definition?
2. Guards first, flat happy path, functions small with ≤ 3 parameters and no boolean flags?
3. Each class/module has one responsibility; IO injected; business rules pure?
4. Any second copy of logic extracted; any `if/else` ladder replaced by a map, strategy or `switch`?
5. Patterns used only where they remove real duplication or coupling — no speculative abstractions?
6. Right data structures, no N+1 or O(n²) loops over growing data, bounded concurrency?
7. Strict types, no `any`/unchecked casts, no magic values, tunables in config?
8. Typed errors with codes and actionable messages; structured logs with a correlation id?
9. Comments only where they explain *why*; public API documented; no commented-out code?
10. Formatter and linter clean; tests next to the source covering the new behaviour?
