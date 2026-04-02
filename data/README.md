# `data/` — core entity model layer

This directory contains core domain/model building blocks used by modules.

## Key file: `CRMEntity.php`

`data/CRMEntity.php` defines the `CRMEntity` base class that most module entity classes extend (e.g., `modules/Contacts/Contacts.php`).

Responsibilities commonly implemented here include:

- Coordinating create/update flows (`save`, internal save helpers)
- Deletion and restoration (`mark_deleted`, `restore`)
- Relationship management (linking/unlinking related records)
- Attachment handling (upload + persist attachment metadata)
- List view / query helpers and report query generation
- Security-aware query building helpers used across modules

Because `CRMEntity` is widely shared, changes to it typically affect many modules.

## Other files

- `Tracker.php` — request/activity tracking support (used by some entrypoints)
- `VTEntityDelta.php` — entity delta/change tracking utilities

## Related docs
- `Docs/Extending-vtiger.md`
- `Docs/Repository-Structure.md`
