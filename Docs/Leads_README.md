# Leads Module (vtiger CRM 5.4.0) — Purpose, Workflows, APIs, and Dependencies

## Overview

The Leads module is the primary “pre-sales intake” area of vtiger CRM. A Lead represents a person or organization that has not yet been fully qualified as a customer. In this codebase, Leads are managed as standard vtiger “entity modules,” which means they use the shared entity framework for CRUD operations, permissions, field-level visibility, auditing, and related lists.

From an implementation point of view, the module consists of a CRM entity class (`modules/Leads/Leads.php`) plus a set of classic vtiger action scripts under `modules/Leads/` that are routed through `index.php` (for example, `index.php?module=Leads&action=DetailView&record=<id>`). The module also integrates with vtiger Webservices for lead conversion, and it participates in the standard related-record model for activities, emails, documents, products, and campaigns.

## Where the module lives in the repository

The Leads module implementation is centered around:

- `modules/Leads/Leads.php`, which defines the `Leads` CRMEntity class, including storage tables, list/search metadata, and related-list handlers.
- `modules/Leads/*.php`, which are action/controller scripts invoked through `index.php` routing.
- `modules/Leads/Leads.js`, which contains client-side behavior for lead conversion validation and some UI helpers.
- `include/Webservices/ConvertLead.php` and `include/Webservices/Utils.php`, which implement the server-side lead conversion engine used by the Leads UI.

## Routing model and entry points

### Standard vtiger routing (index.php)

Most Leads interactions are invoked via vtiger’s front controller:

- `index.php?module=Leads&action=<ActionName>[&record=<id>]`

The front controller is responsible for authentication and permission checks before including the actual module action script. This means Leads scripts generally assume a valid session and rely on `isPermitted(...)` checks for feature visibility and access control.

### Leads module “index” script

The module has an `index.php` under the module directory:

- `modules/Leads/index.php`

This script sets theme variables and includes the list view script (`modules/Leads/ListView.php`). In practice, it is the module’s “default page” handler.

### Common action scripts you will see in this module

Although vtiger has many standard actions, the following are central to Leads workflows in this repository:

- `modules/Leads/ListView.php`  
  This is a thin wrapper that delegates to the generic list view implementation via `require_once('modules/Vtiger/ListView.php')`.

- `modules/Leads/EditView.php`  
  This delegates to the generic edit view controller (`modules/Vtiger/EditView.php`) and then renders `salesEditView.tpl`. It also passes through a `campaignid` request parameter into Smarty when present.

- `modules/Leads/Save.php`  
  Handles create/update submission by copying request values into `Leads::$column_fields`, applying assignment logic (user vs group), saving the entity, optionally managing a Campaign relation when coming from Campaigns, and then redirecting.

- `modules/Leads/DetailView.php`  
  Renders the lead detail page using Smarty, includes “Convert Lead” and email integration controls when permitted, and builds related-list containers in single-pane mode.

- `modules/Leads/DetailViewAjax.php`  
  Provides detail-view inline editing and related-list loading for the Leads module via an `ajxaction` parameter.

- `modules/Leads/ConvertLead.php`  
  Renders the Convert Lead UI and delegates much of the field/mapping behavior to `ConvertLeadUI`.

- `modules/Leads/LeadConvertToEntities.php`  
  Receives the Convert Lead form submission and calls the webservice conversion function `vtws_convertlead(...)`.

- `modules/Leads/ListViewTop.php`  
  Implements the “New Leads” home widget, including filtering by user preference and excluding converted or disqualified leads.

- `modules/Leads/ExportRecords.php`  
  Entry point for the “Export” UI, which includes the shared `include/utils/ExportRecords.php` script to render export templates and perform permission gating.

## Data model and storage

### Core entity tables

The Leads entity is stored across several tables. The `Leads` class declares:

- `vtiger_crmentity` as the shared entity table
- `vtiger_leaddetails` as the primary Leads table (key field: `leadid`)
- `vtiger_leadsubdetails`
- `vtiger_leadaddress`
- `vtiger_leadscf` (custom fields)

In `modules/Leads/Leads.php` these tables are declared as:

- `$table_name = "vtiger_leaddetails"`
- `$table_index = "leadid"`
- `$tab_name = ['vtiger_crmentity','vtiger_leaddetails','vtiger_leadsubdetails','vtiger_leadaddress','vtiger_leadscf']`
- `$customFieldTable = ['vtiger_leadscf', 'leadid']`

A key functional field for this module is the “converted” flag stored in `vtiger_leaddetails.converted`. Several Leads behaviors in this repository depend on this flag. For example, export and “new leads” filtering explicitly include `converted = 0`, and the conversion engine sets it to `1`.

