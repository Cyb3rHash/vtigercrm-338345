# `cron/modules/` — module-specific standalone cron services

This directory is intended for **module-specific cron scripts/services** that can be invoked by the system scheduler directly.

This is separate from the preferred, framework-managed cron tasks run via:
- `vtigercron.php` + DB table `vtiger_cron_task`

## What’s in this repo
- `com_vtiger_workflow/com_vtiger_workflow.service`
- `Reports/ScheduleReports.service`
- `SalesOrder/RecurringInvoice.service`

## How these are typically used
Some deployments create “service scripts” with an access guard:
- validate `$application_unique_key` (see `cron/README-NewCronServiceSetup.txt`)
- write a PID file under `logs/` to prevent duplicate runs

## Related
- `cron/README.md`
- `Docs/Cron-Jobs.md`
