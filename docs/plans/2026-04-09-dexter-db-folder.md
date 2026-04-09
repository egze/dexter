# Move Index DB into `.dexter/` Folder — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Move the dexter index database from `<project>/.dexter.db` to `<project>/.dexter/dexter.db`, with automatic cleanup of legacy files on first open.

**Architecture:** All path knowledge is centralized in `internal/store` via new helpers (`DBPath`, `DBDir`, `LegacyDBPath`, `FindProjectRoot`). `store.Open` deletes any legacy `.dexter.db*` files, creates the `.dexter/` folder, and opens the new location. Callers in `cmd/main.go` and `internal/lsp/server.go` switch to the helpers; the LSP's existing first-run auto-build fills the fresh DB.

**Tech Stack:** Go 1.21+, SQLite (`github.com/mattn/go-sqlite3`), Cobra CLI.

**Reference:** See `docs/plans/2026-04-09-dexter-db-folder-design.md` for the full design rationale and decisions.

---

## Context for the Engineer

### Project layout

- `internal/store/store.go` — SQLite store. Single package, all DB logic.
- `internal/store/store_test.go` — unit tests for the store.
- `cmd/main.go` — Cobra CLI entry point. Defines `init`, `reindex`, `lookup`, `references`, `lsp`, `version` subcommands. Contains `findProjectRoot` and `cmdInit`.
- `internal/lsp/server.go` — LSP server. Contains `findDexterRoot` (walks up looking for an anchor).
- `integration_test.go` — top-level integration tests that build the binary and run it against scaffolded projects.
- `README.md` — user-facing docs. References the DB path in install, editor config, and the "Index database location" section.
- `CHANGELOG.md` — semver-style changelog. Latest entry is `[0.5.3] - 2026-04-09`.
- `.gitignore` — currently has `*.db` and needs `.dexter/` added.

### Key invariants to preserve

- **Monorepo root detection.** When a user opens a subapp in Neovim, the LSP walks up from `rootUri` to find the dexter anchor (either the DB or `.git`). This must keep working after the migration.
- **CLI marker extras.** `cmd/main.go:findProjectRoot` additionally honors `mix.exs` as a fallback marker — the LSP does not, because anchoring on `mix.exs` in a monorepo would pick the wrong root.
- **Corrupt-DB recovery.** `cmd/main.go:cmdInit --force` removes `dbPath`, `dbPath+"-shm"`, `dbPath+"-wal"`. SQLite's WAL siblings will still be `dexter.db-shm` / `dexter.db-wal` inside the new folder, so the cleanup loop keeps working.
- **Tests use `t.TempDir()`.** All tests touching the filesystem use Go's testing temp dir — no cleanup needed.

### Build & test commands

```sh
make build    # build dexter binary
make test     # run all tests (unit + integration)
make lint     # run golangci-lint (required before commit)
```

### Coding conventions

- Performance matters. Don't add allocations or file stats in hot paths unnecessarily.
- When fixing bugs, write the regression test first, verify failure, then fix.
- Keep the CLI working independently of the LSP.
- Use `filepath.Join` for all path construction — cross-platform safety.
- Error messages should mention the full file path to help users.

---

## Task 1: Add path helpers to `internal/store`

**Files:**
- Modify: `internal/store/store.go` (top of file, after existing imports)

**Goal:** Introduce `DBPath`, `DBDir`, `LegacyDBPath` as exported functions. Pure path construction, no side effects. These will be consumed by `store.Open`, `cmd/main.go`, and tests.

**Step 1: Write the failing tests**

Add to `internal/store/store_test.go` (anywhere — convention is to put path-helper tests near the top):

