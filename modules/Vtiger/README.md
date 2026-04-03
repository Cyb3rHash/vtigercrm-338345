# Vtiger module (`modules/Vtiger/`)

This is a core/shared module folder that provides common vtiger UI scaffolding and shared action scripts.

You may find:
- Shared UI scripts used across modules
- Common headers/footers and base behaviors used by the main dispatcher (`index.php`)
- Shared EditView/ListView helpers invoked by module pages

Notes:
- This is not a typical “business module” like Accounts/Contacts; it’s part of the core UI framework.
Related:
- `Smarty/` templates, especially shared templates
- `include/utils/*` and `include/ListView/*`
