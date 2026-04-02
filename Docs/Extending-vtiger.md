# Extending vtiger (modules, entities, vtlib)

This repository uses vtiger’s classic “modules + shared framework” architecture. Most customizations fall into one of these categories:

- adding/modifying a **module** under `modules/`
- extending the shared **entity model** (`data/CRMEntity.php`)
- using **vtlib** APIs for relationships, links, cron registration, and webservice enablement

## Module anatomy (`modules/<ModuleName>/`)

A typical module contains:

- **Action scripts** (controller-like):
  - `ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`, `Delete.php`, `Import.php`, …
  - `*Ajax.php` or `DetailViewAjax.php` for UI-driven async actions

- **Entity class**:
  - Usually `modules/<ModuleName>/<ModuleName>.php`
  - Commonly extends `data/CRMEntity.php`

## How UI routing works

The primary dispatcher is `index.php`:

- Reads request params like `module` and `action`
- Includes `modules/<Module>/<Action>.php` after:
  - session initialization
  - installation/config checks
  - permission checks (e.g., `isPermitted()`)
  - sanitization (e.g., `vtlib_purify()`)

This means:
- adding a new UI “action” usually means adding a corresponding PHP file in the module directory.
- you must ensure permissions and sanitization match existing patterns.

## The base entity model: `data/CRMEntity.php`

`CRMEntity` is a central base class that provides shared behaviors used across modules:

- CRUD coordination (`save`, `mark_deleted`, `restore`)
- relationship handling (link/unlink related records)
- attachment upload helpers
- list view/query utilities and report query building
- event trigger hooks

When modifying or extending entity-level behavior, consider the impact across all modules that inherit from this base.

## vtlib: module lifecycle and extension points

vtlib provides APIs to manage and extend modules programmatically.

Key files:
- `vtlib/Vtiger/Module.php`
  - module instances: `Vtiger_Module::getInstance()`, `getClassInstance()`
  - related lists: `setRelatedList()`, `unsetRelatedList()`
  - links: `addLink()`, `deleteLink()`
  - webservice enablement: `initWebservice()`, `deinitWebservice()`

- `vtlib/Vtiger/Webservice.php`
  - `Vtiger_Webservice::initialize()` / `uninitialize()` which call helpers like `vtws_addDefaultModuleTypeEntity`

- `vtlib/Vtiger/Cron.php`
  - registering and managing tasks in `vtiger_cron_task`

## Creating a new module (skeleton approach)

This repository includes an existing guide:
- `vtlib/README-UsingModuleDir.txt`

It describes using the skeleton module templates under:
- `vtlib/ModuleDir/<target_vtiger_version>/`

High-level workflow:
1. Copy skeleton to `modules/<NewModuleName>/`
2. Rename module files appropriately
3. Update the module’s entity class (table names, indexes, lists, etc.)

## Webservice considerations

If you want a module to be accessible via JSON webservices:

- the module must be registered as a webservice entity (see vtlib webservice initialization)
- the relevant handlers and metadata must exist so `webservice.php` can resolve operations and entity types

See `Docs/Webservice-Internals.md` for how dispatch works.

## Practical tips

- Prefer adding new behavior in module-specific classes/scripts first; changes to `CRMEntity` affect many modules.
- Reuse shared helpers from `include/utils/utils.php` and DB helpers from `include/database/PearDatabase.php`.
- When adding new AJAX endpoints, understand the dispatcher `include/Ajax/CommonAjax.php` which includes module scripts based on request parameters and uses inclusion access checks.
