# `vtigercrm-338345/` — workspace environment configuration (Kavia)

## Purpose
This folder contains **workspace/runtime environment configuration** used by the surrounding development/deployment tooling for this repository (for example, URLs, ports, and CORS settings).

It is **not** part of the core vtiger 5.4.0 application code and is not used by vtiger’s legacy PHP configuration files (such as `config.inc.php`, `config.db.php`).

## What’s inside
- `.env` — environment variables for the workspace environment (backend/frontend URLs, CORS, ports, etc.)

## Notes / change policy
- Treat as **environment/tooling configuration**.
- vtiger itself is primarily configured via root-level PHP config files and the installer (`install.php`).

## Related
- Root `README.md` (setup overview)
- `install/README.md` (vtiger installation and config generation)
