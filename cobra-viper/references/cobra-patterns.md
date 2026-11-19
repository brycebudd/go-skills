# Cobra Best Practices & Version Handling

## Cobra Best Practices

### 1. Use `RunE` for Native Error Handling

Avoid `Run`. If a command fails, use `RunE` to return the error up the execution chain. This allows the root command to handle errors gracefully and consistently, rather than relying on scattered `log.Fatal` calls that bypass `defer` statements.

```go
// Idiomatic: Returning errors to be handled by the executor
func NewServeCmd(v *viper.Viper) *cobra.Command {
    cmd := &cobra.Command{
        Use:   "serve",
        Short: "Starts the primary application server",
        RunE: func(cmd *cobra.Command, args []string) error {
            var cfg engine.Config
            if err := v.Unmarshal(&cfg); err != nil {
                return fmt.Errorf("decoding config: %w", err)
            }
            return engine.Serve(cmd.Context(), cfg)
        },
    }
    cmd.Flags().String("addr", ":8080", "listen address")
    return cmd
}
```

### 2. Silence Usage on Application Errors

By default, Cobra prints the full help text whenever an error is returned. This is confusing if the error was a runtime failure (like a network timeout) rather than a syntax error. Set `SilenceUsage: true` and `SilenceErrors: true` on the root and let `main.go` print the error once.

### 3. Context-Aware Commands

Modern Go relies heavily on `context.Context` for cancellation and timeouts. Pass the Cobra command's context directly to your business logic:

```go
RunE: func(cmd *cobra.Command, args []string) error {
    return engine.Process(cmd.Context(), args)
}
```

To make that context respect `Ctrl+C`, wire signal handling in `main.go`:

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
err := cmd.NewRootCmd().ExecuteContext(ctx)
```

### 4. Validate Positional Arguments with `Args`

Never validate argument counts inside `RunE` — Cobra does it for you, before `RunE` runs, with correct usage errors:

```go
cmd := &cobra.Command{
    Use:  "build [target]",
    Args: cobra.ExactArgs(1),
    // Also: cobra.NoArgs, cobra.MinimumNArgs(n), cobra.MaximumNArgs(n),
    // cobra.RangeArgs(min, max), cobra.OnlyValidArgs (with ValidArgs set)
    // Combine: cobra.MatchAll(cobra.ExactArgs(1), cobra.OnlyValidArgs)
}
```

### 5. `PersistentPreRunE` for Shared Setup

Use `PersistentPreRunE` on the root command to run setup (config loading, logging) after flags are parsed but before any subcommand runs. Cobra runs it for every subcommand automatically.

**Gotcha:** if a subcommand also defines `PersistentPreRunE`, it *replaces* the parent's — Cobra does not chain them by default. Either call the parent's explicitly, or opt into chaining globally with `cobra.EnableTraverseRunHooks = true` (Cobra 1.8+).

### 6. Flag Design

```go
// Persistent flags — inherited by all subcommands
rootCmd.PersistentFlags().String("config", "", "config file path")
rootCmd.PersistentFlags().BoolP("verbose", "v", false, "enable verbose output")

// Local flags — only for this command
serveCmd.Flags().String("addr", ":8080", "listen address")

// Required flags — Cobra validates before RunE is called
serveCmd.Flags().String("name", "", "required name")
serveCmd.MarkFlagRequired("name")

// Flag relationships (Cobra 1.8+ for OneRequired)
serveCmd.MarkFlagsMutuallyExclusive("json", "yaml")
serveCmd.MarkFlagsRequiredTogether("user", "password")
serveCmd.MarkFlagsOneRequired("file", "url")
```

- Use `PersistentFlags` for cross-cutting concerns (config, verbosity, output format).
- Use `Flags` for command-specific options.
- Always provide short flags (`-v`, `-o`) for common options.

### 7. Group Commands in Help Output (Cobra 1.6+)

For CLIs with many subcommands, group them so `--help` reads as a story, not a dump:

```go
rootCmd.AddGroup(&cobra.Group{ID: "core", Title: "Core Commands:"})
rootCmd.AddGroup(&cobra.Group{ID: "admin", Title: "Admin Commands:"})
serveCmd.GroupID = "core"
migrateCmd.GroupID = "admin"
```

### 8. Shell Completion

Cobra generates shell completion for free. Add dynamic completion for both flags *and* positional arguments:

```go
// Flag value completion
serveCmd.RegisterFlagCompletionFunc("output", func(cmd *cobra.Command, args []string, toComplete string) ([]string, cobra.ShellCompDirective) {
    return []string{"json", "yaml", "table"}, cobra.ShellCompDirectiveNoFileComp
})

// Positional argument completion
buildCmd.ValidArgsFunction = func(cmd *cobra.Command, args []string, toComplete string) ([]string, cobra.ShellCompDirective) {
    return listTargets(), cobra.ShellCompDirectiveNoFileComp
}
```

```bash
myapp completion bash        > /etc/bash_completion.d/myapp
myapp completion zsh         > "${fpath[1]}/_myapp"
myapp completion fish        > ~/.config/fish/completions/myapp.fish
myapp completion powershell  | Out-File -Encoding utf8 "$PROFILE\myapp.ps1"
```

### 9. Print Through the Command, Not `fmt.Printf`

Inside any command, write to `cmd.OutOrStdout()` / `cmd.ErrOrStderr()` (or use `cmd.Println`). Direct `fmt.Printf` bypasses `SetOut`/`SetErr`, making output impossible to capture in tests.

## Version Handling

Cobra has a built-in version mechanism — prefer it over a hand-rolled subcommand:

```go
// Set at build time via -ldflags
var (
    version = "dev"
    commit  = "none"
    date    = "unknown"
)

rootCmd := &cobra.Command{
    Use:     "myapp",
    Version: fmt.Sprintf("%s (commit: %s, built: %s)", version, commit, date),
}
// `myapp --version` now works for free.
// Customize the output with rootCmd.SetVersionTemplate if needed.
```

```bash
go build -ldflags="-X 'github.com/spf13/myapp/cmd.version=1.2.3' \
                   -X 'github.com/spf13/myapp/cmd.commit=$(git rev-parse --short HEAD)' \
                   -X 'github.com/spf13/myapp/cmd.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
```

Add a `version` *subcommand* only if you need structured output (`version --json`) — and print via `cmd.OutOrStdout()`, never `fmt.Printf`.
