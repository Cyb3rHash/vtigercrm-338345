# vtiger CRM (v5.4.0) Architecture Overview

## Overview
This repository contains a monolithic, server-rendered PHP CRM application (vtiger CRM 5.4.0) that runs behind a PHP-capable web server and persists data in a relational database (typically MySQL). The system is organized as a classic vtiger/SugarCRM-style codebase where a central web entrypoint dispatches requests to module-specific PHP scripts under `modules/`, while shared framework services (database abstraction, security utilities, UI helpers, and webservice plumbing) live primarily under `include/`, `data/`, and `vtlib/`.

From an operational point of view, the system has three primary execution modes. It serves interactive browser requests via `index.php`, it exposes a JSON webservice endpoint via `webservice.php`, and it runs scheduled background jobs via `vtigercron.php` (plus individual scripts in `cron/`). Installation and initial database bootstrapping are handled via `install.php` and the `install/` directory.

## Architecture Diagram
At a high level, vtiger CRM can be understood as a layered monolith with multiple entrypoints that converge on shared framework services and a shared database.

A useful mental “diagram” is the following set of boxes and arrows:

1. A “Web Browser” box sends HTTP requests to a “Web Server + PHP runtime” box.
2. Inside the PHP runtime, requests are dispatched into one of these entrypoints: `index.php` (main UI), `webservice.php` (JSON API), `install.php` (installation wizard), `vtigercron.php` (scheduled jobs), and specialized entrypoints like `Popup.php` and `graph.php`.
3. Those entrypoints call into “Module Controllers” (scripts under `modules/<ModuleName>/...`) and “Shared Framework Services” (mostly under `include/`, `data/`, and `vtlib/`).
4. The framework services call the “Database Abstraction Layer” (`include/database/PearDatabase.php`, which wraps ADODB) which reads/writes the “CRM Database” (tables such as `vtiger_version`, webservice metadata tables like `vtiger_ws_operation`, and cron tables like `vtiger_cron_task`).
5. Static assets and UI templates flow through the templating subsystem (Smarty) and theme resources under `themes/`.

The key architectural point is that modules are not separate deployables; they are packages of PHP scripts and entity classes that are invoked by the central entrypoints and share the same runtime process, configuration, and database connection.

## Core Components
vtiger’s codebase is large, but the major architectural building blocks can be described in terms of entrypoints, routing/dispatch, module structure, shared framework services, persistence, and background execution.

### Web UI entrypoint and dispatcher (`index.php`)
The primary request dispatcher is `index.php`. It is responsible for starting the PHP session, verifying that the system is installed (by checking for `config.inc.php` and whether database configuration is initialized), checking the installed database version against the code version (`vtigerversion.php` + table `vtiger_version`), and enforcing access control before including the appropriate module action script.

In concrete terms, `index.php` reads `module` and `action` from the HTTP request and constructs a module action filepath such as `modules/<module>/<action>.php`. It includes additional special-case dispatch (for example, `Documents` + `DownloadFile`) and contains logic to decide whether to render standard headers/footers or treat the request as a popup/AJAX/download path.

This file therefore acts as both a front controller and a coarse-grained security and request-validation gate.

### Module layer (`modules/`)
The `modules/` directory contains the CRM’s functional areas such as Accounts, Contacts, Leads, Calendar, Documents, SalesOrder, PurchaseOrder, Reports, Settings, and many others. Each module typically includes:
1. One or more “action scripts” (`ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`, `Delete.php`, `*Ajax.php`) that implement controller-like behavior for that page/action.
2. A module “entity class” (for example `modules/Contacts/Contacts.php`) that represents the module’s domain object and extends the common entity base class.

From the samples in this repository, module entity classes commonly extend `CRMEntity` and implement module-specific behavior such as related list retrieval, export behavior, and module-specific data operations.

### Domain model base (`data/CRMEntity.php`)
`data/CRMEntity.php` provides the base type that module entity classes inherit from (for example, the Contacts module’s `Contacts` class and similar classes in other modules). This base typically encapsulates shared CRUD patterns, auditing hooks, and shared metadata needed by the vtiger runtime to treat modules uniformly (for example, instantiating module objects in `index.php` via `CRMEntity::getInstance($currentModule)`).

