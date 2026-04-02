# Users module

The **Users** module owns authentication and user administration for vtigerCRM. In addition to CRUD for user records, it includes supporting “admin” workflows such as role/profile management, organization-wide sharing rules, and regenerating per-user privilege files.

This module is UI-driven through `index.php` routing (e.g., `module=Users&action=...`) and also provides core user logic via the `Users` CRMEntity class.

---

## Responsibilities

- User authentication (SQL/integrated auth plus optional LDAP/AD auth).
- User lifecycle: create/edit users, change password, activate/deactivate, delete.
- Role & profile administration (and related permission/sharing recalculation flows).
- Supporting settings pages and internal AJAX endpoints for user administration.
- User-related utilities: login history, mail account configuration, notifications/schedulers.

---

## Key entrypoints / scripts

### Core class
- `Users.php`
  - Defines `class Users extends CRMEntity`.
  - Implements authentication helpers (e.g., `doLogin()`, `authenticate_user()`), password hashing (`encrypt_password()`), and user preference/session helpers.

### Authentication & session-related
- `Login.php` / `Authenticate.php`  
  Entry pages used by the classic login flow.
- `Logout.php`  
  Terminates the user session.

### User lifecycle (common UI actions)
- `EditView.php`, `DetailView.php`, `ListView.php`
- `Save.php`, `DeleteUser.php`, `Delete.php`
- `ChangePassword.php`

### Permissions / sharing administration
- `CreateUserPrivilegeFile.php`  
  Generates user privilege artifacts under `user_privileges/` (used by the runtime to quickly load permissions).
- `RecalculateSharingRules.php`  
  Triggers recomputation of sharing rules after role/profile/sharing changes.
- `OrgSharingEditView.php`, `SaveOrgSharing.php`  
  Organization-wide sharing configuration UI actions.

### Other notable utilities
- `LoginHistory.php`  
  Displays login audit/history information.
- `AddMailAccount.php`, `SaveMailAccount.php`  
  Manages per-user mail account settings.
- `UsersAjax.php`, `DetailViewAjax.php`  
  Internal UI AJAX endpoints for user admin operations.

---

## Major workflows (grounded in code)

### 1) Login / authentication
Relevant implementation: `Users.php` (`doLogin()`)

1. UI posts credentials.
2. `Users::doLogin($user_password)` selects authentication mode from `config.php` via `$AUTHCFG['authType']`:
   - `LDAP`: delegates to `modules/Users/authTypes/LDAP.php`
   - `AD`: delegates to `modules/Users/authTypes/adLDAP.php`
   - default (integrated/SQL): reads `vtiger_users.crypt_type`, hashes the provided password via `encrypt_password()`, then verifies against `vtiger_users.user_password`.
3. On failure, login is denied; on success, session is established by the framework.

Notes:
- `authenticate_user()` also exists and validates against `vtiger_users.user_hash` (used in some legacy and internal flows).

### 2) Changing a password
Typical flow:
- User uses `ChangePassword.php` which updates stored password hash fields in `vtiger_users` (implementation depends on the CRMEntity/User logic and the chosen `crypt_type`).

### 3) Updating roles, profiles, and sharing rules
After changes to roles/profiles/sharing:
- `RecalculateSharingRules.php` is used to refresh sharing computations.
- `CreateUserPrivilegeFile.php` is used to regenerate user privilege files under `user_privileges/` so permission checks remain consistent and fast.

---

## Database touchpoints (common tables referenced)

The Users module touches many security- and configuration-related tables. The most central ones are:

### Core user identity / auth
- `vtiger_users` (primary user record; includes `user_name`, `user_password`, `crypt_type`, `user_hash`, status fields, etc.)
- `vtiger_loginhistory` (used by `LoginHistory.php`)

### Roles, groups, sharing, and permissions (representative)
- `vtiger_user2role`, `vtiger_role` (role mapping and hierarchy)
- `vtiger_groups` and related mapping tables (group membership / ownership patterns)
- Organization/share tables frequently referenced by admin flows, such as:
  - `vtiger_def_org_share`
  - `vtiger_org_share_action_mapping`, `vtiger_org_share_action2tab`
  - `vtiger_datashare_*` (module and related-module share rules)

### Misc user-related configuration
- `vtiger_mail_accounts` (mail account configuration for users)
- `vtiger_notificationscheduler` (scheduler/notification configuration used by some user-admin utilities)
- `vtiger_asteriskextensions` (telephony extension mapping when enabled)

---

## Related code
- Base entity behavior: `data/CRMEntity.php`
- Shared permission utilities: `include/utils/UserInfoUtil.php`
- Privilege artifacts directory: `user_privileges/`
