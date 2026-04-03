# vtiger CRM (v5.4.0) — Repository Guide

This repository is a **monolithic, server-rendered PHP application** (vtiger CRM 5.4.0). Browser requests are dispatched through central entrypoints and routed to **module scripts** under `modules/`, backed by shared framework utilities under `include/`, `data/`, and `vtlib/`.

If you're new to the codebase, start by understanding:
- **How requests are routed** (`index.php` → `modules/<Module>/<Action>.php`)
- **Where shared services live** (`include/`, `data/`, `vtlib/`)
- **Where business features live** (`modules/`)
- **How scheduled work runs** (`vtigercron.php` / `cron/`)
- **How APIs work** (`webservice.php` / `vtigerservice.php`)

---

## Quick start (local dev / sandbox)

### Prerequisites
- PHP runtime compatible with vtiger 5.4.0
- MySQL (or supported DB) with a database + user created
- A PHP-capable web server (Apache/Nginx) or PHP-FPM setup

### Install
1. Deploy this repo to your web root (or configure your vhost to point here).
2. Ensure the webserver can write to:
   - `cache/`, `logs/`, and the attachments/upload directories used by your configuration
3. Open the installer:
   - `http(s)://<host>/install.php`
4. The installer will generate/fill configuration files and initialize DB schema/data.

Key installer docs:
- `install/README.md`
- `Docs/VTigerCRM-Architecture-Overview.md`

---

## Key entrypoints (start here)

- **Web UI / main dispatcher:** `index.php`  
  Reads request params (`module`, `action`) and includes `modules/<Module>/<Action>.php` after session/installation/permission checks.

- **Installer wizard:** `install.php` (and `install/`)  
  Step-driven installer that generates configuration and initializes schema/data.

- **JSON Web Services API:** `webservice.php`  
  Metadata-driven API dispatcher (see `include/Webservices/OperationManager.php`).

- **SOAP service dispatcher:** `vtigerservice.php`  
  Routes to `soap/*.php` implementations based on `?service=<name>`.

- **Cron / scheduled jobs runner:** `vtigercron.php`  
  Executes tasks registered in DB table `vtiger_cron_task` using `vtlib/Vtiger/Cron.php`.

- **Popup entrypoint:** `Popup.php`  
  Record-selection popups used across modules (built on list view utilities and module entities).

---

## Directory map (high level)

### Application feature modules
- `modules/` — functional CRM modules (Accounts, Contacts, Leads, …)  
  See: `modules/README.md` and per-module `modules/<Module>/README.md`

### Core shared framework
- `include/` — shared framework services (db abstraction, utils, webservices, query generator, ajax dispatcher, …)  
  See: `include/README.md` and sub-READMEs in `include/*/README.md`

- `data/` — core domain base classes, especially `data/CRMEntity.php`  
  See: `data/README.md`

- `vtlib/` — vtiger extension framework (modules lifecycle, cron registry, webservice init)  
  See: `vtlib/README.md`

### UI rendering / assets
- `Smarty/` — Smarty engine + templates (`Smarty/templates/**`)  
  See: `Smarty/README.md`

- `themes/` — theme resources and static UI assets  
  See: `themes/README.md`

### Ops / integration / support
- `cron/` — legacy/standalone cron scripts (in addition to DB-registered cron tasks)  
  See: `cron/README.md`

- `soap/` — SOAP services (NuSOAP-based)  
  See: `soap/README.md`

- `schema/` — DB schema definition artifact(s)  
  See: `schema/README.md`

- `packages/` — vtiger module ZIP packages (mandatory/optional feature packs)  
  See: `packages/README.md`

---

## Documentation (recommended reading order)

1. `Docs/VTigerCRM-Architecture-Overview.md`
2. `Docs/Repository-Structure.md`
3. `Docs/VTigerCRM-API-Reference.md` (if integrating)
4. `Docs/Webservice-Internals.md` (if extending JSON API)
5. `Docs/Cron-Jobs.md` (if operating cron)
6. `Docs/Extending-vtiger.md` (if adding modules/customizations)

Docs index:
- `Docs/README.md`

---

## Conventions for developers

### Where to implement changes
- **UI routing / permission checks:** `index.php`
- **Module behavior (CRUD/relationships/hooks):** `modules/<Module>/<Module>.php` (usually extends `data/CRMEntity.php`)
- **Shared helpers:** `include/utils/utils.php` + `include/utils/*.php`
- **DB queries:** use `$adb` (PearDatabase), see `include/database/PearDatabase.php`
- **List views / popups:** `include/ListView/*`, `Popup.php`
- **Webservice operations:** `webservice.php` → `include/Webservices/*`

### Third-party code
Directories like `adodb/`, `include/ckeditor/`, `include/htmlpurifier/`, `include/tcpdf/`, etc. are vendor libraries.
Prefer **not** to modify these unless you are intentionally patching a dependency (and documenting why).

---
Task-owner note: This repository now contains a README.md in each major folder and each vtiger module folder under `modules/*` to help onboarding.
