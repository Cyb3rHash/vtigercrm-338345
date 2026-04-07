# HIGH LEVEL DESIGN

System / Project Name: vtiger CRM (v5.4.0)  
Organisation: [To be determined]  
Author: [To be determined]  
Version: 1.0  
Date: 2026-04-07  

IEEE 1016-2009 · C4 Model · OpenAPI 3.1 · AsyncAPI 3.0  

Confidentiality notice: This document contains architectural information about the vtiger CRM deployment and codebase. Distribution and access should be controlled according to organisational policy.

## 1. Document Control

### 1.1 Version History

| Version | Date | Author | Change Summary |
|---|---|---|---|
| 1.0 | 2026-04-07 | [To be determined] | Regenerated HLD from repository evidence, following the HLD generator template. |

### 1.2 Review & Approvals

| Name | Role | Status | Date |
|---|---|---|---|
| [To be determined] | Solution Architect | [To be determined] | [To be determined] |
| [To be determined] | Engineering Lead | [To be determined] | [To be determined] |
| [To be determined] | QA Lead | [To be determined] | [To be determined] |
| [To be determined] | Security Architect | [To be determined] | [To be determined] |

### 1.3 Standards Conformance

| Standard / Framework | Version | Application in this Document |
|---|---|---|
| ISO/IEC/IEEE 1016 (Software Design Description) | 2009 | This document uses an IEEE-aligned, sectioned design description to describe architectural and component-level structure. |
| C4 Model | [To be determined] | Sections 5.2 and 5.3 describe C4 Level 1 and Level 2 views and provide prompts for diagram insertion. |
| OpenAPI | 3.1 | The vtiger implementation does not include an OpenAPI document in the repository; API documentation is captured via code entry points (for example, `webservice.php`) and existing repository docs (for example, `Docs/API_Documentation.md`). |
| AsyncAPI | 3.0 | The implementation is not event-bus based; asynchronous processing is primarily cron-driven (for example, `vtigercron.php` and `vtlib/Vtiger/Cron.php`). |
| ISO/IEC 25010 | 2011 | Section 11 captures NFRs using ISO 25010 quality attributes. |
| OWASP Top 10 | 2021 | Section 9 maps key threats and controls to common web risks such as injection, broken access control, and file inclusion. |

### 1.4 Related Documents

| Document | Reference | Version | Status |
|---|---|---|---|
| Existing HLD (non-template) | `Docs/HLD.md` | [To be determined] | Existing |
| Architecture overview (SAD-style) | `Docs/VTigerCRM-Architecture-Overview.md` | [To be determined] | Existing |
| System documentation | `Docs/system_documentation.md` | [To be determined] | Existing |
| Data model extraction | `Docs/data_model.md` | [To be determined] | Existing |
| Dependency map | `Docs/dependency_map.md` | [To be determined] | Existing |
| API documentation | `Docs/API_Documentation.md` | [To be determined] | Existing |
| API reference | `Docs/VTigerCRM-API-Reference.md` | [To be determined] | Existing |
| Lead module LLD | `Docs/LLd.md` | [To be determined] | Existing |

## 2. Introduction

### 2.1 Purpose

This High Level Design (HLD) provides a software design description that conforms to the intent of ISO/IEC/IEEE 1016-2009 and translates the system’s architectural decisions into container- and component-level design. It describes the vtiger CRM monolithic PHP application as implemented in this repository, with the goal of supporting engineering teams in maintenance, operationalization, and derivation of Low Level Designs (LLDs).

### 2.2 Scope

In Scope: This HLD covers the vtiger CRM PHP monolith entry points and their primary execution modes, including the browser UI dispatcher (`index.php`), the JSON webservice endpoint (`webservice.php` with `include/Webservices/*`), the cron dispatcher (`vtigercron.php` with `vtlib/Vtiger/Cron.php`), the installer (`install.php` with `install/CreateTables.inc.php`), and the legacy SOAP service selector (`vtigerservice.php` with `soap/*`). It also covers the core platform services that these flows depend on, such as the entity framework (`data/CRMEntity.php`), database access (`include/database/PearDatabase.php`), templating (`Smarty_setup.php`), and security/sanitization utilities (`include/utils/CommonUtils.php`, `include/utils/VtlibUtils.php`).

Out of Scope: This HLD does not define the web server configuration (Apache/Nginx), PHP-FPM/process manager configuration, network topology specifics, database provisioning (replication, backups, encryption-at-rest), or a complete formal API contract in OpenAPI/AsyncAPI format, because those artifacts are not present in the repository.

### 2.3 Intended Audience

| Audience | Purpose | Sections of Primary Interest |
|---|---|---|
| Engineering (Backend/PHP) | Understand request entry points, module dispatch, entity lifecycle, and extension points. | 5, 6, 7, 8 |
| QA / Test Engineering | Identify verifiable behaviors and integration surfaces (UI, API, cron). | 3, 8, 11, 14 |
| Security Engineering | Understand security boundaries, key controls, and high-risk flows (dynamic includes, file uploads, sessions). | 9, 11 |
| Operations / SRE | Understand deployment expectations, cron execution, storage requirements, and logging. | 10, 14 |
| Product / Program Management | Understand high-level system boundaries and integration interfaces. | 2, 5, 8 |

