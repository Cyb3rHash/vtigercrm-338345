# Workflow module (`modules/com_vtiger_workflow/`)

This module provides vtiger’s **workflow engine**: define rules/conditions and execute tasks when records change or on schedules.

## What it does
- Define workflows and conditions (expression evaluation)
- Define tasks/actions to execute when workflows trigger
- Provide admin UI pages to manage workflows and tasks

## Key files/folders
- UI/actions: `workflowlist.php`, `editworkflow.php`, `edittask.php`, `tasklist.php`, `com_vtiger_workflowAjax.php`
- Engine/managers: `VTWorkflowManager.inc`, `VTWorkflowApplication.inc`, `VTTaskManager.inc`, `VTTaskQueue.inc`
- Submodules:
  - `tasks/` — task implementations (see `tasks/README.md`)
  - `expression_engine/` — condition/expression evaluation (see `expression_engine/README.md`)
  - `resources/` — supporting assets (see `resources/README.md`)

Related:
- Cron services: `cron/modules/com_vtiger_workflow/`
- Templates: `Smarty/templates/com_vtiger_workflow/`
