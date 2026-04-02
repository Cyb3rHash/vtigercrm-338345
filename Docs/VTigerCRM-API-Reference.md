# VTigerCRM API Reference (Web Services, SOAP, AJAX, and HTTP Endpoints)

## Overview

This repository contains multiple API “surfaces” that can be used to integrate with vtiger CRM. The primary programmable interface is the JSON Web Services endpoint exposed by `webservice.php`. In addition, vtiger includes multiple SOAP services exposed via `vtigerservice.php`, and many in-app “AJAX” endpoints that are invoked by the server-rendered UI through `index.php` routing and module-specific `*Ajax.php` scripts. The codebase also includes a small set of direct HTTP endpoints used for file downloads.

This document focuses on the API entrypoints that are explicitly implemented in the repository and describes how requests are structured, how responses are encoded, and how these endpoints are intended to be used.

## Conventions and base URLs

All examples assume vtiger is hosted at a base URL such as:

- `https://your-vtiger-host/`

In a default deployment, the following entrypoints are available at the web root:

- `webservice.php` (JSON Web Services API)
- `vtigerservice.php` (SOAP service dispatcher)

Some other endpoints are invoked by direct script access under `modules/**` (for example document download endpoints).

## JSON Web Services API (`webservice.php`)

### Endpoint

- Path: `webservice.php`
- Content type: responses are sent as `application/json` by `setResponseHeaders()` in `webservice.php`.
- Version: `webservice.php` sets `$API_VERSION = "0.22"`.

The webservice endpoint routes requests by an `operation` parameter. The operation implementation and its expected parameters are driven by the database tables `vtiger_ws_operation` and `vtiger_ws_operation_parameters` and executed through `include/Webservices/OperationManager.php`.

### Common request parameters

Every request is expected to include:

- `operation`: the operation name. In `webservice.php` it is normalized to lowercase.
- `format` (optional): defaults to `json` in `webservice.php` via `vtws_getParameter($_REQUEST, "format","json")`.
- `sessionName`: required for most operations after login, and returned by `login` (and also by `extendsession`).

The `OperationManager` uses the operation’s configured request type (`GET` or `POST`) to decide where to read parameters from (`$_GET`, `$_POST`, or `$_REQUEST`).

Some operation parameters are declared as type `encoded` (see `OperationManager::handleType()`), meaning the parameter value will be JSON-decoded before being passed into the operation handler.

### Common response format

All responses are wrapped in a `State` object (see `webservice.php`):

Successful response:

```json
{
  "success": true,
  "result": { }
}
```

Error response:

```json
{
  "success": false,
  "error": {
    "code": "SOME_CODE",
    "message": "Human-readable error"
  }
}
```

The error object comes from `WebServiceException` in `include/Webservices/WebServiceError.php`, which stores `code` and `message` as public fields.

### Authentication and session model

#### Access keys and challenge-response login

The login flow is implemented across:

- `include/Webservices/AuthToken.php` (`vtws_getchallenge`)
- `include/Webservices/Login.php` (`vtws_login`)
- `include/Webservices/SessionManager.php` (session creation/validation)
- `webservice.php` (enforces that `sessionName` is present for non-prelogin operations)

The expected login sequence is:

1. `getchallenge` returns a temporary token (valid for 5 minutes).
2. The client computes `md5(token + user_accesskey)` where `user_accesskey` is stored in `vtiger_users.accesskey` (retrieved by `vtws_getUserAccessKey()`).
3. The client calls `login` with `username` and the computed hash (passed as parameter `accessKey` by convention in the JS client code).

#### Sessions (`sessionName`) and extend-session behavior

- After a successful `login`, `OperationManager::runOperation()` sets `"authenticatedUserId"` in the `SessionManager` and returns `sessionName` and the authenticated `userId` (as a webservice ID).
- For most operations, `webservice.php` requires that `sessionName` is present and valid unless `OperationManager::isPreLoginOperation()` is true.
- `extendsession` is handled specially in `webservice.php` and can adopt an existing PHP session by reading `PHPSESSID` from `$_REQUEST` or from cookies and then returning a new `sessionName` and `userId` via `include/Webservices/ExtendSession.php`.

