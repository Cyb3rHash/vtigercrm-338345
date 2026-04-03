# Products module (`modules/Products/`)

The **Products** module manages product catalog items used in Quotes/Orders/Invoices.

## Where to start
- Entity/model: `Products.php` (extends `data/CRMEntity.php`)
- Common UI actions:
  - `ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`
  - AJAX entrypoints (`*Ajax*.php`)

## Related
- `modules/Quotes/`, `modules/SalesOrder/`, `modules/PurchaseOrder/`, `modules/Invoice/`
- Inventory helpers: `include/utils/InventoryUtils.php`
