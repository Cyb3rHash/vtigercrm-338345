# Contacts module (`modules/Contacts/`)

The **Contacts** module manages people/individuals and their relationship to Accounts.

## Where to start
- Entity/model: `Contacts.php` (extends `data/CRMEntity.php`)
- Common UI actions:
  - `ListView.php`, `DetailView.php`, `EditView.php`
  - `Save.php`, `Delete.php`
  - AJAX entrypoints (`*Ajax*.php`)

## Typical relationships
- Linked to Accounts (organization)
- Linked to Activities, Emails, Documents, HelpDesk (depending on enabled features)

Related:
- `modules/Accounts/`
- Base model: `data/CRMEntity.php`
