# LOW LEVEL DESIGN

Component / Module Name: vtiger CRM Core Runtime (UI Dispatcher, Webservices, Cron, Entity Framework)  
Parent System: vtiger CRM (v5.4.0)  
HLD Reference: `../ArchitectureSpecs/vtigercrm-high-level-design-hld.md`  
Component ID: vtigercrm-core-runtime  
Author: [To be determined]  
Version: 1.0  
Date: 2026-04-07  

IEEE 1016-2009 · OpenAPI 3.1 · AsyncAPI 3.0 · IEEE 829  

Confidentiality notice: This document contains implementation-level design details for a legacy vtiger CRM codebase. Distribution and access should be controlled according to organisational policy.

## 1. Document Control

### 1.1 Version History

| Version | Date | Author | Change Summary |
|---|---|---|---|
| 1.0 | 2026-04-07 | [To be determined] | Regenerated LLD from repository evidence, following `skills/dynamic/low_level_design_lld_generator.txt`. |

### 1.2 Review & Approvals

| Name | Role | Status | Date |
|---|---|---|---|
| [To be determined] | Tech Lead / Senior Engineer | [To be determined] | [To be determined] |
| [To be determined] | Security Engineer | [To be determined] | [To be determined] |
| [To be determined] | QA Lead | [To be determined] | [To be determined] |

### 1.3 Standards Conformance

| Standard | Version | Application |
|---|---|---|
| ISO/IEC/IEEE 1016 (Software Design Description) | 2009 | Provides a structured, implementation-oriented design description for the component. |
| OpenAPI | 3.1 | The repository does not include an OpenAPI document; Section 8 documents the observable HTTP interface of `webservice.php` and its operation routing model. |
| AsyncAPI | 3.0 | No message broker/event bus is evidenced; asynchronous processing is cron-driven (Section 6.3 and Section 8.3 document this as schedule-driven execution rather than an AsyncAPI channel). |
| IEEE 829 (Test Documentation) | [To be determined] | Section 15 captures test coverage expectations and scenarios, noting gaps where no automated suite is evidenced. |

### 1.4 Related Documents

| Document | Reference | Version | Relationship |
|---|---|---|---|
| High Level Design (HLD) | `../ArchitectureSpecs/vtigercrm-high-level-design-hld.md` | 1.0 | Parent architecture document that this LLD refines. |
| API documentation | `../../../../Docs/API_Documentation.md` | [To be determined] | Behavioral documentation for `webservice.php` operations and SOAP endpoints. |
| Data model extraction | `../../../../Docs/data_model.md` | [To be determined] | Database tables and relationship patterns (used in Section 7). |
| Legacy Leads module LLD | `../../../../Docs/LLd.md` | [To be determined] | Example of module-level LLD patterns in this repo; complements this “core runtime” LLD. |
| System architecture overview | `../../../../Docs/VTigerCRM-Architecture-Overview.md` | [To be determined] | Additional architectural context; not a substitute for this component-level LLD. |

## 2. Introduction

### 2.1 Purpose

This Low Level Design (LLD) provides the detailed technical specification for the component as defined in the HLD. It focuses on the vtiger CRM “core runtime” concerns: UI request dispatch, JSON Webservices dispatch, cron execution, and the shared entity persistence lifecycle.

### 2.2 Scope

In Scope: This LLD covers the runtime entrypoints and framework classes that are directly evidenced as core execution paths, including `index.php`, `webservice.php`, `vtigercron.php`, `vtigerservice.php`, `data/CRMEntity.php`, `include/Webservices/OperationManager.php`, `include/Webservices/SessionManager.php`, and `vtlib/Vtiger/Cron.php`. It also covers configuration and logging evidence that shapes runtime behavior (`config.template.php`, `log4php.properties`).

Out of Scope: This LLD does not provide low-level design coverage for individual CRM business modules under `modules/*` (except where module dispatch is referenced), and it does not enumerate the full database schema in `schema/DatabaseSchema.xml` because that schema is large and the focus here is on tables directly used by the core runtime flows.

### 2.3 Intended Audience

| Audience | Purpose | Sections of Primary Interest |
|---|---|---|
| Backend / PHP engineers | Understand the concrete dispatch, session, persistence, and cron execution logic. | 3, 4, 6, 7, 8, 11, 12 |
| Integration engineers | Understand the operation-based JSON API entrypoint and session model. | 8, 11, 12 |
| Security engineers | Review safe dynamic include controls, session rules, and file upload handling. | 11, 12, 16 |
| QA engineers | Derive verification scenarios for UI/API/cron and persistence behaviors. | 6, 8, 11, 15 |
| Operations / SRE | Understand cron execution and logging configuration and deployment dependencies. | 10, 13, 16 |

### 2.4 Design Constraints

| ID | Type | Constraint | Impact |
|---|---|---|---|
| CON-001 | Platform | UI execution requires PHP >= 5.2.0 (enforced in `index.php`). | Limits supported hosting/runtime environments. |
| CON-002 | Configuration | Runtime configuration is file-based (generated `config.inc.php` and optional `config_override.php`), not environment-variable driven (as evidenced by `config.template.php`). | Deployment pipelines must manage file generation and secrets without relying on env vars alone. |
| CON-003 | Security | UI routing depends on dynamic file inclusion (`modules/<Module>/<Action>.php`), requiring strict validation and include guards (`index.php` checks module/action existence). | Introduces risk if validation is bypassed; increases need for secure coding/review. |
| CON-004 | Security | Webservice operations and handlers are resolved via DB metadata tables (`vtiger_ws_operation`, `vtiger_ws_operation_parameters`) as implemented by `OperationManager`. | Operational misconfiguration can enable/disable operations or break dispatch. |
| CON-005 | Reliability | Entity persistence uses DB transactions in `CRMEntity::saveentity()` (transaction start/complete). | DB failures can fail whole save; partial persistence is avoided but failures propagate. |
| CON-006 | Scalability | Attachments are written to filesystem locations (via `decideFilePath()` and `CRMEntity::uploadAndSaveFile()`). | Multi-node deployments require shared storage or consistency strategy for attachments. |

## 3. Component Overview

### 3.1 Component Identity

| Attribute | Detail |
|---|---|
| ID | vtigercrm-core-runtime |
| Name | vtiger CRM Core Runtime |
| Type | Backend runtime subsystem (monolithic PHP) |
| Tech Stack | PHP, ADODB/PearDatabase (DB), Smarty (UI templates), log4php (logging), Zend_Json (webservice encoding/decoding), HTMLPurifier (input purification via `vtlib_purify`, referenced in HLD). |
| Primary Entrypoints | `index.php`, `webservice.php`, `vtigercron.php`, `vtigerservice.php` |
| Primary Framework Classes | `CRMEntity` (`data/CRMEntity.php`), `OperationManager` (`include/Webservices/OperationManager.php`), `SessionManager` (`include/Webservices/SessionManager.php`), `Vtiger_Cron` (`vtlib/Vtiger/Cron.php`) |
| Data Stores | Relational DB tables such as `vtiger_crmentity`, `vtiger_ws_operation`, `vtiger_ws_operation_parameters`, `vtiger_cron_task`, and filesystem storage for attachments (`storage/` by default). |
| Owner Team | [To be determined] |
| Repository Location | `vtigercrm-338345/` |

### 3.2 Component Context (C4 Level 3)

This component sits inside the vtiger CRM monolith and provides the core runtime mechanics that all CRM modules depend on: dispatching requests, creating/loading module entity instances, persisting records with a consistent lifecycle, serving webservice API operations, and executing scheduled background tasks.

