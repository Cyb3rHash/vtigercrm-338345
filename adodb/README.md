# `adodb/` — ADODB database abstraction library (third-party)

This folder contains the **ADODB** PHP library bundled with vtiger CRM.

vtiger primarily interacts with ADODB indirectly through:
- `include/database/PearDatabase.php` (`PearDatabase`), which wraps ADODB and exposes vtiger-style DB APIs such as `query()` and `pquery()`.

## Notes for developers
- Treat this as **vendor code**.
- Avoid editing files here unless you are intentionally patching the dependency (and you should document the patch in a PR description / internal changelog).

## Where it is used
- DB layer: `include/database/PearDatabase.php`
- Some DB schema / query utilities also assume ADODB behaviors via `PearDatabase`.