### Relationship tables used by Leads

Leads can have related records through the standard vtiger relationship tables, including:

- Activities and emails: `vtiger_seactivityrel` (and `vtiger_cntactivityrel` after conversion for contact linkage)
- Documents/notes: `vtiger_senotesrel`
- Attachments: `vtiger_seattachmentsrel`
- Products: `vtiger_seproductsrel` (with `setype = 'Leads'` when linked to a lead)
- Campaign membership: `vtiger_campaignleadrel`
- Generic cross-module relationships: `vtiger_crmentityrel`

The Leads module uses some module-specific handling for Campaign and Product relations (see “Related lists and relationships” below).

### Conversion mapping configuration table

Lead conversion relies on a mapping table that determines how lead fields map into Accounts, Contacts, and Potentials during conversion:

- `vtiger_convertleadmapping`

This mapping is configured through Settings screens and persisted by `modules/Settings/SaveConvertLead.php` (which replaces editable mapping entries).

### Schema source of truth

The install process creates tables from:

- `schema/DatabaseSchema.xml` (invoked by `install/CreateTables.inc.php` via `$adb->createTables("schema/DatabaseSchema.xml")`)

If you need to verify column-level details, this schema file is the canonical source in this repository.

## Core class: `Leads` (modules/Leads/Leads.php)

The `Leads` class extends `CRMEntity` and primarily defines:

1. Which tables comprise the entity and how custom fields are stored.
2. List and search configuration (list fields, search fields, and default ordering).
3. Related-list query providers for common related modules.
4. Relationship persistence/unlinking overrides for certain modules (Campaigns and Products).
5. Export query generation that respects field permissions and excludes converted records.

### Export query behavior

`Leads::create_export_query($where)` builds an export query that:

- Uses field-level permission filtering (`getPermittedFieldsQuery("Leads", "detail_view")`)
- Joins the Leads tables and owner tables
- Enforces `vtiger_crmentity.deleted = 0` and `vtiger_leaddetails.converted = 0`
- Applies non-admin access control via `$this->getNonAdminAccessControlQuery('Leads', $current_user)`

This is why converted leads are omitted from exports in this repository.

### Related lists and relationships

The class provides related list handlers used in the detail view related-lists UI:

- `get_activities(...)` for open tasks/events
- `get_history(...)` for completed/held/deferred activities
- `get_emails(...)` for email activities
- `get_campaigns(...)` for Campaigns via `vtiger_campaignleadrel`
- `get_products(...)` for Products via `vtiger_seproductsrel` with `setype='Leads'`

It also overrides relationship behavior:

- `save_related_module(...)` special-cases:
  - Products: inserts into `vtiger_seproductsrel(crmid, productid, setype)`
  - Campaigns: inserts into `vtiger_campaignleadrel(campaignid, leadid, campaignrelstatusid)` (default status `1`)
- `unlinkRelationship(...)` special-cases:
  - Campaigns: deletes from `vtiger_campaignleadrel`
  - Products: deletes from `vtiger_seproductsrel`
  - Others: deletes from `vtiger_crmentityrel` in both directions

### Duplicate merge support

The method `transferRelatedRecords($module, $transferEntityIds, $entityId)` is used to move related records from duplicate leads onto a “primary” lead (for example, in a merge duplicates flow). It updates multiple relation tables (activities, notes, attachments, products, campaigns) while avoiding duplicates.

## Key workflows (web UI)

### Creating and editing a lead

In this repository, lead creation and editing use the shared vtiger edit UI and a Leads-specific save handler.

1. The user opens the edit page:
   - `index.php?module=Leads&action=EditView` (create)
   - `index.php?module=Leads&action=EditView&record=<leadid>` (edit)

   The Leads module delegates most of the form-building to `modules/Vtiger/EditView.php` via `modules/Leads/EditView.php`. If `campaignid` is present in the request, it is assigned to Smarty and can be used by templates.

2. On form submission, the save action is invoked:
   - `index.php?module=Leads&action=Save`

   `modules/Leads/Save.php` iterates over `$focus->column_fields` and copies request values into it. It then applies assignment logic:
   - If `assigntype == 'U'`, it uses `assigned_user_id`
   - If `assigntype == 'T'`, it uses `assigned_group_id`

   Finally, it calls `$focus->save("Leads")`, which triggers the generic `CRMEntity` persistence pipeline and updates `vtiger_crmentity` and the Leads module tables.