Diagram prompt: [Insert C4 Level 3 component context diagram for “vtiger CRM Core Runtime” within the vtiger monolith, showing Browser/API client/Scheduler interactions and dependencies on DB and filesystem.]

### 3.3 Responsibilities

The core runtime is responsible for enforcing consistent execution and safety boundaries across the system. It validates module/action routing and permission checks in `index.php`, enforces webservice sessions and operation dispatch in `webservice.php` and `OperationManager`, executes cron tasks defined in the database using `vtigercron.php` and `Vtiger_Cron`, and provides shared persistence and attachment handling via the `CRMEntity` base class.

### 3.4 Non-Responsibilities (Explicitly Excluded)

This component does not implement business workflows for individual CRM modules (for example, Leads conversion, Accounts hierarchy, or Ticket workflows). Those behaviors live primarily under `modules/<Module>/*` and are out of scope except where they are invoked by the dispatch/persistence framework. This component also does not provide a formal REST resource model; the primary integration API is an operation-based dispatcher (`webservice.php`) whose operations are controlled by database metadata.

## 4. Class / Module Design

### 4.1 Class Diagram

UML class diagram prompt: [Insert UML class diagram for CRMEntity, OperationManager, SessionManager, Vtiger_Cron and show primary relationships (composition/usage) between entry scripts and these classes.]

### 4.2 Package / Namespace Structure

| Package / Namespace | Responsibility / Contents |
|---|---|
| `index.php` | UI front controller: session start, install/version guards, module/action validation, auth gating, permission checks (`isPermitted`), includes module action file. |
| `modules/*` | Module action scripts (controllers) and module entity classes; invoked by `index.php` and often extend `CRMEntity`. |
| `data/CRMEntity.php` | Base class `CRMEntity` implementing persistence transaction, event triggers, soft delete, restore, relationship linking, attachments. |
| `webservice.php` | JSON webservice entrypoint: reads `operation`, starts session, loads operation handler includes, executes handler, returns `State` envelope JSON. |
| `include/Webservices/OperationManager.php` | `OperationManager` class: loads operation metadata from `vtiger_ws_operation`, parameters from `vtiger_ws_operation_parameters`, sanitizes/decodes inputs, includes handler path, invokes handler method. |
| `include/Webservices/SessionManager.php` | `SessionManager` class: cookie-less session start, expiry/idle/invalid checks using `HTTP_Session`. |
| `vtigercron.php` | Cron dispatcher: lists active tasks and executes runnable ones via handler inclusion. |
| `vtlib/Vtiger/Cron.php` | `Vtiger_Cron` class: DB-backed cron registry and task state transitions; creates `vtiger_cron_task` table if missing. |
| `vtigerservice.php` | SOAP dispatcher that includes `soap/*.php` based on `service` request parameter. |
| `config.template.php` | Template for runtime config file; defines DB config keys and settings like `$application_unique_key`. |
| `log4php.properties` | log4php logger/appender configuration and output files under `logs/`. |

### 4.3 Class Specifications

#### 4.3.1 CRMEntity (Base entity lifecycle and persistence)

Attributes:

| Attribute | Detail |
|---|---|
| Type | Base class |
| Package | `data/CRMEntity.php` |
| Responsibility | Shared persistence lifecycle, transactions, attachment storage, soft delete, restore, related module linking, event triggers on save/delete. |
| SOLID Principle | Single Responsibility is partially applied (persistence + attachments + relationship management); Open/Closed is applied via module subclasses overriding behavior and by using metadata-driven fields/tables. |

Properties/Fields:

| Field Name | Type | Visibility | Description/Constraints |
|---|---|---|---|
| `column_fields` | array | public (var) | Field-value map; used as source of persistence. |
| `tab_name` | array | public (var) | List of tables for the module; used by `saveentity()` to persist. |
| `tab_name_index` | array | public (var) | Table-to-primary-key mapping used by `insertIntoEntityTable()` and retrieval. |
| `id` | int|string | public (var) | Current entity ID (`vtiger_crmentity.crmid`) after insert/update. |
| `mode` | string | public (var) | `edit` triggers update path; empty triggers insert path. |

Methods:

| Method Signature | Return Type | Visibility | Throws | Description |
|---|---|---|---|---|
| `static getInstance($module)` | `CRMEntity` | public | [To be determined] | Loads `modules/<Module>/<Class>.php` (with include guard) and instantiates module entity class. |
| `save($module_name, $fileid='')` | void | public | [To be determined] | Triggers entity events (`vtiger.entity.beforesave*`, `vtiger.entity.aftersave*`) and delegates to `saveentity()`. |
| `saveentity($module, $fileid='')` | void | public | [To be determined] | Starts DB transaction, inserts/updates `vtiger_crmentity` and module tables listed in `tab_name`. |
| `uploadAndSaveFile($id, $module, $file_details)` | bool | public | [To be determined] | Saves uploaded file to filesystem (`decideFilePath()`) and inserts metadata into `vtiger_attachments` + relation in `vtiger_seattachmentsrel`. |
| `mark_deleted($id)` | void | public | [To be determined] | Soft deletes a record by updating `vtiger_crmentity.deleted=1` and modified fields. |
| `restore($module, $id)` | void | public | [To be determined] | Restores soft-deleted record (`deleted=0`) and triggers `vtiger.entity.afterrestore`. |
| `retrieve_entity_info($record, $module)` | void | public | [To be determined] | Loads entity field values from each module table and populates `column_fields`; dies if record is deleted/not found. |
| `save_related_module($module, $crmid, $with_module, $with_crmid)` | void | public | [To be determined] | Inserts relations into `vtiger_senotesrel` for Documents or `vtiger_crmentityrel` for other modules (skipping duplicates). |

#### 4.3.2 OperationManager (Webservice operation routing)

Attributes:

| Attribute | Detail |
|---|---|
| Type | Dispatcher helper |
| Package | `include/Webservices/OperationManager.php` |
| Responsibility | Resolve operation metadata from DB tables, sanitize/decode input, load handler includes, call handler method, encode output. |
| SOLID Principle | Single Responsibility: encapsulates operation routing logic for `webservice.php`; Dependency Inversion is limited (direct DB access). |

Properties/Fields:

| Field Name | Type | Visibility | Description/Constraints |
|---|---|---|---|
| `format` | string | private | Normalized format (only `json` evidenced). |
| `pearDB` | object | private | DB handle `$adb` used for metadata queries. |
| `operationName` | string | private | Operation name (lowercased by caller `webservice.php`). |
| `type` | string | private | Operation input source selection (GET/POST/REQUEST) from `vtiger_ws_operation.type`. |
| `handlerPath` | string | private | Include path to handler file from `vtiger_ws_operation.handler_path`. |
| `handlerMethod` | string | private | Handler function name from `vtiger_ws_operation.handler_method`. |
| `preLogin` | int | private | Whether operation is pre-login (`vtiger_ws_operation.prelogin`). |
| `operationId` | int | private | Operation ID (`vtiger_ws_operation.operationid`). |
| `operationParams` | array | private | Parameter names and types from `vtiger_ws_operation_parameters`. |

Methods:

