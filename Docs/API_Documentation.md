# vtiger CRM API Documentation

## Overview

This repository is a vtiger CRM 5.4.0 monolithic PHP application. It exposes multiple HTTP integration surfaces. The most stable and intended-for-integration API is the JSON Web Services endpoint at `webservice.php`. In addition, vtiger includes multiple SOAP services exposed via `vtigerservice.php`, and it contains internal “AJAX” endpoints used by the server-rendered UI (primarily via a dynamic include dispatcher). There are also a few direct HTTP endpoints for downloading attachments/documents.

This document consolidates the available API entrypoints found in this repository, explains how requests and responses are structured, and provides usage examples based on the implementation.

## Base URL and general conventions

All endpoint paths in this document are relative to the vtiger web root. Examples assume a base URL like `https://your-vtiger-host/`.

Many endpoints are classic PHP scripts that accept parameters as query-string and/or `application/x-www-form-urlencoded` POST bodies. Unless otherwise stated, endpoints do not use REST-style JSON request bodies; instead they read from `$_GET`, `$_POST`, or `$_REQUEST`.

## API surfaces at a glance

The repository provides the following top-level API entrypoints.

| API surface | Entrypoint | Primary format | Intended usage |
|---|---|---|---|
| JSON Web Services API | `webservice.php` | JSON over HTTP GET/POST | External integrations and programmatic CRUD/query |
| SOAP service dispatcher | `vtigerservice.php` | SOAP (NuSOAP) | Legacy plugins and customer portal SOAP integrations |
| Internal AJAX dispatcher | `include/Ajax/CommonAjax.php` (via many `*Ajax.php` scripts) | Varies (often HTML fragments / plain text) | Internal UI (not stable for external integrations) |
| File download endpoints | `modules/Documents/DownloadFile.php`, `modules/uploads/downloadfile.php` | Binary download | UI and integration downloads |

## JSON Web Services API (`webservice.php`)

### Endpoint

The Web Services endpoint is implemented by `webservice.php`.

It always responds with JSON (`Content-type: application/json`) and wraps results in a common envelope (see `include/Webservices/State.php` and `webservice.php`).

### Operation routing model

Requests are routed by an `operation` parameter. `webservice.php` normalizes it to lowercase:

- `operation` is required.
- `format` is optional and defaults to `json`.
- `sessionName` is required for all operations except pre-login operations (such as `getchallenge` and `login`), as defined in the database table `vtiger_ws_operation` (see `include/Webservices/OperationManager.php` for the lookup).

Although the handlers for standard operations exist in `include/Webservices/*.php`, the actual list of enabled operations and their parameter definitions are driven by database configuration (`vtiger_ws_operation` and `vtiger_ws_operation_parameters`).

### Request formats (GET vs POST)

The request method used for a given operation is also driven by `vtiger_ws_operation.type` and interpreted by `OperationManager::getOperationInput()`:

- If the operation type is `GET`, inputs are read from `$_GET`.
- If the operation type is `POST`, inputs are read from `$_POST`.
- Otherwise, inputs are read from `$_REQUEST`.

In practice, vtiger’s built-in client uses GET for read-like operations (`getchallenge`, `describe`, `retrieve`, `query`, `listtypes`) and POST for write-like operations (`login`, `create`, `update`, `delete`, `logout`), but you should treat the database configuration as authoritative in a given deployment.

### Encoded (JSON) parameters

Operation parameters can be declared as type `encoded` in `vtiger_ws_operation_parameters`. When a parameter is `encoded`, `OperationManager::handleType()` JSON-decodes it before passing it to the handler.

This is how operations like `create`, `update`, and `revise` receive the `element` parameter as an array/object rather than a raw string.

### Response envelope

All responses follow the same JSON envelope written by `webservice.php`:

Successful response:

```json
{
  "success": true,
  "result": {}
}
```

Error response:

```json
{
  "success": false,
  "error": {
    "code": "SOME_ERROR_CODE",
    "message": "Human-readable message"
  }
}
```

The `error.code` and `error.message` fields come from `WebServiceException` (`include/Webservices/WebServiceError.php`) and standard codes are defined in `include/Webservices/WebServiceErrorCode.php`.

### Common identifiers