The `SessionManager` uses PEAR HTTP_Session and disables cookie usage for the webservice flow with `HTTP_Session::useCookies(false)`. It enforces both maximum lifespan and idle-time constraints and raises specific `WebServiceException`s when a session is expired, idle, or invalid.

### Web Services operations (core set)

The following core operations are implemented by the repository code shown below. Exact availability can depend on the DB configuration in `vtiger_ws_operation`, but these are the standard operations referenced by the built-in JavaScript webservice client (`modules/com_vtiger_workflow/resources/vtigerwebservices.js`) and the server-side implementation files.

#### `getchallenge` (pre-login)

- Handler: `include/Webservices/AuthToken.php` → `vtws_getchallenge($username)`
- Typical method: GET (depends on DB operation configuration)
- Parameters:
  - `username` (string)

Returns a challenge token and expiry details:

```json
{
  "success": true,
  "result": {
    "token": "64f0c8b7a79e2",
    "serverTime": 1730000000,
    "expireTime": 1730000300
  }
}
```

Example (curl):

```bash
curl -s "https://your-vtiger-host/webservice.php?operation=getchallenge&username=admin"
```

#### `login` (pre-login)

- Handler: `include/Webservices/Login.php` → `vtws_login($username,$pwd)`
- Parameters:
  - `username` (string)
  - `accessKey` / `pwd` (string): the code uses `$pwd` as the handler argument, and compares it to `md5(token . accesskey)`.

Returns the created `sessionName`, the authenticated user id (webservice ID), and version fields:

```json
{
  "success": true,
  "result": {
    "sessionName": "9b8d...some-session-id...",
    "userId": "19x1",
    "version": "0.22",
    "vtigerVersion": "5.4.0"
  }
}
```

Example (curl; `ENCODED_KEY` is the md5 computed client-side):

```bash
curl -s -X POST "https://your-vtiger-host/webservice.php" \
  -d "operation=login" \
  -d "username=admin" \
  -d "accessKey=ENCODED_KEY"
```

#### `logout`

- Handler: `include/Webservices/Logout.php` → `vtws_logout($sessionId,$user)`
- Parameters:
  - `sessionName` (passed via `webservice.php` session handling)
  - Operation parameter(s) depend on DB registration; the handler expects `$sessionId` as an argument and also uses `SessionManager`.

Example (using the built-in JS client pattern, `logout` is called as a POST with sessionName attached):

```bash
curl -s -X POST "https://your-vtiger-host/webservice.php" \
  -d "operation=logout" \
  -d "sessionName=YOUR_SESSION_NAME"
```

#### `extendsession`

- Handler: `include/Webservices/ExtendSession.php` → `vtws_extendSession()`
- Parameters: handled specially in `webservice.php`; the server may adopt `PHPSESSID` from request/cookie and requires an `operation` parameter inside the request body to proceed.

Returns:

```json
{
  "success": true,
  "result": {
    "sessionName": "new-session-id",
    "userId": "19x1",
    "version": "0.22",
    "vtigerVersion": "5.4.0"
  }
}
```

#### `listtypes`

- Handler: `include/Webservices/ModuleTypes.php` → `vtws_listtypes($fieldTypeList, $user)`
- Parameters:
  - `fieldTypeList` can be empty (“all types”) or a list of field types; in practice, clients call without it to get all accessible types.

Returns:

- `types`: a list of module/entity names the user can access
- `information`: a map keyed by type name with labels/singular labels and whether it is a CRM entity (`isEntity`)

Example:

```bash
curl -s "https://your-vtiger-host/webservice.php?operation=listtypes&sessionName=YOUR_SESSION_NAME"
```

#### `describe`

- Handler: `include/Webservices/DescribeObject.php` → `vtws_describe($elementType,$user)`
- Parameters:
  - `elementType` (string), for example `Leads`, `Contacts`, `Accounts`.