### 2.4 Design Assumptions & Constraints

| ID | Type | Statement | Impact if Invalid |
|---|---|---|---|
| A-01 | Assumption | The application is deployed behind a PHP-capable web server that routes requests to entry scripts such as `index.php`, `webservice.php`, `install.php`, and `vtigerservice.php`. | Entry-point routing described in this HLD may not match runtime behavior. |
| A-02 | Assumption | A relational database is configured via generated `config.inc.php` (template keys are visible in `config.template.php`) and is reachable from the PHP runtime. | UI startup checks and most features will fail because core operations depend on `$adb` (`include/database/PearDatabase.php`). |
| A-03 | Assumption | A scheduler runs `php vtigercron.php` to execute background services. | Cron-driven automation (workflow processing, reminders, scheduled reports, mail scanning) will not execute. |
| C-01 | Constraint | UI entrypoint enforces PHP >= 5.2.0 (`index.php` checks `version_compare(phpversion(), '5.2.0')`). | Deployments using older PHP will not start. |
| C-02 | Constraint | Installer enforces PHP >= 5.0 (`install.php` checks `version_compare(phpversion(), '5.0')`). | Fresh installs will not run on older PHP. |
| C-03 | Constraint | Dynamic inclusion is guarded to prevent unsafe file inclusion (`checkFileAccessForInclusion` in `include/utils/CommonUtils.php`). | Improper configuration or bypass increases risk of file disclosure or remote code execution. |
| C-04 | Constraint | Cron task registry is database-backed and handler files are loaded at runtime (`Vtiger_Cron::register` and `vtigercron.php` + `Vtiger_Cron::listAllActiveInstances`). | Misconfigured handler paths can break cron execution or cause security issues. |
| C-05 | Constraint | Attachments are written to filesystem paths under `storage/` by default (`decideFilePath` in `include/utils/CommonUtils.php` and `CRMEntity::uploadAndSaveFile` in `data/CRMEntity.php`). | Multi-node deployments require shared storage or an alternative storage strategy. |

## 3. Requirements Traceability

### 3.1 Traceability Matrix

| Req ID | Requirement Summary | Design Element | Section | Verification Method |
|---|---|---|---|---|
| REQ-HLD-01 | The system shall serve a server-rendered CRM web UI with module/action routing. | `index.php` dispatch to `modules/<Module>/<Action>.php` with permission checks. | 5, 6 | Manual verification in a running environment by navigating to modules and observing permission behavior (`isPermitted` in `index.php`). |
| REQ-HLD-02 | The system shall expose a JSON webservice endpoint for programmatic access. | `webservice.php` dispatcher with `include/Webservices/OperationManager.php` and DB-driven `vtiger_ws_operation`. | 6, 8 | Integration verification by invoking `webservice.php` operations described in `Docs/API_Documentation.md`. |
| REQ-HLD-03 | The system shall execute background jobs on a schedule. | `vtigercron.php` + `vtlib/Vtiger/Cron.php` registry (`vtiger_cron_task`). | 6, 10, 14 | CLI execution verification by running `php vtigercron.php` and observing runnable/skip behavior (`Vtiger_Cron::isRunnable`). |
| REQ-HLD-04 | The system shall persist CRM entities to a relational database with a common entity table and module tables. | `data/CRMEntity.php` save pipeline using `$this->tab_name` and `vtiger_crmentity`. | 6, 7 | Data verification by creating a record and checking `vtiger_crmentity` and module-specific tables per module schema. |
| REQ-HLD-05 | The system shall store and serve uploaded attachments linked to CRM records. | `CRMEntity::uploadAndSaveFile` + `vtiger_attachments` + filesystem under `storage/`. | 6, 7, 9 | Upload and download verification through a module supporting attachments (for example, Documents). |

## 4. Architecture Goals & Principles

### 4.1 Architecture Goals

| Goal | Description | Primary NFR |
|---|---|---|
| Secure request dispatch | Ensure dynamic routing and file inclusion cannot be abused to include unsafe paths. | Security |
| Extensible module model | Enable feature growth via modules under `modules/` and shared `CRMEntity` base behaviors. | Maintainability |
| Unified persistence model | Provide consistent CRUD lifecycle and event hooks across modules using `CRMEntity`. | Reliability |
| Metadata-driven integration | Route API operations using database metadata to allow controlled extension (`vtiger_ws_operation`). | Compatibility |
| Operable background processing | Provide a configurable, observable cron task framework (`vtiger_cron_task` via `Vtiger_Cron`). | Reliability |

### 4.2 Design Principles

Separation of concerns is implemented through a layered monolith where request entry points (`index.php`, `webservice.php`, `vtigercron.php`, `install.php`) delegate to specialized subsystems such as modules (`modules/*`), persistence (`data/CRMEntity.php`), and database access (`include/database/PearDatabase.php`). The design relies on convention-based routing for the UI and metadata-driven routing for the API and cron subsystems, which reduces boilerplate but places increased importance on input validation and safe inclusion controls.

