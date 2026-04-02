# vtiger CRM (v5.4.0) — Repository Guide

This repository is a monolithic, server-rendered PHP application (vtiger CRM 5.4.0). Requests are routed through central PHP entrypoints and dispatched to module scripts under `modules/`, backed by a shared database layer and common framework utilities under `include/`, `data/`, and `vtlib/`.

## Key entrypoints (start here)

- **Web UI / main dispatcher:** `index.php`  
  Reads `module` and `action` request params and includes `modules/<Module>/<Action>.php` after session, installation, and permission checks.

- **Installer wizard:** `install.php` (and `install/`)  
  Runs the step-driven installer and generates configuration (`config.inc.php` / DB config) and schema.

- **JSON Web Services API:** `webservice.php`  
  Metadata-driven API dispatcher. Uses `include/Webservices/OperationManager.php` + DB tables (e.g., `vtiger_ws_operation`) to locate and run operations like `login`, `query`, `retrieve`, etc.

- **SOAP service dispatcher:** `vtigerservice.php`  
  Routes to `soap/*.php` implementations based on `?service=<name>`.

- **Cron / scheduled jobs:** `vtigercron.php`  
  Loads `vtlib/Vtiger/Cron.php` and executes tasks registered in the DB table `vtiger_cron_task`.

- **Popup UI entrypoint:** `Popup.php`  
  Record-selection popups used across modules, built on list-view utilities and module entity classes.

## Directory map (high level)

- `modules/` — functional CRM modules (Accounts, Contacts, Leads, …). Each module typically contains:
  - action scripts (`ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`, `*Ajax.php`, …)
  - a module entity class (e.g., `modules/Contacts/Contacts.php`) usually extending `data/CRMEntity.php`

- `data/` — core domain base classes, especially:
  - `data/CRMEntity.php` — base CRUD + relationship + security hooks used by module entity classes

- `include/` — shared platform services (database, utils, webservices, query generation, ajax dispatcher, …)
  - `include/database/PearDatabase.php` — DB abstraction over ADODB (`adodb/`)
  - `include/utils/utils.php` — foundational helpers (language loading, caching, sanitization helpers, etc.)
  - `include/Webservices/` — webservice runtime implementation and operation handlers
  - `include/QueryGenerator/QueryGenerator.php` — SQL query builder used by list views, filters, reporting

- `vtlib/` — vtiger’s extension framework (modules lifecycle, cron registry, webservice registration)
- `Smarty/` — Smarty templating engine and templates (vtiger wrapper: `Smarty_setup.php`)
- `themes/` — theme resources
- `cron/` — legacy/standalone cron scripts (in addition to DB-registered cron tasks)
- `soap/` — SOAP services (NuSOAP-based)

## Documentation

See `Docs/README.md` for a curated index and developer-focused guides that are grounded in this repository’s actual entrypoints and directories.

---
If you are new to vtiger’s codebase, read in this order:
1. `Docs/VTigerCRM-Architecture-Overview.md`
2. `Docs/Repository-Structure.md`
3. `Docs/Webservice-Internals.md` (if integrating via API)
4. `Docs/Cron-Jobs.md` (if operating scheduled jobs)
5. `Docs/Extending-vtiger.md` (if adding modules/customizations)