| Method Signature | Return Type | Visibility | Throws | Description |
|---|---|---|---|---|
| `OperationManager($adb, $operationName, $format, $sessionManager)` | void | public | `WebServiceException` | Initializes supported formats (Zend_Json), loads operation metadata and parameters. |
| `isPreLoginOperation()` | bool | public | [To be determined] | Returns whether the operation is configured as pre-login. |
| `getOperationInput()` | array | public | [To be determined] | Selects input source based on operation type (`GET`, `POST`, else `REQUEST`). |
| `sanitizeOperation($input)` | array | public | [To be determined] | Returns sanitized parameter array according to parameter types (notably `encoded` JSON decode). |
| `getOperationIncludes()` | array | public | [To be determined] | Returns list of include paths (currently only handler path). |
| `runOperation($params, $user)` | mixed | public | `WebServiceException` | Calls handler function. For pre-login operations, sets `authenticatedUserId` and returns session envelope including API version. |
| `encode($param)` | string | public | [To be determined] | Encodes object using Zend_Json encoder. |

#### 4.3.3 SessionManager (Webservice sessions)

Attributes:

| Attribute | Detail |
|---|---|
| Type | Session manager |
| Package | `include/Webservices/SessionManager.php` |
| Responsibility | Manage cookie-less HTTP_Session lifecycle for webservice calls, including expiry and idle checks. |
| SOLID Principle | Single Responsibility: encapsulates session checks for `webservice.php`. |

Properties/Fields:

| Field Name | Type | Visibility | Description/Constraints |
|---|---|---|---|
| `maxLife` | int | private | Session expiration timestamp (now + 86400 seconds). |
| `idleLife` | int | private | Session idle expiration timestamp (now + 1800 seconds). |
| `sessionVar` | string | private | Marker key `__SessionExists` used to validate non-new sessions. |

Methods:

| Method Signature | Return Type | Visibility | Throws | Description |
|---|---|---|---|---|
| `SessionManager()` | void | public | [To be determined] | Disables cookies (`HTTP_Session::useCookies(false)`), sets expire/idle windows. |
| `startSession($sid=null, $adoptSession=false)` | string|null | public | `WebServiceException` | Starts session with provided ID (or new), sets marker, validates session, returns active session id. |
| `isValid()` | bool | public | `WebServiceException` | Throws on expired/idle/invalid session id and destroys session. |
| `getSessionId()` | string | public | [To be determined] | Returns `HTTP_Session::id()`. |
| `set($var_name, $var_value)` | void | public | [To be determined] | Stores session var. |
| `get($name)` | mixed | public | [To be determined] | Retrieves session var. |
| `destroy()` | void | public | [To be determined] | Destroys HTTP session. |

#### 4.3.4 Vtiger_Cron (Cron registry and state machine)

Attributes:

| Attribute | Detail |
|---|---|
| Type | Cron registry / task instance |
| Package | `vtlib/Vtiger/Cron.php` |
| Responsibility | Persist cron task configuration/state in `vtiger_cron_task`, determine runnability by frequency and timestamps, mark running/finished. |
| SOLID Principle | Single Responsibility is mostly applied for cron registry operations; schema initialization is also included. |

Properties/Fields:

| Field Name | Type | Visibility | Description/Constraints |
|---|---|---|---|
| `data` | array | protected | Row from `vtiger_cron_task` describing the task. |
| `bulkMode` | bool | protected | Bulk mode flag that handlers may check (`setBulkMode`). |
| `STATUS_DISABLED` | int | public static | 0 |
| `STATUS_ENABLED` | int | public static | 1 |
| `STATUS_RUNNING` | int | public static | 2 |

Methods:

| Method Signature | Return Type | Visibility | Throws | Description |
|---|---|---|---|---|
| `listAllActiveInstances($byStatus=0)` | array | public static | [To be determined] | Lists tasks with `status <> STATUS_DISABLED` ordered by `sequence`. |
| `getInstance($name)` | `Vtiger_Cron`|false | public static | [To be determined] | Loads a task by name and caches it. |
| `isRunnable()` | bool | public | [To be determined] | Uses last end or last start, checks `elapsedTime >= frequency` when not disabled. |
| `markRunning()` | self | public | [To be determined] | Updates DB: `status=RUNNING`, `laststart=time()`, `lastend=0`. |
| `markFinished()` | self | public | [To be determined] | Updates DB: `status=ENABLED`, `lastend=time()`. |
| `hadTimedout()` | int|null | public | [To be determined] | Detects last run started but never finished (`lastend===0 && laststart!=0`). |
| `register($name, $handler_file, $frequency, $module='Home', $status=1, $sequence=0, $description='')` | void | public static | [To be determined] | Inserts into `vtiger_cron_task` (after creating table if missing). |

## 5. State Machine Design

### 5.1 Entity State Machines

State diagram prompt: [Insert state diagram for `vtiger_cron_task.status` transitions (Disabled, Enabled, Running) and timeout handling.]

States:

| State | Description | Entry Action / Exit Action |
|---|---|---|
| Disabled | Task is not executed by `vtigercron.php` when listing active instances. | Entry: `updateStatus(STATUS_DISABLED)`; Exit: `updateStatus(STATUS_ENABLED)`. |
| Enabled | Task is eligible to run when `isRunnable()` is true. | Entry: `markFinished()` sets Enabled; Exit: `markRunning()` sets Running. |
| Running | Task is currently executing; `lastend` is set to 0. | Entry: `markRunning()`; Exit: `markFinished()` or operator intervention on failure. |

Transitions:

| From State | To State | Trigger | Guard Condition | Action |
|---|---|---|---|---|
| Enabled | Running | Cron dispatcher begins executing task | `isRunnable() == true` | `markRunning()`, include handler file, then `markFinished()`. |
| Running | Enabled | Task handler completes | No exception thrown in `vtigercron.php` loop | `markFinished()`. |
| Running | Enabled | Task restarts after timeout | `hadTimedout()` indicates not completed | `markRunning()` again before including handler. |
| Any | Disabled | Admin disables task | [To be determined] | `updateStatus(STATUS_DISABLED)` updates DB. |
| Disabled | Enabled | Admin enables task | [To be determined] | `updateStatus(STATUS_ENABLED)` updates DB. |

Invalid Transitions:

A transition from Disabled directly to Running is not performed by the standard cron dispatcher path because Disabled tasks are filtered out by `listAllActiveInstances()` when `byStatus==0`. A transition from Enabled to Enabled without updating timestamps does not represent a meaningful execution and is not part of the normal lifecycle.

## 6. Sequence Diagrams

### 6.1 Primary Flow

Sequence diagram prompt: [Insert sequence diagram for a normal authenticated `webservice.php` operation call and response.]

Step-by-step narrative (Webservices request):

1. The client calls `webservice.php` with `operation=<name>`, optional `format=json`, and `sessionName=<sid>` (for non-prelogin operations).
2. `webservice.php` reads and lowercases the operation, creates `SessionManager` and `OperationManager`.
3. `OperationManager` looks up operation metadata from `vtiger_ws_operation` and parameter metadata from `vtiger_ws_operation_parameters`.
4. `webservice.php` starts a session via `SessionManager::startSession()`.
5. If the operation is not prelogin and there is no `sessionName`, `webservice.php` returns an error `State` envelope.
6. `webservice.php` requires the operation handler file given by `OperationManager::getOperationIncludes()` and calls `OperationManager::runOperation(...)`.
7. The handler returns raw output which `webservice.php` wraps in `State(success=true, result=...)` and returns as JSON with `Content-type: application/json`.

### 6.2 Error / Degraded Mode Flow

Sequence diagram prompt: [Insert sequence diagram for an unauthenticated call to a non-prelogin operation returning an error envelope.]

Step-by-step narrative (Auth required failure):