The return value is the entity metadata (fields, types, constraints) as provided by the module handler.

Example:

```bash
curl -s "https://your-vtiger-host/webservice.php?operation=describe&elementType=Leads&sessionName=YOUR_SESSION_NAME"
```

#### `query`

- Handler: `include/Webservices/Query.php` → `vtws_query($q,$user)`
- Parameters:
  - `query` (string): VTQL query.

The server extracts the module from the `FROM <Module>` clause and checks access and read permissions.

Example:

```bash
curl -s --get "https://your-vtiger-host/webservice.php" \
  --data-urlencode "operation=query" \
  --data-urlencode "sessionName=YOUR_SESSION_NAME" \
  --data-urlencode "query=select id, firstname, lastname from Contacts where lastname like 'Sm%';"
```

Returns an array of records:

```json
{
  "success": true,
  "result": [
    { "id": "12x74", "firstname": "John", "lastname": "Smith", "...": "..." }
  ]
}
```

#### `retrieve`

- Handler: `include/Webservices/Retrieve.php` → `vtws_retrieve($id,$user)`
- Parameters:
  - `id` (string): webservice-style ID like `<entityId>x<recordId>`.

Example:

```bash
curl -s "https://your-vtiger-host/webservice.php?operation=retrieve&id=12x74&sessionName=YOUR_SESSION_NAME"
```

#### `create`

- Handler: `include/Webservices/Create.php` → `vtws_create($elementType,$element,$user)`
- Parameters:
  - `elementType` (string)
  - `element` (encoded JSON object): the new entity field values.

The server validates:
- that the module is accessible (`vtws_listtypes`)
- that the user has write access
- reference fields are valid webservice IDs and of permitted types
- owner assignment privileges (`assigned_user_id` and other owner fields)

Example:

```bash
curl -s -X POST "https://your-vtiger-host/webservice.php" \
  -d "operation=create" \
  -d "sessionName=YOUR_SESSION_NAME" \
  -d "elementType=Leads" \
  --data-urlencode 'element={"lastname":"Doe","company":"Example Inc","assigned_user_id":"19x1"}'
```

#### `update`

- Handler: `include/Webservices/Update.php` → `vtws_update($element,$user)`
- Parameters:
  - `element` (encoded JSON object): must include `id`.

Example:

```bash
curl -s -X POST "https://your-vtiger-host/webservice.php" \
  -d "operation=update" \
  -d "sessionName=YOUR_SESSION_NAME" \
  --data-urlencode 'element={"id":"12x74","lastname":"Smith"}'
```

#### `revise`

- Handler: `include/Webservices/Revise.php` → `vtws_revise($element,$user)`
- Parameters:
  - `element` (encoded JSON object): must include `id`.

`revise` is similar to `update`, but it uses `isUpdateMandatoryFields` logic and is typically used when partially revising a record with different mandatory-field semantics.

#### `delete`

- Handler: `include/Webservices/Delete.php` → `vtws_delete($id,$user)`
- Parameters:
  - `id` (string): webservice-style ID.

Example:

```bash
curl -s -X POST "https://your-vtiger-host/webservice.php" \
  -d "operation=delete" \
  -d "sessionName=YOUR_SESSION_NAME" \
  -d "id=12x74"
```

### JavaScript usage example (built-in client)

The repository includes a JS wrapper used by workflow UI:

- `modules/com_vtiger_workflow/resources/vtigerwebservices.js`

It demonstrates:
- `getchallenge` via GET
- `login` via POST
- passing `sessionName` for subsequent calls
- `describe`, `query`, `create`, `update`, `delete`, `extendsession`

This is representative of how vtiger expects the webservice endpoint to be used from browser-side code.

## SOAP services (`vtigerservice.php` dispatcher)

### Endpoint and routing

- Path: `vtigerservice.php`
- Service selection: query parameter `service`

`vtigerservice.php` includes different SOAP server scripts based on `$_REQUEST['service']`:

