# `kcfinder/` — embedded file manager (third-party)

This directory contains **KCFinder**, a web-based file manager commonly integrated with rich text editors.

It includes:
- `core/`, `lib/`, `tpl/`, `js/`, `themes/`, and language resources

Notes for developers:
- Treat as **vendor code**.
- Security-sensitive area: ensure access controls are correct in deployment (KCFinder can expose file browsing/upload features).
- Typically used indirectly via editor integrations in vtiger.
