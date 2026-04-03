# `include/database/` — database abstraction layer

vtiger’s primary DB abstraction is implemented here.

## Key file
- `PearDatabase.php` — defines `PearDatabase`, a wrapper around ADODB with vtiger-style APIs:
  - `query()` — raw queries
  - `pquery()` — prepared statements
  - result helpers like `num_rows()`, `query_result()`, etc.

Other files:
- `Postgres8.php` — DB-specific behavior/support (where applicable)

## Notes
- Most code uses the global `$adb` object.
- Prefer `pquery()` for parameterized queries.

Related:
- ADODB vendor library: `adodb/`
- Repo structure docs: `Docs/Repository-Structure.md`