#### Webservice record IDs

Many operations use a webservice-style record id of the form:

- `<entityId>x<recordId>`

This is created by `vtws_getId($objId, $elemId)` in `include/Webservices/Utils.php`.

For example, `19x1` is typically the admin user (entityId 19, recordId 1), but the entityId values are determined by rows in `vtiger_ws_entity`.

#### Common parameters

Across operations you will typically see:

- `operation` (string): operation name, case-insensitive but normalized to lowercase in `webservice.php`.
- `sessionName` (string): webservice session id. Required for all non-prelogin operations.
- `format` (string): defaults to `json`. The code only defines `json` as an available format.

### Authentication and session model

#### Challenge + access key (md5) login

The login mechanism is challenge-response and uses a per-user “access key” stored in the database (`vtiger_users.accesskey`).

The built-in JS client in `modules/com_vtiger_workflow/resources/vtigerwebservices.js` shows the expected flow:

1. Call `getchallenge` with `username` to obtain a token.
2. Compute `md5(token + accessKey)` where `accessKey` is the user’s access key.
3. Call `login` with `username` and the computed hash.

The server-side implementation is in:

- `include/Webservices/AuthToken.php` (`vtws_getchallenge`)
- `include/Webservices/Login.php` (`vtws_login`)

The token expires after 5 minutes (`AuthToken.php` stores `expireTime = time() + 5 minutes`).

#### Webservice sessions (`sessionName`)

Sessions are managed by `include/Webservices/SessionManager.php` using `include/HTTP_Session/Session.php`. Key characteristics:

- Cookies are disabled for the webservice session flow (`HTTP_Session::useCookies(false)`), so `sessionName` is the explicit session identifier for API calls.
- Maximum session lifespan is configured in code as one day (`$maxWebServiceSessionLifeSpan = 86400`).
- Maximum idle time is configured as 30 minutes (`$maxWebServiceSessionIdleTime = 1800`).
- Expired/idle/invalid sessions raise WebServiceException with codes like `SESSION_EXPIRED`, `SESSION_LEFT_IDLE`, or `INVALID_SESSIONID`.

#### Extending a UI (PHP) session: `extendsession`

`webservice.php` contains special handling for `operation=extendsession` to “adopt” an existing PHP session. It tries to read `PHPSESSID` from `$_REQUEST['PHPSESSID']` or from cookies, then calls `SessionManager::startSession($sessionId, $adoptSession=true)`.

This is mainly used by in-app JavaScript to bridge an interactive UI session to a webservice `sessionName`.

### Core operations and usage

The operation names below are the conventional vtiger 5.4.0 ones implied by the handler files in `include/Webservices/`. Availability still depends on the `vtiger_ws_operation` database table.

#### `getchallenge` (pre-login)

Purpose: Generate a short-lived challenge token for a username.

Request parameters:

- `operation=getchallenge`
- `username` (string)

Example:

```bash
curl -s "https://your-vtiger-host/webservice.php?operation=getchallenge&username=admin"
```

Response `result` (from `include/Webservices/AuthToken.php`):

- `token` (string)
- `serverTime` (epoch seconds)
- `expireTime` (epoch seconds)

#### `login` (pre-login)

Purpose: Authenticate using `md5(token + accessKey)` and start a webservice session.

Request parameters:

- `operation=login`
- `username` (string)
- `accessKey` (string): the computed hash

Example:

```bash
curl -s -X POST "https://your-vtiger-host/webservice.php" \
  -d "operation=login" \
  -d "username=admin" \
  -d "accessKey=MD5_OF_TOKEN_PLUS_USER_ACCESSKEY"
```

Response `result` is constructed by `OperationManager::runOperation()` for pre-login operations and includes:

- `sessionName` (string)
- `userId` (webservice id string)
- `version` (API version, set to `0.22` in `webservice.php`)
- `vtigerVersion` (from `vtiger_version.current_version` via `vtws_getVtigerVersion()`)

#### `logout`

Purpose: Destroy the webservice session.

Request parameters:

- `operation=logout`
- `sessionName` (string)

Example:

```bash
curl -s -X POST "https://your-vtiger-host/webservice.php" \
  -d "operation=logout" \
  -d "sessionName=YOUR_SESSION"
```

