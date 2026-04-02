# Cron jobs / scheduled work

vtiger supports scheduled jobs in two related ways:

1. **Framework-managed cron tasks** executed by `vtigercron.php` and registered in the DB table `vtiger_cron_task` (recommended).
2. **Standalone scripts** under `cron/` that may be invoked directly by system cron (legacy/utility scripts).

## Key files

- Cron runner: `vtigercron.php`
- Cron task framework: `vtlib/Vtiger/Cron.php`
- Legacy scripts folder: `cron/`
- Notes on service setup: `cron/README-NewCronServiceSetup.txt`

## How `vtigercron.php` works (high level)

At runtime, `vtigercron.php`:

- loads configuration (`config.inc.php`)
- uses `Vtiger_Cron::listAllActiveInstances()` from `vtlib/Vtiger/Cron.php` to list enabled tasks
- for each task:
  - checks `isRunnable()` (frequency + last-run timestamps)
  - marks task running (`markRunning()`)
  - includes the task’s handler PHP file (`getHandlerFile()`)
  - marks task finished (`markFinished()`)

The cron framework initializes its schema via `Vtiger_Cron::initializeSchema()` when needed, ensuring the `vtiger_cron_task` table exists.

## Running cron

Typical deployment runs `vtigercron.php` via the system scheduler (e.g., every few minutes). The exact schedule depends on workload and vtiger configuration.

### CLI mode

`vtigercron.php` allows execution from CLI (`PHP_SAPI` check), which is the typical/expected mode for scheduled runs.

### “Service” mode

The repository includes patterns for running a named service task. See:
- `cron/README-NewCronServiceSetup.txt`

That guide shows:
- validating an application key (`$application_unique_key`)
- maintaining a PID file under `logs/`
- wrapping invocation via a `.sh` or `.bat` launcher

## Adding a new cron task

At a conceptual level:

1. Implement the handler PHP script that performs the work.
2. Register it as a `Vtiger_Cron` task, or insert metadata in `vtiger_cron_task` consistent with the framework expectations.
3. Ensure `vtigercron.php` is invoked by your scheduler frequently enough for the configured frequency.

The cron framework API for registration and management is in `vtlib/Vtiger/Cron.php` (methods like `register()`, `deregister()`, `getInstance()`, `listAllActiveInstances()`, etc.).

> Note: Registration details depend on your vtiger runtime environment and DB state. Use the existing tasks in your DB as a reference.

## Where module-specific cron scripts go

The repository includes a placeholder note:

- `cron/modules/README.txt` — “Module specific cron scripts should be here.”

If you maintain module-specific cron logic as standalone scripts (rather than DB-registered tasks), this is where vtiger expects them to live.

## Troubleshooting

- If tasks don’t run:
  - confirm the scheduler is actually invoking `vtigercron.php`
  - check `vtiger_cron_task` entries (enabled/disabled status, frequency)
  - inspect vtiger logs under `logs/` and any handler-specific logging
- If a task appears “stuck”:
  - `Vtiger_Cron::hadTimedout()` is used to detect tasks that were marked running but never finished
  - confirm the handler file completes and does not `exit` early without cleanup
