# Calendar module (`modules/Calendar/`)

The **Calendar** module handles activities (tasks/events), scheduling, and calendar views.

## Where to start
- Entity/model: `Calendar.php` (extends `data/CRMEntity.php`)
- Common UI actions:
  - `ListView.php`, `DetailView.php`, `EditView.php`, `Save.php`
  - AJAX entrypoints (`*Ajax*.php`)
- `iCal/` — iCalendar-related functionality

Related:
- Shared UI/date widgets: `jscalendar/`
- Permission helpers: `include/utils/UserInfoUtil.php`
