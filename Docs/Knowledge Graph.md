# Knowledge Graph

## 1. Purpose and scope

This document provides a repository-wide knowledge graph for the vtiger CRM v5.4.0 codebase in this repository. The intent is to map the major subsystems, the most important nodes (runtime entrypoints, shared services, module layer, and storage), and the key edges (includes/calls/reads-writes/registration) that connect them.

This is not a per-file call graph. The repository contains thousands of PHP files (especially under `modules/` and `include/`), so the graph is organized as a subsystem graph. Where the codebase uses dynamic dispatch (for example, `index.php` including `modules/<Module>/<Action>.php`), the graph models that dispatch as a structured edge rather than enumerating every possible included file.

## 2. Conventions (nodes and edges)

Nodes are grouped by major subsystems. Each node represents one of the following.

A node can be:
1. A runtime entrypoint script (for example, `index.php`).
2. A shared platform/library component (for example, `data/CRMEntity.php` or `include/database/PearDatabase.php`).
3. A module family (for example, “Business modules (modules/*)”).
4. A database concept (for example, “MySQL DB” or “vtiger_ws_operation tables”).
5. A filesystem storage concept (for example, `storage/`, `cache/upload/`).

Edges use consistent verbs:
1. “includes” indicates PHP `include`/`require_once` usage.
2. “dispatches to” indicates dynamic routing to a file path constructed at runtime.
3. “reads/writes” indicates database IO via `$adb` (PearDatabase) or direct SQL.
4. “registers” indicates runtime registration in DB (for example, cron tasks or event handlers).
5. “triggers” indicates the vtiger event framework invoking handlers (for example, workflow event handling).

## 3. Major subsystems (grouped overview)

The repository can be usefully partitioned into these subsystems.

### 3.1 Runtime entrypoints (HTTP / CLI / install)

The system has multiple “front doors” that start a request or job and then converge on shared framework services and the same database.
1. `index.php` is the primary web UI dispatcher.
2. `webservice.php` is the JSON webservice endpoint driven by DB-registered operations.
3. `vtigerservice.php` is the SOAP services endpoint router (includes `soap/*.php` based on `service` parameter).
4. `install.php` is the installer wizard dispatcher (includes `install/<step>.php` files).
5. `vtigercron.php` is the cron runner for DB-registered tasks.

### 3.2 Shared platform services (“vtiger platform layer”)

The shared platform layer is what modules and entrypoints reuse.
1. Core entity model base: `data/CRMEntity.php` (CRUD, attachments, relations, event triggers, access-control join helpers).
2. Database abstraction: `include/database/PearDatabase.php` (wraps ADODB from `adodb/` and provides `query`, `pquery`, caching toggles, charset setup).
3. Utilities and security: `include/utils/utils.php` (pulled in by `index.php` and `data/CRMEntity.php`; provides common helpers, sanitization, and file access checking functions referenced throughout).
4. Events framework: `include/events/include.inc` and related classes (antlr-based condition parsing and handler registration/execution).
5. vtlib extension APIs: `vtlib/Vtiger/*` (module management, webservice enablement, event registration helper, cron task registry).

### 3.3 Module layer (business and settings modules)

The module layer lives under `modules/` and is invoked by entrypoints (mostly by `index.php`).
1. Module “entity classes” typically extend `CRMEntity` (for example `modules/Leads/Leads.php`, `modules/Users/Users.php`).
2. Module “action scripts” implement controller-like behaviors (for example `DetailView.php`, `EditView.php`, `Save.php`, `*Ajax.php`).
3. Cross-module relationships are managed via generic relation tables (for example `vtiger_crmentityrel`) and module-specific relation tables (for example `vtiger_seproductsrel`, `vtiger_campaignleadrel` as seen in `modules/Leads/Leads.php`).

### 3.4 Automation (events, workflows, cron)

Automation is implemented through three main mechanisms.
1. The vtiger event bus triggers handlers during `CRMEntity::save()` (`vtiger.entity.beforesave.*`, `vtiger.entity.aftersave.*`).
2. The workflow engine (`modules/com_vtiger_workflow/VTEventHandler.inc`) evaluates workflow conditions and performs tasks when events occur.
3. Scheduled background jobs are executed via `vtigercron.php` and `vtlib/Vtiger/Cron.php` based on the DB table `vtiger_cron_task`. Install-time registration of cron tasks is performed in `install/CreateTables.inc.php`.

### 3.5 Integration endpoints (webservice JSON + SOAP)

Integration is surfaced through:
1. JSON webservices: `webservice.php` + `include/Webservices/*` using DB-described operations and session management.
2. SOAP services: `vtigerservice.php` includes specific SOAP server scripts under `soap/` based on the `service` request parameter.

### 3.6 Storage and assets

The application uses:
1. A relational DB for system state and CRM records.
2. Filesystem storage for attachments (via `vtiger_attachments.path` and disk writes in `CRMEntity::uploadAndSaveFile()`).
3. UI templates and assets: Smarty templates under `Smarty/` and themes under `themes/`.
4. Bundled third-party libs: `adodb/`, `Smarty/`, `include/ckeditor/`, `include/htmlpurifier/`, `include/tcpdf/`, `log4php/` and `log4php.debug/`.

## 4. System context knowledge graph (actors and external dependencies)

This diagram shows the system boundary (“vtiger CRM monolith”) and the most visible external actors and dependencies. It also calls out bundled third-party libraries as “external dependencies” from the point of view of the vtiger application code, even though they are vendored into the repository.

```mermaid
flowchart LR
  browser["Browser user"] --> web["Web server with PHP runtime"]
  wsclient["API client"] --> web
  scheduler["Scheduler (cron)"] --> phpcli["PHP CLI"]
  admin["Administrator"] --> browser

  subgraph system["vtiger CRM (monolithic PHP app)"]
    ui["Web UI dispatcher (index.php)"]
    api["JSON Webservice endpoint (webservice.php)"]
    soap["SOAP services router (vtigerservice.php)"]
    installer["Installer entry (install.php)"]
    cronrun["Cron runner (vtigercron.php)"]
    modules["Modules (modules/*)"]
    platform["Platform services (data/, include/, vtlib/)"]
  end

  web --> ui
  web --> api
  web --> soap
  web --> installer
  phpcli --> cronrun

  ui --> modules
  ui --> platform
  api --> platform
  cronrun --> platform
  installer --> platform

  db["Relational DB (MySQL or Postgres)"] <--> platform
  fs["Filesystem storage (attachments, cache, templates)"] <--> platform

  subgraph bundled["Bundled dependencies (vendored)"]
    adodb["ADODB (adodb/)"]
    smarty["Smarty templates (Smarty/)"]
    log4php["log4php (log4php/ or log4php.debug/)"]
    zendjson["Zend JSON (include/Zend/Json.php)"]
    antlr["ANTLR runtime (include/antlr/)"]
    cke["CKEditor (include/ckeditor/)"]
    purifier["HTMLPurifier (include/htmlpurifier/)"]
    tcpdf["TCPDF (tcpdf/)"]
  end

  platform --> adodb
  platform --> smarty
  platform --> log4php
  platform --> zendjson
  platform --> antlr
  platform --> cke
  platform --> purifier
  platform --> tcpdf
```

The dominant pattern is that all external interactions (UI, API, background jobs) converge on the same “platform layer” and the same database and filesystem.

## 5. Runtime entrypoints graph (how requests and jobs enter the system)

This diagram enumerates the repository’s primary entrypoints and shows which platform mechanisms they use.

```mermaid
flowchart TB
  subgraph entry["Entrypoints (top-level PHP scripts)"]
    e_index["index.php"]
    e_ws["webservice.php"]
    e_soap["vtigerservice.php"]
    e_install["install.php"]
    e_cron["vtigercron.php"]
  end

  subgraph core["Core platform"]
    c_utils["include/utils/utils.php"]
    c_cfg["config.inc.php"]
    c_crmentity["data/CRMEntity.php"]
    c_db["include/database/PearDatabase.php"]
    c_vtlib["vtlib/Vtiger/*"]
    c_events["include/events/include.inc"]
  end

  subgraph modulelayer["Module layer"]
    m_actions["Action scripts (modules/<Module>/<Action>.php)"]
    m_entities["Entity classes (modules/<Module>/<Module>.php)"]
  end

  subgraph apiimpl["Integration implementations"]
    ws_ops["Webservice operation registry (vtiger_ws_operation tables)"]
    ws_mgr["include/Webservices/OperationManager.php"]
    ws_sess["include/Webservices/SessionManager.php"]
    soap_services["soap/*.php services"]
  end

  subgraph automation["Automation"]
    cron_tasks["Cron tasks registry (vtiger_cron_task)"]
    vtcron["vtlib/Vtiger/Cron.php"]
    wf_handler["modules/com_vtiger_workflow/VTEventHandler.inc"]
  end

  e_index --> c_utils
  e_index --> c_cfg
  e_index --> m_actions
  m_actions --> m_entities
  m_entities --> c_crmentity
  c_crmentity --> c_db
  c_crmentity --> c_events

  e_ws --> c_cfg
  e_ws --> ws_mgr
  ws_mgr --> ws_ops
  ws_mgr --> ws_sess
  e_ws --> c_db

  e_soap --> soap_services
  soap_services --> c_db

  e_cron --> vtcron
  vtcron --> cron_tasks
  e_cron --> c_cfg
  e_cron --> c_db

  e_install --> c_utils
  e_install --> c_vtlib
  e_install --> c_db
  e_install --> cron_tasks
  e_install --> wf_handler
```

This graph is driven directly by the entrypoint scripts:
1. `index.php` requires `include/utils/utils.php`, checks config, and then dynamically includes `modules/<module>/<action>.php`.
2. `webservice.php` uses `OperationManager` to look up operation metadata in DB tables and starts sessions using `SessionManager`.
3. `vtigercron.php` enumerates DB-configured tasks via `Vtiger_Cron` and includes task handler files at runtime.
4. `install.php` chooses install step files under `install/` and delegates install-time schema and metadata creation to `install/CreateTables.inc.php`.

## 6. Subsystem knowledge graph (platform services and their dependencies)

This diagram shows the “platform core” decomposition and highlights how the database and eventing mechanisms are central to most subsystems.

```mermaid
flowchart LR
  subgraph platform["Platform services"]
    p_crmentity["CRMEntity base (data/CRMEntity.php)"]
    p_db["PearDatabase (include/database/PearDatabase.php)"]
    p_logging["Logging bootstrap (include/logging.php)"]
    p_utils["Utilities and security (include/utils/utils.php)"]
    p_vtlib_module["vtlib Module API (vtlib/Vtiger/Module.php)"]
    p_vtlib_event["vtlib Event API (vtlib/Vtiger/Event.php)"]
    p_vtlib_cron["vtlib Cron API (vtlib/Vtiger/Cron.php)"]
    p_ws_utils["Webservices Utils (include/Webservices/Utils.php)"]
    p_ws_mgr["OperationManager (include/Webservices/OperationManager.php)"]
    p_ws_sess["SessionManager (include/Webservices/SessionManager.php)"]
    p_events["Events framework (include/events/include.inc)"]
  end

  subgraph data["Data stores and artifacts"]
    db["DB server"]
    fs_attach["Attachments on disk (vtiger_attachments.path)"]
    fs_cache["Cache directories (cache/*)"]
  end

  subgraph deps["Bundled dependencies"]
    d_adodb["ADODB library (adodb/)"]
    d_log4php["log4php (log4php/ or log4php.debug/)"]
    d_zjson["Zend JSON (include/Zend/Json.php)"]
    d_antlr["ANTLR runtime (include/antlr/)"]
  end

  p_db --> d_adodb
  p_logging --> d_log4php
  p_ws_mgr --> d_zjson
  p_events --> d_antlr

  p_crmentity --> p_logging
  p_crmentity --> p_utils
  p_crmentity --> p_db
  p_crmentity --> p_events

  p_ws_utils --> p_db
  p_ws_mgr --> p_ws_utils
  p_ws_mgr --> p_ws_sess

  p_vtlib_module --> p_db
  p_vtlib_event --> p_db
  p_vtlib_cron --> p_db

  p_db <--> db
  p_crmentity <--> fs_attach
  p_db <--> fs_cache
```

Key facts evidenced by code:
1. `CRMEntity` is the module model base and triggers events on save via `VTEventsManager` (`CRMEntity::save()` requires `include/events/include.inc`).
2. `PearDatabase` wraps ADODB and is the core DB API used throughout (`PearDatabase::getInstance()` returns global `$adb`).
3. Logging is log4php-based (`include/logging.php` selects `log4php` or `log4php.debug` and configures via `log4php.properties`).
4. Webservice operation execution is mediated by `OperationManager` and stored metadata in DB.

## 7. Key end-to-end flows as sequence graphs

### 7.1 Web UI request dispatch (index.php to module action)

This sequence models the “happy path” for a browser request that renders a module page.

```mermaid
sequenceDiagram
  participant B as Browser
  participant W as WebServer
  participant I as index.php
  participant A as ModuleActionScript
  participant E as CRMEntity
  participant D as PearDatabase
  participant DB as Database

  B->>W: HTTP request with module and action
  W->>I: Execute PHP entrypoint
  I->>I: Start session and validate config
  I->>I: Validate module and action
  I->>I: Permission check
  I->>A: Include modules/<Module>/<Action>.php
  A->>E: Use module entity class (extends CRMEntity)
  E->>D: query() or pquery()
  D->>DB: SQL
  DB-->>D: rows
  D-->>E: rows
  A-->>B: Rendered HTML response
```

Evidence anchors:
1. Dispatch: `index.php` builds `modules/$module/$action.php` and includes it after module/action checks.
2. Entity instantiation: `index.php` uses `CRMEntity::getInstance($currentModule)` for DetailView tracking and modules do similar patterns.
3. DB: modules and base classes use `$adb = PearDatabase::getInstance()` or the global `$adb`.

### 7.2 JSON Webservice request execution (webservice.php + OperationManager)

This sequence models how `webservice.php` resolves an operation from DB metadata and runs it.

```mermaid
sequenceDiagram
  participant C as APIClient
  participant WS as webservice.php
  participant OM as OperationManager
  participant SM as SessionManager
  participant H as OperationHandler
  participant D as PearDatabase
  participant DB as Database

  C->>WS: HTTP request with operation and parameters
  WS->>OM: Construct OperationManager(adb, operation, format, sessionManager)
  OM->>DB: Read vtiger_ws_operation and vtiger_ws_operation_parameters
  WS->>SM: startSession(sessionName)
  SM-->>WS: session id or error
  WS->>OM: sanitizeOperation(input)
  OM->>H: require_once(handler_path) and call handler method
  H->>D: query()/pquery()
  D->>DB: SQL
  DB-->>D: rows
  D-->>H: rows
  H-->>OM: operation result
  OM-->>WS: encoded JSON data
  WS-->>C: JSON response envelope (State)
```

Evidence anchors:
1. DB-driven operation lookup: `OperationManager::fillOperationDetails()` queries `vtiger_ws_operation` and `vtiger_ws_operation_parameters`.
2. Session management: `SessionManager` uses `HTTP_Session` and enforces expiry/idle rules.
3. Webservice helper library: `include/Webservices/Utils.php` is loaded and used to resolve entity IDs and other helpers.

### 7.3 Cron task execution (vtigercron.php + vtlib Cron registry)

This sequence models the scheduled job runner.

```mermaid
sequenceDiagram
  participant S as Scheduler
  participant CR as vtigercron.php
  participant VC as Vtiger_Cron
  participant HF as HandlerFile
  participant D as PearDatabase
  participant DB as Database

  S->>CR: Run PHP script (CLI) or authenticated request
  CR->>VC: listAllActiveInstances()
  VC->>DB: Read vtiger_cron_task
  DB-->>VC: tasks list
  CR->>VC: isRunnable() and markRunning()
  VC->>DB: Update vtiger_cron_task status and laststart
  CR->>HF: require_once(handler_file) and execute
  HF->>D: query()/pquery()
  D->>DB: SQL
  DB-->>D: rows
  CR->>VC: markFinished()
  VC->>DB: Update vtiger_cron_task lastend and status
```

Evidence anchors:
1. `vtigercron.php` loads `vtlib/Vtiger/Cron.php`, iterates tasks, includes handler file, marks running/finished.
2. `vtlib/Vtiger/Cron.php` defines schema initialization and registry in `vtiger_cron_task`.

### 7.4 Event-driven workflow execution (CRMEntity save -> VTWorkflowEventHandler)

This sequence shows how CRM entity saves cause the workflow engine to evaluate workflows and run tasks.

```mermaid
sequenceDiagram
  participant UI as ModuleSaveScript
  participant E as CRMEntity
  participant EM as VTEventsManager
  participant WF as VTWorkflowEventHandler
  participant WM as VTWorkflowManager
  participant DB as Database

  UI->>E: save(module)
  E->>EM: trigger vtiger.entity.beforesave
  EM-->>WF: handler (if registered)
  E->>E: saveentity() writes module tables
  E->>EM: trigger vtiger.entity.aftersave
  EM-->>WF: handleEvent(aftersave, entityData)
  WF->>WM: getWorkflowsForModule(module)
  WM->>DB: query workflows tables
  DB-->>WM: workflows
  WF->>WF: evaluate conditions and perform tasks
```

Evidence anchors:
1. `CRMEntity::save()` triggers events in `data/CRMEntity.php`.
2. Install-time event registration in `install/CreateTables.inc.php` registers `VTWorkflowEventHandler` for `vtiger.entity.aftersave`.
3. `VTWorkflowEventHandler` loads webservice utilities and workflow manager classes and performs workflow evaluation and tasks.

## 8. Database-centric nodes (tables as graph anchors)

Even though vtiger has a large schema, some tables are architectural “routing tables” because they control dynamic behavior. These are the most important for understanding system-level flows.

### 8.1 “Meta” tables that drive dynamic behavior

1. `vtiger_version` is queried by `index.php` to ensure the DB version matches the code version.
2. `vtiger_ws_operation` and `vtiger_ws_operation_parameters` are used by `OperationManager` to resolve webservice operations.
3. `vtiger_ws_entity` and related metadata tables are used by webservice utilities (`include/Webservices/Utils.php`) to map module/entity types to handlers and IDs.
4. `vtiger_cron_task` is used by the cron subsystem (`vtlib/Vtiger/Cron.php` and `vtigercron.php`) to list and execute tasks.
5. `vtiger_eventhandlers` and `vtiger_eventhandler_module` are used by the event subsystem (see `vtlib/Vtiger/Event.php`) to register and enumerate event handlers.
6. `vtiger_relatedlists` is used by vtlib module relationship management (`vtlib/Vtiger/Module.php`) to define “related list” UI relationships between modules.

### 8.2 Core entity tables (cross-module linkage)

1. `vtiger_crmentity` is the central table that stores cross-module metadata such as ownership (`smownerid`), deleted flag, timestamps, and setype.
2. `vtiger_crmentityrel` is a generic relation table used by `CRMEntity::save_related_module()` and related-list implementations.
3. Attachment linkage is stored via `vtiger_attachments` and relation table `vtiger_seattachmentsrel`. File data itself is stored on disk; DB stores metadata including the path.

## 9. Subsystem adjacency lists (textual knowledge graph)

This section provides a compact “who depends on whom” list, grouped by subsystem. It is a useful complement to diagrams when you need a quick lookup.

### 9.1 Entrypoints

`index.php`
includes `include/utils/utils.php`, `config.inc.php`, `include/logging.php`, `modules/Users/Users.php`.
reads/writes `vtiger_version` (version check).
dispatches to `modules/<Module>/<Action>.php` (dynamic include).
invokes permission checks (`isPermitted`) before executing module action.

`install.php`
includes `include/install/resources/utils.php`, `vtigerversion.php`.
dispatches to `install/<step>.php` via `Common_Install_Wizard_Utils::checkFileAccessForInclusion`.

`vtigercron.php`
includes `vtlib/Vtiger/Cron.php`, `config.inc.php`.
reads/writes `vtiger_cron_task`.
includes handler files from `vtiger_cron_task.handler_file`.

`webservice.php`
includes `config.inc.php`, `include/Webservices/Utils.php`, `include/Webservices/OperationManager.php`, `include/Webservices/SessionManager.php`, `include/Zend/Json.php`.
reads/writes `vtiger_ws_operation`, `vtiger_ws_operation_parameters`.
loads operation handler path from DB and invokes it.

`vtigerservice.php`
includes `soap/*.php` based on `service` request parameter.

### 9.2 Platform layer

`data/CRMEntity.php`
includes `include/logging.php`, `include/utils/utils.php`, `include/utils/UserInfoUtil.php`, `include/events/include.inc`, `include/Zend/Json.php`.
reads/writes module tables listed in `$this->tab_name`.
reads/writes `vtiger_crmentity`, `vtiger_crmentityrel`, attachment tables.
triggers events via `VTEventsManager` during save and delete flows.

`include/database/PearDatabase.php`
includes `adodb/adodb.inc.php`.
provides `query`, `pquery`, `num_rows`, `query_result` and optional caching.
connects using values from `config.inc.php` (via global `$dbconfig` when constructed without explicit params).

`include/logging.php`
chooses between `log4php/` and `log4php.debug/` based on `config.performance.php`.
configures log4php via `log4php.properties`.

`vtlib/Vtiger/Module.php`
reads/writes module metadata tables such as `vtiger_tab` and relationship tables such as `vtiger_relatedlists`.
supports adding custom links and initializing webservice support (`Vtiger_Webservice::initialize`).

`vtlib/Vtiger/Cron.php`
reads/writes `vtiger_cron_task`, can create the table if missing.
registers/deregisters tasks.

`vtlib/Vtiger/Event.php`
registers handlers via `VTEventsManager`, queries `vtiger_eventhandlers` and module mapping tables.

### 9.3 Automation

`modules/com_vtiger_workflow/VTEventHandler.inc`
depends on events framework (`VTEventHandler`), workflow manager classes, and Webservices utilities.
evaluates workflows and performs tasks on save events.

Install-time automation registration (`install/CreateTables.inc.php`)
registers cron tasks (`Vtiger_Cron::register`).
registers event handlers via `VTEventsManager->registerHandler`, including workflow handler.

### 9.4 Modules (representative examples)

`modules/Users/Users.php`
extends `CRMEntity`.
uses `PearDatabase`, `include/Webservices/Utils.php` for generating access keys and ownership transfer.
reads/writes `vtiger_users` and related tables and writes “privileges flat files” (via `modules/Users/CreateUserPrivilegeFile.php`).

`modules/Leads/Leads.php`
extends `CRMEntity`.
implements module-specific related-list functions for Activities, Campaigns, Emails, Products.
uses module-specific relation tables such as `vtiger_seproductsrel` and `vtiger_campaignleadrel`.

## 10. Sources (evidence trail)

This knowledge graph is grounded in the following repository sources.

### 10.1 Runtime entrypoints

1. `index.php`
2. `install.php`
3. `vtigercron.php`
4. `webservice.php`
5. `vtigerservice.php`

### 10.2 Core platform components

1. `data/CRMEntity.php`
2. `include/database/PearDatabase.php`
3. `include/logging.php`
4. `include/utils/utils.php`
5. `vtlib/Vtiger/Module.php`
6. `vtlib/Vtiger/Cron.php`
7. `vtlib/Vtiger/Webservice.php`
8. `vtlib/Vtiger/Event.php`

### 10.3 Webservice stack

1. `include/Webservices/Utils.php`
2. `include/Webservices/OperationManager.php`
3. `include/Webservices/SessionManager.php`

### 10.4 Automation stack (events + workflows)

1. `include/events/include.inc`
2. `modules/com_vtiger_workflow/VTEventHandler.inc`

### 10.5 Install-time initialization

1. `install/CreateTables.inc.php`
2. `include/install/resources/utils.php`
3. `config.template.php`

### 10.6 Supporting repository docs used for alignment

1. `Docs/VTigerCRM-Architecture-Overview.md`
2. `Docs/VTigerCRM-Modules-MindMap.md`