3. Campaign return special-case  
   If the request indicates the save is returning to Campaigns (`return_module == "Campaigns"`), the save handler re-establishes the campaign relation in `vtiger_campaignleadrel`, attempting to preserve an existing `campaignrelstatusid` if present.

4. The script redirects back to the requested return action/module and tries to preserve list view context (`return_viewname`, `pagenumber`, and `search_url`).

### Inline edits in Detail View (AJAX)

The Leads detail view supports inline edits using:

- `modules/Leads/DetailViewAjax.php`

When `ajxaction == "DETAILVIEW"`, the handler:

- Loads entity data (`retrieve_entity_info`)
- Updates a single field in `$modObj->column_fields[$fieldname]`
- Saves the record using `$modObj->save("Leads")`
- Returns `:#:SUCCESS` or `:#:FAILURE`

The same script also delegates related-list loading when `ajxaction == "LOADRELATEDLIST"` by including `include/ListView/RelatedListViewContents.php`.

### Viewing lead details and related lists

The lead detail page is rendered by:

- `modules/Leads/DetailView.php`

This script:

- Retrieves lead entity information using `retrieve_entity_info(...)`.
- Builds the display blocks using `getBlocks($currentModule, "detail_view", '', $focus->column_fields)`.
- Shows action buttons based on `isPermitted(...)` checks for edit, delete, merge, email, and conversion capabilities.
- Marks the record as viewed (`$focus->markAsViewed($current_user->id)`).

If `singlepane_view == 'true'`, related lists are computed via `getRelatedLists(...)` and displayed within the same page.

### “New Leads” dashboard widget

The home widget “New Leads” is implemented by:

- `modules/Leads/ListViewTop.php` (function `getNewLeads($maxval, $calCnt)`)

This function:

- Reads the user preference `vtiger_users.lead_view` if `$_REQUEST['lead_view']` is not provided.
- Computes a start datetime based on the preference (Today, Last 2 Days, Last Week).
- Queries for leads that match:
  - `vtiger_crmentity.deleted = 0`
  - `vtiger_leaddetails.converted = 0`
  - Excludes lead statuses “Lost Lead” and “Junk Lead” (including translated strings)
  - Created time after the computed start datetime
  - Assigned to the current user (`vtiger_crmentity.smownerid = current_user.id`)
- Returns entries plus an “advanced filter” query payload (`advft_criteria`) encoded with `Zend_Json` so the UI can open the full list view with a matching filter.

### Exporting leads

The export UI entry point for Leads is:

- `modules/Leads/ExportRecords.php`

This script includes:

- `include/utils/ExportRecords.php`

The shared export handler renders an export template (or a “not permitted” template if the user lacks export permission) and uses request/session context such as `$_SESSION['export_where']` and selected record IDs.

The CSV export query itself for Leads is generated by `Leads::create_export_query(...)`, which enforces permissions and excludes converted leads.

## Lead conversion (Lead → Accounts/Contacts/Potentials)

Lead conversion is a central workflow in this repository. It is implemented as a combination of Leads UI scripts and a webservice-based conversion engine.

### Conversion UI rendering

The conversion UI page is:

- `modules/Leads/ConvertLead.php`

It:

- Creates a `ConvertLeadUI` instance for the lead record.
- Assigns it into Smarty as `UIINFO`.
- Renders the module template `ConvertLead.tpl`.

The `ConvertLeadUI` helper (`modules/Leads/ConvertLeadUI.php`) loads lead values from `vtiger_leaddetails`, `vtiger_leadscf`, and `vtiger_crmentity`, and provides helper methods for:

- Determining module activation and edit permission (Accounts/Contacts/Potentials).
- Determining whether fields are active and mandatory.
- Populating owner selection lists.
- Resolving mapped field values via `vtiger_convertleadmapping` combined with `vtws_describe(...)` metadata.

### Client-side validation before submit

`modules/Leads/Leads.js` defines `verifyConvertLeadData(form)`, which enforces a set of client-side checks, including:

- At least one of Account or Contact must be selected when both are available.
- Mandatory fields for selected entities (Accounts/Contacts/Potentials) must be present.
- Potential closing date format must be valid when provided.
- Potential amount must be numeric.
- Contact email format must be valid when provided.
- “Transfer related records to” selection must be consistent with selected entities (it prevents selecting transfer-to-Account without selecting Account creation, and similarly for Contact).

These checks are meant to reduce conversion failures before the server-side conversion engine runs.

### Conversion submission and server-side conversion engine

The conversion form submits to:

- `modules/Leads/LeadConvertToEntities.php`

This script constructs an `$entityValues` payload and calls:

- `vtws_convertlead($entityValues, $current_user)` from `include/Webservices/ConvertLead.php`

