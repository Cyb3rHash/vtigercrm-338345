# Repository Structure (vtiger CRM v5.4.0)

This document is a practical “map” of the repository so you can quickly find the right code when debugging, extending, or integrating.

## Top-level execution entrypoints

These are the most important “entry” scripts that drive runtime behavior:

- `index.php` — main web UI dispatcher
  - Starts session and validates installation state
  - Loads shared utilities (`include/utils/utils.php`) and user model (`modules/Users/Users.php`)
  - Routes based on `module` + `action` and includes `modules/<Module>/<Action>.php`
  - Enforces permission checks using helpers like `isPermitted()` and uses `vtlib_purify()` for sanitization

- `webservice.php` — JSON webservices endpoint
  - Uses `include/Webservices/OperationManager.php` to resolve the requested `operation`
  - Uses `include/Webservices/SessionManager.php` to manage/validate sessions (`sessionName`)
  - Loads handlers from `include/Webservices/*.php` (and module handlers resolved via `VtigerWebserviceObject`)

- `vtigerservice.php` — SOAP dispatcher
  - Includes SOAP endpoints from `soap/` based on `?service=...`

- `vtigercron.php` — cron runner
  - Loads `vtlib/Vtiger/Cron.php` and executes tasks configured in the DB table `vtiger_cron_task`

- `install.php` — installer front controller
  - Loads `install/` step scripts safely (via inclusion checks)

Other notable entrypoints:
- `Popup.php` — record selection popups used throughout the UI
- `graph.php` — graphing/report-style output (varies by feature usage)

## Where core “framework” code lives

### `include/` — shared platform layer

Key areas:
- `include/utils/utils.php`  
  General utilities used by most entrypoints (language loading, caching helpers, sanitization patterns, UI helpers, etc.).

- `include/database/PearDatabase.php`  
  Database wrapper around ADODB (`adodb/`), providing methods like `query()` and `pquery()` (prepared statements).

- `include/Webservices/`  
  Webservice runtime pieces:
  - `OperationManager.php` — operation metadata loading + execution
  - `SessionManager.php` — webservice session lifecycle
  - operation handlers like `Login.php`, `Query.php`, `Retrieve.php`, etc.
  - `Utils.php` — webservice utilities: ID conversion, module handler loading, entity registration helpers

- `include/QueryGenerator/QueryGenerator.php`  
  SQL query builder used in list views, filters, reporting, and sometimes webservices.

- `include/Ajax/CommonAjax.php`  
  Internal AJAX inclusion dispatcher: includes `modules/<module>/<file>.php` (or `modules/Vtiger/<file>.php`) after access checks.

### `data/` — core CRM entity model

- `data/CRMEntity.php`  
  The base class for module entity classes. Implements shared CRUD flows, relationship operations, file attachment helpers, list view support, and report query helpers.

This is one of the most important files to understand when you’re changing behavior that affects many modules.

### `vtlib/` — extension & lifecycle framework

- `vtlib/Vtiger/Module.php` — module lifecycle, links, relationships, and webservice init/deinit for modules
- `vtlib/Vtiger/Cron.php` — cron task registry APIs and `vtiger_cron_task` schema initialization
- `vtlib/Vtiger/Webservice.php` — helper to initialize/uninitialize module webservice support

## Where “business features” live

### `modules/` — functional modules

Each module folder (e.g., `modules/Contacts/`) commonly contains:
- Controller-like action scripts: `ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`, `Delete.php`, `Import.php`, …
- AJAX entry scripts: `*Ajax.php`, `DetailViewAjax.php`, etc.
- Entity class: `modules/<Module>/<Module>.php` (typically extends `CRMEntity`)

Routing for UI requests is primarily driven by:
- `index.php` reading `module` + `action` → includes `modules/<Module>/<Action>.php`

## UI rendering

- `Smarty_setup.php` defines `vtigerCRM_Smarty` (vtiger’s Smarty wrapper)
- `Smarty/` contains Smarty itself and templates under `Smarty/templates/**`
- `themes/` contains theme assets and images

## Integrations

- JSON API: `webservice.php` (see `Docs/Webservice-Internals.md` and `Docs/VTigerCRM-API-Reference.md`)
- SOAP: `vtigerservice.php` + `soap/` (see `Docs/VTigerCRM-API-Reference.md`)

## Scheduled work

- Preferred framework-driven cron: `vtigercron.php` + `vtlib/Vtiger/Cron.php`
- Legacy/standalone cron scripts: `cron/` (some may be invoked by system cron directly)

See `Docs/Cron-Jobs.md`.

## “How do I…?” quick pointers

- **Find what code runs for a page:** look at the request’s `module` and `action`, then open `modules/<Module>/<Action>.php`
- **Find the module’s data behavior:** open `modules/<Module>/<Module>.php` and the base `data/CRMEntity.php`
- **Find a DB query wrapper:** check `include/database/PearDatabase.php`
- **Find webservice handler:** start at `webservice.php` → `include/Webservices/OperationManager.php` → handler file under `include/Webservices/`
- **Find popup record selection logic:** `Popup.php`