```go
func TestDBPath(t *testing.T) {
    got := DBPath("/project/root")
    want := filepath.Join("/project/root", ".dexter", "dexter.db")
    if got != want {
        t.Errorf("DBPath = %q, want %q", got, want)
    }
}

func TestDBDir(t *testing.T) {
    got := DBDir("/project/root")
    want := filepath.Join("/project/root", ".dexter")
    if got != want {
        t.Errorf("DBDir = %q, want %q", got, want)
    }
}

func TestLegacyDBPath(t *testing.T) {
    got := LegacyDBPath("/project/root")
    want := filepath.Join("/project/root", ".dexter.db")
    if got != want {
        t.Errorf("LegacyDBPath = %q, want %q", got, want)
    }
}
```

**Step 2: Run the tests to verify they fail**

```sh
go test ./internal/store/ -run 'TestDBPath|TestDBDir|TestLegacyDBPath' -v
```

Expected: `undefined: DBPath`, `undefined: DBDir`, `undefined: LegacyDBPath` — compile failure.

**Step 3: Implement the helpers**

Add to `internal/store/store.go` after the `Store` struct definition (around line 17, before `Open`):

```go
// DBPath returns the canonical database path for a project root:
// <root>/.dexter/dexter.db
func DBPath(projectRoot string) string {
    return filepath.Join(projectRoot, ".dexter", "dexter.db")
}

// DBDir returns the directory that holds the database: <root>/.dexter
func DBDir(projectRoot string) string {
    return filepath.Join(projectRoot, ".dexter")
}

// LegacyDBPath returns the pre-migration database path: <root>/.dexter.db
// Used only for detecting and deleting databases created before the
// .dexter/ folder layout.
func LegacyDBPath(projectRoot string) string {
    return filepath.Join(projectRoot, ".dexter.db")
}
```

**Step 4: Run the tests to verify they pass**

```sh
go test ./internal/store/ -run 'TestDBPath|TestDBDir|TestLegacyDBPath' -v
```

Expected: all three PASS.

**Step 5: Commit**

```sh
git add internal/store/store.go internal/store/store_test.go
git commit -m "Add DBPath/DBDir/LegacyDBPath helpers in store"
```

---

## Task 2: Add `FindProjectRoot` to `internal/store`

**Files:**
- Modify: `internal/store/store.go`
- Modify: `internal/store/store_test.go`

**Goal:** Centralize the root-marker walk used by both `cmd/main.go` and `internal/lsp/server.go`. Default markers are the new `.dexter/dexter.db` path, the legacy `.dexter.db`, and `.git`. Callers can pass extra markers (the CLI passes `mix.exs`).

**Step 1: Write the failing tests**

Add to `internal/store/store_test.go`:

```go
func TestFindProjectRoot(t *testing.T) {
    // Helper: create a directory tree inside t.TempDir() and return the root.
    mktree := func(t *testing.T, files []string) string {
        t.Helper()
        root := t.TempDir()
        for _, rel := range files {
            full := filepath.Join(root, rel)
            if strings.HasSuffix(rel, "/") {
                if err := os.MkdirAll(full, 0o755); err != nil {
                    t.Fatal(err)
                }
                continue
            }
            if err := os.MkdirAll(filepath.Dir(full), 0o755); err != nil {
                t.Fatal(err)
            }
            if err := os.WriteFile(full, []byte(""), 0o644); err != nil {
                t.Fatal(err)
            }
        }
        return root
    }

    t.Run("new layout at root", func(t *testing.T) {
        root := mktree(t, []string{".dexter/dexter.db", "apps/app/lib/foo.ex"})
        got := FindProjectRoot(filepath.Join(root, "apps", "app"))
        if got != root {
            t.Errorf("got %q, want %q", got, root)
        }
    })

    t.Run("legacy file at root", func(t *testing.T) {
        root := mktree(t, []string{".dexter.db", "apps/app/lib/foo.ex"})
        got := FindProjectRoot(filepath.Join(root, "apps", "app"))
        if got != root {
            t.Errorf("got %q, want %q", got, root)
        }
    })

    t.Run("git fallback", func(t *testing.T) {
        root := mktree(t, []string{".git/", "apps/app/lib/foo.ex"})
        got := FindProjectRoot(filepath.Join(root, "apps", "app"))
        if got != root {
            t.Errorf("got %q, want %q", got, root)
        }
    })

    t.Run("mix.exs extra marker", func(t *testing.T) {
        root := mktree(t, []string{"apps/app/mix.exs", "apps/app/lib/foo.ex"})
        start := filepath.Join(root, "apps", "app", "lib")
        got := FindProjectRoot(start, "mix.exs")
        want := filepath.Join(root, "apps", "app")
        if got != want {
            t.Errorf("got %q, want %q", got, want)
        }
    })

    t.Run("no marker returns input", func(t *testing.T) {
        root := mktree(t, []string{"lib/foo.ex"})
        start := filepath.Join(root, "lib")
        got := FindProjectRoot(start)
        if got != start {
            t.Errorf("got %q, want %q", got, start)
        }
    })

    t.Run("new layout preferred over legacy", func(t *testing.T) {
        // Both exist — new layout should win because it's first in the marker list.
        root := mktree(t, []string{".dexter/dexter.db", ".dexter.db"})
        got := FindProjectRoot(root)
        if got != root {
            t.Errorf("got %q, want %q", got, root)
        }
    })
}
```

