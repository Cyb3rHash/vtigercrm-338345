# PurchaseOrder module (`modules/PurchaseOrder/`)

The **PurchaseOrder** module handles purchase orders (vendor-facing documents).

## Where to start
- Entity/model: `PurchaseOrder.php` (extends `data/CRMEntity.php`)
- Common UI actions:
  - `ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`
  - AJAX entrypoints (`*Ajax*.php`)
- PDF templates: `pdf_templates/`

## Related
- `modules/Vendors/`, `modules/Products/`
