# Go Architecture, Packages, Interfaces & Error Handling

## Package Organization: Flat by Default

**Anti-Pattern:** Using deeply nested directory trees or relying heavily on an `internal/` folder by default to artificially enforce "Clean Architecture" layers. This leads to circular dependencies and difficult navigation.

### 1. The Single-Package Default

Start flat. If you are building a microservice or a simple tool, put everything in the root directory (or alongside your `main.go`). Only create a new package when you truly need a new namespace to clarify the code, or when you need to decouple a strictly independent domain.

### 2. The Proper Use of `internal/`

The `internal/` directory has a specific compiler enforcement: it prevents other modules from importing the code inside it.

- **For Applications:** If you are building an executable binary, nobody can import your code anyway. Using `internal/` here is usually just adding unnecessary path depth.
- **For Libraries:** Use `internal/` sparingly. It should be reserved for complex subsystems where you need to share exported types between your own packages, but absolutely must prevent end-users from relying on those types.

```
// Idiomatic: A flat, feature-focused library or simple app
myproject/
├── main.go           # Entry point (if application)
├── server.go         # Core logic
├── config.go         # Configuration
├── parser.go         # Domain specific parsing
├── parser_test.go
├── go.mod
└── go.sum
```

### 3. Domain Packages for Service Applications

When an application has genuinely distinct, independently-testable domains, give each its own top-level package. The rule is still **one level deep** — no `internal/` nesting, no Clean Architecture layers. Each package owns one concern. `main.go` wires them together.

The signal to create a package: can this domain be described in one sentence, and does it have no knowledge of the other packages? If yes, it earns its own package.

```
// Idiomatic: A web service with distinct domain packages
myservice/
├── main.go        # wires everything together; no business logic here
├── config/        # Config struct, env loading
├── auth/          # identity verification, session middleware
├── db/            # data store client + all queries
├── storage/       # blob storage (S3, R2, GCS)
├── billing/       # payment provider + credit ledger
├── jobs/          # job lifecycle + queue dispatch + worker handler (same domain, one package)
├── web/           # HTTP handlers + HTML templates + static assets (tightly coupled by design)
├── transcribe/    # domain-specific processing — independent pure functions
├── Makefile
├── Dockerfile
└── .env.example
```

**Rules for domain packages:**
- Each package has **one clear purpose** — name it after what it does, not what layer it is (`jobs/`, not `service/`)
- Packages do not import each other sideways — cycles are a sign of wrong boundaries; `main` is the wiring point
- Related sub-concerns that always travel together stay in one package (e.g. job creation + worker handler both live in `jobs/` because they share the job lifecycle domain)
- HTTP handlers and the templates they render belong together in `web/` — they are tightly coupled by design
- Do not create `utils/`, `helpers/`, or `common/` packages — these are symptoms of unclear ownership

**Anti-pattern to reject:** Clean Architecture / DDD layers (`service/`, `repository/`, `controller/`, `domain/`). These layer-named packages cause circular imports, force interface proliferation, and add zero clarity in Go. Domain-named packages (`auth/`, `billing/`, `jobs/`) are the correct middle ground between "too flat" and "over-engineered."

## Interface Design

### 1. Interfaces are Discovered, Not Designed Upfront

Write concrete types first. Only define an interface when you discover that multiple types need to be used interchangeably by a consumer.

### 2. Define Interfaces Where They Are Used

Interfaces belong in the package that *consumes* them, not the package that *implements* them. This decouples your packages.

```go
// processor/processor.go

// Idiomatic: The consumer defines exactly what it needs.
// The concrete 'UserStore' doesn't even need to know this interface exists.
type UserFetcher interface {
    FetchUser(id string) (*User, error)
}

type Processor struct {
    fetcher UserFetcher
}
```

### 3. Accept Interfaces, Return Structs

Require the smallest interface possible as an input parameter (e.g., `io.Reader` instead of `*os.File`), but return a concrete struct so callers aren't forced to use type assertions to access specific fields or methods.

## Library API Design

### Domain Object as Entry Point

When designing a Go library that wraps a stateful resource (a vault, a database connection, a config store), make that resource struct the primary object. Its methods return domain-typed sub-objects. **Avoid:**

- Passing a config/resource struct as the first argument to every package-level function
- Package-level global state (like pflag's default `FlagSet`) for library code that may be embedded

**Do this instead:**

```go
// Open the primary resource once
v, err := vault.Open(path)

// Domain operations are methods on the primary object
// Each call is stateless (reloads fresh) — no cached state on the struct
idx, err := v.People()             // returns *people.Index, error
note, err := v.Daily(time.Now())   // returns *daily.Note, error
mtgs, err := v.Meetings()          // returns *meetings.Index, error

// Domain types live in sub-packages — callers use type inference
p, err := idx.FindOne("Steve")     // *people.Person
```

**Why this pattern:**

- The primary struct (`Vault`) is the single entry point — callers need just one import to get started
- Sub-packages define the rich domain types (`people.Person`, `daily.Note`) — each type lives with the logic that owns it
- No global state means the library is safe for concurrent use, multiple instances, and testing
- Stateless method calls (reload fresh each time) keep the struct simple — no cache invalidation logic needed
- Type inference (`:=`) means callers rarely need to explicitly import sub-packages for variable declarations

**When to use a global instance instead:** Only for CLI-only tools (like `pflag` itself) where there is truly only ever one instance and ease of use for end-users outweighs library correctness.

## Configuration and Struct Design

### Functional Options for Complex Initialization

When a struct has many optional configuration parameters, avoid massive constructors. Use the Functional Options pattern.

```go
type Server struct {
    addr    string
    timeout time.Duration
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        addr:    addr,
        timeout: 30 * time.Second, // Sane default
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

## Error Handling

### 1. Errors are Values

Errors aren't exceptions to be caught; they are values to be handled. Check them explicitly.

### 2. Wrap for Context, Not for Stack Traces

When returning an error, add context about what you were trying to do.

```go
// Idiomatic
data, err := os.ReadFile(path)
if err != nil {
    return fmt.Errorf("loading config file %s: %w", path, err)
}
```
