# `modules/` — vtiger functional modules

Each subdirectory here represents a vtiger CRM module (Accounts, Contacts, Leads, Calendar, Documents, Reports, Settings, …).

## How modules are executed (UI)

The main dispatcher is `index.php`.

At a high level:
- UI requests include `module=<ModuleName>` and `action=<ActionName>`
- `index.php` validates the request (installation, session, permissions, sanitization)
- then includes: `modules/<ModuleName>/<ActionName>.php`

Examples of common action scripts:
- `ListView.php`, `DetailView.php`, `EditView.php`
- `Save.php`, `Delete.php`, `Import.php`
- `*Ajax.php` or `DetailViewAjax.php` (often for internal UI async flows)

## Entity classes

Most modules have an entity class file named after the module, for example:
- `modules/Contacts/Contacts.php`

These usually extend:
- `data/CRMEntity.php`

The `CRMEntity` base provides shared CRUD, relationships, attachments, and list/report utilities used across modules.

## AJAX patterns

Many module `*Ajax.php` scripts include the shared dispatcher:
- `include/Ajax/CommonAjax.php`

That dispatcher includes `modules/<module>/<file>.php` (or falls back to `modules/Vtiger/<file>.php`) after access checks. Treat these endpoints primarily as **internal UI APIs**.

## Related docs
- `Docs/Repository-Structure.md`
- `Docs/Extending-vtiger.md`
