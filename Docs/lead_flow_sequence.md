# Lead Creation Flow (Sequence Diagram)

## Overview

This document describes the lead creation flow in vtiger CRM (v5.4.x) from the initial browser request, through `index.php` routing, into the Leads save controller, through the `CRMEntity` save pipeline and database writes, and finally through post-save event and workflow hooks. The diagram also includes the redirect to the Lead DetailView and the “recently viewed” tracking write that happens when the DetailView is opened.

## Mermaid sequence diagram

```mermaid
sequenceDiagram
autonumber
actor Browser
participant Index as "index.php (Front Controller)"
participant ACL as "Auth and Permission Checks"
participant LeadSave as "modules/Leads/Save.php"
participant Lead as "Leads (modules/Leads/Leads.php)"
participant Entity as "CRMEntity (data/CRMEntity.php)"
participant Events as "VTEventsManager"
participant Trigger as "VTEventTrigger"
participant Delta as "VTEntityDelta"
participant Workflow as "VTWorkflowEventHandler"
participant WFMgr as "VTWorkflowManager"
participant DB as "Database (MySQL via PearDatabase)"
participant Tracker as "Tracker (data/Tracker.php)"

Browser->>Index: "POST module=Leads action=Save (form submit)"
Index->>ACL: "Validate module/action file exists (anti path traversal)"
ACL-->>Index: "OK"
Index->>ACL: "Ensure session is authenticated"
ACL-->>Index: "OK"
Index->>ACL: "isPermitted(Leads, Save, record?)"
ACL-->>Index: "Allowed"
Index->>LeadSave: "include modules/Leads/Save.php"

LeadSave->>LeadSave: "Instantiate Leads and map request into column_fields"
LeadSave->>LeadSave: "Set assigned_user_id from assigntype (U or T)"
LeadSave->>Lead: "save('Leads')"
Lead->>Entity: "save('Leads')"

Entity->>Entity: "Load event framework (include/events/include.inc)"
Entity->>Events: "initTriggerCache()"
Entity->>Entity: "entityData = VTEntityData::fromCRMEntity(this)"

Entity->>Events: "trigger vtiger.entity.beforesave.modifiable"
Events->>Trigger: "trigger(entityData)"
Trigger->>DB: "Read vtiger_eventhandlers (if not cached)"
Trigger-->>Events: "Run matching handlers (if any)"

Entity->>Events: "trigger vtiger.entity.beforesave"
Events->>Trigger: "trigger(entityData)"
Trigger->>Delta: "handleEvent(beforesave) (if registered)"

Entity->>Events: "trigger vtiger.entity.beforesave.final"
Events->>Trigger: "trigger(entityData)"

Entity->>Entity: "saveentity() startTransaction()"
Entity->>DB: "INSERT vtiger_crmentity (base row, timestamps, owner)"
opt "If module sequence field (uitype 4) is present"
Entity->>DB: "UPDATE vtiger_modentity_num (increment sequence)"
end
Entity->>DB: "INSERT vtiger_leaddetails"
Entity->>DB: "INSERT vtiger_leadsubdetails"
Entity->>DB: "INSERT vtiger_leadaddress"
Entity->>DB: "INSERT vtiger_leadscf (custom field table)"
opt "If owner differs from current user"
Entity->>DB: "DELETE/INSERT vtiger_ownernotify"
end
Entity->>Lead: "save_module('Leads')"
Lead-->>Entity: "No-op (empty implementation)"
Entity->>DB: "commitTransaction()"

Entity->>Events: "trigger vtiger.entity.aftersave"
Events->>Trigger: "trigger(entityData)"
Trigger->>Delta: "handleEvent(aftersave) (if registered)"
Trigger->>Workflow: "handleEvent(aftersave) (if registered)"

Workflow->>WFMgr: "getWorkflowsForModule('Leads')"
loop "For each workflow"
Workflow->>WFMgr: "Check execution condition and evaluate(entityData)"
alt "Workflow condition matches"
Workflow->>Workflow: "performTasks(entityData)"
else "No match"
Workflow-->>Workflow: "Skip"
end
end

Entity->>Events: "trigger vtiger.entity.aftersave.final"
Events->>Trigger: "trigger(entityData)"

LeadSave->>LeadSave: "return_id = focus->id"
opt "If return_module == Campaigns"
LeadSave->>DB: "Update vtiger_campaignleadrel (preserve campaign lead status)"
end
LeadSave-->>Browser: "302 redirect to DetailView (record=return_id)"

Browser->>Index: "GET module=Leads action=DetailView record=return_id"
Index->>Entity: "CRMEntity::getInstance('Leads') and retrieve_entity_info()"
Index->>Entity: "track_view(user_id, 'Leads', record)"
Entity->>Tracker: "track_view(...)"
Tracker->>DB: "INSERT vtiger_tracker (recently viewed)"
Index-->>Browser: "Render DetailView HTML"
```

## Notes on hooks and extensibility

The pre-save and post-save hook points shown in the diagram are implemented in `CRMEntity::save()` using the `VTEventsManager` and `VTEventTrigger` classes. Which handlers execute is determined by rows in the `vtiger_eventhandlers` table, and vtiger caches the active handler set via `VTEventsManager::initTriggerCache()` / `VTEventTrigger` caching logic. In a default installation, `install/CreateTables.inc.php` registers `VTEntityDelta` on both `vtiger.entity.beforesave` and `vtiger.entity.aftersave`, and it registers `VTWorkflowEventHandler` on `vtiger.entity.aftersave` with a dependency on `VTEntityDelta`, which ensures the delta calculation runs before workflows are evaluated.

The actual workflow tasks are data-driven: `VTWorkflowEventHandler` loads workflows for the module (for Leads, the module name resolves to `Leads`) and evaluates each workflow’s execution condition and expression-based rule. When a workflow matches, it calls `performTasks`, which may send emails, update fields, or execute entity methods depending on how workflows are configured in the database.

Although the flow above covers the most common “create lead from UI” path, the same `CRMEntity::save()` pipeline (including events) is also used by other entry points that call `$focus->save($module)` for Leads.