Ensure `"strings"` is imported in `store_test.go` — it likely already is, but verify.

**Step 2: Run the test to verify it fails**

```sh
go test ./internal/store/ -run TestFindProjectRoot -v
```

Expected: `undefined: FindProjectRoot` — compile failure.

**Step 3: Implement `FindProjectRoot`**

Add to `internal/store/store.go` after the `LegacyDBPath` helper:

```go
// FindProjectRoot walks up from path looking for known dexter/project
// markers. The default markers (in priority order) are:
//
//   1. .dexter/dexter.db — the current database layout
//   2. .dexter.db         — the legacy layout (pre-.dexter/ folder)
//   3. .git               — repository root fallback
//
// Additional markers can be passed via extraMarkers; they are tried after
// the defaults, in the order given. The CLI passes "mix.exs" to fall back
// to the nearest Mix project when no dexter/git marker is found.
//
// Returns the original path if no marker is found.
func FindProjectRoot(path string, extraMarkers ...string) string {
    markers := append([]string{
        filepath.Join(".dexter", "dexter.db"),
        ".dexter.db",
        ".git",
    }, extraMarkers...)

    for _, marker := range markers {
        dir := path
        for {
            if _, err := os.Stat(filepath.Join(dir, marker)); err == nil {
                return dir
            }
            parent := filepath.Dir(dir)
            if parent == dir {
                break
            }
            dir = parent
        }
    }
    return path
}
```

`os` and `filepath` are already imported.

**Step 4: Run the test to verify it passes**

```sh
go test ./internal/store/ -run TestFindProjectRoot -v
```

Expected: all subtests PASS.

**Step 5: Commit**

```sh
git add internal/store/store.go internal/store/store_test.go
git commit -m "Add store.FindProjectRoot centralized marker walk"
```

---

## Task 3: Migrate legacy files and create `.dexter/` in `store.Open`

**Files:**
- Modify: `internal/store/store.go` (the `Open` function and a new internal helper)
- Modify: `internal/store/store_test.go`

**Goal:** Make `store.Open` responsible for detecting and deleting legacy files, creating the new folder, and opening the new path. No callers need to know.

**Step 1: Write the failing tests**

Add to `internal/store/store_test.go`:

