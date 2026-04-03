# Docs index — vtiger CRM (v5.4.0)

This folder contains architecture and developer documentation for this vtigerCRM repository. The docs here are intended to be **grounded in the actual code structure and entrypoints** present in the repo.

---

## Start here

- **Architecture overview:** `VTigerCRM-Architecture-Overview.md`  
  High-level components, primary entrypoints (`index.php`, `webservice.php`, `install.php`, `vtigercron.php`) and data flows.

- **Repository map:** `Repository-Structure.md`  
  A practical map of the codebase (directories, key files, and what to read when debugging).

- **API reference:** `VTigerCRM-API-Reference.md`  
  JSON webservices (`webservice.php`), SOAP (`vtigerservice.php` + `soap/`), and internal AJAX patterns.

---

## Developer-oriented deep dives (repo-grounded)

- `Webservice-Internals.md`  
  How `webservice.php` dispatches operations via `include/Webservices/OperationManager.php`.

- `Cron-Jobs.md`  
  How scheduled work runs via `vtigercron.php` + `vtlib/Vtiger/Cron.php`, and how `cron/` scripts fit in.

- `Extending-vtiger.md`  
  Adding/understanding modules, entity classes (`data/CRMEntity.php`), and vtlib extension points.

---

## Design docs / diagrams

- `HLD.md` — high-level design
- `LLd.md` — low-level design notes
- `component_diagram.md`, `dependency_map.md`, `knowledge_graph.md`, `mindmap.md`
- `VTigerCRM-Modules-MindMap.md` — module inventory/mindmap
- `Leads-Accounts-Contacts-Opportunities-Interactions.md` — module interaction notes
- Other domain diagrams: `data_model.md`, `contacts_class_diagram.md`, `lead_flow_sequence.md`

---

## “Local README” documentation in the codebase

To help onboarding, most major folders now have their own `README.md` explaining what the folder is for and where to start.

### Top-level
- `README.md` (repo guide + entrypoints + quick start)
- `Docs/README.md` (this file)

### Core framework
- `include/README.md`
  - plus sub-READMEs:
    - `include/Webservices/README.md`
    - `include/utils/README.md`
    - `include/database/README.md`
    - `include/Ajax/README.md`
    - `include/QueryGenerator/README.md`
    - `include/ListView/README.md`
    - and vendor subfolders (ckeditor/htmlpurifier/tcpdf/etc.)

- `data/README.md`
- `vtlib/README.md`
- `Smarty/README.md`

### Feature modules
- `modules/README.md`
- `modules/<Module>/README.md` for every module under `modules/*`
  - Additional submodule docs exist for:
    - `modules/Settings/*`
    - `modules/com_vtiger_workflow/*`
    - `modules/Migration/DBChanges/`

### Ops / integration / supporting folders
- `install/README.md`
- `cron/README.md` and `cron/modules/README.md`
- `soap/README.md`
- `schema/README.md`
- `packages/README.md`
- `themes/README.md`
- `user_privileges/README.md`
- `test/README.md`
- plus other top-level library/support folders (`adodb/`, `Image/`, `kcfinder/`, …)

---
If you’re onboarding: read `README.md` at the repo root, then open the README in the folder you’re working in.