1. The client calls `webservice.php` for an operation that requires authentication without providing `sessionName`.
2. After session start attempt, `webservice.php` checks `if(!$sessionId && !$operationManager->isPreLoginOperation())`.
3. The system returns `State(success=false, error={code: AUTHENTICATION_REQUIRED, message: "Authencation required"})` (as a `WebServiceException`), encoded via Zend_Json.

### 6.3 Async / Event-Driven Flow

Sequence diagram prompt: [Insert sequence diagram for `vtigercron.php` executing a runnable cron task instance.]

Step-by-step narrative (Cron execution):

1. A scheduler executes `php vtigercron.php` (CLI path is permitted by `vtigercron.php` gate `PHP_SAPI === "cli"`).
2. `vtigercron.php` lists tasks from the DB via `Vtiger_Cron::listAllActiveInstances()`.
3. For each task, it checks runnability via `Vtiger_Cron::isRunnable()` based on `frequency` and `lastend/laststart`.
4. If runnable, it calls `markRunning()`, validates file access (`checkFileAccess`), `require_once` includes the handler file, then calls `markFinished()`.
5. If the handler throws an exception, `vtigercron.php` catches it and prints an error message; the task state may remain Running until the next cycle detects timeout (based on timestamps).

## 7. Data Model

### 7.1 Entity Relationship Diagram

Physical ER diagram prompt: [Insert ER diagram for tables used by core runtime: vtiger_crmentity, vtiger_attachments, vtiger_seattachmentsrel, vtiger_ws_operation, vtiger_ws_operation_parameters, vtiger_cron_task, vtiger_version.]

### 7.2 Table / Entity Definitions

#### 7.2.1 `vtiger_cron_task`

Purpose: Stores cron task configuration and runtime state for tasks executed by `vtigercron.php` and managed by `Vtiger_Cron`. The schema is created programmatically by `Vtiger_Cron::initializeSchema()` if missing.

Column definitions:

| Column | Data Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | INT | NOT NULL, PRIMARY KEY, AUTO_INCREMENT | [To be determined] | Unique task id. |
| `name` | VARCHAR(100) | UNIQUE KEY | [To be determined] | Stable task name used to find instance by name. |
| `handler_file` | VARCHAR(100) | UNIQUE KEY | [To be determined] | PHP handler include path executed by cron. |
| `frequency` | int | [To be determined] | [To be determined] | Minimum time between runs (seconds) used by `isRunnable()`. |
| `laststart` | long | [To be determined] | [To be determined] | Unix timestamp when last run started. |
| `lastend` | long | [To be determined] | [To be determined] | Unix timestamp when last run ended; set to 0 while running. |
| `status` | int | [To be determined] | [To be determined] | 0 Disabled, 1 Enabled, 2 Running (`Vtiger_Cron::$STATUS_*`). |
| `module` | VARCHAR(100) | [To be determined] | [To be determined] | Module owning the task (logical grouping). |
| `sequence` | int | [To be determined] | [To be determined] | Execution ordering. |
| `description` | TEXT | [To be determined] | [To be determined] | Human-readable description. |

Indexes:

| Index Name | Columns Included | Type / Purpose |
|---|---|---|
| PRIMARY | `id` | Primary key. |
| UNIQUE (name) | `name` | Prevent duplicate task names. |
| UNIQUE (handler_file) | `handler_file` | Prevent duplicate handler file entries. |

Triggers: No triggers are evidenced in the inspected code.

Relationships: No foreign keys are evidenced in the schema creation statement in `Vtiger_Cron::initializeSchema()`.

#### 7.2.2 `vtiger_ws_operation`

Purpose: Stores webservice operation definitions used by `OperationManager::fillOperationDetails($operationName)` to determine how to route an operation and which handler to call.

Column definitions:

| Column | Data Type | Constraints | Default | Description |
|---|---|---|---|---|
| `operationid` | [To be determined] | [To be determined] | [To be determined] | Operation primary key referenced by `vtiger_ws_operation_parameters.operationid`. |
| `name` | [To be determined] | [To be determined] | [To be determined] | Operation name matched by `webservice.php` `operation` parameter (lowercased). |
| `type` | [To be determined] | [To be determined] | [To be determined] | Determines whether inputs are read from `$_GET`, `$_POST`, or `$_REQUEST`. |
| `handler_path` | [To be determined] | [To be determined] | [To be determined] | PHP file path included by `webservice.php`. |
| `handler_method` | [To be determined] | [To be determined] | [To be determined] | Callable function invoked by `OperationManager::runOperation`. |
| `prelogin` | [To be determined] | [To be determined] | [To be determined] | Flag indicating prelogin operations (e.g., `login`) that can run without `sessionName`. |

Indexes:

| Index Name | Columns Included | Type / Purpose |
|---|---|---|
| [To be determined] | [To be determined] | [To be determined] |

Triggers: No triggers are evidenced in the inspected code.

Relationships: Has a one-to-many relationship to `vtiger_ws_operation_parameters` via `operationid` (used by `OperationManager::fillOperationParameters()`).

#### 7.2.3 `vtiger_ws_operation_parameters`

Purpose: Stores the ordered parameter list and type information for each operation, which drives `OperationManager::sanitizeInputForType()` and decoding of `encoded` parameters.

Column definitions:

| Column | Data Type | Constraints | Default | Description |
|---|---|---|---|---|
| `operationid` | [To be determined] | [To be determined] | [To be determined] | Foreign key-like reference to `vtiger_ws_operation.operationid` (used in query). |
| `sequence` | [To be determined] | [To be determined] | [To be determined] | Parameter ordering used by `ORDER BY sequence`. |
| `name` | [To be determined] | [To be determined] | [To be determined] | Parameter name to extract from input and pass to handler. |
| `type` | [To be determined] | [To be determined] | [To be determined] | Parameter type; `encoded` triggers JSON decoding. |

Indexes:

| Index Name | Columns Included | Type / Purpose |
|---|---|---|
| [To be determined] | [To be determined] | [To be determined] |

Triggers: No triggers are evidenced in the inspected code.

Relationships: Many-to-one to `vtiger_ws_operation` via `operationid`.

#### 7.2.4 `vtiger_crmentity`

Purpose: Central table for most CRM records. `CRMEntity` inserts/updates this table during `saveentity()` and uses it for soft delete via `mark_deleted()`.

Column definitions:

| Column | Data Type | Constraints | Default | Description |
|---|---|---|---|---|
| `crmid` | [To be determined] | [To be determined] | [To be determined] | Primary entity ID shared by module tables. |
| `smcreatorid` | [To be determined] | [To be determined] | [To be determined] | Creator user ID. |
| `smownerid` | [To be determined] | [To be determined] | [To be determined] | Owner user/group ID. |
| `setype` | [To be determined] | [To be determined] | [To be determined] | Module name or subtype label. |
| `description` | [To be determined] | [To be determined] | [To be determined] | Description field used by `insertIntoCrmEntity()`. |
| `modifiedby` | [To be determined] | [To be determined] | [To be determined] | User ID of last modifier. |
| `createdtime` | [To be determined] | [To be determined] | [To be determined] | Creation timestamp. |
| `modifiedtime` | [To be determined] | [To be determined] | [To be determined] | Modification timestamp. |
| `deleted` | [To be determined] | [To be determined] | [To be determined] | Soft delete flag set by `mark_deleted()`. |
| `viewedtime` | [To be determined] | [To be determined] | [To be determined] | Updated by `markAsViewed()` for record view notification. |

Indexes:

| Index Name | Columns Included | Type / Purpose |
|---|---|---|
| [To be determined] | [To be determined] | [To be determined] |

Triggers: No triggers are evidenced in inspected code.

