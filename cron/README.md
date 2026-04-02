# `cron/` — standalone cron scripts (legacy/utility)

This folder contains standalone scripts that may be executed by the system scheduler directly, separate from vtiger’s framework-managed cron task registry (`vtiger_cron_task`).

## Prefer framework-managed tasks when possible

The repository also supports DB-registered cron tasks run through:
- `vtigercron.php` (runner)
- `vtlib/Vtiger/Cron.php` (task registry + schema)

See `Docs/Cron-Jobs.md` for details.

## Service setup helper

See:
- `cron/README-NewCronServiceSetup.txt`

This file contains patterns for:
- validating `$application_unique_key` to restrict access
- writing a PID file under `logs/` to prevent duplicate service instances
- wrapper `.sh` / `.bat` launchers invoking `vtigercron.php`

## Module-specific cron scripts

`cron/modules/` is intended for module-specific cron logic (see `cron/modules/README.txt`).