```go
func TestOpen_CreatesDexterFolder(t *testing.T) {
    dir := t.TempDir()

    s, err := Open(dir)
    if err != nil {
        t.Fatalf("Open: %v", err)
    }
    defer func() { _ = s.Close() }()

    if info, err := os.Stat(filepath.Join(dir, ".dexter")); err != nil || !info.IsDir() {
        t.Errorf(".dexter/ directory was not created: %v", err)
    }
    if _, err := os.Stat(filepath.Join(dir, ".dexter", "dexter.db")); err != nil {
        t.Errorf(".dexter/dexter.db was not created: %v", err)
    }
}

func TestOpen_MigratesLegacyLayout(t *testing.T) {
    dir := t.TempDir()

    // Seed a fake legacy database and its WAL siblings.
    legacy := filepath.Join(dir, ".dexter.db")
    legacyShm := legacy + "-shm"
    legacyWal := legacy + "-wal"
    for _, f := range []string{legacy, legacyShm, legacyWal} {
        if err := os.WriteFile(f, []byte("legacy placeholder"), 0o644); err != nil {
            t.Fatal(err)
        }
    }

    s, err := Open(dir)
    if err != nil {
        t.Fatalf("Open: %v", err)
    }
    defer func() { _ = s.Close() }()

    for _, f := range []string{legacy, legacyShm, legacyWal} {
        if _, err := os.Stat(f); !os.IsNotExist(err) {
            t.Errorf("legacy file %s still exists (err=%v)", f, err)
        }
    }
    if _, err := os.Stat(filepath.Join(dir, ".dexter", "dexter.db")); err != nil {
        t.Errorf("new DB was not created: %v", err)
    }

    // Smoke test: the store should be functional after migration.
    if _, err := s.db.Exec("INSERT INTO files (path, mtime) VALUES (?, ?)", "/fake.ex", 1); err != nil {
        t.Errorf("store not functional after migration: %v", err)
    }
}

func TestOpen_MigrationWithPartialLegacyFiles(t *testing.T) {
    // Legacy DB present but no WAL siblings — should still migrate cleanly.
    dir := t.TempDir()
    legacy := filepath.Join(dir, ".dexter.db")
    if err := os.WriteFile(legacy, []byte("legacy"), 0o644); err != nil {
        t.Fatal(err)
    }

    s, err := Open(dir)
    if err != nil {
        t.Fatalf("Open: %v", err)
    }
    defer func() { _ = s.Close() }()

    if _, err := os.Stat(legacy); !os.IsNotExist(err) {
        t.Errorf("legacy .dexter.db still exists: %v", err)
    }
}
```

Also update `TestOpenCorruptedDB` (currently at `internal/store/store_test.go:956`):

```go
func TestOpenCorruptedDB(t *testing.T) {
    dir := t.TempDir()

    // Pre-create the .dexter/ folder and plant a garbage DB file in the
    // new location so Open's migration path doesn't touch it.
    if err := os.MkdirAll(DBDir(dir), 0o755); err != nil {
        t.Fatal(err)
    }
    dbPath := DBPath(dir)

    if err := os.WriteFile(dbPath, []byte("this is not a sqlite database"), 0o644); err != nil {
        t.Fatal(err)
    }

    _, err := Open(dir)
    if err == nil {
        t.Fatal("expected Open to fail on a corrupted DB file, got nil")
    }
}
```

**Step 2: Run the tests to verify they fail**

```sh
go test ./internal/store/ -run 'TestOpen_CreatesDexterFolder|TestOpen_MigratesLegacyLayout|TestOpen_MigrationWithPartialLegacyFiles' -v
```

Expected: all three FAIL because `.dexter/` isn't created and legacy files aren't deleted.

**Step 3: Implement `migrateLegacyLayout` and update `Open`**

In `internal/store/store.go`, replace the existing `Open` function (lines 19–33) with:

