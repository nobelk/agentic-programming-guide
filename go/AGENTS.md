# AGENTS.md

Instructions for AI coding agents working in this repository. Read before making changes.

## Stack

- **Language:** Go (see `go.mod` for version)
- **Layout:** `cmd/<binary>/main.go` entry points, business logic in `internal/`, public packages in `pkg/` (rare)
- **Build:** `make` targets wrap the commands below

## Commands

Run these before declaring any task complete:

```bash
gofmt -s -w .                  # format
goimports -w .                 # organize imports
go vet ./...                   # static analysis
golangci-lint run ./...        # lint suite
go test -race -cover ./...     # tests with race detector
govulncheck ./...              # vulnerability scan
go build ./...                 # compile check
```

If any of these fail, fix before proceeding. Do not disable lints to silence errors.

## Workflow: TDD is mandatory

Every behavioral change follows **red → green → refactor**:

1. **Red.** Write the smallest failing test that describes the new behavior. Run it. Confirm it fails for the right reason.
2. **Green.** Write the minimum code to pass. Resist gold-plating.
3. **Refactor.** Clean up names, duplication, structure. Re-run tests.

Bug fixes start with a failing test that reproduces the bug. No test, no fix.

## Code style

- `gofmt` is law. Never argue with it.
- Package names: lowercase, short, singular, no underscores. No `util`/`common`/`helpers` packages.
- Exported identifiers: `MixedCaps`. Acronyms: `HTTPServer`, `userID` (not `HttpServer`, `userId`).
- Every exported identifier has a doc comment starting with its name: `// Parse returns ...`.
- Prefer early returns. Max nesting depth: 3.
- Line length: soft limit ~120 chars.
- Declare variables close to use. Use `:=` for declaration+assignment.

## Error handling

- **Always handle errors.** Never `_ = fn()` without a comment justifying it.
- **Wrap with context using `%w`:** `fmt.Errorf("insert user %q: %w", email, err)`.
- **Check with `errors.Is` and `errors.As`.** Never `==` across package boundaries.
- **Log or return, never both.** Log at the boundary that handles the error.
- Error strings: lowercase, no trailing punctuation.
- No `panic` in library code. Panics are for unrecoverable init failures only.

## Interfaces

- **Accept interfaces, return structs.**
- Define interfaces at the consumer, not alongside the implementation.
- Keep them small (1–3 methods). Don't define interfaces speculatively.
- Never use `interface{}`/`any` as a default parameter type. Prefer generics or concrete types.

## Concurrency

- **Every goroutine must have a defined exit path.** Leaks are bugs.
- **First parameter is `ctx context.Context`** for any function that does I/O, blocks, or spawns goroutines. Name it `ctx`.
- Never store a context in a struct. Never pass `nil`; use `context.TODO()` and mark it.
- Sender closes channels, not receiver.
- Use `golang.org/x/sync/errgroup` for concurrent operations that can fail.
- Run `go test -race` locally before pushing.

## Testing

- Table-driven tests by default. Subtests via `t.Run(name, ...)`.
- Mark tests `t.Parallel()` unless they share mutable state.
- Use `t.Helper()` in test helpers so failures point at the caller.
- Use `t.Cleanup(...)` for teardown, not shared fixtures.
- Prefer hand-written fakes over generated mocks.
- Fuzz any function that parses untrusted input.
- Integration tests: `//go:build integration` build tag, use testcontainers for real dependencies.
- Coverage floor: 80% on business logic, 100% on pure functions.

## Production readiness

- **HTTP servers:** always set `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout`. Defaults are unsafe.
- **HTTP clients:** always set `Timeout` or use per-request context deadlines. The default client hangs forever.
- **Graceful shutdown:** `signal.NotifyContext` + `srv.Shutdown(ctx)` on SIGTERM.
- **Config:** from environment variables. Validate on startup. Fail fast on missing required values.
- **Secrets:** never log, never commit, never hardcode.
- **Logging:** `log/slog` with JSON handler. Include request/trace IDs. Never log secrets, PII, or full payloads.
- **Metrics:** Prometheus. Bounded label cardinality (no user IDs, request IDs, or URLs as labels).

## Performance

- **Measure before optimizing.** Benchmark + pprof, never guess.
- Pre-size slices and maps: `make([]T, 0, n)`.
- Use `strings.Builder` for concatenation in loops.
- Use `sync.Pool` for hot-path reusable allocations.
- Small structs (<~64 bytes) pass by value, not pointer.
- Order struct fields largest-to-smallest to minimize padding.

## Security

- **Parameterized queries only.** No `fmt.Sprintf` into SQL, ever.
- Validate all external input: length, range, encoding, format.
- `html/template` for HTML output, never `text/template`.
- TLS: minimum `tls.VersionTLS12`, prefer 1.3.
- `crypto/rand` for anything security-related; `math/rand` is for simulations.
- Passwords: `bcrypt` or `argon2`. Never roll your own.

## Dependencies

- Prefer the standard library.
- Before adding a dependency: check license, maintenance, vulnerability history. Justify it in the commit message.
- Keep `go.mod` and `go.sum` committed. Run `go mod tidy` after changes.

## Before submitting

Checklist:

- [ ] Failing test existed first; it now passes.
- [ ] `gofmt`, `go vet`, `golangci-lint`, `go test -race -cover`, `govulncheck` all green.
- [ ] Every `goroutine` has a bounded lifetime.
- [ ] Every external call has a timeout or context deadline.
- [ ] Every error is handled or explicitly justified.
- [ ] No secrets, PII, or large payloads in logs.
- [ ] Exported identifiers have doc comments.
- [ ] Commit message explains *why*, not just *what*.

## When in doubt

- Re-read the existing code in the target package and match its patterns.
- Check the standard library source — it's the best Go style reference available.
- Prefer simplicity, explicitness, and testability over cleverness.
- `Clear is better than clever.`