Defense in depth is applied through multiple, overlapping controls. For example, `index.php` validates module/action paths and enforces permission checks, while shared file inclusion guards (`checkFileAccessForInclusion` in `include/utils/CommonUtils.php`) and input purification (`vtlib_purify` in `include/utils/VtlibUtils.php`) support secure handling across request surfaces.

Configuration-driven behavior is favored for operations and background tasks. Webservice operations are resolved via the `vtiger_ws_operation` registry in `include/Webservices/OperationManager.php`, and cron tasks are resolved via `vtiger_cron_task` in `vtlib/Vtiger/Cron.php`. This improves flexibility, but requires governance around configuration changes.

## 5. Logical Architecture

### 5.1 Architecture Pattern

| Pattern | Rationale / Trade-offs |
|---|---|
| Monolithic PHP application | A single deployable codebase is consistent with the repository layout and entry scripts. The trade-off is tight coupling and shared deployment lifecycle for all modules. |
| Front controller (UI) | `index.php` acts as the central dispatcher that includes module action scripts. The trade-off is reliance on safe dynamic inclusion and URL parameter validation. |
| Script-based MVC (module actions + templates) | Module action scripts behave as controllers and render via Smarty templates (`Smarty_setup.php`). The trade-off is limited compile-time structure and higher reliance on conventions. |
| Metadata-driven API dispatcher | `webservice.php` dispatches based on DB metadata (`vtiger_ws_operation`) via `OperationManager`. The trade-off is operational risk if metadata is misconfigured. |
| Registry-driven cron scheduling | `vtigercron.php` runs tasks configured in `vtiger_cron_task` via `Vtiger_Cron`. The trade-off is that task handler safety and correctness depends on registry integrity. |

### 5.2 System Context (C4 Level 1)

The vtiger CRM system boundary includes the PHP codebase and its dependencies on a relational database and filesystem storage. External actors include browser users (UI), API clients (JSON webservices), SOAP clients (legacy integrations selected via `vtigerservice.php`), and a scheduler that triggers cron execution.

Diagram prompt: [Insert C4 Level 1 System Context diagram here.]

### 5.3 Container Architecture (C4 Level 2)

Although the application is a monolith, it runs in multiple execution modes represented by entry scripts: a UI request mode (`index.php`), an API request mode (`webservice.php`), an installation mode (`install.php`), a cron execution mode (`vtigercron.php`), and a SOAP dispatch mode (`vtigerservice.php`). All modes share the same database abstraction (`include/database/PearDatabase.php`) and entity framework (`data/CRMEntity.php`).

Diagram prompt: [Insert C4 Level 2 Container diagram here.]

### 5.4 Logical Layers

| Layer | Responsibility | Key Components |
|---|---|---|
| Presentation | Render server-side UI and templates. | `Smarty_setup.php`, `Smarty/templates/*`, `themes/*`. |
| UI Routing & Access Control | Start session, validate request, enforce authn/authz, dispatch to modules. | `index.php`, `include/utils/CommonUtils.php`, `include/utils/VtlibUtils.php`. |
| Module/Application Logic | Implement per-feature actions and workflows for CRM modules. | `modules/*` action scripts and module classes. |
| Domain Entity Lifecycle | Provide shared CRUD, relationship handling, event triggering, attachments, soft delete. | `data/CRMEntity.php`. |
| Integration (API & SOAP) | Provide programmatic access via JSON webservice operations and legacy SOAP endpoints. | `webservice.php`, `include/Webservices/*`, `vtigerservice.php`, `soap/*`. |
| Background Processing | Execute scheduled tasks and maintain task state. | `vtigercron.php`, `vtlib/Vtiger/Cron.php`, `cron/*`. |
| Persistence | Database abstraction and schema creation utilities. | `include/database/PearDatabase.php`, `adodb/*`, `schema/DatabaseSchema.xml`. |

## 6. Component Architecture

### 6.1 Component Overview

| Component ID | Component Name | Responsibility | Technology | LLD Reference |
|---|---|---|---|---|
| C-01 | UI Front Controller | UI request dispatch, authentication/authorization gating, module/action include routing. | PHP (`index.php`) | [To be determined] |
| C-02 | Module Action Scripts | Implement per-module actions such as ListView, DetailView, Save, Delete, Ajax endpoints. | PHP (`modules/*`) | `Docs/LLd.md` (Leads only) |
| C-03 | Entity Framework | Shared entity lifecycle, transactions, events, relations, attachments. | PHP (`data/CRMEntity.php`) | [To be determined] |
| C-04 | Database Access Layer | DB connection, query/prepared query, transactions, schema creation. | PHP + ADODB (`include/database/PearDatabase.php`, `adodb/*`) | [To be determined] |
| C-05 | JSON Webservice Dispatcher | Operation-based webservice routing and response envelope. | PHP (`webservice.php`, `include/Webservices/*`) | `Docs/API_Documentation.md` |
| C-06 | Webservice Session Manager | Cookie-less session lifecycle management for API requests. | PHP (`include/Webservices/SessionManager.php`) | [To be determined] |
| C-07 | Cron Dispatcher | Discover and execute runnable scheduled tasks. | PHP (`vtigercron.php`) | [To be determined] |
| C-08 | Cron Registry | Persist cron task configuration and state; schema auto-init if missing. | PHP (`vtlib/Vtiger/Cron.php`) + DB `vtiger_cron_task` | [To be determined] |
| C-09 | Installer Wizard | Include installation steps and bootstrap schema and metadata. | PHP (`install.php`, `install/CreateTables.inc.php`) | [To be determined] |
| C-10 | Templating Wrapper | Configure Smarty directories and assign global UI flags. | PHP (`Smarty_setup.php`) | [To be determined] |
| C-11 | Safe Include & Sanitization Utilities | Prevent restricted file access and purify inputs. | PHP (`include/utils/CommonUtils.php`, `include/utils/VtlibUtils.php`) | [To be determined] |
| C-12 | SOAP Service Selector | Route SOAP service endpoints by `service` parameter. | PHP (`vtigerservice.php`, `soap/*`) | [To be determined] |