```go
func Open(projectRoot string) (*Store, error) {
    if err := migrateLegacyLayout(projectRoot); err != nil {
        return nil, fmt.Errorf("migrate legacy layout: %w", err)
    }
    if err := os.MkdirAll(DBDir(projectRoot), 0o755); err != nil {
        return nil, fmt.Errorf("create dexter dir: %w", err)
    }

    dbPath := DBPath(projectRoot)
    db, err := sql.Open("sqlite3", dbPath+"?_journal_mode=WAL&_synchronous=NORMAL&_busy_timeout=5000&_foreign_keys=ON")
    if err != nil {
        return nil, err
    }
    db.SetMaxOpenConns(2)

    if err := migrate(db); err != nil {
        _ = db.Close()
        return nil, err
    }

    return &Store{db: db}, nil
}

// migrateLegacyLayout deletes any pre-.dexter/ folder artifacts so that a
// fresh database will be built at the new location on the next Open. The
// index is a derived cache, so deletion (rather than move) is safe and
// avoids WAL/SHM consistency edge cases.
//
// Returns nil when there is nothing to migrate.
func migrateLegacyLayout(projectRoot string) error {
    legacy := LegacyDBPath(projectRoot)
    if _, err := os.Stat(legacy); err != nil {
        if os.IsNotExist(err) {
            return nil
        }
        return err
    }
    for _, f := range []string{legacy, legacy + "-shm", legacy + "-wal"} {
        if err := os.Remove(f); err != nil && !os.IsNotExist(err) {
            return fmt.Errorf("remove %s: %w", f, err)
        }
    }
    return nil
}
```

Verify `fmt` is already imported (it is — used elsewhere in this file).

**Step 4: Run the tests to verify they pass**

```sh
go test ./internal/store/ -run 'TestOpen_|TestOpenCorruptedDB' -v
```

Expected: all four tests PASS.

Also run the full store test suite to catch regressions:

```sh
go test ./internal/store/ -v
```

Expected: all existing tests PASS.

**Step 5: Commit**

```sh
git add internal/store/store.go internal/store/store_test.go
git commit -m "Auto-create .dexter/ and migrate legacy DB in store.Open"
```

---

## Task 4: Update `cmd/main.go` to use store helpers

**Files:**
- Modify: `cmd/main.go`

**Goal:** Replace the three `".dexter.db"` hardcoded references with calls to the store helpers, and delegate the walk in `findProjectRoot` to `store.FindProjectRoot`.

**Step 1: Update `findProjectRoot` (around line 141)**

Replace the existing function:

```go
func findProjectRoot(path string) string {
    info, err := os.Stat(path)
    if err != nil {
        fatal(err)
    }
    if !info.IsDir() {
        path = filepath.Dir(path)
    }
    return store.FindProjectRoot(path, "mix.exs")
}
```

The file/directory preamble stays because CLI users can pass file paths (e.g. `dexter reindex some/file.ex`).

**Step 2: Update `cmdInit` (around line 166)**

Replace the `dbPath := filepath.Join(projectRoot, ".dexter.db")` line and the surrounding user-facing message with:

```go
func cmdInit(projectRoot string, force bool, profile bool) {
    dbPath := store.DBPath(projectRoot)
    if _, err := os.Stat(dbPath); err == nil {
        if !force {
            fmt.Fprintf(os.Stderr, "Index already exists at %s\n", dbPath)
            fmt.Fprintf(os.Stderr, "Run `dexter reindex` to update, or `dexter init --force` to delete and rebuild from scratch.\n")
            os.Exit(1)
        }
        for _, f := range []string{dbPath, dbPath + "-shm", dbPath + "-wal"} {
            _ = os.Remove(f) // files may not exist
        }
    }
    // ... rest of function unchanged
```

Leave the rest of `cmdInit` alone.

**Step 3: Build and run all tests**

```sh
make build
make test
```

Expected: build succeeds. All tests pass (including existing integration tests — Task 5 will update those that rely on the old path).

Note: integration tests that hardcode `.dexter.db` paths may fail here. **That's fine** — the next task updates them. If any fail now, note which ones and move on to Task 5.

**Step 4: Commit**

```sh
git add cmd/main.go
git commit -m "Use store helpers for DB path in cmd/main.go"
```

---

## Task 5: Update `internal/lsp/server.go` to use `store.FindProjectRoot`