Successful response:

```json
{ "success": true, "result": { "message": "successfull" } }
```

#### `listtypes`

Purpose: Return the list of modules/entities accessible to the user.

Request parameters:

- `operation=listtypes`
- `sessionName`

The handler `include/Webservices/ModuleTypes.php` returns:

- `types`: list of accessible types
- `information`: label and singular label per type and whether it is an “entity” module

Example:

```bash
curl -s "https://your-vtiger-host/webservice.php?operation=listtypes&sessionName=YOUR_SESSION"
```

#### `describe`

Purpose: Return metadata about a module/entity, including fields and types.

Request parameters:

- `operation=describe`
- `elementType` (string, e.g., `Leads`, `Contacts`)
- `sessionName`

Example:

```bash
curl -s "https://your-vtiger-host/webservice.php?operation=describe&elementType=Contacts&sessionName=YOUR_SESSION"
```

The result structure is produced by the entity handler (for modules this is typically `include/Webservices/VtigerModuleOperation.php`), and includes a `fields` array with field-level metadata such as `name`, `label`, `mandatory`, `editable`, and `type`.

#### `query`

Purpose: Execute a VTQL query string and return records.

Request parameters:

- `operation=query`
- `query` (string VTQL, e.g. `select id, firstname from Contacts;`)
- `sessionName`

Example:

```bash
curl -s --get "https://your-vtiger-host/webservice.php" \
  --data-urlencode "operation=query" \
  --data-urlencode "sessionName=YOUR_SESSION" \
  --data-urlencode "query=select id, firstname, lastname from Contacts where lastname like 'Sm%';"
```

The handler in `include/Webservices/Query.php` determines the module from the `FROM` clause and enforces module access and read permission.

#### `retrieve`

Purpose: Retrieve one record by webservice record ID.

Request parameters:

- `operation=retrieve`
- `id` (string, webservice id like `12x74`)
- `sessionName`

Example:

```bash
curl -s "https://your-vtiger-host/webservice.php?operation=retrieve&id=12x74&sessionName=YOUR_SESSION"
```

#### `create`

Purpose: Create a record in a module.

Request parameters:

- `operation=create`
- `elementType` (string)
- `element` (encoded JSON object containing field values)
- `sessionName`

Example:

```bash
curl -s -X POST "https://your-vtiger-host/webservice.php" \
  -d "operation=create" \
  -d "sessionName=YOUR_SESSION" \
  -d "elementType=Leads" \
  --data-urlencode 'element={"lastname":"Doe","company":"Example Inc","assigned_user_id":"19x1"}'
```

The handler in `include/Webservices/Create.php` enforces:

- the module is accessible (`vtws_listtypes`)
- write access (`$meta->hasWriteAccess()`)
- reference-field integrity (IDs must refer to permitted modules)
- assignment privileges for owner fields like `assigned_user_id`

#### `update`

Purpose: Update an existing record. The `element` must include `id`.

Request parameters:

- `operation=update`
- `element` (encoded JSON object; must contain `id`)
- `sessionName`

Example:

```bash
curl -s -X POST "https://your-vtiger-host/webservice.php" \
  -d "operation=update" \
  -d "sessionName=YOUR_SESSION" \
  --data-urlencode 'element={"id":"12x74","lastname":"Smith"}'
```

The handler in `include/Webservices/Update.php` checks existence, permissions, and reference fields similarly to `create`.

#### `revise`

Purpose: Revise an existing record with slightly different mandatory-field semantics.

Request parameters:

- `operation=revise`
- `element` (encoded JSON object; must contain `id`)
- `sessionName`

Implementation: `include/Webservices/Revise.php`.

#### `delete`

Purpose: Delete one record by webservice record ID.

Request parameters:

- `operation=delete`
- `id` (webservice id)
- `sessionName`

Example:

```bash
curl -s -X POST "https://your-vtiger-host/webservice.php" \
  -d "operation=delete" \
  -d "sessionName=YOUR_SESSION" \
  -d "id=12x74"
```

Implementation: `include/Webservices/Delete.php`.

#### `sync` (record updates and deletes since a timestamp)