### 6.2 Component Detail

### 6.2.1 UI Front Controller (C-01)

| Field | Value |
|---|---|
| Component ID | C-01 |
| Type | Front controller / dispatcher |
| Responsibility | Starts sessions, validates installation state, validates module/action, enforces authentication and authorization, includes module action scripts, conditionally renders headers/footers. |
| Interfaces Exposed | HTTP UI via `index.php` with parameters such as `module` and `action`. |
| Interfaces Consumed | Database via `$adb` (`include/database/PearDatabase.php`); entity framework via `CRMEntity::getInstance`; permission checking via `isPermitted`; include guards via `checkFileAccessForInclusion` (indirectly via loaded libraries). |
| Data Owned | Session state (`$_SESSION`) including `authenticated_user_id` and `app_unique_key`; request routing decisions (`$currentModuleFile`). |
| Dependencies | `include/utils/utils.php`, `modules/Users/Users.php`, `vtigerversion.php`, `include/logging.php`, `include/utils/UserInfoUtil.php`, and module scripts under `modules/*`. |
| Technology Stack | PHP, log4php (`LoggerManager`), Smarty for rendering via included headers/footers. |
| Scalability Approach | Horizontal scaling via multiple PHP web instances is possible, but shared DB and shared file storage for uploads/attachments are required. |
| Key Failure Modes | Misconfigured `config.inc.php` triggers redirect to `install.php`; DB version mismatch (`vtiger_version` vs `vtigerversion.php`) blocks UI; invalid module/action triggers termination; permission denied blocks action inclusion. |
| LLD Reference | [To be determined] |

### 6.2.2 JSON Webservice Subsystem (C-05)

| Field | Value |
|---|---|
| Component ID | C-05 |
| Type | API gateway / dispatcher within monolith |
| Responsibility | Accepts requests to `webservice.php`, resolves `operation` via `OperationManager`, manages sessions via `SessionManager`, executes handler method, returns JSON `State` response. |
| Interfaces Exposed | HTTP JSON endpoint `webservice.php` with parameters including `operation`, `format`, and `sessionName`. |
| Interfaces Consumed | Database metadata tables `vtiger_ws_operation` and `vtiger_ws_operation_parameters` via `OperationManager::fillOperationDetails` and `fillOperationParameters`; session layer via `HTTP_Session` (`include/HTTP_Session/Session.php`). |
| Data Owned | API session identifier returned as `sessionName`; response envelope fields `success`, `result`, `error` (`include/Webservices/State.php`). |
| Dependencies | `include/Webservices/Utils.php`, `include/Webservices/OperationManager.php`, `include/Webservices/SessionManager.php`, `include/Zend/Json.php`. |
| Technology Stack | PHP, Zend JSON (`Zend_Json`), vtiger webservices framework in `include/Webservices/*`. |
| Scalability Approach | Stateless request handling is possible if session storage is managed consistently for API calls; DB remains central bottleneck; handler performance depends on operation implementation. |
| Key Failure Modes | Unknown operation throws `WebServiceException`; missing/invalid session blocks non-prelogin operations; session expiry/idle invalidates access; misconfigured handler path breaks operation include/dispatch. |
| LLD Reference | `Docs/API_Documentation.md` (behavioral guidance); [To be determined] for per-operation design. |

### 6.2.3 Entity Framework (C-03)

