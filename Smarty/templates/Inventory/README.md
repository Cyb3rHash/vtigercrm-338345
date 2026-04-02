# Inventory subsystem (shared)

vtiger “Inventory” is a **shared subsystem** used by inventory-enabled transaction modules such as **Quotes**, **SalesOrder**, **PurchaseOrder**, and **Invoice**. It is not a standalone `modules/Inventory/` module in this codebase; instead, it is implemented through:

- Shared **Smarty templates** (this directory),
- Shared **PHP utilities/handlers** under `include/`,
- Shared **PDF rendering** under `include/` and `vtlib/`.

This README documents the major entrypoints and database touchpoints used by the inventory layer.

---

## Responsibilities

- Render and process the **product line-item block** for inventory documents:
  - product/service rows, quantities, list price, discounts
  - item-level taxes, group taxes, shipping/handling, adjustments, totals
- Maintain inventory line-item relations in the database.
- Support stock update and history tracking for inventory transactions.
- Provide reusable UI templates and JS for inventory edit/detail views.
- Generate PDF output for inventory modules.

---

## Key templates (this directory)

These templates are used by inventory-enabled modules to render consistent line-item UI:

- `ProductDetailsEditView.tpl`  
  Edit UI for product/service rows (quantities, prices, taxes, etc.).
- `InventoryEditView.tpl`  
  Wraps the editable inventory block (totals, adjustments, tax blocks).
- `InventoryDetailView.tpl`  
  Read-only view of inventory rows and computed totals.
- `InventoryCreateView.tpl`  
  Create UI version of the inventory block.
- `ProductDetails.tpl`  
  Shared line-item presentation template.
- `InventoryActions.tpl`  
  UI controls/actions related to the inventory block.

---

## Backend entrypoints (PHP)

Although the templates live here, most inventory logic is implemented in shared PHP code:

### Core inventory utilities
- `include/utils/InventoryUtils.php`
  - Contains the bulk of inventory/tax/stock helper functions.
  - Examples of responsibilities implemented here:
    - Build product details block structures (`getProductDetailsBlockInfo`)
    - Tax lookup and calculations (`getTaxId`, `getTaxPercentage`, `getProductTaxPercentage`, `getAllTaxes`, `getTaxDetailsForProduct`)
    - Inventory line-item persistence and cleanup (e.g., deleting/inserting into inventory relation tables)
    - Inventory history tracking (`addInventoryHistory`)
    - Stock-related helpers (`getPrdQtyInStck`, `getPrdReOrderLevel`, reorder notifications via `vtiger_inventorynotification`)

### Inventory update handler hook
- `include/InventoryHandler.php`
  - Defines `handleInventoryProductRel($entity)`
  - Delegates to `updateInventoryProductRel($entity)` in `include/utils/InventoryUtils.php`
  - Used as a central hook to reconcile product relations/stock updates after inventory records change.

### Shared JS
- `include/js/Inventory.js`
  - Frontend behaviors for inventory edit screens (dynamic row updates, calculations, etc.).

---

## PDF generation

- `include/InventoryPDFController.php`
  - Defines `Vtiger_InventoryPDFController`
  - Loads the record and associated products, builds models (line items + summary totals), and renders PDFs using the vtiger PDF framework.
  - Pulls organization header data from the DB (e.g., company details) for PDF headers.
- `vtlib/Vtiger/PDF/inventory/`
  - PDF “inventory” viewers/models used by the controller to render header/content/footer sections.

---

## Major workflows (grounded in code)

### 1) Rendering an inventory edit/detail UI
1. An inventory-enabled module’s EditView/DetailView includes inventory UI.
2. The module uses templates in `Smarty/templates/Inventory/` to render:
   - product rows and totals sections
   - tax/shipping/adjustment panels
3. `include/js/Inventory.js` supports dynamic UI behavior.

### 2) Saving line items and taxes
Relevant implementation: `include/utils/InventoryUtils.php`

Typical save flow (module-dependent, but uses shared helpers):
1. Existing line items may be removed via helper logic (e.g., deletion from inventory relation tables).
2. New line item rows are stored in inventory relation tables.
3. Taxes are resolved from configured inventory tax tables and product-tax relations when required.
4. Optional inventory history/status tracking is written for modules that record status transitions.

### 3) Stock update and reorder notification
Relevant implementation: `updateStk()`, `sendPrdStckMail()` in `include/utils/InventoryUtils.php`

- Reads current stock / reorder level from `vtiger_products`.
- Uses `vtiger_inventorynotification` templates to build and send reorder emails when stock drops below configured levels.

### 4) PDF export of an inventory record
Relevant implementation: `include/InventoryPDFController.php`

1. Loads the entity and associated products (`getAssociatedProducts()`).
2. Builds product models and summary totals (taxes, shipping, adjustments, grand total).
3. Renders the PDF using inventory viewers and outputs it.

---

## Database touchpoints (common inventory tables)

Inventory touches both generic entity tables and inventory-specific relations:

### Line items / inventory relations
- `vtiger_inventoryproductrel` (main line-item rows)
- `vtiger_inventorysubproductrel` (sub-product relationships)
- `vtiger_inventoryshippingrel` (shipping/handling related details)

### Taxes
- `vtiger_inventorytaxinfo` (configured tax definitions and percentages)
- `vtiger_producttaxrel` (product-to-tax mapping)
- `vtiger_shippingtaxinfo` (shipping tax definitions)

### Stock and products/services
- `vtiger_products` (qty-in-stock, reorder level)
- `vtiger_service` (service items)
- `vtiger_productcurrencyrel` / `vtiger_currency_info` (currency context for pricing)

### Notifications and history tracking
- `vtiger_inventorynotification` (reorder mail templates and enablement flags)
- Status history tables used by inventory modules (examples referenced by helpers):
  - `vtiger_sostatushistory`, `vtiger_postatushistory`
  - `vtiger_invoicestatushistory`, `vtiger_quotestagehistory`

### PDF/company header data
- `vtiger_organizationdetails` (company info used in PDF headers)

---

## Related modules (consumers)
Inventory is consumed by modules like:
- `modules/Quotes/`
- `modules/SalesOrder/`
- `modules/PurchaseOrder/`
- `modules/Invoice/`

The shared inventory subsystem ensures consistent UI, persistence, tax logic, and PDF rendering across these modules.