The repository includes a synchronization handler in `include/Webservices/GetUpdates.php` named `vtws_sync($mtime,$elementType,$syncType,$user)`. In typical vtiger deployments, the operation name is `sync`, but the exact operation name still depends on the `vtiger_ws_operation` configuration.

Functionality summary:

- Returns `updated` (records) and `deleted` (ids) since a given modification time.
- Supports limiting, and a `more` flag to indicate additional pages.
- Accepts `mtime` as epoch seconds and returns `lastModifiedTime`.

Because the operation mapping is database-driven, confirm the exact parameter names and operation name in `vtiger_ws_operation` and `vtiger_ws_operation_parameters` for your instance before relying on it.

#### `convertlead`

The repository includes a webservice lead conversion handler in `include/Webservices/ConvertLead.php` named `vtws_convertlead($entityvalues, $user)`. Operation availability and parameter wiring are database-driven; if enabled, the operation typically receives an encoded structure describing the lead id and target entities (Accounts/Contacts/Potentials) and returns newly created entity ids.

### Common error codes

Standard error codes are defined in `include/Webservices/WebServiceErrorCode.php`. Common ones you will see during integration include:

- `AUTHENTICATION_REQUIRED`, `AUTHENTICATION_FAILURE`
- `INVALID_USER_CREDENTIALS`, `INVALID_AUTH_TOKEN`, `ACCESSKEY_UNDEFINED`
- `ACCESS_DENIED`
- `RECORD_NOT_FOUND`, `INVALID_ID_ATTRIBUTE`, `REFERENCE_INVALID`
- `SESSION_EXPIRED`, `SESSION_LEFT_IDLE`, `INVALID_SESSIONID`
- `DATABASE_QUERY_ERROR`, `INTERNAL_SERVER_ERROR`

## SOAP services (`vtigerservice.php`)

### Endpoint and service selection

SOAP services are dispatched by `vtigerservice.php` using a `service` query parameter (see `vtigerservice.php`):

- `vtigerservice.php?service=outlook` includes `soap/vtigerolservice.php`
- `vtigerservice.php?service=customerportal` includes `soap/customerportal.php`
- `vtigerservice.php?service=webforms` includes `soap/webforms.php`
- `vtigerservice.php?service=firefox` includes `soap/firefoxtoolbar.php`
- `vtigerservice.php?service=wordplugin` includes `soap/wordplugin.php`
- `vtigerservice.php?service=thunderbird` includes `soap/thunderbirdplugin.php`

Each service script uses NuSOAP (`include/nusoap/nusoap.php`) and acts as a SOAP server that reads the raw request from `php://input` and processes it with `$server->service($HTTP_RAW_POST_DATA)`.

### SOAP service namespaces and WSDL

The Firefox toolbar and Customer Portal services configure their WSDL and namespace in code:

- Firefox toolbar uses `$server->configureWSDL('vtigersoap')` and `$NAMESPACE = 'http://www.vtiger.com/products/crm'` (`soap/firefoxtoolbar.php`).
- Customer Portal uses `$server->configureWSDL('customerportal')` and the same namespace (`soap/customerportal.php`).

In a running instance, the WSDL is typically retrievable by appending `?wsdl` to the SOAP endpoint URL, subject to the NuSOAP configuration and server environment.

### Firefox toolbar SOAP service (`service=firefox`)

Endpoint:

- `vtigerservice.php?service=firefox`

Authentication model:

- The method `LogintoVtigerCRM(user_name, password, version)` creates a session id and stores it in the database table `vtiger_soapservice` with type `FireFox`.
- Most other methods accept `(username, session)` and validate the session with `validateSession($username, $sessionid)` which compares against `vtiger_soapservice`.

Registered methods include (see `soap/firefoxtoolbar.php`):

- `LogintoVtigerCRM(user_name, password, version) -> logindetails`
- Permission checks:
  - `CheckLeadPermission(username, session) -> string`
  - `CheckContactPermission(username, session) -> string`
  - `CheckAccountPermission(username, session) -> string`
  - `CheckTicketPermission(username, session) -> string`
  - `CheckVendorPermission(username, session) -> string`
  - `CheckProductPermission(username, session) -> string`
  - `CheckNotePermission(username, session) -> string`
  - `CheckSitePermission(username, session) -> string`
  - `CheckRssPermission(username, session) -> string`