| Field | Value |
|---|---|
| Component ID | C-03 |
| Type | Domain framework / persistence orchestrator |
| Responsibility | Provides base class `CRMEntity` for modules, orchestrates save transactions (`saveentity`), triggers events (`save`), manages attachments (`uploadAndSaveFile`), supports soft delete (`mark_deleted`) and restore (`restore`). |
| Interfaces Exposed | PHP class `CRMEntity` used by module classes and runtime flows. |
| Interfaces Consumed | DB via `$this->db` / `$adb` (`PearDatabase`); event engine via `include/events/include.inc` and `VTEventsManager`; filesystem via `decideFilePath` for attachments. |
| Data Owned | Entity `column_fields`, module tables list (`$tab_name`), entity id (`$this->id`). |
| Dependencies | `include/utils/utils.php`, `include/utils/UserInfoUtil.php`, `include/Zend/Json.php`, `include/events/include.inc`, `include/utils/CommonUtils.php`. |
| Technology Stack | PHP, PearDatabase/ADODB, Zend JSON, vtiger events engine. |
| Scalability Approach | Shared DB transactions per save; bulk save mode exists (`CRMEntity::isBulkSaveMode`) to reduce overhead in batch scenarios. |
| Key Failure Modes | Transaction failures cause rollback; missing required data can abort saves; filesystem upload failures prevent attachment persistence; misconfigured events/handlers can impact save performance or correctness. |
| LLD Reference | `Docs/LLd.md` (Leads module patterns); [To be determined] for general entity lifecycle LLD. |

### 6.2.4 Cron Subsystem (C-07 / C-08)

| Field | Value |
|---|---|
| Component ID | C-07 / C-08 |
| Type | Scheduler dispatcher and task registry |
| Responsibility | `vtigercron.php` loads active task instances and executes runnable handlers. `Vtiger_Cron` provides DB-backed registry (`vtiger_cron_task`), schema init (`initializeSchema`), and task state transitions (`markRunning`, `markFinished`). |
| Interfaces Exposed | CLI execution of `php vtigercron.php`; optional request parameter `service` selects a specific task. |
| Interfaces Consumed | Database table `vtiger_cron_task`; handler files referenced by `handler_file` field; safe file access check `checkFileAccess` in `vtigercron.php`. |
| Data Owned | Cron task state: `status`, `laststart`, `lastend`, `frequency`, `handler_file`, `module`, `sequence`, `description`. |
| Dependencies | `vtlib/Vtiger/Cron.php`, `include/database/PearDatabase.php`, handler files under `cron/*`. |
| Technology Stack | PHP CLI, vtlib utilities (`vtlib/Vtiger/Utils.php`), PearDatabase. |
| Scalability Approach | Single-runner execution per schedule is typical to avoid overlapping execution; runnability is frequency-gated (`Vtiger_Cron::isRunnable`). |
| Key Failure Modes | Handler exceptions are caught and printed; timed-out tasks are restarted; misconfigured handler paths break execution; concurrent execution can cause duplicate processing if not controlled operationally. |
| LLD Reference | [To be determined] |

### 6.2.5 Installer Bootstrap (C-09)

| Field | Value |
|---|---|
| Component ID | C-09 |
| Type | Installer and bootstrap process |
| Responsibility | `install.php` includes install step scripts under `install/` after access validation; `install/CreateTables.inc.php` creates schema (`$adb->createTables("schema/DatabaseSchema.xml")`) and registers core metadata such as events and cron tasks. |
| Interfaces Exposed | HTTP installer wizard via `install.php`. |
| Interfaces Consumed | ADODB schema parser via `PearDatabase::createTables`; vtlib APIs for cron registration; event manager `VTEventsManager`. |
| Data Owned | Installation-time configuration values (provided via wizard); initial roles/profiles/users; generated privilege files. |
| Dependencies | `install.php`, `include/install/resources/utils.php`, `install/CreateTables.inc.php`, `schema/DatabaseSchema.xml`, `vtlib/Vtiger/Cron.php`. |
| Technology Stack | PHP, PearDatabase/ADODB, vtlib, Zend JSON. |
| Scalability Approach | One-time operation; not a runtime scaling concern. |
| Key Failure Modes | Schema creation failure aborts install; permission/unsafe include protections can block step inclusion; DB connectivity failures block install. |
| LLD Reference | [To be determined] |

## 7. Data Architecture

### 7.1 Data Flow Diagram

Diagram prompt: [Insert a data flow diagram annotated with data classifications (Public/Internal/Confidential) here.]

### 7.2 Data Stores

| Store ID | Store Name | Technology | Data Domain | Classification |
|---|---|---|---|---|
| DS-01 | CRM relational database | MySQL-compatible RDBMS via ADODB (`include/database/PearDatabase.php`) | CRM entities, metadata, permissions, audit trails, cron registry, webservice operation registry | Confidential |
| DS-02 | Attachment storage | Filesystem under `storage/` (`decideFilePath` in `include/utils/CommonUtils.php`) | Uploaded files and attachments referenced by `vtiger_attachments.path` | Confidential |
| DS-03 | Cache directories | Filesystem (`cache/`, `Smarty/templates_c/`, `Smarty/cache/`) | Cached assets, compiled templates, runtime caches | Internal |
| DS-04 | Privilege flat files | Filesystem (`user_privileges/`) generated during install (`createUserPrivilegesfile`) | Derived permission and sharing configuration per user | Confidential |
| DS-05 | Logs | Filesystem under `logs/` (configured by log4php) | Operational and security logs | Internal |

### 7.3 Data Governance

