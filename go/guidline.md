# Go Engineering Guidelines for Agents

> A practical, directive guide for AI agents writing idiomatic, production-grade, high-performance Go. Follow these rules by default; deviate only with explicit justification.

---

## Table of Contents

1. [Core Philosophy](#1-core-philosophy)
2. [Project Layout](#2-project-layout)
3. [Naming & Code Style](#3-naming--code-style)
4. [Error Handling](#4-error-handling)
5. [Interface Design](#5-interface-design)
6. [Concurrency](#6-concurrency)
7. [Idiomatic Design Patterns](#7-idiomatic-design-patterns)
8. [Test-Driven Development](#8-test-driven-development)
9. [Performance Engineering](#9-performance-engineering)
10. [Observability](#10-observability)
11. [Production Readiness](#11-production-readiness)
12. [Security](#12-security)
13. [Dependencies & Tooling](#13-dependencies--tooling)
14. [Agent Workflow Checklist](#14-agent-workflow-checklist)

---

## 1. Core Philosophy

Go values simplicity, explicitness, and composition. Internalize these proverbs (Rob Pike) and apply them reflexively:

- **Clear is better than clever.** Prefer obvious code over compact code.
- **Errors are values.** Handle them with normal control flow, not exceptions.
- **Don't communicate by sharing memory; share memory by communicating.** Prefer channels for coordination, mutexes for protecting state.
- **A little copying is better than a little dependency.** Don't pull in a library for trivial functionality.
- **The bigger the interface, the weaker the abstraction.** Keep interfaces minimal.
- **Make the zero value useful.** Design types so their zero state is valid.
- **`gofmt`'s style is nobody's favorite, yet `gofmt` is everybody's favorite.** Never argue about formatting.
- **Don't panic.** Reserve panics for truly unrecoverable conditions during initialization.
- **Composition over inheritance.** Go has no inheritance; use embedding and interfaces.

### Decision heuristics

- If two implementations are equally correct, pick the simpler one.
- If a feature isn't needed right now, don't build it (YAGNI).
- If a function is doing two things, split it.
- If you can't test it easily, the design is wrong.

---

## 2. Project Layout

Follow the community-standard layout. Do not invent new top-level directories without reason.

```
myservice/
├── cmd/
│   └── myservice/
│       └── main.go          # Entry point; wiring only, no business logic
├── internal/                # Private packages (enforced by the compiler)
│   ├── config/
│   ├── domain/              # Core business types and rules
│   ├── transport/           # HTTP/gRPC handlers
│   ├── storage/             # Persistence adapters
│   └── service/             # Application services orchestrating domain
├── pkg/                     # Public, importable packages (use sparingly)
├── api/                     # OpenAPI, protobuf, schema definitions
├── migrations/              # Database migrations
├── test/                    # Integration/E2E tests requiring fixtures
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

**Rules:**

- Prefer `internal/` over `pkg/`. Code under `internal/` cannot be imported by other modules, which is almost always what you want.
- `cmd/<binary>/main.go` should be thin: parse config, construct dependencies, start the process. No business logic.
- One package per directory. Package name matches the directory name, not the path.
- Avoid deeply nested packages. Flat is better than nested.
- Do not create a `util`, `common`, `helpers`, or `misc` package. These become garbage dumps. Put utilities in the package that owns the concept.

---

## 3. Naming & Code Style

### Formatting

- Always run `gofmt` (or `goimports`). No exceptions.
- Use `golangci-lint` with `staticcheck`, `govet`, `errcheck`, `ineffassign`, `gosec`, `revive` enabled.

### Naming conventions

| Item | Convention | Example |
|---|---|---|
| Packages | lowercase, short, singular, no underscores | `user`, `httpclient` |
| Files | lowercase with underscores | `user_repository.go` |
| Exported identifiers | `MixedCaps` | `ParseConfig`, `UserID` |
| Unexported identifiers | `mixedCaps` | `parseConfig`, `userID` |
| Interfaces (single method) | `-er` suffix | `Reader`, `Notifier` |
| Receivers | 1–2 letter abbreviation, consistent across methods | `u *User`, not `self` or `this` |
| Acronyms | All caps or all lower | `userID`, `HTTPServer`, not `UserId` or `HttpServer` |
| Test files | `_test.go` suffix | `user_test.go` |

### Style rules

- Package comment on the `package` line: `// Package user provides ...`
- Every exported identifier gets a doc comment starting with its name: `// Parse returns ...`
- Line length: no hard limit, but break long lines at logical points. ~100–120 chars is a reasonable soft ceiling.
- Prefer early returns over nested conditionals. Maximum nesting depth: 3.
- Declare variables as close to use as possible.
- Use `:=` for declaration + assignment; use `var` for zero-value declarations or when the type must be explicit.
- Group imports: stdlib, then third-party, then local, separated by blank lines. `goimports` handles this.
- No getters/setters unless there's real logic. Export the field directly if it's just a field.
- Don't stutter: `user.User` is bad; rename to `user.Account` or move to a different package.

---

## 4. Error Handling

Errors are the most important cross-cutting concern in Go. Get this right.

### Rules

1. **Always handle errors.** Never `_ = someFn()` to silence them. If you genuinely want to ignore, comment why.
2. **Return errors, don't panic.** Panic is for programmer errors (nil pointer, index out of bounds on a known-good input) or unrecoverable initialization failures.
3. **Wrap errors with context using `%w`.** This preserves the chain for `errors.Is`/`errors.As`.
4. **Don't log and return.** Pick one. Logging at every layer produces duplicate noise. Log at the boundary that handles the error.
5. **Error strings start lowercase and don't end with punctuation.** They get concatenated with other strings.

### Patterns

```go
// Wrapping
if err := db.Insert(u); err != nil {
    return fmt.Errorf("insert user %q: %w", u.Email, err)
}

// Sentinel errors for expected, matchable conditions
var ErrNotFound = errors.New("not found")

// Typed errors when callers need structured data
type ValidationError struct {
    Field string
    Msg   string
}
func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation: %s: %s", e.Field, e.Msg)
}

// Checking
if errors.Is(err, ErrNotFound) { ... }

var vErr *ValidationError
if errors.As(err, &vErr) {
    // access vErr.Field
}
```

### Anti-patterns

- ❌ `if err != nil { return err }` without wrapping when context would help debugging.
- ❌ `panic(err)` in library code.
- ❌ Comparing errors with `==` across package boundaries; use `errors.Is`.
- ❌ Creating error hierarchies with deep inheritance; prefer wrapping.
- ❌ Returning `nil, nil` from functions that return `(T, error)`. Either return a value or an error.

---

## 5. Interface Design

### Rules

- **Accept interfaces, return structs.** Functions take the narrowest interface they need and return concrete types so callers get full functionality.
- **Define interfaces where they are consumed, not where implementations live.** The consumer knows what it needs; the producer doesn't.
- **Keep interfaces small.** 1–3 methods is typical. `io.Reader` has one method and is the most useful interface in Go.
- **Don't define interfaces speculatively.** Add an interface when you have a second implementation or when you need it for testing — not before.

### Example

```go
// GOOD: defined next to the consumer, narrow.
package billing

type InvoiceSender interface {
    Send(ctx context.Context, inv *Invoice) error
}

type Service struct {
    sender InvoiceSender
}

func (s *Service) Charge(ctx context.Context, inv *Invoice) error {
    // ...
    return s.sender.Send(ctx, inv)
}
```

```go
// GOOD: concrete implementation lives elsewhere, doesn't know about the interface.
package email

type Client struct{ /* ... */ }

func (c *Client) Send(ctx context.Context, inv *billing.Invoice) error { /* ... */ }
```

### Anti-patterns

- ❌ Interfaces that mirror a struct 1:1 ("header interfaces"). This just doubles the API surface.
- ❌ `interface{}` / `any` as a default parameter type. Use generics or concrete types.
- ❌ Embedding multiple interfaces into a giant "service" interface. Pass the small interfaces the function actually needs.

---

## 6. Concurrency

Concurrency is Go's differentiator. Misuse causes most production bugs. Apply these rules strictly.

### Goroutine lifecycle

**Every goroutine must have a defined exit condition.** Leaking goroutines is a bug.

```go
// GOOD: context controls lifetime, WaitGroup confirms cleanup.
func (s *Server) Start(ctx context.Context) error {
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        defer wg.Done()
        s.runLoop(ctx)
    }()
    <-ctx.Done()
    wg.Wait()
    return nil
}
```

### Context propagation

- **Every function that does I/O, blocks, or spawns goroutines accepts a `context.Context` as its first parameter.** Parameter name is always `ctx`.
- Never store a context in a struct. Pass it through the call stack.
- Never pass `nil` as a context. Use `context.TODO()` if you genuinely don't have one yet and mark it for follow-up.
- Derive new contexts with `WithCancel`, `WithTimeout`, `WithDeadline`, or `WithValue`. Call the cancel function (typically `defer cancel()`).

### Channel rules

- The sender closes the channel, not the receiver. Closing a closed channel panics.
- Unbuffered channels synchronize; buffered channels queue. Don't use buffered channels to "fix" a race — that's hiding a bug.
- Reading from a closed channel returns the zero value immediately. Use the two-value form: `v, ok := <-ch`.

### Synchronization primitives

| Use case | Primitive |
|---|---|
| Protect shared mutable state | `sync.Mutex` |
| Read-mostly shared state | `sync.RWMutex` |
| Run exactly once | `sync.Once` |
| Wait for N goroutines | `sync.WaitGroup` |
| Coordinate errors across goroutines | `golang.org/x/sync/errgroup` |
| Reuse allocations | `sync.Pool` |
| Lock-free counters | `sync/atomic` |

### errgroup pattern

```go
func fetchAll(ctx context.Context, ids []string) ([]Item, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]Item, len(ids))

    for i, id := range ids {
        i, id := i, id // capture (unnecessary in Go 1.22+ but harmless)
        g.Go(func() error {
            item, err := fetch(ctx, id)
            if err != nil {
                return fmt.Errorf("fetch %s: %w", id, err)
            }
            results[i] = item
            return nil
        })
    }
    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

### Testing for races

Always run tests with `-race` in CI. The race detector finds real bugs.

```bash
go test -race ./...
```

---

## 7. Idiomatic Design Patterns

Don't port GoF patterns verbatim. Go has its own idioms.

### Functional options

Use for constructors with many optional parameters. This is the canonical Go pattern for configuration.

```go
type Server struct {
    addr    string
    timeout time.Duration
    logger  *slog.Logger
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func WithLogger(l *slog.Logger) Option {
    return func(s *Server) { s.logger = l }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        addr:    addr,
        timeout: 30 * time.Second,       // sensible default
        logger:  slog.Default(),
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

### Constructor functions

- Name them `New` or `NewFoo`. Return `(*T, error)` if construction can fail.
- Validate inputs. Fail fast on invalid configuration.

### Dependency injection

Pass dependencies as constructor arguments. Do not use reflection-based DI frameworks. If wiring gets complex, use [`google/wire`](https://github.com/google/wire) for compile-time generation.

```go
func NewOrderService(
    repo OrderRepository,
    payments PaymentGateway,
    clock Clock,
    logger *slog.Logger,
) *OrderService {
    return &OrderService{repo: repo, payments: payments, clock: clock, logger: logger}
}
```

### Embedding (composition)

Use struct embedding to compose behavior. Don't abuse it for "inheritance."

```go
type AuditedRepo struct {
    *UserRepo                // promotes methods; wrap selectively if you need interception
    auditor Auditor
}
```

### Middleware (decorator pattern for handlers)

```go
type Middleware func(http.Handler) http.Handler

func Logging(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            next.ServeHTTP(w, r)
            logger.Info("request",
                "method", r.Method,
                "path", r.URL.Path,
                "duration", time.Since(start),
            )
        })
    }
}

func Chain(h http.Handler, mws ...Middleware) http.Handler {
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}
```

### Worker pool

```go
func worker(ctx context.Context, jobs <-chan Job, results chan<- Result) {
    for {
        select {
        case <-ctx.Done():
            return
        case j, ok := <-jobs:
            if !ok {
                return
            }
            results <- process(j)
        }
    }
}
```

### Pipeline

Chain stages via channels. Each stage reads from its input channel and writes to its output.

### Null Object / Zero value

Make zero values useful. A `sync.Mutex{}` is ready to use. A `bytes.Buffer{}` is ready to read/write. Design your types the same way when possible.

---

## 8. Test-Driven Development

TDD is not optional for production code. Write the test first, watch it fail, make it pass, then refactor.

### The cycle

1. **Red:** Write the smallest failing test that describes a behavior you want.
2. **Green:** Write the minimum code to make it pass. Resist gold-plating.
3. **Refactor:** Improve names, structure, and duplication with the safety net of green tests.
4. Commit at each green. Commits should be small and reviewable.

### Rules

- Every exported function has tests.
- Every bug fix begins with a failing test that reproduces the bug.
- Tests live in `foo_test.go` alongside `foo.go`, in either the same package (white-box) or `foo_test` package (black-box, preferred for library APIs).
- Prefer the standard `testing` package. Add [`testify`](https://github.com/stretchr/testify) only if the team wants richer assertions — it's not required.
- No test should depend on another test's state. Each test is independent.
- Tests must be deterministic. No sleeps-for-timing. Use fakes and clocks.

### Table-driven tests

The canonical Go test style. Use it by default for multi-case tests.

```go
func TestParseDuration(t *testing.T) {
    t.Parallel()

    tests := []struct {
        name    string
        input   string
        want    time.Duration
        wantErr bool
    }{
        {"seconds", "30s", 30 * time.Second, false},
        {"minutes", "5m", 5 * time.Minute, false},
        {"invalid", "banana", 0, true},
        {"empty", "", 0, true},
    }

    for _, tt := range tests {
        tt := tt
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got, err := ParseDuration(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("err = %v, wantErr = %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("got %v, want %v", got, tt.want)
            }
        })
    }
}
```

### Parallel tests

- Mark tests `t.Parallel()` unless they share mutable state.
- Capture loop variables (`tt := tt`) in subtests in Go <1.22.

### Test helpers

```go
func mustOpenDB(t *testing.T) *sql.DB {
    t.Helper()                       // error reports show the caller's line
    db, err := sql.Open("pgx", testDSN)
    if err != nil {
        t.Fatalf("open db: %v", err)
    }
    t.Cleanup(func() { db.Close() }) // runs after test completes
    return db
}
```

### Fakes over mocks

Prefer hand-written fakes (in-memory implementations of your interfaces) over mock-generation frameworks. Fakes are reusable, type-safe, and refactor cleanly. Generate mocks (with `mockgen` or `moq`) only for large interfaces or external dependencies you can't fake ergonomically.

### Fuzzing

For any function that parses untrusted input, write a fuzz test.

```go
func FuzzParse(f *testing.F) {
    f.Add("30s")
    f.Add("")
    f.Fuzz(func(t *testing.T, s string) {
        _, _ = ParseDuration(s) // must not panic
    })
}
```

Run with `go test -fuzz=FuzzParse -fuzztime=30s`.

### Benchmarks

For performance-sensitive code, add a benchmark before optimizing. Measure, don't guess.

```go
func BenchmarkParse(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _, _ = ParseDuration("30s")
    }
    b.ReportAllocs()
}
```

### Coverage

- Target ≥80% for business logic; 100% for pure functions and critical paths.
- `go test -cover ./...`
- Coverage is a floor, not a ceiling. 90% coverage of the wrong things is useless.

### Integration tests

- Use build tags (`//go:build integration`) to separate from unit tests.
- Use [testcontainers-go](https://github.com/testcontainers/testcontainers-go) for real dependencies (Postgres, Redis) rather than mocks.
- Seed data in `t.Cleanup` teardown, not shared fixtures.

---

## 9. Performance Engineering

**Rule zero: measure first.** Never optimize without a benchmark and a profile.

### Profiling workflow

1. Add a benchmark that exercises the hot path.
2. Profile: `go test -bench=. -cpuprofile=cpu.out -memprofile=mem.out`
3. Analyze: `go tool pprof cpu.out`
4. For running services, expose `net/http/pprof` on an internal port.

### Common wins

- **Pre-size slices and maps.** `make([]T, 0, n)` avoids repeated allocation during `append`.
- **Reuse buffers.** Use `strings.Builder` for string concatenation in loops. Use `sync.Pool` for hot-path temporary objects.
- **Avoid unnecessary pointers.** Small structs (≤ ~64 bytes, rule of thumb) are faster passed by value than by pointer due to escape analysis and GC pressure.
- **Avoid `interface{}` on hot paths.** Boxing allocates. Use generics instead.
- **Preallocate with `copy` over `append` when size is known.**
- **Use `io.Copy` for stream copying**; it uses internal buffering and avoids allocating intermediate slices.
- **Minimize lock scope.** Hold mutexes for the shortest possible time. Consider sharding or `sync.Map` for highly contended maps.

### Escape analysis

Check where values allocate: `go build -gcflags="-m" ./...`. Escape to heap is not always bad, but on hot paths it matters.

### Memory layout

Order struct fields from largest to smallest to minimize padding:

```go
// GOOD: 24 bytes
type User struct {
    ID      int64   // 8
    Name    string  // 16
    Active  bool    // 1 (+ 7 padding)
}

// BAD: 32 bytes due to padding
type User struct {
    Active  bool    // 1 (+ 7 padding)
    ID      int64   // 8
    Name    string  // 16
}
```

### Anti-patterns

- ❌ Premature optimization. Readability first; benchmark before changing.
- ❌ Using goroutines for work that's CPU-bound and already parallelized by the runtime. Goroutines aren't free.
- ❌ `time.Sleep` as a sync primitive.
- ❌ Reflection on hot paths. Use code generation or generics.

---

## 10. Observability

### Structured logging

Use `log/slog` (stdlib, Go 1.21+). Do not use `fmt.Println` or `log.Printf` for production logs.

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

logger.InfoContext(ctx, "order created",
    "order_id", order.ID,
    "user_id", order.UserID,
    "amount_cents", order.Amount,
)
```

**Rules:**

- Log at boundaries: service start/stop, request in/out, external call in/out, unrecoverable errors.
- Include a request ID / trace ID in every log line. Propagate it through `context`.
- Never log secrets, tokens, passwords, PII, or full request bodies. Redact or hash.
- Use keys consistently: `user_id`, not sometimes `user` and sometimes `uid`.
- Log levels: `DEBUG` (dev only), `INFO` (normal operations), `WARN` (recoverable anomaly), `ERROR` (operator attention needed).

### Metrics

Use [Prometheus client](https://github.com/prometheus/client_golang). Export the four golden signals: latency, traffic, errors, saturation.

- **Counters:** monotonically increasing events (requests, errors).
- **Gauges:** current state (goroutines, queue depth).
- **Histograms:** distributions (request duration).
- Keep cardinality bounded. Never use user IDs, request IDs, or URLs-with-params as labels.

### Distributed tracing

Use [OpenTelemetry](https://opentelemetry.io/docs/languages/go/) for tracing. Wrap inbound handlers and outbound clients. Propagate trace context via the `context.Context`.

---

## 11. Production Readiness

### Graceful shutdown

Every long-running process must drain in-flight work on signal.

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           handler,
        ReadHeaderTimeout: 5 * time.Second,
        ReadTimeout:       15 * time.Second,
        WriteTimeout:      15 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    go func() {
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            slog.Error("server failed", "err", err)
            os.Exit(1)
        }
    }()

    <-ctx.Done()
    slog.Info("shutting down")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(shutdownCtx); err != nil {
        slog.Error("shutdown failed", "err", err)
    }
}
```

### HTTP server hardening

- **Always set timeouts** on `http.Server` (`ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout`). Defaults are unsafe (no timeout).
- **Always set timeouts on `http.Client`** (`Timeout` or per-request contexts). The default client has no timeout and will hang forever.
- Limit request body size with `http.MaxBytesReader`.
- Use `context` for per-request cancellation and propagate it to downstream calls and DB queries.

### Configuration

- Read configuration from environment variables (12-factor). Use a library like [`envconfig`](https://github.com/kelseyhightower/envconfig) or [`koanf`](https://github.com/knadh/koanf).
- Validate on startup. Fail fast with a clear message if required config is missing.
- Never commit secrets. Use your platform's secret manager.
- Provide sensible defaults for development.

### Health checks

Expose two separate endpoints:

- `/healthz` (liveness): "Am I alive?" — return 200 unless the process is irrecoverable.
- `/readyz` (readiness): "Can I serve traffic?" — check dependencies (DB, queues). Return 503 while dependencies are unavailable.

### Retries and timeouts

- Use exponential backoff with jitter for retries. Don't retry non-idempotent operations without care.
- Set a deadline on every external call. Propagate via `context.WithTimeout`.
- Implement a circuit breaker for flaky dependencies ([`sony/gobreaker`](https://github.com/sony/gobreaker)).

### Rate limiting

Use `golang.org/x/time/rate` for in-process limits; Redis-based token buckets for distributed limits.

---

## 12. Security

- **Parameterized queries only.** Never `fmt.Sprintf` SQL. Use `database/sql` placeholders or a query builder like `sqlc`.
- **Validate all external input.** Length, range, encoding, format.
- **Escape output contextually.** Use `html/template`, not `text/template`, for HTML. Use `template.JS` / `template.URL` for typed escaping.
- **TLS by default.** Use `crypto/tls` with sane defaults; use `tls.Config{MinVersion: tls.VersionTLS12}` at minimum (prefer 1.3).
- **Secrets:** never log, never commit, never embed in binaries. Load from environment or secret manager.
- **Password hashing:** use `golang.org/x/crypto/bcrypt` or `argon2`. Never roll your own.
- **Randomness:** `crypto/rand` for anything security-related. `math/rand` for simulations only.
- **Run `govulncheck` in CI:** `go install golang.org/x/vuln/cmd/govulncheck@latest && govulncheck ./...`
- **Run `gosec` in CI** via golangci-lint.

---

## 13. Dependencies & Tooling

### Go modules

- Use Go modules. Keep `go.mod` and `go.sum` committed.
- Pin to minor versions; upgrade deliberately.
- Prefer the standard library. Add a dependency only when it provides substantial value.
- Audit transitive dependencies before adopting. Check license, maintenance activity, vulnerability history.
- Vendor (`go mod vendor`) for reproducible builds if required by policy.

### Required CI checks

```bash
go fmt ./...                  # formatting
go vet ./...                  # static checks
golangci-lint run ./...       # lint suite
go test -race -cover ./...    # unit tests with race detector
govulncheck ./...             # known vulnerabilities
go build ./...                # compile succeeds
```

### Recommended golangci-lint config

Enable at minimum: `errcheck`, `govet`, `ineffassign`, `staticcheck`, `unused`, `gosec`, `revive`, `gocyclo`, `misspell`, `errorlint`, `bodyclose`, `contextcheck`, `nilerr`.

### Makefile skeleton

```makefile
.PHONY: all fmt lint test build

all: fmt lint test build

fmt:
	gofmt -s -w .
	goimports -w .

lint:
	golangci-lint run ./...

test:
	go test -race -cover ./...

build:
	go build -trimpath -ldflags="-s -w" -o bin/ ./cmd/...
```

### Build flags

- `-trimpath` removes file system paths from binaries (reproducibility, security).
- `-ldflags="-s -w"` strips debug info (smaller binaries).
- `-ldflags="-X main.version=$(git describe)"` embeds version info.

---

## 14. Agent Workflow Checklist

When writing or modifying Go code, an agent should proceed as follows:

### Before writing code

- [ ] Read existing code in the target package. Match its style and patterns.
- [ ] Check `go.mod` for the Go version and available dependencies. Don't add new deps without reason.
- [ ] Identify the interface boundaries. What's the smallest unit that can be tested?

### Red phase

- [ ] Write a failing test that describes the new behavior.
- [ ] Run the test. Confirm it fails for the right reason (not a compile error).

### Green phase

- [ ] Write the minimum code to pass the test.
- [ ] Run `go test -race ./...`. Confirm pass.

### Refactor phase

- [ ] Remove duplication.
- [ ] Improve names.
- [ ] Extract functions when a block needs a comment to explain what it does.
- [ ] Re-run tests.

### Before submitting

- [ ] `gofmt -s -w .`
- [ ] `go vet ./...`
- [ ] `golangci-lint run ./...`
- [ ] `go test -race -cover ./...`
- [ ] `govulncheck ./...`
- [ ] All errors are handled or explicitly ignored with justification.
- [ ] All goroutines have bounded lifetimes.
- [ ] All contexts are propagated; no `context.TODO()` in production paths.
- [ ] All external calls have timeouts.
- [ ] No secrets, PII, or large payloads are logged.
- [ ] Exported identifiers have doc comments.
- [ ] New public APIs have examples (`ExampleFoo` functions) where useful.
- [ ] CHANGELOG / commit message describes *why*, not just *what*.

### When stuck

- Re-read the standard library source. It's the best Go style reference available.
- Search `pkg.go.dev` before inventing something.
- If you need a pattern, check `golang.org/x/...` — the extensions repo is closest to blessed.

---

## Appendix A: Go Proverbs (Rob Pike, abbreviated)

- Don't communicate by sharing memory; share memory by communicating.
- Concurrency is not parallelism.
- Channels orchestrate; mutexes serialize.
- The bigger the interface, the weaker the abstraction.
- Make the zero value useful.
- `interface{}` says nothing.
- Gofmt's style is no one's favorite, yet gofmt is everyone's favorite.
- A little copying is better than a little dependency.
- Syscall must always be guarded with build tags.
- Cgo must always be guarded with build tags.
- Cgo is not Go.
- With the unsafe package there are no guarantees.
- Clear is better than clever.
- Reflection is never clear.
- Errors are values.
- Don't just check errors, handle them gracefully.
- Design the architecture, name the components, document the details.
- Documentation is for users.
- Don't panic.

---

## Appendix B: Further Reading (canonical sources)

- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
- [Google Go Style Guide](https://google.github.io/styleguide/go/)
- [Go Proverbs](https://go-proverbs.github.io/)
- [The Go Memory Model](https://go.dev/ref/mem)
- [Go Blog](https://go.dev/blog/)

---

*End of guidelines. When in doubt, prefer simplicity, explicitness, and testability over cleverness.*