- `vtigerservice.php?service=outlook` → `soap/vtigerolservice.php`
- `vtigerservice.php?service=customerportal` → `soap/customerportal.php`
- `vtigerservice.php?service=webforms` → `soap/webforms.php`
- `vtigerservice.php?service=firefox` → `soap/firefoxtoolbar.php`
- `vtigerservice.php?service=wordplugin` → `soap/wordplugin.php`
- `vtigerservice.php?service=thunderbird` → `soap/thunderbirdplugin.php`

Each SOAP service uses NuSOAP (`include/nusoap/nusoap.php`) and serves requests by reading raw POST data from `php://input` into `$HTTP_RAW_POST_DATA` and invoking `$server->service($HTTP_RAW_POST_DATA)`.

### SOAP authentication pattern (Firefox toolbar service)

The Firefox toolbar SOAP service provides a simple session model backed by the database table `vtiger_soapservice`:

- Login method: `LogintoVtigerCRM($user_name,$password,$version)`
  - Generates a random `sessionid`
  - Stores `(userid, 'FireFox', sessionid)` into `vtiger_soapservice`
- Most operations accept `username` and `session` and call `validateSession($username, $sessionid)` before executing.

The validation retrieves the stored session id from `vtiger_soapservice` for the given user and service type.

### Representative SOAP methods (Firefox toolbar)

The file `soap/firefoxtoolbar.php` registers many SOAP methods. The following are explicitly registered at the top of the file and represent its key API surface:

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

Because these are SOAP operations, clients send a SOAP envelope to `vtigerservice.php?service=firefox` with the operation name as the SOAP body method, and include the arguments exactly as registered by the NuSOAP server.

### Customer Portal SOAP service

The customer portal SOAP service (`soap/customerportal.php`) is a large API surface used by the vtiger Customer Portal integration. It registers methods that commonly accept a single structured parameter (`common_array`) that NuSOAP passes as an associative array.

Representative registered methods include:

- `authenticate_user(fieldname: common_array) -> common_array`
- `change_password(fieldname: common_array) -> common_array`
- `create_ticket(fieldname: common_array) -> common_array`
- `get_tickets_list(fieldname: common_array) -> common_array`
- `get_ticket_comments(fieldname: common_array) -> common_array`
- `get_combo_values(fieldname: common_array) -> common_array`
- `get_KBase_details(fieldname: common_array) -> common_array1`
- `update_ticket_comment(fieldname: common_array) -> common_array`
- `close_current_ticket(fieldname: common_array) -> string`
- `send_mail_for_password(email: string) -> string`
- `get_picklists(fieldname: common_array) -> common_array`
- Attachment operations:
  - `get_ticket_attachments(fieldname: common_array) -> common_array`
  - `get_filecontent(fieldname: common_array) -> common_array`
  - `add_ticket_attachment(fieldname: common_array) -> common_array`

The portal session model is also backed by `vtiger_soapservice` but uses `type='customer'` and the customer/contact id as `id`. Many methods begin by calling `validateSession($id, $sessionid)`.

## AJAX / action endpoints in the web UI

### Overview

In vtiger’s classic UI, many interactive features use “AJAX” endpoints that are routed through module-specific `*Ajax.php` entry scripts. A common pattern in the repository is that a module’s `*Ajax.php` script simply includes the shared dispatcher `include/Ajax/CommonAjax.php`, which then includes a module file based on request parameters.

This makes the “API” surface more dynamic: the actual code executed depends on the `module` and `file` request parameters.

### Common AJAX dispatcher (`include/Ajax/CommonAjax.php`)

The dispatcher constructs a module file path:

- Primary: `modules/<module>/<file>.php`
- Fallback: `modules/Vtiger/<file>.php`

It then calls `checkFileAccessForInclusion($moduleFilepath)` and includes the file.

This means requests routed to a module’s `*Ajax.php` entrypoint must typically include:

- `module`: the module name (e.g., `Contacts`)
- `file`: the script name to include (without `.php`)