| Concern | Policy / Approach |
|---|---|
| Soft delete | Records are soft-deleted via `vtiger_crmentity.deleted` (`CRMEntity::mark_deleted`). |
| Attachment handling | Attachments are stored on filesystem and linked via DB (`vtiger_attachments`, `vtiger_seattachmentsrel`) (`CRMEntity::uploadAndSaveFile`). |
| Access control | UI permissions are enforced by `isPermitted` in `index.php`, and field visibility rules are applied in entity logic and UI helpers. |
| Auditability | UI can write to `vtiger_audit_trial` when auditing is enabled (see `index.php` audit insert). |
| Backup and retention | [To be determined] based on operational requirements; DB and filesystem stores must be backed up consistently. |
| Data classification | CRM content is treated as Confidential by default; classification should be refined per organisation policy. |

## 8. Integration Architecture

### 8.1 Integration Patterns

| Pattern | When Used / Rationale |
|---|---|
| HTTP server-rendered UI | Primary user interaction through `index.php` and module scripts. |
| Operation-based JSON API | Programmatic integration through `webservice.php` using DB-configured operations (`vtiger_ws_operation`). |
| Legacy SOAP endpoints | Compatibility integrations selected through `vtigerservice.php?service=...` and implemented in `soap/*`. |
| Cron-driven asynchronous processing | Background work executed via `vtigercron.php` and `cron/*` handlers registered through `Vtiger_Cron`. |
| Filesystem-backed attachments | Efficient storage for binary data with DB metadata pointers (`vtiger_attachments.path`). |

### 8.2 Integration Map

| Source | Target | Pattern / Standard | Data / Event | SLA / Timeout |
|---|---|---|---|---|
| Browser | `index.php` | HTTP (server-rendered HTML) | UI actions, module operations | [To be determined] |
| API client | `webservice.php` | HTTP JSON (vtiger webservices) | Operation request/response | Session idle 1800s and lifespan 86400s (as set in `include/Webservices/SessionManager.php`) |
| SOAP client | `vtigerservice.php` -> `soap/*` | SOAP over HTTP | Plugin/portal-style SOAP functions | [To be determined] |
| Scheduler | `vtigercron.php` | CLI execution | Run configured cron services | Frequency-based run gating (`Vtiger_Cron::isRunnable`) |
| vtiger runtime | Database | RDBMS protocol | CRUD and metadata queries | [To be determined] |
| vtiger runtime | Filesystem | Local I/O | Attachments and caches | [To be determined] |

### 8.3 API Standards & Conventions

| Concern | Standard / Policy |
|---|---|
| REST API Specification | Not represented as REST resources; API is operation-driven through `webservice.php` and DB metadata (`vtiger_ws_operation`). |
| Authentication | Session-based: clients supply `sessionName` and server enforces session validity (`include/Webservices/SessionManager.php`). |
| Error envelope | JSON responses use a `State` wrapper with `success`, `result`, and `error` (`include/Webservices/State.php`). |
| Input sanitization | Parameters are sanitized through `OperationManager::sanitizeOperation` and `vtws_getParameter` in `include/Webservices/Utils.php`. |
| Pagination | [To be determined] per operation (vtiger webservices support query/list operations, but conventions are not centralized in a spec file). |
| Rate limiting | [To be determined] (not present in repository code reviewed). |

### 8.4 Interface Control Document (ICD) Summary

| Interface ID | Interface Name | Standard | Specification Location |
|---|---|---|---|
| ICD-UI-01 | Browser UI | HTTP (HTML) | `index.php`, `modules/*`, `Smarty_setup.php`, `Smarty/templates/*` |
| ICD-API-01 | vtiger JSON Webservices | vtiger operation-based JSON | `webservice.php`, `include/Webservices/*`, `Docs/API_Documentation.md` |
| ICD-SOAP-01 | SOAP service selector | SOAP | `vtigerservice.php`, `soap/*` |
| ICD-CRON-01 | Cron task execution | CLI | `vtigercron.php`, `vtlib/Vtiger/Cron.php`, `cron/*` |

## 9. Security Architecture

### 9.1 Security Model

The security posture is based on session-based authentication for UI and API flows, role/profile-based authorization for module actions, and safe dynamic inclusion controls to mitigate file inclusion and disclosure risks. UI routing is validated and permission-checked in `index.php`. API routing is validated through a combination of operation metadata lookup (`OperationManager` querying `vtiger_ws_operation`) and session enforcement (`SessionManager` using `HTTP_Session`).

Additionally, vtiger includes an explicit safe inclusion guard (`checkFileAccessForInclusion` in `include/utils/CommonUtils.php`) and a shared input purification function (`vtlib_purify` in `include/utils/VtlibUtils.php`) implemented using HTMLPurifier.

Diagram prompt: [Insert security/trust boundary diagram here.]

### 9.2 Threat Model Summary

