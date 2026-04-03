# Accounts module (`modules/Accounts/`)

The **Accounts** module manages organizations/companies in vtiger CRM.

## Where to start
- Entity/model: `Accounts.php` (extends `data/CRMEntity.php`)
- Common UI actions:
  - `ListView.php`, `DetailView.php`, `EditView.php`
  - `Save.php`, `Delete.php`
  - AJAX entrypoints (`*Ajax*.php`)

## How it’s used
Accounts are linked to:
- Contacts (people at an account)
- Potentials (opportunities)
- Activities/Emails/Documents (via relations)

Related:
- `modules/Contacts/`, `modules/Potentials/`
- Base model: `data/CRMEntity.php`
