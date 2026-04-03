# CustomView module (`modules/CustomView/`)

CustomView is vtiger’s “custom filter” / saved list-view definition system.

What it does:
- Manage saved filters for module list views
- Support default filters, filter visibility, and filter sharing (in some contexts)

Key files:
- `CustomView.php`
- `PopulateCustomView.php`
- UI/actions: `EditView.php`, `Save.php`, `Delete.php`, etc.
- AJAX: `CustomViewAjax.php`

Related:
- `include/ListView/`
- `include/QueryGenerator/`