Relationships: Module base tables typically reference `vtiger_crmentity.crmid` as their primary key (pattern described in `Docs/data_model.md`).

#### 7.2.5 `vtiger_attachments` and `vtiger_seattachmentsrel`

Purpose: Persist attachment metadata and link it to CRM records. Used by `CRMEntity::uploadAndSaveFile()`.

`vtiger_attachments` column definitions:

| Column | Data Type | Constraints | Default | Description |
|---|---|---|---|---|
| `attachmentsid` | [To be determined] | [To be determined] | [To be determined] | Attachment ID (also a `vtiger_crmentity.crmid` row with setype like “<Module> Attachment”). |
| `name` | [To be determined] | [To be determined] | [To be determined] | Original filename (sanitized). |
| `description` | [To be determined] | [To be determined] | [To be determined] | Description from entity column_fields. |
| `type` | [To be determined] | [To be determined] | [To be determined] | MIME type. |
| `path` | [To be determined] | [To be determined] | [To be determined] | Filesystem path prefix returned by `decideFilePath()`. |

`vtiger_seattachmentsrel` column definitions:

| Column | Data Type | Constraints | Default | Description |
|---|---|---|---|---|
| `crmid` | [To be determined] | [To be determined] | [To be determined] | CRM record ID that owns the attachment (entity id or notes id). |
| `attachmentsid` | [To be determined] | [To be determined] | [To be determined] | Attachment id referencing `vtiger_attachments.attachmentsid`. |

Indexes:

| Index Name | Columns Included | Type / Purpose |
|---|---|---|
| [To be determined] | [To be determined] | [To be determined] |

Triggers: No triggers are evidenced in inspected code.

Relationships: `vtiger_seattachmentsrel` links an entity record (`crmid`) to an attachment (`attachmentsid`). Attachment metadata and the physical file location must be consistent for downloads to work.

#### 7.2.6 `vtiger_version`

Purpose: Stores DB version `current_version` used by `index.php` to block UI access when code and DB versions mismatch.

Column definitions:

| Column | Data Type | Constraints | Default | Description |
|---|---|---|---|---|
| `current_version` | [To be determined] | [To be determined] | [To be determined] | DB schema/application version used for migration gating. |

Indexes:

| Index Name | Columns Included | Type / Purpose |
|---|---|---|
| [To be determined] | [To be determined] | [To be determined] |

Triggers: No triggers are evidenced in inspected code.

Relationships: Conceptually related to code version `vtigerversion.php` but no FK.

## 8. API Specifications

### 8.1 API Overview

| Attribute | Detail |
|---|---|
| API Style | Operation-based (RPC-like) JSON API. |
| Base URL | `webservice.php` |
| Auth | Session-based using `sessionName`. Prelogin operations are permitted when `vtiger_ws_operation.prelogin=1`. |
| Versioning | `$API_VERSION = "0.22"` in `webservice.php`; vtiger version retrieved via `vtws_getVtigerVersion()`. |
| Request Formats | Input source depends on `vtiger_ws_operation.type`: GET uses `$_GET`, POST uses `$_POST`, else `$_REQUEST` (`OperationManager::getOperationInput()`). |
| Response Envelope | JSON `State` with `success`, `result` on success, and `error` on failure (`webservice.php`). |
| Content Type | `application/json` (set by `webservice.php` in `setResponseHeaders()`). |

### 8.2 REST Endpoints

Note: The implementation is not REST resource-oriented; operations are selected by the `operation` parameter. The “endpoint tables” below document the observed HTTP interface for the dispatcher.

#### 8.2.1 `webservice.php` dispatcher (all operations)

Endpoint Details:

| Field | Value | Notes |
|---|---|---|
| Path | `webservice.php` | Single endpoint for all operations. |
| Methods | GET and POST | Effective input source depends on operation metadata in `vtiger_ws_operation.type`. |
| Query/Body Parameters | `operation`, `format`, `sessionName`, plus operation-specific params | `operation` is lowercased in `webservice.php`. |
| Auth | Required unless operation is prelogin | Enforced by `OperationManager::isPreLoginOperation()` and `webservice.php` gate. |
| Success Envelope | `{ "success": true, "result": ... }` | Written by `writeOutput()`. |
| Error Envelope | `{ "success": false, "error": { "code": "...", "message": "..." } }` | Written by `writeErrorOutput()` using `WebServiceException`. |

Path Parameters:

| Name | In | Type | Description |
|---|---|---|---|
| N/A | N/A | N/A | `webservice.php` does not use path parameters; it uses query/body parameters. |

Error Responses:

| HTTP Status | Problem Type | Description | Example Cause |
|---|---|---|---|
| 200 (typical) | `AUTHENTICATION_REQUIRED` | Returned in JSON envelope when auth is missing for non-prelogin operation. | Missing `sessionName`. |
| 200 (typical) | `UNKNOWN_OPERATION` | Returned when `vtiger_ws_operation` has no row for requested operation. | Invalid `operation` value. |
| 200 (typical) | `INTERNAL_SERVER_ERROR` | Returned on unexpected exceptions. | PHP exception thrown in handler or dispatcher. |

Success response JSON example:

```json
{
  "success": true,
  "result": {}
}
```

Failure response JSON example:

```json
{
  "success": false,
  "error": {
    "code": "AUTHENTICATION_REQUIRED",
    "message": "Authencation required"
  }
}
```

#### 8.2.2 Canonical operations (examples)

The operation list is DB-driven. The following are common vtiger operations described in repo docs (`Docs/API_Documentation.md`) and supported by the dispatcher pattern.

##### Operation: `getchallenge` (prelogin)

Endpoint Details:

| Field | Value | Notes |
|---|---|---|
| Operation | `getchallenge` | Prelogin operation. |
| Method | GET (typical) | DB-driven. |
| Parameters | `username` | Used to generate short-lived token. |
| Auth | Not required | Prelogin. |

Path Parameters:

| Name | In | Type | Description |
|---|---|---|---|
| N/A | N/A | N/A | N/A |

Error Responses:

| HTTP Status | Problem Type | Description | Example Cause |
|---|---|---|---|
| 200 (typical) | [To be determined] | [To be determined] | Missing username or user not found. |

Success response JSON example:

```json
{
  "success": true,
  "result": {
    "token": "TOKEN",
    "serverTime": 0,
    "expireTime": 0
  }
}
```

##### Operation: `login` (prelogin)

Endpoint Details:

| Field | Value | Notes |
|---|---|---|
| Operation | `login` | Prelogin operation that sets authenticated user in session. |
| Method | POST (typical) | DB-driven. |
| Parameters | `username`, `accessKey` | `accessKey` is typically an md5 hash computed client-side (documented in `Docs/API_Documentation.md`). |
| Auth | Not required | Prelogin. |

Path Parameters:

| Name | In | Type | Description |
|---|---|---|---|
| N/A | N/A | N/A | N/A |

Error Responses:

| HTTP Status | Problem Type | Description | Example Cause |
|---|---|---|---|
| 200 (typical) | `AUTHENTICATION_FAILURE` | Login failed. | Incorrect credentials or invalid access key. |

Success response JSON example (from `OperationManager::runOperation()` prelogin path):

```json
{
  "success": true,
  "result": {
    "sessionName": "SESSION_ID",
    "userId": "19x1",
    "version": "0.22",
    "vtigerVersion": "5.4.0"
  }
}
```

##### Operation: `extendsession` (special handling)

Endpoint Details:

