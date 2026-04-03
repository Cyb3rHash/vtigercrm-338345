# `include/utils/` — core utility layer

This folder contains **high-traffic utility code** used by most entrypoints and modules.

## Start here
- `utils.php` — foundational helpers referenced widely across the application
- `UserInfoUtil.php` — permission, role, sharing, and user/security utilities
- `ListViewUtils.php` — list view query/render helpers
- `EditViewUtils.php`, `DetailViewUtils.php` — form/view helpers
- `VTCacheUtils.php` — caching helpers used by several modules (e.g., Reports)

## Notes for developers
- Changes here can affect large parts of the application.
- Prefer reusing existing helper functions rather than duplicating logic inside modules.

Related:
- `Docs/Repository-Structure.md`
- `include/README.md`
