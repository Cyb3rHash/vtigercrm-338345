# `vtlib/` — vtiger extension framework

`vtlib` provides vtiger’s internal APIs for module lifecycle management, relationships/links, cron task registration, and webservice enablement.

## Key files in this repository

- `Vtiger/Module.php`
  - Module APIs: `Vtiger_Module::getInstance()`, `getClassInstance()`
  - Related lists: `setRelatedList()`, `unsetRelatedList()`
  - Links: `addLink()`, `deleteLink()`, `getLinks()`
  - Webservice enablement for modules: `initWebservice()`, `deinitWebservice()`

- `Vtiger/Cron.php`
  - Cron task registry and schema for table `vtiger_cron_task`
  - APIs: `register()`, `deregister()`, `listAllActiveInstances()`, `getInstance()`, etc.
  - Executed by `vtigercron.php`

- `Vtiger/Webservice.php`
  - `Vtiger_Webservice::initialize()` / `uninitialize()` for module webservice support

## Creating a new module (skeleton)

See the existing guide:
- `README-UsingModuleDir.txt`

It explains using templates under `vtlib/ModuleDir/<vtiger_version>/` to scaffold a new module directory under `modules/`.

## Related docs
- `Docs/Extending-vtiger.md`
- `Docs/Cron-Jobs.md`
- `Docs/Webservice-Internals.md`