**Files:**
- Modify: `internal/lsp/server.go`

**Goal:** Delete the local `findDexterRoot` and call `store.FindProjectRoot` directly from the two call sites in `Initialize`.

**Step 1: Remove `findDexterRoot`**

Delete lines 735–752 of `internal/lsp/server.go` — the entire `findDexterRoot` function and its comment.

**Step 2: Update call sites in `Initialize` (lines 293 and 298)**

Replace both occurrences of `findDexterRoot(root)` with `store.FindProjectRoot(root)`.

The `store` package is already imported at line 27.

**Step 3: Build and run all tests**

```sh
make build
make test
```

Expected: build succeeds. LSP tests pass. Integration tests that reference the old `.dexter.db` path may still fail — Task 6 handles those.

**Step 4: Commit**

```sh
git add internal/lsp/server.go
git commit -m "Delete findDexterRoot; use store.FindProjectRoot in LSP"
```

---

## Task 6: Update integration tests

**Files:**
- Modify: `integration_test.go`

**Goal:** Replace hardcoded `.dexter.db` references (lines 462 and 508) with the new path, and add a new integration test that verifies the legacy-to-new migration through the CLI binary.

**Step 1: Update existing assertions**

In `integration_test.go`, change line 462:

```go
// before
if _, err := os.Stat(filepath.Join(subapp, ".dexter.db")); err == nil {
    t.Error("should not have created a second .dexter.db in the subapp")
}

// after
if _, err := os.Stat(filepath.Join(subapp, ".dexter", "dexter.db")); err == nil {
    t.Error("should not have created a second .dexter/dexter.db in the subapp")
}
```

Change line 508:

```go
// before
dbPath := filepath.Join(root, ".dexter.db")

// after
dbPath := store.DBPath(root)
```

Ensure `"github.com/remoteoss/dexter/internal/store"` is imported at the top of `integration_test.go`. If it isn't, add it.

**Step 2: Add a migration integration test**

Add at the end of `integration_test.go`:

```go
// TestIntegration_LegacyMigration simulates an upgrade path: a project with
// the pre-.dexter/ folder layout (legacy .dexter.db file and its WAL
// siblings at the root) is migrated automatically on the next dexter
// invocation. The legacy files should be deleted and a fresh index built
// at .dexter/dexter.db.
func TestIntegration_LegacyMigration(t *testing.T) {
    binary := buildDexter(t)
    root := scaffoldProject(t)

    // Seed the legacy layout.
    legacy := filepath.Join(root, ".dexter.db")
    for _, f := range []string{legacy, legacy + "-shm", legacy + "-wal"} {
        if err := os.WriteFile(f, []byte("legacy placeholder"), 0644); err != nil {
            t.Fatal(err)
        }
    }

    // Run init — migration runs inside store.Open.
    runDexter(t, binary, root, "init", "--force", root)

    // Legacy files should be gone.
    for _, f := range []string{legacy, legacy + "-shm", legacy + "-wal"} {
        if _, err := os.Stat(f); !os.IsNotExist(err) {
            t.Errorf("legacy file %s still exists after migration", f)
        }
    }

    // New DB should exist and be functional.
    newDB := filepath.Join(root, ".dexter", "dexter.db")
    if _, err := os.Stat(newDB); err != nil {
        t.Errorf("new DB not created: %v", err)
    }

    // Lookups should work against the migrated index.
    out := runDexter(t, binary, root, "lookup", "MyApp.Repo")
    if !strings.Contains(out, "repo.ex:1") {
        t.Errorf("expected lookup to work after migration, got: %s", out)
    }
}
```

**Step 3: Run the integration tests**

```sh
go test -run 'TestIntegration_' -v
```

Expected: all integration tests pass, including the new `TestIntegration_LegacyMigration`.