Key payload elements include:

- `leadId` as a webservice entity ID (`vtws_getWebserviceEntityId('Leads', $recordId)`)
- `assignedTo` as a Users or Groups webservice entity ID
- `transferRelatedRecordsTo`, defaulting to `Contacts` when not provided
- `entities` describing which of Accounts, Contacts, and Potentials to create, with some direct values (for example `accountname`, `lastname`, `potentialname`, `sales_stage`)

At a high level, the conversion engine:

- Blocks double conversion by checking `vtiger_leaddetails.converted`.
- Creates or reuses an Account (it de-duplicates by Account name).
- Creates a Contact and links it to the Account when applicable.
- Creates a Potential and sets `related_to` to the Account or Contact depending on what exists.
- Transfers related records (documents, attachments, products, generic relations, campaigns, and optionally comments) from the Lead to the selected destination entity.
- Transfers activities and email relations so the new Account/Contact retains context.
- Marks the Lead as converted (`vtiger_leaddetails.converted = 1`) and performs related cleanup (for example, campaign lead relation cleanup and tracker cleanup).

After success, `LeadConvertToEntities.php` redirects the user to the created Account or Contact detail view (favoring Account if present).

### Conversion mapping configuration (Settings)

Conversion mapping is persisted by:

- `modules/Settings/SaveConvertLead.php`

This settings handler replaces mapping rows in `vtiger_convertleadmapping` (for `editable = 1`) based on user input.

The mapping UI logic is supported by:

- `modules/Settings/LeadCustomFieldMapping.php`

These settings are critical because conversion field population depends on `vtiger_convertleadmapping`, and conversion failures can occur if mappings are incomplete for required fields.

## APIs and integration surfaces related to Leads

vtiger in this repository exposes multiple “API surfaces.” Not all are intended for external integrations, but they are important to understand for automation and extensions.

### Web UI endpoints (index.php actions)

These are “application endpoints” in the sense that they are part of the server-rendered UI and are routed through `index.php`. For Leads, common actions include:

- `ListView`, `EditView`, `Save`, `DetailView`
- `ConvertLead`, `LeadConvertToEntities`
- `DetailViewAjax` (AJAX inline edit and related-list loading)
- `ExportRecords` (export UI)

Because these actions depend on vtiger session state and permission checks, they are primarily used by the UI rather than external clients.

### vtiger JSON Webservices (webservice.php)

The primary programmatic API surface is:

- `webservice.php`

While Leads does not define a bespoke endpoint file for webservice operations, vtiger’s standard webservice operations support Leads as an entity type. Typical operations include:

- `describe` (entity metadata): `elementType=Leads`
- `create` (create a lead): `elementType=Leads`, `element=<json>`
- `retrieve` (retrieve a lead): `id=<wsid>`
- `update` / `revise` (update fields): `element=<json with id>`
- `query` (VTQL query): `select ... from Leads ...`

A key detail is the webservice ID format. vtiger uses “webservice entity IDs” of the form `<entityId>x<recordId>`. Internally, conversion scripts generate these IDs via `vtws_getWebserviceEntityId(...)`.

For repository-level details on how `webservice.php` works and how responses are structured, see `Docs/VTigerCRM-API-Reference.md`.

### Lead conversion webservice function (`vtws_convertlead`)

Lead conversion in this repository is implemented as a server-side webservice function:

- `vtws_convertlead(...)` in `include/Webservices/ConvertLead.php`

This function is invoked by Leads UI conversion submission (`modules/Leads/LeadConvertToEntities.php`). If you are extending conversion behavior, this function and the utilities it calls in `include/Webservices/Utils.php` are the most important code paths.

### SOAP services (plugin-oriented)

This repository includes SOAP services routed through:

- `vtigerservice.php?service=<service>`

Several SOAP service implementations include Leads-related operations, including:

- `soap/thunderbirdplugin.php` (functions include `AddLead` and lead permission checks as part of a plugin integration surface)
- `soap/firefoxtoolbar.php` (includes a `create_lead_from_webform(...)` operation among other module creation operations)
- `soap/wordplugin.php` (supports retrieving merge field information, including lead columns)

These SOAP surfaces are implemented using NuSOAP and have their own authentication/session patterns (often backed by `vtiger_soapservice`), separate from `webservice.php`.

### Export endpoints and data extraction

Leads export is primarily routed through UI actions and server-side export utilities:

- `modules/Leads/ExportRecords.php` → `include/utils/ExportRecords.php` for UI preparation.
- `include/utils/export.php` contains shared export logic used across modules, including Leads.