- Create operations:
  - `create_lead_from_webform(username, session, lastname, firstname, email, phone, company, country, description) -> string`
  - `create_contacts(user_name, session, firstname, lastname, phone, mobile, email, street, city, state, country, zipcode) -> string`
  - `create_account(username, session, accountname, email, phone, primary_address_street, primary_address_city, primary_address_state, primary_address_postalcode, primary_address_country) -> string`
  - `create_ticket_from_toolbar(username, session, title, description, priority, severity, category, user_name, parent_id, product_id) -> string`
  - `create_vendor_from_webform(username, session, vendorname, email, phone, website) -> string`
  - `create_product_from_webform(username, session, productname, productcode, website) -> string`
  - `create_note_from_webform(username, session, title, notecontent) -> string`
  - `create_site_from_webform(username, session, portalname, portalurl) -> string`
  - `create_rss_from_webform(username, session, rssurl) -> string`
- Picklist retrieval:
  - `GetPicklistValues(username, session) -> combo_values_array`

Return types are declared in the service using NuSOAP complex types `logindetails` and `combo_values_array`.

### Customer Portal SOAP service (`service=customerportal`)

Endpoint:

- `vtigerservice.php?service=customerportal`

Authentication model:

- `authenticate_user` issues a session id and stores it in `vtiger_soapservice` with type `customer`.
- Many methods take a “common_array” input that includes keys like `id` (customer/contact id) and `sessionid` and then call `validateSession($id, $sessionid)` which compares with the stored session.

The service registers a large set of methods in `soap/customerportal.php`. Commonly used operations include:

- Authentication and session:
  - `authenticate_user(fieldname: common_array) -> common_array`
  - `update_login_details(fieldname: common_array) -> string`
  - `send_mail_for_password(email: string) -> string`
- Ticket operations:
  - `create_ticket(fieldname: common_array) -> common_array`
  - `get_tickets_list(fieldname: common_array) -> common_array`
  - `get_ticket_comments(fieldname: common_array) -> common_array`
  - `update_ticket_comment(fieldname: common_array) -> common_array`
  - `close_current_ticket(fieldname: common_array) -> string`
  - `get_ticket_attachments(fieldname: common_array) -> common_array`
  - `add_ticket_attachment(fieldname: common_array) -> common_array`
  - `get_filecontent(fieldname: common_array) -> common_array`
- Knowledge base (FAQ):
  - `get_KBase_details(fieldname: common_array) -> common_array1`
  - `save_faq_comment(fieldname: common_array) -> common_array`
- Picklists and metadata:
  - `get_combo_values(fieldname: common_array) -> common_array`
  - `get_picklists(fieldname: common_array) -> common_array`
  - `get_modules() -> field_details_array`
  - `show_all(module: string) -> string`
- Documents, quotes, invoices, products and related lists:
  - `get_list_values(id, block, sessionid, only_mine) -> field_datalist_array`
  - `get_details(id, block, contactid, sessionid) -> field_details_array`
  - `get_pdf(id, block, contactid, sessionid) -> field_datalist_array`
  - `get_documents(id, module, customerid, sessionid) -> field_details_array`
  - `get_filecontent_detail(id, folderid, block, contactid, sessionid) -> get_ticket_attachments_array`
  - `updateCount(id) -> string`
- Project-related:
  - `get_project_components(id, module, customerid, sessionid) -> field_details_array`
  - `get_project_tickets(id, module, customerid, sessionid) -> field_details_array`

In this repository, Customer Portal methods often return arrays of field/value pairs or HTML fragments embedded in strings, because the portal is designed as a tightly coupled companion application rather than a general-purpose API.

### Other SOAP services

The repository also includes SOAP services intended for legacy plugins:

- Outlook: `vtigerservice.php?service=outlook` (`soap/vtigerolservice.php`)
- Thunderbird: `vtigerservice.php?service=thunderbird` (`soap/thunderbirdplugin.php`)
- Word plugin: `vtigerservice.php?service=wordplugin` (`soap/wordplugin.php`)
- Webforms: `vtigerservice.php?service=webforms` (`soap/webforms.php`)

