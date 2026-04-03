# Migration module (`modules/Migration/`)

Migration/upgrade tooling used when moving between vtiger versions.

Key folder:
- `DBChanges/` — versioned change scripts (see `DBChanges/README.md`)

Notes:
- Treat migration scripts carefully; they can modify schema and critical data.
