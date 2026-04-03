# Quotes module (`modules/Quotes/`)

The **Quotes** module handles customer quotations and pricing documents.

## Where to start
- Entity/model: `Quotes.php` (extends `data/CRMEntity.php`)
- Common UI actions:
  - `ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`
  - AJAX entrypoints (`*Ajax*.php`)
- PDF templates: `pdf_templates/`

## Related
- `modules/Products/`
- `modules/Accounts/`, `modules/Contacts/`
- Inventory helpers: `include/utils/InventoryUtils.php`
