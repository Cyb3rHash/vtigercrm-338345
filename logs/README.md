# `logs/` — runtime logs and PID files

This directory is used for runtime-generated logs and (for some cron/service patterns) PID files.

## What you may find here
- Application logs (depending on logging configuration)
- Cron/service PID files (see `cron/README.md` and `cron/README-NewCronServiceSetup.txt`)

## Operational notes
- Ensure the PHP/webserver user can write to `logs/`.
- Log format and verbosity are influenced by vtiger configuration and the bundled logging stack (`log4php/` + `log4php.properties`).
