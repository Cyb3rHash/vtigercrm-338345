# `include/Webservices/` — JSON Web Services runtime

This folder implements vtiger’s **JSON Web Services** (API) used by `webservice.php`.

## How requests flow
1. Client calls `webservice.php` with `operation=<name>`
2. `webservice.php` uses `OperationManager.php` to load operation metadata from DB (e.g., `vtiger_ws_operation`)
3. `SessionManager.php` validates/creates the webservice session (`sessionName`)
4. The handler file (often in this folder) is loaded and executed

## Key files
- `OperationManager.php` — resolves operation → handler and executes it
- `SessionManager.php` — session lifecycle for API sessions
- `State.php` — success/error envelope model
- Operation handlers (examples):
  - `Login.php`, `Logout.php`
  - `Query.php`, `Retrieve.php`, `Create.php`, `Update.php`, `Delete.php`
  - `DescribeObject.php`, `ModuleTypes.php`
- `Utils.php` — webservice utility helpers (ID conversion, entity registration, etc.)

## Related docs
- `Docs/VTigerCRM-API-Reference.md`
- `Docs/Webservice-Internals.md`
- `include/README.md`
