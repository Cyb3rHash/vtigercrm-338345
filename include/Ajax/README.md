# `include/Ajax/` — internal AJAX dispatch and helpers

This folder contains shared AJAX dispatch/support used by many modules.

Key files:
- `CommonAjax.php` — includes module PHP scripts based on request parameters (after access checks).
  - Many module `*Ajax.php` entrypoints delegate here.
- `TagCloud.php` — tag cloud helper endpoint/logic.

Important note:
- These endpoints are primarily **internal UI APIs** for the vtiger web app, not a public stable API.

Related:
- `modules/*/*Ajax*.php`
- `include/utils/utils.php`
- `Docs/VTigerCRM-API-Reference.md` (internal AJAX notes)
