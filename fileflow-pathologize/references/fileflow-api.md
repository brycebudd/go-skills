# `fileflow` API Reference

Safely move/copy/rename files, including across filesystems, without overwriting non-identical data. The package-level functions use the zero-value defaults; use a `Flow` when behavior must be configured per instance.

## Functions

```go
func Move(src, dst string) (string, error)          // rename if same FS, else copy+delete
func Rename(src, dst string) (string, error)         // same-FS only; fails on cross-device
func Copy(src, dst string) (string, error)           // atomic copy; returns final path
func Exists(path string) bool
func Equal(file1, file2 string) (bool, error)        // byte-for-byte comparison
```

`Move`, `Rename`, and `Copy` all return the **final destination path**, which matters because of conflict handling below. Prefer `Move` over `Rename` unless you specifically *want* to fail across filesystems. `Exists` reports only an accessible regular file; it returns false for directories.

All three operations create missing destination directories by default. `CopyWithPaths` no longer exists because `Copy` itself now creates them.

## Configure behavior with `Flow`

```go
type Flow struct {
    FindAvailableName func(string) (string, error)
    DirMode           fs.FileMode
    NoCreateDirs      bool
}
```

The zero value is ready to use. A nil `FindAvailableName` selects `FindAvailableNameInc`; a zero `DirMode` selects `fileflow.DefaultDirMode` (`0o755`). Set `NoCreateDirs: true` when destination directories must already exist—for example, when a destination is a required mount point and creating the path on the wrong volume would be dangerous. The operation then returns an error matching `fs.ErrNotExist` via `errors.Is` instead of creating the directory.

```go
f := fileflow.Flow{
    FindAvailableName: fileflow.FindAvailableNameAuto,
    DirMode:           0o700,
    NoCreateDirs:      true,
}

final, err := f.Move(src, dst)
```

`Flow` also has `Move`, `Rename`, and `Copy` methods with the package-level signatures. Configure an instance before first use; do not mutate it while its methods may be running. Older mutable package globals are gone.

## Conflict handling (the key feature)

When `dst` already exists, fileflow does **not** blindly overwrite:

- If the existing file is **identical** to the source → `Copy` leaves `src` untouched and returns `dst` without writing; `Move`/`Rename` remove `src` and return `dst`.
- If it **differs** → an incrementing suffix is added: `photo.jpg` → `photo-1.jpg` → `photo-2.jpg`, up to 100 candidates.

Always use the returned path; never assume the file landed at `dst`.

## Naming strategies

```go
// Default: preserve the full requested name, then append -1, -2, ...
f := fileflow.Flow{FindAvailableName: fileflow.FindAvailableNameInc}

// Continue a trailing one-or-more-digit counter: file-3.txt -> file-4.txt.
f.FindAvailableName = fileflow.FindAvailableNameCont

// Continue one- or two-digit counters, but preserve years/zero-padded IDs.
f.FindAvailableName = fileflow.FindAvailableNameAuto

// Append a nanosecond timestamp; fall back to incrementing on collision.
f.FindAvailableName = fileflow.FindAvailableNameTS

// Or supply any func(baseName string) (string, error).
f.FindAvailableName = myStrategy
```

Choose deliberately:

- `FindAvailableNameInc` (default): `file-1.txt` → `file-1-1.txt` and `report-2024.txt` → `report-2024-1.txt`.
- `FindAvailableNameCont`: `file-1.txt` → the next available sequence number, but also interprets `report-2024.txt` as a counter.
- `FindAvailableNameAuto`: continues one- or two-digit suffixes, while treating three or more digits as meaningful (`report-2024-1.txt`, not `report-2025.txt`).
- `FindAvailableNameTS`: appends `20060102-150405.000000000`.

The 100-candidate limit is internal and not configurable.

## Write safety and durability

- `Rename` uses no-replace primitives, so a racing writer cannot normally clobber an occupied destination. Windows uses `MoveFileEx` without replace; Unix uses atomic link-and-remove.
- On Unix filesystems without hard-link support (such as FAT and some network mounts), rename falls back to `os.Rename`; the no-overwrite guarantee is then best-effort.
- `Copy` writes to a temporary file in the destination directory, syncs it, preserves the source permission mode, and renames it into place with the same no-replace logic. A failed copy does not expose a partial destination or leave its temporary file behind.

## Errors (`errors.Is` for sentinels, `errors.As` for structured errors)

```go
fileflow.ErrSameFile             // src and dst are the same file
fileflow.ErrMaxAttemptsReached   // ran out of conflict-suffix attempts
*fileflow.ErrFailedMovingFile    // fields: Src, Dst, Err
*fileflow.ErrFailedRemovingOriginal // fields: File, Err
```

`ErrSameFile` detects equivalent paths, symlinks, and case-only differences on case-insensitive filesystems—not just identical strings. Treat it as a no-op success only when that matches the caller's semantics. A removal error may return the final destination together with `ErrFailedRemovingOriginal`, because the content arrived safely but the source could not be deleted.
