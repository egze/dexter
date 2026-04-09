# Move index database into a `.dexter/` folder

**Date:** 2026-04-09
**Branch:** `move-db-to-dexter-folder`
**Status:** Design approved

## Summary

Move the dexter index database from `<project>/.dexter.db` to `<project>/.dexter/dexter.db`. Auto-migrate existing projects by deleting the legacy file on startup and letting the LSP's normal first-run auto-build populate the new location.

## Motivation

A single dotfile at the repo root is fine for a DB, but as dexter grows it may want to stash sibling artifacts (logs, profiling output, scratch files). Moving to a dedicated `.dexter/` folder gives us a namespaced scratch space and stops cluttering the repo root with `.dexter.db`, `.dexter.db-shm`, and `.dexter.db-wal`.

## Design decisions

### Migration strategy: auto-rebuild

When the LSP or CLI encounters a legacy `<root>/.dexter.db`, delete it (plus `-shm` / `-wal` siblings) and rebuild from scratch into the new location. Rationale:

- The index is fully derivable from source; rebuilding is fast.
- Moving the DB file mid-flight introduces edge cases (WAL checkpoint state, partial moves on crash).
- Matches the existing `IndexVersion` mismatch flow, which already triggers forced rebuilds.

### Root-marker detection

`FindProjectRoot` walks up looking for (in priority order):

1. `.dexter/dexter.db` — new layout
2. `.dexter.db` — legacy (needed during the brief window between upgrade and first `store.Open`)
3. `.git`
4. extra markers passed by the caller (e.g. `mix.exs` from the CLI)

Once `store.Open` runs and deletes the legacy file, subsequent walks match the new marker first.

### Legacy file cleanup

Delete. Not rename, not leave-in-place. The DB is a gitignored derived cache; preserving it just creates different clutter.

### `.gitignore`

Add `.dexter/` as a new line in the repo's own `.gitignore`. Keep `*.db` — it's orthogonal. Update the README to recommend `.dexter/` for user projects.

### Path knowledge centralization

All path knowledge lives in `internal/store`:

- `store.DBPath(projectRoot) string` — canonical DB path.
- `store.DBDir(projectRoot) string` — the `.dexter/` folder.
- `store.LegacyDBPath(projectRoot) string` — pre-migration path.
- `store.FindProjectRoot(path, extraMarkers ...string) string` — centralized walk, replaces duplicated loops in `cmd/main.go` and `internal/lsp/server.go`.

`store.Open(projectRoot)` becomes:

1. `migrateLegacyLayout(projectRoot)` — delete legacy files if present.
2. `os.MkdirAll(DBDir(projectRoot), 0o755)` — create the folder.
3. Open the DB at `DBPath(projectRoot)`.

## Call-site updates

### `internal/store/store.go`
- Add `DBPath`, `DBDir`, `LegacyDBPath`, `FindProjectRoot` exported helpers.
- Add unexported `migrateLegacyLayout`.
- `Open` calls migrate → mkdir → open.

### `cmd/main.go`
- `findProjectRoot` keeps its file-vs-dir preamble, then delegates to `store.FindProjectRoot(path, "mix.exs")`.
- `cmdInit` uses `store.DBPath(projectRoot)`; the existing `[dbPath, dbPath+"-shm", dbPath+"-wal"]` cleanup loop keeps working unchanged.
- Any stderr messages mentioning `.dexter.db` updated to `.dexter/dexter.db`.

### `internal/lsp/server.go`
- `findDexterRoot` deleted. Call sites use `store.FindProjectRoot(path)` directly (no `mix.exs` — LSP anchors on `.git` for monorepo correctness).

## Tests

### Updated
- `integration_test.go:462` — check `filepath.Join(subapp, ".dexter", "dexter.db")` is not created.
- `integration_test.go:508` — corrupt-DB recovery uses `store.DBPath(root)`.
- `internal/store/store_test.go:958` — uses `store.DBPath(dir)`.

### New
1. `TestStore_CreatesDexterFolder` — opening a store in an empty dir creates `.dexter/` and `.dexter/dexter.db`.
2. `TestStore_MigratesLegacyLayout` — fake legacy `.dexter.db` + `-shm` + `-wal` are deleted on Open; new DB exists and is functional.
3. `TestStore_FindProjectRoot` — table-driven walk for new marker, legacy marker, `.git`, `mix.exs` extra marker, no marker.
4. `TestIntegration_LegacyMigration` — scaffolds a project with fake legacy files, runs `dexter init`, asserts legacy deleted and new index working.

## Documentation

### `README.md`
- Line 31: TOC entry `(.dexter.db)` → `(.dexter/)`.
- Lines 104–105: gitignore snippet → `echo ".dexter/" >> .gitignore`.
- Lines 158, 190: Neovim `root_markers` / `root_pattern` — add `.dexter/dexter.db` as primary marker, keep `.git` and `mix.exs` fallbacks.
- Lines 382–400: Rewrite prose to describe the new location and the one-time migration.

### `CHANGELOG.md`
- Add `[Unreleased]` section (or next-version heading if convention differs) noting the path change, auto-migration, and gitignore update guidance. Flag that editor-extension repos (`dexter-vscode`, `dexter-zed`) will continue working via `mix.exs` / `.git` fallbacks and can be updated separately.

### `docs/architecture.md`
- Check for `.dexter.db` references during implementation; update any found.

## Non-goals

- No config option for DB location.
- No `dexter migrate` subcommand.
- No sibling files in `.dexter/` beyond the DB itself.
- No `IndexVersion` bump (schema unchanged).
- No backward-compat shim that reads from the old path.

## Rollout

Everything lands as one commit. Order within the commit:

1. `store` helpers and `Open` changes.
2. `cmd/main.go` and `internal/lsp/server.go` call sites.
3. Updated existing tests.
4. New migration tests.
5. `README.md`, `CHANGELOG.md`, `.gitignore`.
6. `make test && make lint`.

## Open questions

1. **Migration error handling.** If `migrateLegacyLayout` fails (permissions, etc.), `store.Open` returns the error and init/LSP startup fails loudly. No silent fallback. User resolves manually.
2. **CHANGELOG `[Unreleased]` section.** Existing entries are all versioned. Add `[Unreleased]` or stage under the next version heading — decided during implementation.