| Field | Value | Notes |
|---|---|---|
| Operation | `extendsession` | Adopt an existing PHP session (`PHPSESSID`) to establish a webservice session. |
| Method | [To be determined] | DB-driven; `webservice.php` includes special-case logic for this operation. |
| Parameters | `PHPSESSID` (request or cookie) | `webservice.php` reads `$_REQUEST['PHPSESSID']` or cookie `PHPSESSID`. |
| Auth | Required in the sense of having an adoptable session | `webservice.php` sets `$adoptSession=true` when it sees expected input. |

Path Parameters:

| Name | In | Type | Description |
|---|---|---|---|
| N/A | N/A | N/A | N/A |

Error Responses:

| HTTP Status | Problem Type | Description | Example Cause |
|---|---|---|---|
| 200 (typical) | `AUTHENTICATION_REQUIRED` | Returned if request input does not allow adoption. | Missing `operation` input or missing session. |

Success response JSON example:

```json
{
  "success": true,
  "result": {}
}
```

### 8.3 Event / Async Interfaces

| Channel / Topic | Direction | Schema Ref | Consumer / Producer |
|---|---|---|---|
| `cron:<task name>` | Producer | N/A (PHP include execution) | Producer: `vtigercron.php`; Consumer: handler file in `vtiger_cron_task.handler_file`. |

## 9. Business Logic & Algorithms

### 9.1 Business Process Name

Business process: Core Dispatch, Persistence, and Automation.

Triggers:
- HTTP requests to `index.php` and `webservice.php`.
- Scheduled execution of `vtigercron.php` from an external scheduler.

Pre-conditions:
- `config.inc.php` exists and DB is configured (enforced in `index.php`).
- DB is reachable and schema is consistent with code version (migration gate in `index.php` comparing `vtiger_version.current_version` with `$vtiger_current_version` in `vtigerversion.php`).
- For authenticated operations: session exists and is valid.

Algorithm logic:
- UI dispatch validates `module` exists under `modules/` and action file exists (`index.php` scandir checks), validates `record` numeric when present, enforces authentication and permissions via `isPermitted`, and includes the module action script.
- Webservice dispatch lowercases `operation`, resolves handler path/method and parameter list from DB, starts a cookie-less session, enforces authentication for non-prelogin operations, decodes `encoded` parameters using Zend_Json, invokes handler callable, and returns a JSON envelope.
- Cron execution lists enabled tasks, checks runnability based on frequency and timestamps, marks task running, includes handler file, and marks finished.

Post-conditions:
- UI action scripts may mutate DB state using `CRMEntity::save()` and related helpers.
- Webservice operations return consistent JSON envelopes.
- Cron task state timestamps are updated in `vtiger_cron_task`.

Complexity:

| Metric | Value | Notes |
|---|---|---|
| Time complexity (UI dispatch) | O(1) per request (excluding module logic) | Dispatch checks scandir results and includes one PHP file. |
| Time complexity (webservice dispatch) | O(P) | P is number of parameters for an operation loaded from `vtiger_ws_operation_parameters`. |
| Time complexity (cron dispatch cycle) | O(T) | T is number of active cron tasks. |

### 9.2 Business Rules Register

| Rule ID | Rule Description | Applies To | Violation Response |
|---|---|---|---|
| BR-001 | A requested UI module must exist as a directory under `modules/` and must not contain `.` or `/`. | `index.php` | Request terminates with “Module name is missing. Please check the module name.” |
| BR-002 | A requested UI action must exist as `modules/<module>/<action>.php`. | `index.php` | Request terminates with “Action name is missing. Please check the action name.” |
| BR-003 | `record` parameter must be numeric when provided. | `index.php` | Request terminates with “An invalid record number specified to view details.” |
| BR-004 | A non-prelogin webservice operation requires a valid session. | `webservice.php` | JSON error envelope with authentication required. |
| BR-005 | Cron tasks run no more frequently than configured frequency. | `Vtiger_Cron::isRunnable()` | Task is skipped with “[INFO] not ready to run…” output. |

### 9.3 Input Validation Rules

| Field / Parameter | Validation Rule | Error Code | Error Detail Message |
|---|---|---|---|
| `module` (UI) | Must be present, must exist in `modules/` directory list, must not contain `.` or `/`. | N/A (UI) | “Module name is missing. Please check the module name.” |
| `action` (UI) | Must correspond to an existing `*.php` file inside module directory. | N/A (UI) | “Action name is missing. Please check the action name.” |
| `record` (UI) | Must be numeric if not empty. | N/A (UI) | “An invalid record number specified to view details.” |
| `operation` (API) | Must map to an entry in `vtiger_ws_operation.name` (case-insensitive; lowercased). | `UNKNOWN_OPERATION` | “Unknown operation requested” (thrown by `OperationManager`). |
| `sessionName` (API) | Required for non-prelogin operations; must represent a valid `HTTP_Session`. | `AUTHENTICATION_REQUIRED` / `SESSIONIDINVALID` / `SESSLIFEOVER` / `SESSIONIDLE` | Session invalid/expired/idle or auth required. |
| File upload name | Sanitized via `sanitizeUploadFileName(...)` and bad extensions list from config. | [To be determined] | Upload may be rejected or saved with `.txt` appended depending on sanitization rules. |

## 10. Concurrency & Resource Management

### 10.1 Concurrency Model

| Concern | Design Decision |
|---|---|
| Web request concurrency | Each HTTP request is handled independently by PHP runtime; shared DB state is managed with transactions and DB-level constraints. |
| Cron concurrency | `vtigercron.php` iterates tasks in a single process; concurrency must be controlled operationally to avoid multiple schedulers running overlapping cron cycles. |
| DB concurrency | `CRMEntity::saveentity()` uses transactions; concurrency correctness also relies on DB schema constraints and module-specific logic. |
| Session concurrency | Webservice sessions are managed by `HTTP_Session` and can expire/idle; cookies are disabled in webservice session manager (`HTTP_Session::useCookies(false)`). |

### 10.2 Resource Management

| Resource | Acquisition | Release Strategy | Leak Prevention |
|---|---|---|---|
| DB connection (`$adb`) | Initialized globally via vtiger bootstrap (used by many scripts/classes). | PHP request lifecycle ends; connections are closed by runtime. | Use prepared queries (`pquery`) to avoid injection; transaction boundaries in `saveentity()`. |
| Filesystem storage for attachments | `CRMEntity::uploadAndSaveFile()` computes path via `decideFilePath()` and moves uploaded file. | File remains on disk; DB stores path metadata. | Ensure upload directory is writable and protected; sanitize filenames and block bad extensions. |
| Cron handler includes | `vtigercron.php` includes handler file based on DB row `handler_file`. | PHP script ends; included code remains loaded for process duration. | Validate handler file with `checkFileAccess()` before inclusion. |
| HTTP session | `SessionManager::startSession()` creates/adopts session. | `SessionManager::destroy()` on invalid sessions; expiry/idle destroys session. | Use expiry and idle windows to prevent indefinite sessions. |

### 10.3 Memory Management

| Concern | Approach |
|---|---|
| PHP memory limit | `config.template.php` sets `ini_set('memory_limit','64M')`. |
| Bulk save optimization | `CRMEntity::isBulkSaveMode()` exists to reduce overhead; insert date/time preservation logic and caching of field queries in `insertIntoEntityTable()`. |
| Caching | `CRMEntity::insertIntoEntityTable()` uses a static `_privatecache` for field metadata results to reduce repeated DB reads (notably in bulk save). |

## 11. Error Handling

### 11.1 Error Handling Strategy

