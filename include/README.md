# `include/` — shared framework services

This directory contains the shared “platform” code used by most of vtigerCRM. Many entrypoints (`index.php`, `Popup.php`, `webservice.php`, etc.) require files from here.

## Key areas

### Utilities
- `utils/utils.php`  
  Foundational utilities used across the app (language loading, caching helpers, UI helpers, and common functions referenced by entrypoints).

### Database
- `database/PearDatabase.php`  
  Primary DB abstraction layer built on ADODB (`/adodb`). Common APIs include `query()` and `pquery()` (prepared statements) and optional caching.

### Webservices (JSON API)
- `Webservices/`  
  Implementation of JSON webservices dispatched by `webservice.php`. Key files include:
  - `OperationManager.php` — resolves operations and executes them
  - `SessionManager.php` — session lifecycle for `sessionName`
  - `Utils.php` — shared webservice utilities (ID conversion, entity registration helpers)
  - handlers like `Login.php`, `Query.php`, `Retrieve.php`, etc.

### Query generation
- `QueryGenerator/QueryGenerator.php`  
  Builds SQL queries for modules with filters/joins; used in list views, reporting, and some webservice flows.

### AJAX dispatcher
- `Ajax/CommonAjax.php`  
  Includes module PHP scripts based on request parameters (after access checks). Many module `*Ajax.php` scripts delegate into this.

## Related docs
- `Docs/Repository-Structure.md`
- `Docs/Webservice-Internals.md`
- `Docs/VTigerCRM-API-Reference.md`