If an integration test fails because `runDexter` signature differs from what I assumed — inspect it and adjust. `scaffoldProject` returns the project root; `buildDexter` returns the binary path; `runDexter(t, binary, dir, args...)` runs the binary and returns stdout.

**Step 4: Commit**

```sh
git add integration_test.go
git commit -m "Update integration tests for .dexter/ folder layout"
```

---

## Task 7: Run the full test suite and lint

**Files:** none (verification step)

**Goal:** Catch anything the previous per-task test runs missed and make sure linting is clean before touching docs.

**Step 1: Run the full test suite**

```sh
make test
```

Expected: all tests pass. Any failure is a regression introduced by Tasks 1–6; stop and fix it before continuing.

**Step 2: Run the linter**

```sh
make lint
```

Expected: zero issues. Fix anything golangci-lint reports.

**Step 3: No commit** — this task only verifies.

---

## Task 8: Update `.gitignore`

**Files:**
- Modify: `.gitignore`

**Step 1: Add `.dexter/` line**

Current `.gitignore`:

```
# Binary
dexter

# Database
*.db

# OS
.DS_Store

.claude
dist/
```

Update the `# Database` section:

```
# Database
*.db
.dexter/
```

Keep `*.db` — it's a safety net for any stray `.db` files users might drop in this repo while developing.

**Step 2: Commit**

```sh
git add .gitignore
git commit -m "Ignore .dexter/ folder"
```

---

## Task 9: Update `README.md`

**Files:**
- Modify: `README.md`

**Goal:** Update every reference to `.dexter.db` to reflect the new layout, plus rewrite the "Index database location" section.

**Step 1: TOC entry (line 31)**

```
# before
- [Index database location (.dexter.db)](#index-database-location-dexterdb)

# after
- [Index database location (.dexter/)](#index-database-location-dexter)
```

**Step 2: Quick-start gitignore snippet (lines 104–105)**

```
# before
# 3. Add .dexter.db to your .gitignore
echo ".dexter.db*" >> .gitignore

# after
# 3. Add .dexter/ to your .gitignore
echo ".dexter/" >> .gitignore
```

**Step 3: Neovim root_markers (line 158)**

```
# before
root_markers = { '.dexter.db', '.git', 'mix.exs' },

# after
root_markers = { '.dexter/dexter.db', '.dexter.db', '.git', 'mix.exs' },
```

Keep `.dexter.db` so users mid-upgrade still get anchored correctly.

**Step 4: Neovim root_pattern for older lspconfig (line 190)**

```
# before
root_dir = lspconfig.util.root_pattern(".dexter.db", "mix.exs", ".git"),

# after
root_dir = lspconfig.util.root_pattern(".dexter/dexter.db", ".dexter.db", "mix.exs", ".git"),
```

**Step 5: "Index database location" section (lines 382–400)**

Replace the heading and prose. Keep the monorepo-vs-single-app structure but update all references:

```markdown
## Index database location (.dexter/)

Dexter creates `.dexter/dexter.db` at the root of your project when you start the LSP for the first time. But if you prefer, you can run `dexter init` yourself in the root of your project. Where you place it determines what gets indexed.

When the LSP server starts, it walks up from the project root looking for `.dexter/dexter.db`, preferring `.git` as the anchor point. This means if you initialised from the monorepo root, the server will find the right database even when Neovim's `rootUri` points to a sub-app (e.g. because `mix.exs` is there).

If you're upgrading from a pre-`.dexter/` version of dexter, any existing `.dexter.db` file at your project root will be automatically deleted and rebuilt into the new `.dexter/` folder on the next `dexter init` or LSP startup. Update your `.gitignore` to use `.dexter/` in place of `.dexter.db*`.

**Monorepo root (recommended if using an Elixir monorepo or umbrella structure)** — Put the index at the root of your repository, next to `.git`. This indexes everything: all apps, all shared libraries, and all deps. Go-to-definition works across the entire codebase.

```sh
cd ~/code/my-monorepo   # where .git lives
dexter init .
```

**Single app** — Put the index inside a specific Mix project. Go-to-definition works within that app and its deps, but not across other apps in the monorepo.

```sh
cd ~/code/my-monorepo/apps/my_app
dexter init .
```
```