The system uses a mix of exception-based handling (webservice and cron flows) and immediate termination (`die`) for some UI validation failures. Webservice errors are consistently encoded into a JSON envelope by `webservice.php`, while cron errors are printed to stdout and do not necessarily update task status beyond the last successful state transition. Entity persistence relies on DB transactions so that persistence failures avoid partial writes.

### 11.2 Error Classification

| Error Class | HTTP Status | Retry | Handling |
|---|---|---|---|
| `WebServiceException` | 200 (typical) | Client-dependent | Caught in `webservice.php` and returned in JSON `error` envelope. |
| Generic `Exception` in webservice | 200 (typical) | Client-dependent | Caught and mapped to `INTERNAL_SERVER_ERROR` code/message. |
| Cron task exception | N/A (CLI) | Manual/next cycle | Caught in `vtigercron.php`, prints error text; subsequent cycle may restart. |
| UI validation failure | N/A (HTML) | User corrects input | `index.php` terminates request with message. |
| DB failure during entity save | N/A | [To be determined] | Transaction behavior depends on DB layer and error mode; failure aborts save. |

### 11.3 Retry Policy

| Parameter | Policy |
|---|---|
| Webservice network/client errors | [To be determined] (client-side policy). |
| Cron task failures | Implicit retry on next cycle if scheduler re-invokes; timeout restart is logged as “[INFO] ... had timedout ... restarting”. |
| DB transient errors | [To be determined] (no explicit retry/backoff is evidenced in the inspected code). |

### 11.4 Circuit Breaker Configuration

| Parameter | Value / Policy |
|---|---|
| Circuit breaker | Not implemented (no circuit breaker library/pattern evidenced). |

## 12. Security Implementation

### 12.1 Authentication & Authorisation

| Concern | Implementation Requirement |
|---|---|
| UI authentication | `index.php` checks `$_SESSION["authenticated_user_id"]` and validates `$_SESSION["app_unique_key"] == $application_unique_key` (key configured in config). |
| UI authorization | `index.php` computes effective action and checks `isPermitted($module, $now_action, [$record])` before including module script. |
| API authentication | `webservice.php` requires `sessionName` for non-prelogin operations, starts session via `SessionManager`, and uses `authenticatedUserId` session variable. |
| API authorization | Operation handlers typically enforce permissions; dispatcher ensures user context is loaded when `authenticatedUserId` exists. |
| Cron access control | `vtigercron.php` allows execution only in CLI or when authenticated session has matching app unique key. |

### 12.2 Input Validation & Injection Prevention

The UI dispatcher rejects unsafe module/action names by verifying module directory existence via `scandir` and prohibiting `.` or `/` via regex checks. It validates `record` is numeric and uses `vtlib_purify()` for some output string building. Webservice inputs are normalized and sanitized according to parameter definitions; `encoded` parameters are JSON-decoded. Database operations throughout core runtime use prepared queries (`$adb->pquery`) rather than raw interpolation.

### 12.3 Sensitive Data Handling

| Data Type | Handling Requirement |
|---|---|
| Database password | Stored in generated config file (template placeholders exist in `config.template.php`). Do not log; restrict file permissions. |
| Application unique key | Stored in config file as `$application_unique_key`; used for session validation and cron execution gate. Treat as secret. |
| Attachment contents | Stored on filesystem under `storage/`-like paths; ensure directory permissions, avoid direct web access if possible. |
| Webservice session identifiers | Passed as `sessionName`; treat as secret/session token; avoid logging and protect via transport (TLS). |

### 12.4 OWASP Top 10 Compliance Checklist

| OWASP Category | Control Applied | Implementation Location |
|---|---|---|
| Broken Access Control | Central permission checks in dispatcher | `index.php` (`isPermitted` and module activation checks). |
| Cryptographic Failures | [To be determined] | Not evidenced in inspected files; TLS/transport is out of repo scope. |
| Injection | Prepared statements and sanitization | `$adb->pquery` usage throughout; `OperationManager` parameter handling; `vtlib_purify` referenced in HLD. |
| Insecure Design | Defense-in-depth via dispatcher validation and include guards | `index.php` validation; cron include checks. |
| Security Misconfiguration | DB-driven operation and cron registries require governance | `vtiger_ws_operation*` and `vtiger_cron_task` tables. |
| Vulnerable and Outdated Components | Legacy third-party libs exist | Not assessed here; requires dependency audit. |
| Identification and Authentication Failures | Session expiry/idle enforcement for webservice | `SessionManager` uses max life (86400) and idle (1800). |
| Software and Data Integrity Failures | Safe include checks for cron handlers | `vtigercron.php` calls `checkFileAccess(...)` before `require_once`. |
| Security Logging and Monitoring Failures | Dedicated SECURITY logger | `log4php.properties` defines SECURITY logger output to `logs/security.log`. |
| Server-Side Request Forgery (SSRF) | [To be determined] | Not evidenced in inspected core runtime paths. |

## 13. Observability

### 13.1 Structured Logging

Fields (recommended for consistent logging across entrypoints):

| Field | Type | Description/Example |
|---|---|---|
| `component` | string | For example, `index`, `webservice`, `cron`. |
| `operation` | string | Webservice operation name (lowercased). |
| `user_id` | string|int | Authenticated user id when available (UI: `$_SESSION["authenticated_user_id"]`, API: `authenticatedUserId`). |
| `module` | string | UI module name. |
| `action` | string | UI action name. |
| `cron_task` | string | Cron task name (`Vtiger_Cron::getName()`). |
| `error_code` | string | Webservice error code. |
| `message` | string | Human-readable message. |

Log Level Guide:

| Level | When to Use |
|---|---|
| FATAL | Security critical errors or unrecoverable failures (default for root logger). |
| INFO | High-level operational messages (e.g., cron not runnable). |
| DEBUG | Diagnostic output (enabled for INSTALL and MIGRATION loggers). |

Concrete log configuration evidenced in `log4php.properties`:

- SECURITY logger writes to `logs/security.log`.
- INSTALL logger writes to `logs/installation.log`.
- MIGRATION logger writes to `logs/migration.log`.
- SOAP logger writes to `logs/soap.log`.
- PLATFORM logger writes to `logs/platform.log`.
- Root logger writes to `logs/vtigercrm.log`.
- SQLTIME logger writes to `logs/sqltime.log`.

### 13.2 Metrics

| Metric Name | Type | Labels | Description |
|---|---|---|---|
| [To be determined] | [To be determined] | [To be determined] | No metrics subsystem is evidenced in inspected code; metrics design requires additional instrumentation. |

### 13.3 Distributed Tracing

No distributed tracing implementation is evidenced in the inspected code. If tracing is introduced, standard practice should include request correlation IDs propagated through UI, webservice, and cron logs, and trace context propagation across any external calls.

### 13.4 Health Check Endpoints

| Endpoint | Check Performed | Success Response | Failure Response |
|---|---|---|---|
| [To be determined] | [To be determined] | [To be determined] | [To be determined] |

## 14. Performance Design

### 14.1 Performance Budget

| Operation | p50 Target | p95 Target | p99 Target |
|---|---|---|---|
| UI dispatch (`index.php` include + permission check) | [To be determined] | [To be determined] | [To be determined] |
| Webservice operation dispatch overhead | [To be determined] | [To be determined] | [To be determined] |
| Entity save transaction (`CRMEntity::saveentity`) | [To be determined] | [To be determined] | [To be determined] |
| Cron cycle per task overhead | [To be determined] | [To be determined] | [To be determined] |

### 14.2 Caching Strategy

