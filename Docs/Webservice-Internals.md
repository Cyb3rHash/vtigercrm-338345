# Webservice internals (JSON) — `webservice.php`

This document explains how the JSON webservices layer works internally in this repository (vtiger CRM v5.4.0). For request/response shapes and example calls, see `Docs/VTigerCRM-API-Reference.md`.

## Key files

- Entry point: `webservice.php`
- Dispatcher/operation runner: `include/Webservices/OperationManager.php`
- Session lifecycle: `include/Webservices/SessionManager.php`
- Shared utilities: `include/Webservices/Utils.php`
- Representative handlers:
  - `include/Webservices/Login.php` (`vtws_login`)
  - `include/Webservices/Query.php` (`vtws_query`)
  - `include/Webservices/Retrieve.php` (`vtws_retrieve`)

## Dispatch model: operation registry in the database

Unlike typical “hardcoded routes”, vtiger’s webservice endpoint is **metadata-driven**.

At runtime:

1. The client calls `webservice.php` with an `operation` parameter (e.g., `login`, `query`, `retrieve`).
2. `webservice.php` constructs an `OperationManager` instance and asks it to **load operation details** from the database.
3. The DB describes:
   - whether the operation is allowed pre-login
   - the HTTP method/parameter types
   - which PHP file(s) must be included to run the operation
   - which function to call

`OperationManager` is responsible for looking up the operation and collecting its parameter definitions.

## Where operation code lives

Most built-in operations are implemented as functions in `include/Webservices/*.php`.

Examples:
- `vtws_login` is implemented in `include/Webservices/Login.php`
- `vtws_query` is implemented in `include/Webservices/Query.php`
- `vtws_retrieve` is implemented in `include/Webservices/Retrieve.php`

Handlers often load module-specific webservice handlers via `VtigerWebserviceObject` and meta classes (see `include/Webservices/VtigerWebserviceObject.php`, `include/Webservices/EntityMeta.php` referenced by many handlers).

## Parameter decoding/sanitization

`OperationManager` inspects parameter types and reads parameters from the correct source (`$_GET`, `$_POST`, or `$_REQUEST`) depending on the configured request type.

One important handler feature is support for parameters declared as `encoded`: those are decoded (JSON) before being passed to the handler function.

This is used frequently for operations like `create`, `update`, `revise` where the client sends a JSON object in a parameter such as `element`.

## Session model (`sessionName`)

Webservices typically use a `sessionName` token rather than relying on browser cookies.

`webservice.php` uses `include/Webservices/SessionManager.php` which wraps `include/HTTP_Session/Session.php` and configures it for API usage (e.g., cookie usage may be disabled for this flow).

- Pre-login operations (e.g., `getchallenge`, `login`) are allowed without an existing session.
- Other operations require a valid `sessionName` and will return a structured `WebServiceException` error when invalid/expired.

## Error and response format

`webservice.php` writes JSON responses with a consistent envelope using a `State` object:

- success response: `{ "success": true, "result": ... }`
- error response: `{ "success": false, "error": { "code": "...", "message": "..." } }`

The error object originates from `WebServiceException` (see `include/Webservices/WebServiceError.php` as referenced by the API docs).

## Extending: registering new operations

Conceptually, adding a new operation involves two parts:

1. **Implementation**: a PHP function (often placed under `include/Webservices/`) that performs the operation.
2. **Registration**: a DB entry that maps an `operation` name to:
   - handler include path(s)
   - handler method/function name
   - operation parameters and types

Utility helpers for operation/entity registration live in `include/Webservices/Utils.php` (e.g., functions like `vtws_addWebserviceOperation` and `vtws_addWebserviceOperationParam` are published from there).

> Note: The actual DB schema and expected fields are part of vtiger’s webservice metadata tables; consult the vtiger documentation or your existing DB state when registering new operations.

## Debugging tips

- If an operation is “unknown”, start with:
  - the request’s `operation` value
  - operation lookup in `OperationManager`
  - whether the DB has an entry for that operation

- If an operation fails after inclusion:
  - confirm the handler file exists in `include/Webservices/`
  - inspect the handler function signature and parameter names/types

- If you see session-related errors:
  - inspect `SessionManager::isValid()`
  - confirm the client is sending `sessionName`
  - confirm idle/expire settings in SessionManager configuration
