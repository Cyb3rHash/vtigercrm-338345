# Leads module (`modules/Leads/`)

The **Leads** module manages unqualified prospects (potential customers) before conversion.

## Where to start
- Entity/model: `Leads.php` (extends `data/CRMEntity.php`)
- Common UI actions:
  - `ListView.php`, `DetailView.php`, `EditView.php`
  - `Save.php`
  - Conversion flows typically call lead conversion utilities (also referenced by webservices via `include/Webservices/ConvertLead.php`)

Related:
- `modules/Accounts/`, `modules/Contacts/`, `modules/Potentials/`
- Docs: `Docs/Leads-Accounts-Contacts-Opportunities-Interactions.md`
