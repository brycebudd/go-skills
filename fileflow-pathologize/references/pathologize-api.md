# `pathologize` API Reference & Common Mistakes

Make a filename or path safe on **every** modern OS, not just the host. It is intentionally restrictive: strips illegal characters, trims trailing dots/spaces, enforces a 255-byte length, and neutralizes reserved names (`CON` → `CON_`).

## Functions

```go
func Clean(filename string) string             // sanitize ONE name/segment
func CleanPath(path string) string             // make a full path VALID (volume-aware)
func Join(root string, parts ...string) string // trusted root + untrusted parts, contained
func IsClean(filename string) bool             // true if Clean(name) == name
```

```go
pathologize.Clean("CON..")            // "CON_"
pathologize.Clean("a/b:c*d?.txt")     // "abcd.txt"  (/, :, *, ? are illegal)
```

Characters stripped by `Clean`: control chars plus `` \ / : * ? " < > | ``. Invalid UTF-8 becomes U+FFFD; leading/trailing whitespace and trailing dots are trimmed; names are capped at 255 bytes; reserved device names are defused (`CON` → `CON_`). `Clean` is idempotent and never returns an empty string (it falls back to `"file"`).

## `Join` — the safe way to build a destination (use this by default)

```go
func Join(root string, parts ...string) string
```

`Join` combines a **trusted** root with one or more **untrusted** parts:

- `root` is passed through **verbatim** — its separators, drive prefix, and trailing slash are preserved, and it is never sanitized. It must come from a trusted source (a configured destination, not user input).
- Each `part` is fully sanitized: split on both `/` and `\`, structural segments (empty, `.`, `..`) dropped, every remaining segment run through `Clean`. A part that sanitizes to nothing contributes `"file"`.
- The result is guaranteed to stay lexically **under** `root` — a part cannot escape it or inject an absolute path. This is the containment `CleanPath` does *not* provide.

```go
pathologize.Join("/data", "../../etc", "pass:wd")   // "/data/etc/passwd" — contained + cleaned
pathologize.Join(`C:\Sorted`, "ig", userName)       // root verbatim, userName sanitized
```

## `CleanPath` — make a full path valid (NOT safe)

`CleanPath` runs `path.Clean` semantics (collapse `//`, resolve `.` and internal `..`, drop a trailing separator) and additionally `Clean`s each component. In v1.1 it is **volume-aware and separator-agnostic**:

- Both `/` and `\` are accepted as separators; output always uses `/`.
- A Windows **drive** prefix (`C:`) passes through verbatim.
- A **UNC** prefix (`\\server\share`) is preserved as `//server/share`.

```go
pathologize.CleanPath(`C:\Users\me\report:final`) // "C:/Users/me/reportfinal"
pathologize.CleanPath("a//b/../c")                 // "a/c"
```

**CleanPath makes a path VALID, not SAFE.** Like `path.Clean`, it preserves a leading `..` in a relative path and keeps an absolute path absolute — it does **not** neutralize traversal or contain the result under any root. Use it only on paths you control (e.g. an author-configured absolute destination). For untrusted input, use `Join`, or `Clean` each segment yourself.

## Security: path-traversal defense

Untrusted input must never decide where a file lands. Two safe options:

1. **`Join`** (preferred) — pass the trusted root and untrusted parts; parts are sanitized and contained under the root:

   ```go
   dst := pathologize.Join(root, untrusted)   // can't escape root
   ```

2. **`Clean` per segment** — when you assemble the path yourself, `Clean` each untrusted segment before joining. `Clean` strips separators and resolves traversal tokens to a harmless single segment (`".."` → `"file"`), so a captured value can't introduce a new path level.

Do **not** rely on `CleanPath` for this — it preserves `..` and absolute paths by design.

## Common Mistakes

| Mistake | Fix |
|---|---|
| `os.Rename` across volumes → `EXDEV` | `fileflow.Move` (copy+delete fallback) |
| Assuming the file landed at `dst` | Use the path returned by `Move`/`Rename`/`Copy` |
| Overwriting existing files | fileflow already suffixes non-identical files — don't pre-check |
| Setting old package globals | Configure a `fileflow.Flow`; the mutable globals were removed |
| Calling removed `CopyWithPaths` | Use `Copy`; it creates parent directories by default |
| Requiring a mount point but allowing directory creation | Use `Flow{NoCreateDirs: true}` |
| Building a dest from untrusted input by hand | `pathologize.Join(root, parts...)` — sanitized + contained |
| Using `CleanPath` on untrusted input | `CleanPath` makes paths VALID, not SAFE — use `Join` |
| Sanitizing the trusted root too | `Join` takes root verbatim; only parts are untrusted |
| Manual `os.MkdirAll` before an operation | `Move`/`Rename`/`Copy` create parents by default |

## Installation

```sh
go get github.com/spf13/fileflow
go get github.com/spf13/pathologize
```