In the web UI flow, this base is a key part of the “model layer” that sits behind the module action scripts.

### Shared framework services (`include/`)
The `include/` directory holds shared, cross-module functionality. While the repository has many subareas, the architecture-critical parts are:

1. **Utility and security helpers** (`include/utils/utils.php` and other `include/utils/*` files). `index.php`, `Popup.php`, and other entrypoints require `include/utils/utils.php`, indicating it is one of the primary shared foundations for request handling, sanitization (for example, `vtlib_purify()` is used heavily), and general system utilities.
2. **Query building** (`include/QueryGenerator/QueryGenerator.php`), which supports building module queries in a structured way and is typically used by list views, filters, and reporting.
3. **List view framework** (`include/ListView/*`), which provides shared list-view rendering and behavior used across modules (for example, popups and list pages use list-view services).
4. **Webservices framework** (`include/Webservices/*`), described separately below.

In architectural terms, `include/` is the application “platform layer” used by all modules.

### Database access layer (`include/database/PearDatabase.php` + `adodb/`)
vtiger uses a database abstraction layer centered around `include/database/PearDatabase.php`, which wraps ADODB (`adodb/adodb.inc.php`) and provides commonly used APIs such as `query`, `pquery` (prepared statement execution), `num_rows`, and `query_result`. It also includes optional performance preferences support via `config.performance.php` and introduces optional in-process query result caching (`PearDatabaseCache`).

Most code interacts with the database through the global `$adb` object obtained from `PearDatabase::getInstance()` or initialized at the bottom of `PearDatabase.php`. This structure makes the database layer a shared singleton-like service across the monolith.

### Templating and UI rendering (Smarty)
The system uses Smarty templates and a vtiger-specific Smarty wrapper (for example, `Popup.php` uses `require_once('Smarty_setup.php')` and creates `new vtigerCRM_Smarty`). Templates are stored under `Smarty/templates/` and module templates exist under subfolders such as `Smarty/templates/modules/` and module-specific areas like `Smarty/templates/com_vtiger_workflow/`.

Themes and static assets are under `themes/`, and the entrypoint (`index.php`) includes standard headers/footers from `modules/Vtiger/header.php` and `modules/Vtiger/footer.php` for normal page requests.

### Webservice API entrypoint (`webservice.php`) and operation registry (`include/Webservices/*`)
`webservice.php` exposes a JSON-based API endpoint. Architecturally, it does not hardcode all operations directly; instead, it:
1. Loads system configuration (`config.inc.php`) and webservice utilities.
2. Reads the requested operation name from the request (parameter `operation`) and uses `OperationManager` (`include/Webservices/OperationManager.php`) to look up operation metadata in the database (`vtiger_ws_operation` and `vtiger_ws_operation_parameters`).
3. Starts or adopts a session via `SessionManager` and enforces authentication for non-prelogin operations.
4. Dynamically `require_once()`s the operation handler path (`handler_path` stored in `vtiger_ws_operation`) and invokes the registered handler method.
5. Returns JSON output in a consistent “State” wrapper (success/error).

This design makes the webservice layer metadata-driven. New operations can be registered in the database and mapped to handler implementations, while the runtime remains a stable dispatcher.

Several core operations (for example, login and query) are implemented in `include/Webservices/` files such as `Login.php`, `Query.php`, and `DescribeObject.php`, and use `VtigerWebserviceObject` (`include/Webservices/VtigerWebserviceObject.php`) to resolve entity metadata (`vtiger_ws_entity`) and instantiate handlers for entity types.

### Cron / scheduled execution (`vtigercron.php` + `vtlib/Vtiger/Cron.php` + `cron/`)
Background execution is driven by `vtigercron.php`, which can be run in a CLI context (or via an authenticated session with the correct application key). It loads `vtlib/Vtiger/Cron.php` and uses `Vtiger_Cron::listAllActiveInstances()` to retrieve enabled tasks from the database table `vtiger_cron_task`.

For each active cron task, `vtigercron.php` checks whether it is runnable based on frequency and last-run timestamps, marks it running, includes the task handler file (`handler_file`), and then marks it finished. `vtlib/Vtiger/Cron.php` is responsible for maintaining the cron task registry and schema, including creating `vtiger_cron_task` if it does not exist and providing registration/deregistration APIs.

