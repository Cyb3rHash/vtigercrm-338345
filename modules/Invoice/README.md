# Invoice module (`modules/Invoice/`)

The **Invoice** module handles invoicing documents and related inventory line items.

## Where to start
- Entity/model: `Invoice.php` (extends `data/CRMEntity.php`)
- Common UI actions:
  - `ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`
  - AJAX entrypoints (`*Ajax*.php`)
- PDF templates: `pdf_templates/`

## Related
- `modules/SalesOrder/`, `modules/Quotes/`
- Inventory helpers: `include/utils/InventoryUtils.php`
