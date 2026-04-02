# `Smarty/` — templating engine and templates

vtigerCRM uses Smarty for server-rendered UI composition.

## Key integration point

- `Smarty_setup.php` (at repository root) defines `vtigerCRM_Smarty`, a vtiger-specific Smarty wrapper that:
  - configures compile/cache directories
  - assigns commonly used template variables
  - implements helper caching (e.g., tag cloud view caching per user)

Entry scripts such as `Popup.php` typically require `Smarty_setup.php` and instantiate `vtigerCRM_Smarty` to render templates.

## Templates

Templates are stored under:
- `Smarty/templates/`

You will find module-related templates under paths like:
- `Smarty/templates/modules/`
- module-specific areas such as `Smarty/templates/com_vtiger_workflow/`

## Related docs
- `Docs/Repository-Structure.md`
