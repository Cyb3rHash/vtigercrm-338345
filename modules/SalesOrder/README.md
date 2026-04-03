# SalesOrder module (`modules/SalesOrder/`)

The **SalesOrder** module handles customer sales orders.

## Where to start
- Entity/model: `SalesOrder.php` (extends `data/CRMEntity.php`)
- Common UI actions:
  - `ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`
  - AJAX entrypoints (`*Ajax*.php`)
- PDF templates: `pdf_templates/`

## Scheduled services
- Recurring invoice related service: `cron/modules/SalesOrder/RecurringInvoice.service`

## Related
- `modules/Quotes/`, `modules/Invoice/`, `modules/Products/`