| What is Cached | Cache Location | TTL | Invalidation Trigger |
|---|---|---|---|
| Field metadata query results during `insertIntoEntityTable()` | In-process static cache (`$_privatecache`) | Process lifetime | Process restart; bulk save operation boundaries. |
| List view and other UI caching | [To be determined] | [To be determined] | [To be determined] |
| Smarty compiled templates | Filesystem (`Smarty/templates_c/`) | [To be determined] | Template change / cache clear. |

### 14.3 Database Query Optimisation

The core runtime reduces repeated metadata reads by caching field query results in `CRMEntity::insertIntoEntityTable()`. It uses prepared statements (`pquery`) throughout the inspected code to allow DB optimization and avoid SQL injection. Additional performance optimization is expected to rely on DB indexing and careful module query design, which are not fully enumerated in this LLD.

### 14.4 Load Test Requirements

| Test Type | Tool | Pass Criteria |
|---|---|---|
| Webservice dispatch throughput | [To be determined] | [To be determined] |
| UI navigation smoke load | [To be determined] | [To be determined] |
| Cron task execution duration | [To be determined] | No task should exceed schedule window; timeouts should be investigated. |

## 15. Testing Design

### 15.1 Coverage Requirements

| Test Type | Coverage Target | Tooling | CI Gate |
|---|---|---|---|
| Unit | [To be determined] | [To be determined] | [To be determined] |
| Integration | [To be determined] | [To be determined] | [To be determined] |
| E2E | [To be determined] | [To be determined] | [To be determined] |

### 15.2 Key Test Scenarios

| Scenario ID | Test Scenario | Test Type | Expected Outcome |
|---|---|---|---|
| TS-001 | UI dispatch rejects non-existent module/action | Integration | Request terminates with error message; no file inclusion occurs. |
| TS-002 | UI dispatch rejects non-numeric `record` | Integration | Request terminates with “invalid record number” message. |
| TS-003 | Webservice non-prelogin op without session returns error envelope | Integration | JSON `success=false` with auth-required code. |
| TS-004 | Webservice prelogin `login` returns `sessionName` and `userId` envelope | Integration | JSON `success=true` with expected result fields. |
| TS-005 | Cron skips task when frequency not elapsed | Integration | CLI output includes “not ready to run”; DB timestamps remain unchanged. |
| TS-006 | CRMEntity save persists to `vtiger_crmentity` and module tables in a transaction | Integration | Record appears in DB consistently across tables. |
| TS-007 | Attachment upload writes file and creates `vtiger_attachments` + `vtiger_seattachmentsrel` records | Integration | File exists on disk; DB links present. |

### 15.3 Test Data Strategy

Test data should be managed as DB fixtures or installer-seeded datasets rather than ad-hoc manual entries. Because persistence spans `vtiger_crmentity` and module tables, test data creation should use the same API/UI paths that production uses (UI save flows or webservice create operations) to ensure that workflows and events are exercised.

## 16. Deployment & Configuration

### 16.1 Environment Variables

| Variable | Description | Required | Default | Sensitive? |
|---|---|---|---|---|
| N/A | No environment-variable based configuration is evidenced in the inspected code paths; vtiger uses PHP config files generated from `config.template.php` (e.g., `config.inc.php`) and optionally `config_override.php`. | N/A | N/A | N/A |

### 16.2 Infrastructure Dependencies

| Dependency | Type | Purpose | Failure Behaviour |
|---|---|---|---|
| Relational database | Database | Store CRM entities and runtime metadata tables (`vtiger_crmentity`, `vtiger_ws_operation*`, `vtiger_cron_task`, `vtiger_version`). | UI/API fail; `index.php` cannot pass version check and `webservice.php` cannot resolve operations. |
| Filesystem storage | Filesystem | Store attachments and caches; default attachment root under `storage/` as returned by `decideFilePath()`. | Upload/download breaks; some modules may fail. |
| Scheduler | External service | Execute `php vtigercron.php` periodically. | Background automation does not run. |
| Web server + PHP runtime | Runtime | Serve `index.php`, `webservice.php`, `vtigerservice.php`, installer. | Entire application unavailable. |
| Logs directory | Filesystem | log4php appenders write to `logs/*.log`. | Logging is lost; troubleshooting/security monitoring degraded. |

### 16.3 Rollback & Migration Procedure

Database and application rollback must consider the UI migration gate: `index.php` compares `$vtiger_current_version` (from `vtigerversion.php`) with `vtiger_version.current_version`. If they mismatch, the UI prints “Migration Incompleted” and exits. Therefore, rollback procedure must keep code and DB in a compatible pair.

Suggested rollback steps (environment-specific details to be determined):

1. Put the application in maintenance mode (prevent new writes) using web server controls.
2. Restore application code to previous known-good version (matching the DB schema version recorded in `vtiger_version.current_version`).
3. Restore database backup (or apply reverse migration) so `vtiger_version.current_version` matches the code version.
4. Verify core entrypoints:
   - UI: load `index.php` and authenticate.
   - API: call a prelogin operation on `webservice.php`.
   - Cron: run `php vtigercron.php` and confirm tasks list/skip/run as expected.
5. Monitor logs in `logs/vtigercrm.log` and `logs/security.log`.

### 16.4 Code Standards & Conventions

| Concern | Standard / Reference |
|---|---|
| Safe dynamic includes | UI dispatch validates module/action existence; cron validates handler file via `checkFileAccess()`; avoid any new request-driven include without validation. |
| DB access | Use `$adb->pquery(...)` prepared queries and keep transaction boundaries clear for persistence. |
| Error envelopes | Webservice endpoint must return `State` JSON envelope consistently for new operations. |
| Logging | Use appropriate loggers (SECURITY vs root) and ensure logs are written under `logs/` per `log4php.properties`. |
| File uploads | Sanitize filenames (`sanitizeUploadFileName`) and enforce bad extension list (`$upload_badext` from config). |

## 17. Design Review Checklist

| Item | Category | Status | Reviewer / Date |
|---|---|---|---|
| All LLD sections present and conform to template | Documentation | [To be determined] | [To be determined] |
| Webservice auth/session behavior matches code | Security | [To be determined] | [To be determined] |
| Cron state machine and runnability logic is correct | Reliability | [To be determined] | [To be determined] |
| Entity save transaction boundaries are clear | Data | [To be determined] | [To be determined] |
| Attachment storage risks documented (shared storage) | Deployment | [To be determined] | [To be determined] |
| Logging configuration and destinations documented | Observability | [To be determined] | [To be determined] |
| Tables and columns in Section 7 match actual schema | Data | [To be determined] | [To be determined] |

## 18. Glossary

| Term | Definition |
|---|---|
| Action script | A module PHP script under `modules/<Module>/<Action>.php` included by `index.php`. |
| Cron task | A scheduled job represented by a row in `vtiger_cron_task` and executed by `vtigercron.php`. |
| Dispatcher | An entrypoint script (e.g., `index.php`, `webservice.php`, `vtigercron.php`) that routes control to module/handler code. |
| Encoded parameter | A webservice parameter type that is JSON-decoded by `OperationManager::handleType()` when type is `encoded`. |
| Entity | A CRM record represented by `vtiger_crmentity` and module tables, persisted through `CRMEntity`. |
| Operation | A webservice function configured in `vtiger_ws_operation` and executed via handler method call. |
| Prelogin operation | A webservice operation that does not require `sessionName` and establishes session/user context on success. |
| Soft delete | Marking a record deleted by setting `vtiger_crmentity.deleted=1` rather than removing rows. |
| Webservice session | Cookie-less session created by `SessionManager` and referenced by `sessionName`. |
