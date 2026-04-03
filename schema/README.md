# `schema/` — database schema artifact(s)

This folder contains the vtiger CRM database schema definition.

Key file:
- `DatabaseSchema.xml` — vtiger schema definition used by installer and some upgrade/migration tooling.

## When you care about this folder
- During initial installation (`install.php`)
- While understanding what tables/columns a module depends on
- During migrations/upgrades (see `modules/Migration/`)

## Related
- `install/README.md`
- `Docs/Cron-Jobs.md` (cron schema is also managed in DB)
- `Docs/Repository-Structure.md`
