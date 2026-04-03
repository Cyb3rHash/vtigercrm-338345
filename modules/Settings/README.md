# Settings module (`modules/Settings/`)

The **Settings** area is vtiger’s primary **administration/configuration hub**. Unlike most business modules, Settings is implemented as a collection of admin action scripts rather than a single CRMEntity.

## What it does
- User/role/profile administration (and permission/sharing recalculation)
- Module management (install/enable/disable/import packages)
- Company/organization configuration
- Picklists, currencies, email templates, notifications, audit trail, etc.
- Mail scanner configuration (`MailScanner/`)

## Key entrypoints/patterns
- Many admin pages are direct action scripts (e.g., `EditCompanyDetails.php`, `ListProfiles.php`, `TaxConfig.php`, …)
- AJAX: `SettingsAjax.php`
- Submodules:
  - `ModuleManager/` (see `ModuleManager/README.md`)
  - `MailScanner/` (see `MailScanner/README.md`)

Related:
- `modules/Users/README.md`
- `Docs/Repository-Structure.md`
