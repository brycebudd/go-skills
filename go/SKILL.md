---
name: go
description: >
  Expert Go programming skill authored by spf13 (former Go team lead, author of Cobra, Viper, Hugo, Afero).
  Covers idiomatic Go — package design, error handling, interfaces, concurrency, testing, and
  project layout, current through Go 1.25. Use whenever Go code is written, reviewed, debugged, or
  refactored — any .go file, go.mod, CLI tool, or web service, and whenever the user mentions
  Go or golang, even if they don't ask for "idiomatic" code.
---

# Idiomatic Go: The Go Way

Idiomatic Go patterns and best practices for building robust, efficient, and maintainable applications (current through Go 1.25).

## When to Activate

- Writing, reviewing, auditing, or refactoring Go code (especially code that looks like Java/Spring Boot patterns)
- Designing Go packages, modules, or APIs
- Choosing between stdlib and third-party libraries
- Any question about Go project structure, error handling, concurrency, or testing

## Core Principles

### 1. Clear is Better than Clever
Go favors readability and simplicity over abstraction and cleverness. No `Get` prefix on getters.

```go
func LookupUser(id string) (*User, error) {
    user, err := db.FindUser(id)
    if err != nil {
        return nil, fmt.Errorf("finding user %s: %w", id, err)
    }
    return user, nil
}
```

### 2. Make the Zero Value Useful
Design types so their zero value is immediately usable without constructors (like `sync.Mutex` and `bytes.Buffer`).

### 3. Return Early, Keep the Happy Path Left
Handle errors and edge cases immediately and return. Avoid `else` blocks for main logic.

## Quick Rules

- **Packages**: Flat by default; domain-named packages (`auth/`, `billing/`, `jobs/`) one level deep; never `service/`, `repository/`, `utils/`, or `common/`.
- **Interfaces**: Discovered, not designed upfront; defined by the *consumer*; accept interfaces, return concrete structs.
- **Errors**: Check explicitly; wrap with `fmt.Errorf("doing x: %w", err)`; combine with `errors.Join`.
- **Concurrency**: `errgroup.WithContext` + `SetLimit` or `sync.WaitGroup.Go` (Go 1.25); never start a goroutine without an exit condition.
- **Stdlib First**: `slices`, `maps`, `cmp.Or`, `iter.Seq`, `math/rand/v2`, `net/http` ServeMux (Go 1.22+), `log/slog`, `omitzero` (Go 1.24).
- **Testing**: Table-driven tests with `t.Run()`, `t.Helper()`, `t.Context()`, `b.Loop()`, `testing/synctest` (Go 1.25), `afero.NewMemMapFs()`, `go-cmp`.

## Reference Guides

Load the relevant reference file on demand:

- **Architecture, Packages, Interfaces & Error Handling**: [references/architecture-and-apis.md](./references/architecture-and-apis.md) — Flat/domain package layout, `internal/`, consumer-side interfaces, Domain Object entry points, functional options, error wrapping.
- **Concurrency, Testing & Toolchain Debugging**: [references/concurrency-and-testing.md](./references/concurrency-and-testing.md) — `errgroup`, `sync.WaitGroup.Go`, typed `sync/atomic`, `context.WithoutCancel`, table-driven tests, Afero, `testing/synctest`, and why the Go toolchain is not the bug.
- **Standard Library, Generics, HTTP, Syntax & Logging**: [references/stdlib-and-http.md](./references/stdlib-and-http.md) — Generics guidelines, `slices`/`maps`/`cmp`/`iter`/`math/rand/v2`/`omitzero`, Go 1.22 ServeMux + timeouts + graceful shutdown + middleware, `//go:build`, `tool` in `go.mod`, `log/slog`.