In addition, the repository contains standalone cron scripts under `cron/` (for example, `cron/intimateTaskStatus.php`) which implement specific scheduled behaviors like sending reminders and notifications. The architecture therefore supports both “registered cron tasks” and “direct cron scripts,” with the registered approach being the preferred centralized mechanism.

### Installation entrypoint (`install.php` + `install/`)
Installation is initiated via `install.php`, which:
1. Ensures required PHP version constraints.
2. Loads installation language and utility helpers (for example, `include/install/resources/utils.php`).
3. Chooses which installation step file to include from `install/` based on request parameters, and validates safe inclusion via `Common_Install_Wizard_Utils::checkFileAccessForInclusion()`.

This makes installation a step-driven wizard controlled by files under `install/`.

### Extension and module lifecycle framework (`vtlib/`)
`vtlib/` is vtiger’s internal extension framework. For example, `vtlib/Vtiger/Module.php` exposes APIs to manage module relationships and links, and can initialize or de-initialize webservice support for a module via `Vtiger_Webservice::initialize()` / `uninitialize()`. This layer is what enables module import/export, module activation checks, and other system-level “meta” operations.

## Data Flow
### Interactive UI request (typical page view)
A standard browser request flows through the system as follows:
1. The browser sends an HTTP request with `module=<ModuleName>` and `action=<ActionName>` to `index.php`.
2. `index.php` starts the session, checks installation configuration, loads `config.inc.php`, validates module/action inputs, and verifies user authentication and permissions.
3. `index.php` includes the module action script (for example, `modules/Contacts/DetailView.php`).
4. The module action script uses shared utilities and a module entity instance (typically derived from `CRMEntity`) to query and manipulate data.
5. Database operations are executed via `$adb` (an instance of `PearDatabase`) which uses ADODB under the hood.
6. The module action assigns data to Smarty and renders templates, producing HTML output to the browser.

### Webservice request (JSON API)
A webservice request flows through the system as follows:
1. A client sends an HTTP request to `webservice.php` with `operation=<name>`, optional session identifier (for example, `sessionName`), and input parameters.
2. `webservice.php` uses `OperationManager` to resolve operation metadata stored in database tables such as `vtiger_ws_operation` and `vtiger_ws_operation_parameters`.
3. The session is started (or adopted for `extendsession`), and authentication is enforced for operations that are not marked as pre-login.
4. The handler implementation is loaded dynamically by path and invoked, returning structured results.
5. The response is encoded as JSON and returned with a consistent success/error envelope.

### Scheduled job run
A scheduled task run flows through the system as follows:
1. A scheduler calls `php vtigercron.php` (CLI) optionally with `?service=<TaskName>` to run a specific task.
2. `vtigercron.php` enumerates active tasks via `Vtiger_Cron::listAllActiveInstances()`.
3. For each runnable task, it loads the handler file defined in the database (`vtiger_cron_task.handler_file`) and executes it within the vtiger runtime context.
4. The cron runtime updates task state in the database by marking running/finished and tracking timestamps.

## Integration Points
vtiger CRM integrates with several internal and external components, mostly as bundled libraries within this repository.

1. The database is accessed via `include/database/PearDatabase.php`, which wraps the ADODB library in `adodb/`. The database schema and runtime metadata tables are core to operation resolution (webservice operations), cron task scheduling, and version checks (`vtiger_version`).
2. The JSON webservice layer uses Zend JSON (`include/Zend/Json.php`) as configured by `include/Webservices/OperationManager.php`.
3. Templating is handled via Smarty (directory `Smarty/`), with vtiger’s own wrapper used in entrypoints like `Popup.php`.
4. Scheduled tasks are registered and managed through `vtlib/Vtiger/Cron.php` and executed by `vtigercron.php`.
5. Optional remote content retrieval exists via the HTTP utility in `class_http/class_http.php` (useful for integrations that fetch external web content), though its usage depends on specific modules.
6. The repository also contains SOAP-related code under `soap/`, indicating legacy or alternative integration methods beyond the JSON webservice endpoint (the exact SOAP surface is not fully characterized here beyond file presence).

