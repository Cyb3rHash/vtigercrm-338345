# Reports module

The **Reports** module provides end-user report creation, execution, export, and scheduling. Reports are defined in the database (modules, selected columns, filters, grouping, sharing) and executed by dynamically building SQL queries with access checks.

---

## Responsibilities

- Create and edit report definitions (columns, filters, grouping, totals).
- Enforce report sharing rules and editability (owner/admin/subordinates).
- Execute reports by building dynamic SQL with field-level access checks.
- Export report results (e.g., Excel, PDF) and render printable views.
- Schedule reports to be emailed at intervals.

---

## Key scripts / classes

### Core definition class
- `Reports.php`
  - Defines `class Reports extends CRMEntity`
  - Loads report metadata from database (primary/secondary modules, type, name, description, folder, owner)
  - Enforces view/edit permissions using report sharing logic and cached subordinate user information (`VTCacheUtils`).

### Report execution
- `ReportRun.php`
  - Defines the execution logic for a report:
    - Reads report definition tables (`vtiger_selectquery`, `vtiger_selectcolumn`, etc.)
    - Builds SELECT column expressions with special handling for certain field types
    - Performs per-field permission checks (e.g., `CheckFieldPermission`) and profile-based access logic.

### Report creation / editing workflow screens
- `NewReport0.php`, `NewReport1.php` (wizard steps)
- `SaveReport.php`, `Save.php`, `SaveAndRun.php`
- `DuplicateReport.php`

### Running / viewing / exporting
- `ReportRun.php`, `PrintReport.php`
- `CreateXL.php` (Excel export)
- `CreatePDF.php` (PDF export)
- `ReportColumns.php`, `ReportFilters.php`, `ReportGrouping.php`, `ReportColumnsTotal.php` (definition sub-screens)

### Sharing, folders, and scheduling
- `ReportSharing.php`
- `ChangeFolder.php`, `SaveReportFolder.php`, `DeleteReportFolder.php`
- `ScheduledReports.php`, `ReportsScheduleEmail.php`

### AJAX
- `ReportsAjax.php`
- UI JS: `Reports.js`

---

## Major workflows (grounded in code)

### 1) Loading a report definition (permission-aware)
Relevant implementation: `Reports::Reports($reportid)` in `Reports.php`

1. Attempts to load report metadata from cache (`VTCacheUtils::lookupReport_Info`).
2. If not cached, queries `vtiger_report` joined with `vtiger_reportmodules`.
3. For non-admin users, applies sharing constraints by checking:
   - `vtiger_reportsharing` (user/group shares)
   - public reports
   - ownership
   - owner within subordinate roles (role hierarchy lookup via `vtiger_role.parentrole`)
4. Determines editability (`$this->is_editable`) based on admin/owner/subordinate ownership.

### 2) Executing a report
Relevant implementation: `ReportRun` class in `ReportRun.php`

1. Initializes by constructing a `Reports` object for the report id.
2. Builds the report query based on:
   - selected columns (`vtiger_selectquery` + `vtiger_selectcolumn`)
   - filters and date filters (report filter tables)
   - grouping and totals tables (when configured)
3. Applies field-level access checks:
   - skips columns the user is not permitted to view (`CheckFieldPermission` and profile-derived permissions).
4. Produces a result set for HTML rendering and for export flows.

### 3) Exporting and scheduling
- Export flows (`CreateXL.php`, `CreatePDF.php`) execute the report and render in the desired format.
- Scheduling (`ScheduledReports.php`, `ReportsScheduleEmail.php`) uses the scheduled report table and related logic to send report results periodically.

---

## Database touchpoints (report definition + runtime)

### Core report definition
- `vtiger_report` (report header: name, description, owner, sharing type, folder)
- `vtiger_reportmodules` (primary module + secondary modules)
- `vtiger_reportfolder` (folders)
- `vtiger_reportsharing` (user/group sharing)

### Query definition (what to select)
- `vtiger_selectquery`
- `vtiger_selectcolumn`

### Filters / grouping / totals
- `vtiger_reportfilters`
- `vtiger_reportdatefilter`
- `vtiger_reportgroupbycolumn`
- `vtiger_reportsummary`
- `vtiger_reportsortcol`
- `vtiger_relcriteria`, `vtiger_relcriteria_grouping` (advanced filter criteria structures)

### Scheduling
- `vtiger_scheduled_reports`

### Permission support (access checks)
- `vtiger_profile2field`, `vtiger_def_org_field`
- Core metadata tables used to resolve fields/blocks/tabs:
  - `vtiger_field`, `vtiger_blocks`, `vtiger_tab`, `vtiger_entityname`

---

## Related code
- Shared report helpers: `ReportUtils.php`, `CustomReportUtils.php`
- Shared utilities: `include/utils/UserInfoUtil.php`, `include/utils/utils.php`
