# `cache/` — runtime cache directory

This directory contains files generated at runtime to improve performance and support uploads/imports.

Common subfolders:
- `cache/images/` — cached/generated images
- `cache/import/` — temporary import files
- `cache/upload/` — temporary upload-related files
- `cache/index.html` — prevents directory listing

## Operational notes
- In many deployments it is safe to **clear this directory** when troubleshooting UI/caching issues, as long as the web server user has permission to recreate needed files.
- Ensure the PHP/webserver user can write to `cache/`.