## Technology Stack
The main technologies evidenced in the repository are:
1. PHP (with runtime checks in `index.php` and `install.php` for PHP version compatibility).
2. ADODB for database abstraction (`adodb/`, used by `include/database/PearDatabase.php`).
3. Smarty templating engine (`Smarty/`), used for server-rendered UI composition.
4. Zend JSON (`include/Zend/Json.php`) used by the webservice stack.
5. vtlib (`vtlib/`) as the vtiger module/extension framework.
6. A variety of bundled third-party libraries and assets (for example, CKEditor under `include/ckeditor/`, HTMLPurifier under `include/htmlpurifier/`, and PDF tooling such as TCPDF under `tcpdf/`), which support rich text editing, sanitization, and document/PDF generation.

## Key Design Decisions
vtiger’s architecture reflects a set of consistent design decisions that shape how features are built and extended.

1. The system uses a single front controller (`index.php`) that dispatches into module action scripts by including PHP files. This keeps routing simple but tightly couples request handling to filesystem structure.
2. Business functionality is modularized by module directories under `modules/`, while shared infrastructure is centralized under `include/`, `data/`, and `vtlib/`. This reduces duplication across modules and allows vtiger to treat modules uniformly through base classes like `CRMEntity`.
3. The webservice API is metadata-driven. Operations are discovered from database tables (such as `vtiger_ws_operation`) and mapped to handler paths and methods at runtime through `OperationManager`. This makes the webservice surface extensible without changing the dispatcher.
4. Scheduled jobs are also metadata-driven. Cron tasks are stored in `vtiger_cron_task` and executed dynamically by `vtigercron.php` by loading handler files. This provides a centralized framework for scheduled work.
5. The database abstraction layer is centralized through a global `$adb` object and a wrapper class (`PearDatabase`). This enables uniform SQL execution and optional performance features (for example, query result caching controlled by performance preferences).

## Scalability & Performance
vtiger CRM is implemented as a single PHP application and is therefore typically scaled horizontally by running multiple PHP worker processes behind a load balancer, with a shared database and shared storage for uploads/attachments.

Performance-related features evidenced in this repository include:
1. A dedicated performance configuration file (`config.performance.php`) that contains toggles affecting list view computation, record navigation behavior, database charset optimizations, and other runtime settings.
2. `PearDatabase` includes performance-oriented behaviors such as optional query result caching (`PearDatabaseCache`) and the ability to skip `SET NAMES utf8` if the database default charset is already UTF-8 (`DB_DEFAULT_CHARSET_UTF8`).
3. List view performance can be influenced by whether page count is computed on each load (`LISTVIEW_COMPUTE_PAGE_COUNT`), which can become expensive on large datasets; the configuration shows this is intended as a tunable knob.

Because the repository does not include deployment descriptors in the files examined here (for example, web server config or process manager configuration), the exact recommended scaling topology is not fully determined from current sources. However, the code and configuration patterns clearly anticipate performance tuning primarily via database efficiency and page-level behaviors rather than microservice decomposition.

## Security Considerations
Security in vtiger CRM is implemented in multiple layers, with notable controls visible in the primary dispatcher and in the webservice endpoint.

1. `index.php` includes explicit request validation checks, including path traversal defenses by verifying that `module` is a real directory under `modules/` and that the requested action file exists, and by rejecting module/action strings containing path separators. It also performs permission checks via `isPermitted()` before including module action scripts.
2. Authentication is session-based for the web UI. If there is no authenticated user in the session, `index.php` forces routing to the login action and includes `modules/Users/Login.php`.
3. The JSON webservice endpoint (`webservice.php`) enforces authentication by requiring a valid session for non-prelogin operations and by validating login/token behavior through code in `include/Webservices/Login.php`.
4. Sanitization utilities such as `vtlib_purify()` appear throughout entrypoints (for example, `Popup.php`) and are part of the shared `include/utils/utils.php` toolbox, indicating a consistent strategy of input cleaning before use.
5. Cron execution (`vtigercron.php`) contains an access gate: it allows execution via CLI or via a session that includes a matching application unique key, reducing the risk of unauthorized web-triggered cron runs.

There are additional security implications not fully verifiable from the limited files analyzed (for example, how file uploads are validated and stored, how secrets are managed in `config.inc.php`, and how webservice permissions are enforced across all operations). Those details would typically be confirmed by reviewing `config.inc.php`, module upload handlers, and the full webservice handler set.
