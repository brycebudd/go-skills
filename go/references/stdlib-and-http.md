# Go Standard Library, Generics, HTTP, Syntax & Logging

## Generics (Go 1.18+)

Generics exist to eliminate duplicated algorithms, not to create type hierarchies. If you are thinking about generics in terms of inheritance or polymorphism, stop — you are writing Java.

### When to Use Generics

Use generics when you have the **same algorithm** that needs to operate on **multiple concrete types**:

```go
// Good: generic algorithm, concrete types as inputs
func Map[S, T any](slice []S, f func(S) T) []T {
    result := make([]T, len(slice))
    for i, v := range slice {
        result[i] = f(v)
    }
    return result
}

// Good: constraint expresses a meaningful requirement
func Min[T cmp.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}
```

### When NOT to Use Generics

```go
// Bad: generic interface for polymorphism — this is Java
type Repository[T any] interface {
    Find(id string) (T, error)
    Save(entity T) error
}

// Good: a concrete interface for what you actually need
type UserStore interface {
    FindUser(id string) (*User, error)
    SaveUser(u *User) error
}
```

- **Do not** create generic base types, generic services, or generic repositories.
- **Do not** use `any` as a constraint to mean "I don't know the type yet." That's a design smell.
- **Do** use `comparable` when you need map keys or equality checks.
- **Do** use `cmp.Ordered` when you need `<`, `>`, `<=`, `>=`.
- Start with a concrete implementation. Generify only when you have the same logic repeated across 3+ types.
- Generic type aliases are fully supported since Go 1.24.

## Standard Library: Use the New Packages

LLMs frequently suggest third-party utilities or write manual helpers that have been in the standard library since Go 1.21. **Always check stdlib first.**

### `slices` package (Go 1.21)

```go
import "slices"

// Searching and testing
slices.Contains(s, "value")
slices.Index(s, "value")           // returns -1 if not found
slices.ContainsFunc(s, func(v string) bool { return v == "x" })

// Sorting
slices.Sort(s)                     // sorts in place, works on any ordered type
slices.SortFunc(s, func(a, b T) int { return cmp.Compare(a.Name, b.Name) })
slices.IsSorted(s)

// Manipulation
slices.Reverse(s)
slices.Compact(s)                  // removes consecutive duplicates
slices.Delete(s, i, j)            // removes elements [i, j)
slices.Clone(s)                   // shallow copy
slices.Concat(s1, s2, s3)         // concatenate multiple slices (1.22)

// Iterator bridging (1.23)
slices.Collect(it)                // iterator -> slice
slices.Sorted(it)                 // iterator -> sorted slice
slices.Values(s)                  // slice -> iterator
```

Never write `sort.Slice(s, func(i, j int) bool { return s[i] < s[j] })` when `slices.Sort(s)` exists.

### `maps` package (Go 1.21; iterators 1.23)

```go
import "maps"

maps.Keys(m)        // iterator over keys — collect with slices.Collect / slices.Sorted
maps.Values(m)      // iterator over values
maps.Clone(m)       // shallow copy
maps.Copy(dst, src) // copies all entries from src into dst
maps.DeleteFunc(m, func(k K, v V) bool { ... })  // delete entries matching predicate
maps.Equal(m1, m2)  // reports whether two maps are equal

// Common idiom: sorted keys in one line
keys := slices.Sorted(maps.Keys(m))
```

### `cmp` package (Go 1.21)

```go
import "cmp"

cmp.Compare(a, b)    // returns -1, 0, or 1; works on any cmp.Ordered type
cmp.Or(a, b, c)      // returns first non-zero value — replaces ternary workarounds
min(a, b)            // built-in since Go 1.21
max(a, b)            // built-in since Go 1.21
```

`cmp.Or` is especially useful for default-value patterns:

```go
// Instead of: if cfg.Timeout == 0 { cfg.Timeout = 30 * time.Second }
cfg.Timeout = cmp.Or(cfg.Timeout, 30*time.Second)
```

### `errors.Join` (Go 1.20)

```go
// Combine multiple errors — no third-party library needed
err := errors.Join(err1, err2, err3)

// Works correctly with errors.Is and errors.As
if errors.Is(err, ErrNotFound) { ... }
```

Use this instead of `fmt.Errorf("%w; %w", err1, err2)` or any `multierr` package.

### Iterators: `iter` and Range-over-Func (Go 1.23)

The standard way to expose a sequence from an API without allocating a slice. Return `iter.Seq[T]` or `iter.Seq2[K, V]`; callers use plain `range`:

```go
func (idx *Index) All() iter.Seq[*Person] {
    return func(yield func(*Person) bool) {
        for _, p := range idx.people {
            if !yield(p) {
                return
            }
        }
    }
}

// Caller — just a normal range loop
for p := range idx.All() { ... }
```

Prefer returning iterators over slices for large or lazily-produced sequences. Don't invent a custom `Next()/HasNext()` iterator type — that's Java.

### `math/rand/v2` (Go 1.22)

Always import `math/rand/v2`, never the old `math/rand`. Auto-seeded, cleaner API:

```go
import "math/rand/v2"

rand.IntN(100)            // was rand.Intn — note the capital N
rand.N(10 * time.Second)  // generic: random value in [0, n) for any integer type
```

### `encoding/json`: `omitzero` (Go 1.24)

`omitzero` omits any zero value — including `time.Time{}` and zero structs, which `omitempty` never handled correctly:

```go
type Event struct {
    Name      string    `json:"name"`
    StartedAt time.Time `json:"started_at,omitzero"` // omitempty would emit "0001-01-01T..."
}
```

## HTTP: Use the Improved stdlib Router (Go 1.22)

LLMs reflexively recommend gorilla/mux or chi for any routing beyond the trivial. Since Go 1.22, the standard `net/http` ServeMux handles method and path-parameter routing natively.

```go
mux := http.NewServeMux()

// Method-scoped routes
mux.HandleFunc("GET /users", listUsers)
mux.HandleFunc("POST /users", createUser)

// Path parameters — accessed via r.PathValue
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // ...
})

// Wildcard
mux.HandleFunc("GET /files/{path...}", serveFile)
```

Reach for chi or gorilla/mux only when you need named route generation or regex constraints — middleware doesn't justify a framework (see below). For pure method + path routing, the stdlib is sufficient.

### Production Servers: Always Set Timeouts

`http.ListenAndServe(addr, mux)` ships with **no timeouts** — a single slow client can hold a connection open forever (slow-loris). LLM-generated servers almost never set these. Never emit a production server without them:

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,   // slow-loris protection
    ReadTimeout:       10 * time.Second,  // full request read
    WriteTimeout:      30 * time.Second,  // response write (covers handler time)
    IdleTimeout:       120 * time.Second, // keep-alive connections
}
```

For per-route control beyond `WriteTimeout`, use `http.TimeoutHandler` or handler-level `context.WithTimeout`. Outbound calls need the same discipline: `http.DefaultClient` has no timeout either — construct a client with one.

### Graceful Shutdown

Every production server needs a shutdown path that stops accepting connections and drains in-flight requests. The pattern:

```go
func run(ctx context.Context) error {
    ctx, stop := signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)
    defer stop()

    srv := &http.Server{ /* ... timeouts as above ... */ }

    errCh := make(chan error, 1)
    go func() { errCh <- srv.ListenAndServe() }()

    select {
    case err := <-errCh:
        return err // ListenAndServe failed at startup
    case <-ctx.Done():
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        return srv.Shutdown(shutdownCtx) // drain in-flight, then exit
    }
}
```

- `Shutdown` needs a fresh context with its own deadline — the signal context is already canceled.
- Long-lived connections (SSE, websockets) must watch `r.Context()` or they'll hold shutdown until the drain deadline.

### Middleware Is Just a Function

No framework needed. A middleware is `func(http.Handler) http.Handler`:

```go
func withRequestLog(logger *slog.Logger, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        logger.Info("request", "method", r.Method, "path", r.URL.Path, "dur", time.Since(start))
    })
}

// Compose by wrapping — outermost runs first
handler := withRequestLog(logger, withAuth(mux))
```

Don't import a middleware framework for what function composition already does.

## Syntax: Use Current Go

LLMs frequently generate outdated syntax. Know the current idioms:

### Range over Integer (Go 1.22)

```go
// Old
for i := 0; i < 10; i++ { ... }

// New
for i := range 10 { ... }
```

### Build Constraints

```go
// Old (deprecated — do not generate)
// +build linux darwin

// Current
//go:build linux || darwin
```

The `//go:build` form is required since Go 1.17. Never emit the old `// +build` syntax.

### `any` Instead of `interface{}`

```go
// Old
func Print(v interface{}) { ... }

// Current — `any` is a built-in alias for interface{} since Go 1.18
func Print(v any) { ... }
```

### Tool Dependencies in `go.mod` (Go 1.24)

Never generate a `tools.go` file with blank imports. Track dev tools with the `tool` directive:

```bash
go get -tool golang.org/x/tools/cmd/stringer
go tool stringer -type=Color   # run it
```

## Structured Logging with `log/slog` (Go 1.21)

```go
// Pass a logger via context for request-scoped logging
func HandleRequest(ctx context.Context, r *Request) {
    logger := slog.With("request_id", r.ID, "user_id", r.UserID)
    // All subsequent log calls carry these fields automatically
    logger.Info("handling request")
    process(ctx, logger, r)
}

// Group related fields
logger.With(slog.Group("http",
    slog.String("method", r.Method),
    slog.String("path", r.URL.Path),
    slog.Int("status", status),
))

// Log at the right level — LLMs over-use Info
logger.Debug("cache miss", "key", key)     // internal state, high volume
logger.Info("server started", "addr", addr) // lifecycle events
logger.Warn("retrying", "attempt", n)       // recoverable problems
logger.Error("request failed", "err", err)  // needs attention
```

- **Never** use a package-level `log` or `slog` global beyond `main`. Pass `*slog.Logger` as a dependency.
- **Never** log and return an error. Log at the boundary, return the error through the call stack.
- Use `slog.Default()` as the fallback only in `main` or in libraries when no logger is provided.
