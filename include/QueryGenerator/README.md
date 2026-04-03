# `include/QueryGenerator/` — SQL query builder

This folder contains vtiger’s `QueryGenerator`, a utility used to build module queries with filters/joins in a structured way.

Key file:
- `QueryGenerator.php`

Where it is used
- Module list views and filters
- Reporting / dynamic queries
- Some webservice flows that rely on metadata-driven queries

Related:
- `include/utils/ListViewUtils.php`
- `modules/Reports/`
