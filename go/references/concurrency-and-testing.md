# Go Concurrency, Testing & Toolchain Debugging

## Concurrency Patterns

**Anti-Pattern:** Heavy, static Worker Pools. Go's scheduler is incredibly efficient; you don't need to manually manage pools of workers like OS threads in other languages.

### 1. Share Memory by Communicating

Don't use mutexes to protect shared data if you can pass that data over a channel instead. Channels orchestrate execution; mutexes serialize execution.

### 2. Bounded Concurrency

To limit concurrency, use `errgroup` with `SetLimit`. Don't hand-roll a semaphore channel or a rigid worker pool when this exists.

```go
func FetchAll(ctx context.Context, urls []string, maxConcurrent int) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(maxConcurrent)

    for _, url := range urls {
        g.Go(func() error {
            return fetch(ctx, url)
        })
    }

    return g.Wait()
}
```

Loop variables are per-iteration since Go 1.22 — never emit the old `url := url` capture line.

When you don't need error propagation, `sync.WaitGroup.Go` (Go 1.25) removes the Add/Done boilerplate:

```go
var wg sync.WaitGroup
for _, url := range urls {
    wg.Go(func() { process(url) })
}
wg.Wait()
```

### 3. Never Start a Goroutine Without Knowing How It Stops

Every `go func()` must have a clear exit condition, usually governed by a `context.Context` or a closed channel.

### 4. `sync/atomic` Typed Values (Go 1.19)

Use the typed atomic values instead of the function-based API:

```go
// Old (still works but avoid for new code)
var count int64
atomic.AddInt64(&count, 1)
val := atomic.LoadInt64(&count)

// New — type-safe, no pointer arithmetic
var count atomic.Int64
count.Add(1)
val := count.Load()

// Other typed atomics
var flag  atomic.Bool
var ptr   atomic.Pointer[MyStruct]
var val32 atomic.Int32
var val64 atomic.Uint64
```

### 5. `context.WithoutCancel` (Go 1.21)

When you need to detach a context's cancellation but preserve its values (e.g., for a background task that should outlive a request):

```go
// The background job should keep running even after the HTTP request context cancels
bgCtx := context.WithoutCancel(requestCtx)
go doBackgroundWork(bgCtx)
```

## Testing Patterns

**Anti-Pattern:** Relying on heavy BDD frameworks (like Ginkgo) or complex mocking generation tools. Go testing should just be Go programming.

### 1. Table-Driven Tests

The absolute standard for unit testing in Go. Iterate over a slice of structs containing inputs and expected outputs using `t.Run()`.

```go
func TestParseConfig(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr bool
    }{
        {"valid config", "port=8080", false},
        {"invalid format", "port=abc", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := ParseConfig(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("ParseConfig() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

### 2. Meaningful Helpers with `t.Helper()`

When extracting repeated assertion logic, always call `t.Helper()` to ensure failures point to the actual test case, not the helper function line.

### 3. Fakes and Stubs over Heavy Mocks

Leverage Go's implicit interfaces to write simple, manual fakes. This keeps test dependencies lightweight and test logic transparent.

### 4. Golden Files and the `testdata` Directory

For tests requiring complex inputs or producing large outputs, use a directory named `testdata`. The `go test` tool explicitly ignores these directories.

### 5. Filesystem Abstraction (The Afero Pattern)

Do not hardcode `os` package calls deep within business logic. Accept an interface for the filesystem so tests can run in memory without touching the disk. `github.com/spf13/afero` is the industry standard for this.

```go
import "github.com/spf13/afero"

type FileProcessor struct {
    fs afero.Fs
}

func NewFileProcessor(fs afero.Fs) *FileProcessor {
    return &FileProcessor{fs: fs}
}
```

In tests, inject `afero.NewMemMapFs()` to completely eliminate disk I/O and prevent flaky, slow tests.

### 6. `cmp` over DeepEqual

For comparing complex structs or maps, use `github.com/google/go-cmp/cmp` for rich, readable diffs instead of the strict `reflect.DeepEqual`.

### 7. Modern `testing` Additions (Go 1.24+)

```go
// Test-scoped context — canceled automatically when the test ends.
// Use instead of context.Background() in tests.
func TestFetch(t *testing.T) {
    ctx := t.Context()
    ...
}

// Benchmarks: b.Loop() replaces the classic b.N loop — more accurate,
// prevents the compiler from optimizing the benchmarked call away.
func BenchmarkParse(b *testing.B) {
    for b.Loop() {
        Parse(input)
    }
}

// t.Chdir(dir) changes working directory for the test and restores it after.
```

### 8. `testing/synctest` for Concurrent Code (Go 1.25)

Test goroutines and timeouts deterministically with a fake clock — no `time.Sleep` in tests, no flaky timing assumptions:

```go
func TestTimeout(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        ctx, cancel := context.WithTimeout(t.Context(), time.Second)
        defer cancel()
        time.Sleep(2 * time.Second) // instant — fake clock advances when all goroutines block
        if ctx.Err() == nil {
            t.Fatal("expected timeout")
        }
    })
}
```

Never write `time.Sleep(100 * time.Millisecond)` to "wait for a goroutine" in a test. Use synctest, channels, or explicit synchronization.

## Debugging: The Go Toolchain Is Not the Problem

**The Go tool is extremely reliable. It is almost never the source of a bug.**

When debugging, do not waste time suspecting `go run`, `go build`, `go test`, or the build cache. The Go toolchain does what it says:

- `go run` always recompiles from source. It does not use a stale cached binary.
- `go build` is deterministic and correct.
- `go test` runs the actual compiled test binary.
- The build cache is keyed by source content — if the source changed, the cache is invalidated automatically.

**If an error persists after you edit the code, the explanation is one of these — in order of likelihood:**

1. The edit did not fix the underlying logic error.
2. The edit was made in the wrong file, wrong function, or wrong package.
3. There is a second call site with the same bug that was not updated.
4. The error is coming from a different code path than the one being edited.

**What to do instead of blaming the tool:**

- Re-read the error message carefully. Go's error messages are accurate.
- Confirm the file you edited is actually the file being compiled (`go list -f '{{.GoFiles}}' .`).
- Add a `fmt.Println` or `t.Log` at the exact site to verify execution reaches it.
- Check that all call sites of a changed function were updated.

Do not suggest clearing the build cache (`go clean -cache`), restarting the Go toolchain, or any other tool-level intervention before first exhausting all code-level explanations. The tool is not lying to you.