If you are integrating at the data level, prefer JSON Webservices where possible, since the export utilities are tightly coupled to UI permissions, templates, and session state.

## Dependencies

### vtiger framework dependencies

The Leads module is built on shared vtiger framework components, including:

- `CRMEntity` and entity persistence pipeline (generic save/retrieve/mark deleted logic).
- `PearDatabase` (database abstraction).
- `Smarty` (`vtigerCRM_Smarty`) for server-rendered templates.
- Permission checks (`isPermitted`) and field visibility (`getFieldVisibilityPermission`).
- Related list rendering helpers (`GetRelatedList`, `getRelatedLists`, and related-list session handling in single-pane mode).

### Cross-module dependencies

Leads integrates with several other vtiger modules by including their classes and by using shared relationship tables:

- Calendar/Activity (`modules/Calendar/Activity.php`) for tasks/events and activity relationships.
- Campaigns (`modules/Campaigns/Campaigns.php`) for campaign membership (`vtiger_campaignleadrel`) and conversion migration into account/contact campaign relations.
- Documents (`modules/Documents/Documents.php`) for notes and attachments relations.
- Emails (`modules/Emails/Emails.php`) for email activities linked through `vtiger_seactivityrel`.
- Products (`modules/Products/...`) via `vtiger_seproductsrel` relations.

### External libraries used in Leads flows

The Leads module and its widget logic also depend on shared libraries already vendored in this repository, such as:

- Zend JSON encoding (`Zend_Json`) used in `modules/Leads/ListViewTop.php` for advanced filter query encoding.
- NuSOAP (`include/nusoap/*`) for SOAP-based plugin services.
- ADOdb (via vtiger database layer) for schema creation and database access patterns.

## Configuration and user preferences

### Lead view preference for “New Leads”

The “New Leads” widget uses a user preference field:

- `vtiger_users.lead_view`

If the request does not specify `lead_view`, the widget reads this field for the current user and adjusts the start date accordingly.

### Convert Lead field mapping (admin/config)

Lead conversion depends on:

- `vtiger_convertleadmapping`

Administrators can configure these mappings through Settings actions backed by the mapping scripts described above. Incomplete mappings can prevent conversion or cause placeholder values to be inserted for mandatory fields during conversion.

## Operational notes and troubleshooting

### “Convert Lead” button visibility in DetailView

`modules/Leads/DetailView.php` only enables conversion UI when multiple conditions are met, including:

- The user has `EditView` permission on the Lead record and `ConvertLead` permission.
- Accounts and/or Contacts modules are active and the user can edit them.
- The lead is not already converted.
- The conversion UI helper indicates necessary data is present (for example company availability or Contacts module availability).

If users report missing “Convert Lead” controls, these permission and module-activation checks are the first things to verify.

### Common conversion failure causes

When conversion fails in `LeadConvertToEntities.php`, the UI renders a reason-oriented error screen. The messages indicate repository-evidenced causes such as:

- Lead field mapping is incomplete.
- Mandatory fields are empty.

In practice, this points directly to Settings mapping completeness and to form data validation for mandatory fields.

### Converted leads being “missing” from lists/exports

Many code paths filter out converted leads by checking `vtiger_leaddetails.converted = 0`, including export query generation and the “New Leads” widget. If converted leads appear missing, it is likely due to expected filtering rather than deletion.

## Sources

This document is grounded in repository code and docs, primarily:

- `modules/Leads/Leads.php`
- `modules/Leads/Save.php`
- `modules/Leads/DetailView.php`
- `modules/Leads/DetailViewAjax.php`
- `modules/Leads/EditView.php`
- `modules/Leads/index.php`
- `modules/Leads/ListView.php`
- `modules/Leads/ListViewTop.php`
- `modules/Leads/ConvertLead.php`
- `modules/Leads/ConvertLeadUI.php`
- `modules/Leads/LeadConvertToEntities.php`
- `modules/Leads/Leads.js`
- `include/Webservices/ConvertLead.php`
- `include/Webservices/Query.php`
- `include/Webservices/Utils.php`
- `include/Ajax/CommonAjax.php`
- `include/utils/ExportRecords.php`
- `include/utils/export.php`
- `modules/Settings/SaveConvertLead.php`
- `modules/Settings/LeadCustomFieldMapping.php`
- `soap/thunderbirdplugin.php`
- `soap/firefoxtoolbar.php`
- `soap/wordplugin.php`
- `install/CreateTables.inc.php`
- `schema/DatabaseSchema.xml`
- `Docs/LLd.md`
- `Docs/VTigerCRM-API-Reference.md`
- `Docs/Leads-Accounts-Contacts-Opportunities-Interactions.md`
