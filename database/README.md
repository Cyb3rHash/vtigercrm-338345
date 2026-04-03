# `database/` — database connection helpers (root-level)

This folder contains repository-local database connection helper code used by vtiger components.

Key file:
- `DatabaseConnection.php` — helper to create/manage DB connections used during runtime and/or install/health-check flows.

## Related
- Main DB abstraction used by most code: `include/database/PearDatabase.php`
- DB schema artifact: `schema/DatabaseSchema.xml`
- Installer: `install.php` and `install/`
