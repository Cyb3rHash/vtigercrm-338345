# `install/` — installer wizard steps

The vtiger installer is driven by:

- `install.php` — installer front controller / dispatcher
- `install/` — step scripts and resources invoked by `install.php`

## How installation dispatch works

`install.php`:
- performs PHP version checks
- loads installer language/resources
- includes the appropriate step file under `install/` based on request parameters
- uses inclusion safety checks (via `Common_Install_Wizard_Utils::checkFileAccessForInclusion()`)

## Outputs of installation

In a standard vtiger deployment, installation generates configuration and initializes database schema/data. This repository includes config templates such as:

- `config.template.php`
- `config.db.php`
- `config.inc.php` (often written/filled during install)

## Related docs
- `Docs/VTigerCRM-Architecture-Overview.md`
- `Docs/Repository-Structure.md`