Because the inclusion is runtime-driven, these endpoints should be treated as internal UI endpoints rather than stable external APIs.

### Example: TagCloud AJAX API (`include/Ajax/TagCloud.php`)

`include/Ajax/TagCloud.php` is a concrete AJAX endpoint with clearly defined `ajxaction` values and request parameters.

It reads:

- `ajxaction`: one of `SAVETAG`, `GETTAGCLOUD`, `DELETETAG`
- `recordid`: CRM record ID (numeric) for the tagged record
- `module`: module name for context
- `tagfields`: tag text (used for `SAVETAG`)
- `tagid`: numeric tag id (used for `DELETETAG`)

Responses:

- `SAVETAG`: returns HTML for the updated tag cloud (or `:#:FAILURE`).
- `GETTAGCLOUD`: returns HTML for the tag cloud.
- `DELETETAG`: returns `SUCCESS` or terminates with an error if `tagid` is invalid.

A typical request to save a tag (exact routing depends on which module’s TagCloud wrapper is used) sends the parameters above and expects an HTML fragment response.

### Notable module-level AJAX entrypoints

The repository includes many module entry scripts that are intended to be invoked via `index.php` routing, for example:

- `modules/Contacts/ContactsAjax.php`
- `modules/Documents/DocumentsAjax.php`
- `modules/Emails/EmailsAjax.php`
- `modules/Webmails/WebmailsAjax.php`
- `modules/Calendar/CalendarAjax.php`
- `modules/Reports/ReportsAjax.php`

Many of these include `include/Ajax/CommonAjax.php` and then rely on `file=...` to select a specific module script, while some “DetailViewAjax.php” scripts implement record updates and related list loading.

Because the request/response structure varies per module script, the most reliable way to identify the actual parameters for a specific AJAX call is to locate the JavaScript caller (often under `include/js/**` or `modules/**/**.js`) and the corresponding included PHP handler referenced by `file=...`.

## Direct HTTP endpoints for file downloads

### Documents download (`modules/Documents/DownloadFile.php`)

This script streams a document file back to the browser and sets download headers.

- Path: `modules/Documents/DownloadFile.php`
- Parameters:
  - `fileid`: attachment id (typically from `vtiger_attachments.attachmentsid`)
  - `folderid`: document folder id

Behavior (high level):

- Resolves the related `notesid` via `vtiger_seattachmentsrel`.
- Validates the note exists in the specified folder.
- Reads the attachment file at `<path>/<fileid>_<filename>`.
- Streams it with `Content-Disposition: attachment`.

Example URL shape:

- `https://your-vtiger-host/modules/Documents/DownloadFile.php?fileid=<ATTACHMENT_ID>&folderid=<FOLDER_ID>`

### Generic uploads download (`modules/uploads/downloadfile.php`)

This script downloads an attachment and can check that a related entity record has not been deleted.

- Path: `modules/uploads/downloadfile.php`
- Parameters:
  - `fileid`: attachment id
  - `entityid`: related CRM entity id (used to check `vtiger_crmentity.deleted`)
  - `return_module`: module name (used for UI flows)

Behavior:

- If `entityid` is provided and the record is deleted, it returns a localized “record deleted” message.
- Otherwise, it streams `<path>/<fileid>_<name>` from `vtiger_attachments`.

Example URL shape:

- `https://your-vtiger-host/modules/uploads/downloadfile.php?fileid=<ATTACHMENT_ID>&entityid=<CRMID>&return_module=<MODULE>`

## Security and stability notes

The JSON Web Services API enforces session validation for non-prelogin operations and performs module/record permission checks in individual handlers like `vtws_retrieve`, `vtws_query`, and `vtws_create`. The SOAP services implement their own session validation (often by comparing a `sessionid` stored in `vtiger_soapservice`). The AJAX endpoints are largely internal to the UI and frequently rely on server-side inclusion of specific scripts based on request parameters, so they should be considered less stable as an external integration surface.

For external integrations, `webservice.php` is the most consistent and strongly typed interface available in this repository.