| STRIDE Category | Threats Identified | Controls Applied | Residual Risk |
|---|---|---|---|
| Spoofing | Session hijack or session fixation for UI/API. | UI checks `$_SESSION['app_unique_key']` against `$application_unique_key` in `index.php`; API session enforced by `include/Webservices/SessionManager.php`. | Medium |
| Tampering | SQL injection or parameter tampering. | Prepared queries via `$adb->pquery` (`PearDatabase`); request purification via `vtlib_purify`; `index.php` rejects non-numeric `record`. | Medium |
| Repudiation | Users deny actions due to insufficient audit trails. | Optional audit insert into `vtiger_audit_trial` in `index.php` when enabled; logging via log4php. | Medium |
| Information Disclosure | Unsafe dynamic includes or direct file access to sensitive directories. | `index.php` validates module/action paths; `checkFileAccessForInclusion` blocks unsafe directories (`storage`, `cache`, `test`). | Medium |
| Denial of Service | Expensive list views/queries or repeated API calls. | Performance flags in `config.performance.php` and caching features in `PearDatabase`; cron frequency gating. | Medium |
| Elevation of Privilege | Bypassing `isPermitted` checks or abusing pre-login API operations. | Centralized permission checks in `index.php`; prelogin operations explicitly flagged in operation metadata and enforced by dispatcher. | Medium |

### 9.3 Security Controls by Layer

| Layer | OWASP / Control | Implementation |
|---|---|---|
| Presentation | Output encoding | `PearDatabase` query helpers apply `to_html` encoding in multiple fetch methods; templates use Smarty rendering. |
| UI Routing | Broken access control prevention | `index.php` uses `isPermitted` and module activation checks (`vtlib_isModuleActive`). |
| Input handling | Injection / XSS mitigation | `vtlib_purify` uses HTMLPurifier; webservice input sanitization through `OperationManager`. |
| File inclusion | File inclusion hardening | `checkFileAccessForInclusion` in `include/utils/CommonUtils.php` blocks restricted directories and out-of-root access. |
| Persistence | SQL injection mitigation | Prepared statements through `$adb->pquery` (in `include/database/PearDatabase.php`). |
| Background jobs | Unauthorized execution prevention | `vtigercron.php` allows execution in CLI or authenticated session with matching app key. |

## 10. Deployment Architecture

### 10.1 Deployment Diagram

Diagram prompt: [Insert deployment diagram here.]

### 10.2 High Availability Design

| Concern | Design Decision |
|---|---|
| Web tier availability | Multiple PHP instances behind a load balancer are feasible, as the codebase is shared and stateless between requests except for sessions and filesystem state. |
| Database availability | [To be determined] (typically a primary/replica or managed DB offering, but not defined in repository). |
| Filesystem availability | Shared filesystem or object storage gateway is required for attachments if multiple web nodes are used, because attachments are written under `storage/`. |
| Background processing | Run `vtigercron.php` from a single scheduler node to reduce duplicate execution risk unless tasks are designed for concurrency. |

### 10.3 Environment Strategy

| Environment | Purpose | Configuration Differences | Promotion Gate |
|---|---|---|---|
| Development | Local development and debugging | [To be determined] | Code review + basic smoke tests |
| Test / QA | Verification and regression | [To be determined] | Automated and/or manual acceptance tests |
| Production | Live CRM usage | [To be determined] | Change management approval + rollout plan |

### 10.4 Capacity Planning & Sizing

| Component | Baseline Load | Peak Load Assumption | Initial Sizing | Scaling Trigger |
|---|---|---|---|---|
| PHP web runtime | [To be determined] | [To be determined] | [To be determined] | Increased request latency / CPU saturation |
| Database | [To be determined] | [To be determined] | [To be determined] | Slow query rate / connection saturation |
| Filesystem (attachments) | [To be determined] | [To be determined] | [To be determined] | Storage utilization thresholds |
| Cron runner | [To be determined] | [To be determined] | [To be determined] | Task backlog or timeouts |

### 10.5 CI/CD Pipeline Design

| Stage | Action |
|---|---|
| Source | Pull from VCS and validate repository integrity. |
| Build | [To be determined] (PHP projects often require linting and packaging rather than compilation). |
| Deploy to Dev | [To be determined] |
| Deploy to Staging | [To be determined] |
| Deploy to Production | [To be determined] |
| Post-Deploy | Run smoke checks for `index.php`, `webservice.php`, and `vtigercron.php`; verify logs and database connectivity. |

## 11. Non-Functional Requirements

| NFR ID | ISO 25010 Quality Attribute | Target | Design Response | Acceptance Test |
|---|---|---|---|---|
| NFR-01 | Security | Prevent unsafe file inclusion | Validate module/action in `index.php` and restrict includes via `checkFileAccessForInclusion`. | Attempt to include a file under `storage/` through an include path should fail with restricted access. |
| NFR-02 | Reliability | Consistent transactional saves | `CRMEntity::saveentity` uses DB transactions via `PearDatabase::startTransaction` and `completeTransaction`. | Save an entity; verify module tables and `vtiger_crmentity` are consistent even on partial failure. |
| NFR-03 | Maintainability | Standardized module lifecycle | Modules extend/compose `CRMEntity` and follow consistent action script conventions. | Add a module action following existing conventions and verify dispatch works. |
| NFR-04 | Performance efficiency | Reduce repeated DB work | `PearDatabase` supports query result caching and performance flags (`config.performance.php`). | Enable/disable caching and compare response times for repeated queries. |
| NFR-05 | Portability | Support common PHP-hosted environments | Entry points are standard PHP scripts; DB abstraction via ADODB. | Deploy on a standard PHP web server with supported DB driver. |
| NFR-06 | Operability | Observable behavior via logs and status | Loggers via log4php and cron status fields (`vtiger_cron_task.laststart/lastend/status`). | Execute cron and confirm state transitions in DB and log outputs. |

