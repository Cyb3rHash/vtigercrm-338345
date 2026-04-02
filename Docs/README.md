# Docs index — vtiger CRM (v5.4.0)

This folder contains architecture and developer documentation for this vtigerCRM repository. The docs here are intended to be **grounded in the actual code structure and entrypoints** present in the repo.

## Start here

- **Architecture overview:** `VTigerCRM-Architecture-Overview.md`  
  High-level components, primary entrypoints (`index.php`, `webservice.php`, `install.php`, `vtigercron.php`) and data flows.

- **API reference:** `VTigerCRM-API-Reference.md`  
  JSON webservices (`webservice.php`), SOAP (`vtigerservice.php` + `soap/`), and internal AJAX endpoints.

## Design docs

- `HLD.md` — high-level design
- `LLd.md` — low-level design notes
- `VTigerCRM-Modules-MindMap.md` — module inventory/mindmap
- `Leads-Accounts-Contacts-Opportunities-Interactions.md` — module interaction notes

## Developer-oriented guides (repo-grounded)

- `Repository-Structure.md`  
  A practical map of the codebase (directories, key files, and what to read when debugging).

- `Webservice-Internals.md`  
  How `webservice.php` dispatches operations via `include/Webservices/OperationManager.php` and how handlers are organized.

- `Cron-Jobs.md`  
  How scheduled work runs via `vtigercron.php` + `vtlib/Vtiger/Cron.php`, and how `cron/` scripts fit in.

- `Extending-vtiger.md`  
  Adding/understanding modules, entity classes (`data/CRMEntity.php`), and vtlib extension points.

## Module READMEs in the codebase

Several major directories also contain concise `README.md` files for local orientation:

- `include/README.md`
- `modules/README.md`
- `vtlib/README.md`
- `data/README.md`
- `cron/README.md`
- `install/README.md`
- `soap/README.md`
- `Smarty/README.md`
