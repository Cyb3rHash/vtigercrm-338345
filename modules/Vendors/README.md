# Vendors module (`modules/Vendors/`)

The **Vendors** module manages supplier/vendor records (used by purchasing/inventory flows).

## Where to start
- Entity/model: `Vendors.php` (extends `data/CRMEntity.php`)
- Common UI actions:
  - `ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`
  - AJAX entrypoints (`*Ajax*.php`)

## Related
- `modules/PurchaseOrder/`
- `modules/Products/`