These files implement additional NuSOAP servers and expose methods for client-side integrations (for example, contact lookup, email tracking, or office plugin interactions). The exact WSDL and method signatures are defined in the corresponding `soap/*.php` file, and should be treated as the authoritative source for those specific services in this repository.

## Internal AJAX endpoints (UI-oriented)

### Common dispatcher: `include/Ajax/CommonAjax.php`

vtiger’s classic UI uses many module-specific `*Ajax.php` scripts that (directly or indirectly) include `include/Ajax/CommonAjax.php`.

This dispatcher is dynamic (see `include/Ajax/CommonAjax.php`):

- It constructs a file path: `modules/<module>/<file>.php`.
- If that file does not exist, it falls back to: `modules/Vtiger/<file>.php`.
- It then includes the file after calling `checkFileAccessForInclusion($moduleFilepath)`.

This design means the request/response formats of AJAX calls vary widely and are not stable as an external API. Integrations should prefer `webservice.php` whenever possible.

### TagCloud AJAX handler: `include/Ajax/TagCloud.php`

`include/Ajax/TagCloud.php` is a concrete example of an internal AJAX endpoint with explicit parameters.

It expects:

- `ajxaction`: one of `SAVETAG`, `GETTAGCLOUD`, `DELETETAG`
- `recordid`: numeric CRM record id
- `module`: module name (string)
- `tagfields`: tag content (used for `SAVETAG`)
- `tagid`: numeric tag id (used for `DELETETAG`)

The handler returns HTML fragments (tag cloud HTML) or short status strings such as `SUCCESS` or `:#:FAILURE`.

Because output is HTML and not JSON, this endpoint is best understood as an internal UI API.

## Direct HTTP endpoints for downloads

### Documents download: `modules/Documents/DownloadFile.php`

Endpoint:

- `modules/Documents/DownloadFile.php`

Parameters:

- `fileid`: attachment id (`vtiger_attachments.attachmentsid`)
- `folderid`: document folder id (checked against `vtiger_notes.folderid`)

Response:

- Streams the file bytes and sets download headers such as `Content-Disposition: attachment; filename=<originalname>`.

Example shape:

- `https://your-vtiger-host/modules/Documents/DownloadFile.php?fileid=<ATTACHMENT_ID>&folderid=<FOLDER_ID>`

### Generic attachment download: `modules/uploads/downloadfile.php`

Endpoint:

- `modules/uploads/downloadfile.php`

Parameters:

- `fileid`: attachment id (`vtiger_attachments.attachmentsid`)
- `entityid`: optional record id used to check deletion status in `vtiger_crmentity.deleted`
- `return_module`: module name (used by UI flows)

Response:

- If `entityid` is provided and marked deleted, it prints a localized “record deleted” message (`$app_strings['LBL_RECORD_DELETE']`).
- Otherwise, it streams the attachment content with download headers.

Example shape:

- `https://your-vtiger-host/modules/uploads/downloadfile.php?fileid=<ATTACHMENT_ID>&entityid=<CRMID>&return_module=<MODULE>`

## Security and stability notes

The JSON Web Services API enforces authentication (`sessionName`) for non-prelogin operations, and most handlers enforce module and record-level access checks. SOAP services implement separate session models (often backed by `vtiger_soapservice`) and may have different authentication and authorization behavior.

The internal AJAX dispatcher (`include/Ajax/CommonAjax.php`) dynamically includes PHP scripts based on request parameters. Even though it uses `checkFileAccessForInclusion`, this mechanism is intended for the UI and should not be treated as a stable public API.

For external integrations, `webservice.php` is the recommended and most consistent interface implemented in this repository.

## Sources

This document is based on the following repository sources:

- JSON Web Services API: `webservice.php`, `include/Webservices/*.php`, and `modules/com_vtiger_workflow/resources/vtigerwebservices.js`
- SOAP services: `vtigerservice.php`, `soap/firefoxtoolbar.php`, and `soap/customerportal.php`
- Internal AJAX and download endpoints: `include/Ajax/CommonAjax.php`, `include/Ajax/TagCloud.php`, `modules/Documents/DownloadFile.php`, and `modules/uploads/downloadfile.php`
- Existing API reference: `Docs/VTigerCRM-API-Reference.md`
