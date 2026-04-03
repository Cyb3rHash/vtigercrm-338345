# HelpDesk module (`modules/HelpDesk/`)

The **HelpDesk** module manages support tickets/cases.

## Where to start
- Entity/model: `HelpDesk.php` (extends `data/CRMEntity.php`)
- Common UI actions:
  - `ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`
  - AJAX entrypoints (`*Ajax*.php`)

Typical relationships:
- Contacts/Accounts (customer)
- Documents/Emails/Activities (ticket-related communication)

Related:
- `modules/Contacts/`, `modules/Accounts/`, `modules/Documents/`