## 12. Architecture Decision Records (ADRs)

| Field | Value |
|---|---|
| ID / Status | ADR-H001 / Accepted (legacy, observed) |
| Date | [To be determined] |
| Decision Makers | [To be determined] |
| SAD ADR Reference | [To be determined] |
| Context | The codebase must support many CRM modules with a shared persistence model and a convention-based UI routing mechanism. |
| Decision | Use a monolithic PHP architecture with a front controller (`index.php`) that dynamically includes module action scripts, and a shared entity framework (`data/CRMEntity.php`) to standardize lifecycle and persistence. |
| Alternatives Considered | [To be determined] |
| Rationale | The repository structure and runtime behavior demonstrate that this approach is the implemented design and supports rapid module addition via conventions. |
| Consequences | Security controls around dynamic includes and input handling are critical; deployments must manage shared database and filesystem storage. |

## 13. Risks & Open Issues

### 13.1 Design Risks

| ID | Risk | Impact | Mitigation |
|---|---|---|---|
| R-01 | Misconfiguration of DB-driven webservice operations (`vtiger_ws_operation`) can expose unintended handlers. | High | Restrict admin access to operation configuration; review handler paths; enforce safe include practices. |
| R-02 | Multi-node deployments without shared attachment storage can break file access. | High | Use shared storage or refactor attachment storage strategy. |
| R-03 | Dynamic inclusion patterns increase attack surface if new call sites omit include guards. | High | Enforce use of `checkFileAccessForInclusion` for all request-driven includes. |
| R-04 | Cron tasks may overlap or duplicate work if scheduled concurrently. | Medium | Run a single cron dispatcher instance; design tasks to be idempotent where possible. |

### 13.2 Open Design Issues

| ID | Issue | Owner | Target Resolution |
|---|---|---|---|
| OI-01 | Define and publish formal API contracts (OpenAPI where applicable). | [To be determined] | [To be determined] |
| OI-02 | Define HA and backup strategy for DB and filesystem stores. | [To be determined] | [To be determined] |
| OI-03 | Define SLOs and operational monitoring standards. | [To be determined] | [To be determined] |

## 14. Operational Architecture

### 14.1 Service Level Objectives (SLOs)

| SLO ID | SLO Definition | Target | Error Budget (30 days) |
|---|---|---|---|
| SLO-01 | UI availability for authenticated users | [To be determined] | [To be determined] |
| SLO-02 | Webservice availability for authenticated operations | [To be determined] | [To be determined] |
| SLO-03 | Cron task execution completion within schedule | [To be determined] | [To be determined] |

### 14.2 Observability Design

| Pillar | Standard / Tooling |
|---|---|
| Logging | log4php configuration (see `include/logging.php` usage and `log4php.properties` in repository). |
| Security logging | Dedicated security logger usage (`LoggerManager::getLogger('SECURITY')` in `index.php`). |
| Job observability | Cron task state persisted in DB (`vtiger_cron_task`) and printed execution output in `vtigercron.php`. |
| Audit trails | Optional DB audit insert in `index.php` (`vtiger_audit_trial`) when enabled by `user_privileges/audit_trail.php`. |

### 14.3 Incident Management

| Stage | Process |
|---|---|
| Detect | Monitor logs and user reports; monitor cron output and `vtiger_cron_task` status transitions. |
| Triage | Identify affected entry point (UI/API/Cron/Installer) and isolate failing component. |
| Mitigate | Roll back recent changes, disable failing cron tasks, apply configuration corrections. |
| Resolve | Apply code fix or configuration fix; validate via smoke tests on entry scripts. |
| Learn | Record ADR/incident notes and update this HLD and operational runbooks. |

## 15. Glossary

| Term | Definition |
|---|---|
| ADR | Architecture Decision Record, a structured record of an architectural choice and its rationale. |
| C4 Model | A set of hierarchical diagrams (Context, Container, Component, Code) for describing software architecture. |
| Cron | A scheduler mechanism; in vtiger, it refers to `vtigercron.php` running tasks from `vtiger_cron_task`. |
| CRMEntity | vtiger base class for CRM modules implementing shared persistence and lifecycle behaviors (`data/CRMEntity.php`). |
| Handler file | A PHP file executed as part of an operation (webservice handler) or cron task (cron handler). |
| HLD | High Level Design document describing architecture, components, data, integration, deployment, and NFRs. |
| Operation (webservice) | A named API operation resolved from `vtiger_ws_operation` and executed by `OperationManager`. |
| PearDatabase | vtiger database abstraction built on ADODB (`include/database/PearDatabase.php`). |
| SessionName | Webservice session identifier returned and required for authenticated operations (`webservice.php`). |
| vtlib | vtiger library layer providing extension and runtime frameworks such as `Vtiger_Cron`. |