**Step 6: Grep for any remaining `.dexter.db` references**

```sh
grep -n "\.dexter\.db" README.md
```

Expected: only the lines you intentionally kept as legacy fallbacks in the Neovim config snippets. Nothing else.

**Step 7: Commit**

```sh
git add README.md
git commit -m "Update README for .dexter/ folder layout"
```

---

## Task 10: Update `CHANGELOG.md`

**Files:**
- Modify: `CHANGELOG.md`

**Goal:** Add an `[Unreleased]` section at the top documenting the path change.

**Step 1: Add `[Unreleased]` entry**

Insert after line 1 (`# Changelog`) and before `## [0.5.3]`:

```markdown
## [Unreleased]

### Changed

- **Index database location** — the index database has moved from `<project>/.dexter.db` to `<project>/.dexter/dexter.db`. Existing databases are automatically migrated on the next `dexter init` or LSP startup: the legacy `.dexter.db` (and its `-shm`/`-wal` siblings) is deleted and a fresh index is built in the new folder. Update your `.gitignore` to use `.dexter/` instead of `.dexter.db*`. Editor-extension repos (`dexter-vscode`, `dexter-zed`) continue to work via their existing `mix.exs` / `.git` fallbacks and can be updated separately.
```

**Step 2: Commit**

```sh
git add CHANGELOG.md
git commit -m "Changelog: note .dexter/ folder migration"
```

---

## Task 11: Check `docs/architecture.md` for stray references

**Files:**
- Possibly modify: `docs/architecture.md`

**Step 1: Grep**

```sh
grep -n "\.dexter\.db\|dexter\.db" docs/architecture.md
```

**Step 2: If any references found**

Update them in line with the section-5 design decisions (`.dexter/dexter.db` primary, mention migration if relevant). If none are found, skip to step 3.

**Step 3: Commit (only if changes were made)**

```sh
git add docs/architecture.md
git commit -m "Update architecture doc for .dexter/ folder layout"
```

---

## Task 12: Final verification

**Step 1: Full clean build and test**

```sh
make build
make test
make lint
```

Expected: everything green.

**Step 2: Manual smoke test**

```sh
# Fresh project
TMP=$(mktemp -d)
cd "$TMP"
git init -q
echo 'defmodule MyApp do
  def hello, do: :world
end' > lib/my_app.ex 2>/dev/null || (mkdir -p lib && echo 'defmodule MyApp do
  def hello, do: :world
end' > lib/my_app.ex)

# Init should create .dexter/dexter.db, not .dexter.db
~/Projects/dexter/dexter init .
test -f .dexter/dexter.db && echo "OK: new layout created"
test -f .dexter.db && echo "FAIL: legacy file exists"

# Legacy migration smoke test
cd "$TMP"
rm -rf .dexter
touch .dexter.db .dexter.db-shm .dexter.db-wal
~/Projects/dexter/dexter init --force .
test -f .dexter/dexter.db && echo "OK: migrated to new layout"
test -f .dexter.db && echo "FAIL: legacy file still present"
test -f .dexter.db-shm && echo "FAIL: legacy -shm still present"
test -f .dexter.db-wal && echo "FAIL: legacy -wal still present"

cd ~/Projects/dexter
rm -rf "$TMP"
```

All `OK` lines should print; no `FAIL` lines.

**Step 3: Review the commit series**

```sh
git log --oneline main..HEAD
```

Expected: a clean series of small, focused commits (one per task). No stray work, no reverts.

**Step 4: No commit** — this task only verifies.

---

## Done

The branch `move-db-to-dexter-folder` is ready for review / merge. The migration is complete, transparent to users, and fully tested.
