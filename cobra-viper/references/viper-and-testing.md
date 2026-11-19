# Viper Configuration, CLI Testing & Common Mistakes

## Viper Configuration Patterns

### 1. Prefer a Viper Instance over the Global Singleton

`viper.New()` gives you an instance you can inject, isolate in tests, and run concurrently. The package-level `viper.Get*` singleton exists for convenience in small tools — the same trade-off as pflag's default FlagSet. For anything with tests or multiple commands, pass `*viper.Viper` explicitly.

### 2. Unmarshal into Typed Structs

**Anti-Pattern:** Calling `v.GetString("database.host")` deep inside your business logic. This tightly couples your domain to Viper and scatters magic strings throughout your codebase.

Instead, define a strongly-typed configuration struct, unmarshal Viper's state into it at the routing layer (`cmd/`), and pass that struct down.

```go
type Config struct {
    Host string `mapstructure:"host"`
    Port int    `mapstructure:"port"`
}

var cfg Config
if err := v.Unmarshal(&cfg); err != nil {
    return fmt.Errorf("unable to decode config: %w", err)
}
```

### 3. The Binding Hierarchy

Viper merges configuration sources in this order (highest → lowest priority):

1. **Explicit `Set()`** calls in code
2. **Flags** (bound via `BindPFlag` / `BindPFlags`)
3. **Environment variables** (`MYCLI_PORT`)
4. **Config file** (`~/.mycli.yaml`, `./.mycli.yaml`)
5. **Defaults** (`v.SetDefault`)

You must explicitly bind each source. Bind a whole flag set at once rather than flag-by-flag:

```go
// In PersistentPreRunE — after flags are parsed, before RunE
if err := v.BindPFlags(cmd.Flags()); err != nil {
    return err
}
```

Binding in `PersistentPreRunE` (rather than `init()`) also avoids the classic collision where two subcommands bind different flags to the same Viper key and the last `init()` silently wins.

### 4. Environment Variable Mapping

With `v.SetEnvPrefix("MYAPP")` and `v.AutomaticEnv()`:

| Viper key | Environment variable |
|-----------|---------------------|
| `log-level` | `MYAPP_LOG_LEVEL` |
| `serve.addr` | `MYAPP_SERVE_ADDR` |
| `db.password` | `MYAPP_DB_PASSWORD` |

Nested keys with dots or dashes require a replacer to map correctly to env var names:

```go
v.SetEnvKeyReplacer(strings.NewReplacer("-", "_", ".", "_"))
```

**Critical gotcha — `AutomaticEnv` + `Unmarshal`:** `Unmarshal` only walks keys Viper already knows about (from defaults, config file, or explicit binds). A value set *only* via environment variable is invisible to `Unmarshal` unless the key was registered. The fix: call `v.SetDefault` for every key in your config struct, or `v.BindEnv` each key explicitly. This is the most common "my env var is ignored" bug in Viper applications.

### 5. Config File Setup

```go
func initConfig(v *viper.Viper, cmd *cobra.Command) error {
    if cfgFile, _ := cmd.Flags().GetString("config"); cfgFile != "" {
        v.SetConfigFile(cfgFile)
    } else {
        home, err := os.UserHomeDir()
        if err != nil {
            return err
        }
        v.AddConfigPath(home)
        v.AddConfigPath(".")
        v.SetConfigType("yaml")
        v.SetConfigName(".myapp")
    }

    v.SetEnvPrefix("MYAPP")
    v.SetEnvKeyReplacer(strings.NewReplacer("-", "_", ".", "_"))
    v.AutomaticEnv()

    if err := v.ReadInConfig(); err != nil {
        var notFound viper.ConfigFileNotFoundError
        if !errors.As(err, &notFound) {
            return fmt.Errorf("reading config: %w", err)
        }
        // Not found is fine — defaults, env, and flags still apply
    }

    return v.BindPFlags(cmd.Flags())
}
```

Use YAML as the default format — it's readable and supports nesting:

```yaml
# ~/.myapp.yaml
log-level: debug

serve:
  addr: ":9090"
  port: 9090

db:
  host: localhost
  port: 5432
  name: myapp
```

## Testing CLI Commands

**Anti-Pattern:** Testing CLI commands by compiling the binary and using `os/exec`. This is extremely slow, brittle, and makes it difficult to measure test coverage.

**Second anti-pattern:** executing a package-level subcommand directly. `cmd.Execute()` always executes from the *root* of the command tree, and mutating a global command leaks flag state between tests. This is exactly why commands are built by factories.

```go
// Idiomatic: In-memory CLI testing via the factory
func TestServeCommand(t *testing.T) {
    tests := []struct {
        name    string
        args    []string
        want    string
        wantErr bool
    }{
        {"default port", []string{"serve"}, "listening on :8080", false},
        {"custom port", []string{"serve", "--addr", ":9090"}, "listening on :9090", false},
        {"bad flag", []string{"serve", "--bogus"}, "", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            buf := new(bytes.Buffer)

            root := cmd.NewRootCmd() // fresh tree + fresh Viper every test
            root.SetOut(buf)
            root.SetErr(buf)
            root.SetArgs(tt.args) // full CLI invocation, exactly as a user types it

            err := root.ExecuteContext(t.Context())
            if (err != nil) != tt.wantErr {
                t.Fatalf("Execute() error = %v, wantErr %v", err, tt.wantErr)
            }
            if !strings.Contains(buf.String(), tt.want) {
                t.Errorf("output = %q, want substring %q", buf.String(), tt.want)
            }
        })
    }
}
```

- **Always execute through the root** with `SetArgs` containing the subcommand name — that's the real code path users hit, including flag parsing and `PersistentPreRunE`.
- A fresh `NewRootCmd()` per test case means no shared state, no `viper.Reset()`, and tests can run in parallel.
- If you're stuck with the global Viper singleton, call `viper.Reset()` between tests — its state bleeds across test cases.
- Use `SetOut` / `SetErr` to capture output — which only works if commands print via `cmd.OutOrStdout()`.
- Set config env vars with `t.Setenv` — it restores them automatically and blocks accidental `t.Parallel` misuse.

## Common Mistakes

- **Executing a subcommand variable directly in tests**: `serverCmd.Execute()` runs the whole tree from the root, not just that command. Build fresh trees with a factory and drive them through `SetArgs`.
- **Env-only values missing after `Unmarshal`**: `AutomaticEnv` doesn't register keys. `SetDefault` or `BindEnv` every key in your config struct.
- **Accessing Viper before config is loaded**: values are empty until your setup has run (`PersistentPreRunE` or `cobra.OnInitialize`). Don't read Viper in `init()` functions or `var` blocks.
- **Forgetting `BindPFlags`**: flags are not automatically visible to Viper. Bind the command's flag set explicitly.
- **Missing `SetEnvKeyReplacer`**: nested keys with dots (e.g., `serve.addr`) won't match `MYAPP_SERVE_ADDR` without a replacer.
- **Cobra/Viper imports in business logic**: the `engine` package must never import Cobra or Viper. Pass typed config structs instead.
- **`fmt.Printf` inside commands**: bypasses `SetOut`, breaking test capture. Use `cmd.OutOrStdout()`.
- **Over-nesting subcommands**: two levels (`app command subcommand`) is usually the right depth. Three or more levels confuse users.
