# Contacts Module Class Diagram (Mermaid)

## Scope and intent

This document extracts the class structure and the most important inter-class relationships implemented by the vtiger CRM **Contacts** module. The module is centered around the `Contacts` entity class, which extends the vtiger framework base entity `CRMEntity`.

Because vtiger modules also include procedural “controller” scripts (for example `EditView.php` and `CallRelatedList.php`), this diagram represents those scripts as UML classes with the `<<script>>` stereotype so their dependencies on the core entity classes are visible in a single class diagram. These “script” nodes are not PHP classes in the codebase; they are file-level entrypoints.

## Mermaid class diagram

```mermaid
classDiagram
direction LR

class CRMEntity {
  +retrieve_entity_info(id, module)
  +save(module)
  +mark_deleted(id)
  +restore(id)
  +uploadAndSaveFile(id, module, fileDetails)
  +save_related_module(module, crmid, with_module, with_crmids)
  +unlinkRelationship(id, return_module, return_id)
  +setRelationTables(secmodule)
  +transferRelatedRecords(module, transferEntityIds, entityId)
}

class Contacts {
  +Contacts()
  +getCount(user_name)
  +get_opportunities(id, cur_tab_id, rel_tab_id, actions)
  +get_activities(id, cur_tab_id, rel_tab_id, actions)
  +get_history(id)
  +get_tickets(id, cur_tab_id, rel_tab_id, actions)
  +get_quotes(id, cur_tab_id, rel_tab_id, actions)
  +get_salesorder(id, cur_tab_id, rel_tab_id, actions)
  +get_products(id, cur_tab_id, rel_tab_id, actions)
  +get_purchase_orders(id, cur_tab_id, rel_tab_id, actions)
  +get_emails(id, cur_tab_id, rel_tab_id, actions)
  +get_campaigns(id, cur_tab_id, rel_tab_id, actions)
  +get_invoices(id, cur_tab_id, rel_tab_id, actions)
  +create_export_query(where)
  +getColumnNames()
  +get_searchbyemailid(username, emailaddress)
  +get_contactsforol(user_name)
  +save_module(module)
  +insertIntoAttachment(id, module)
  +generateReportsSecQuery(module, secmodule)
  +setRelationTables(secmodule)
  +unlinkDependencies(module, id)
  +unlinkRelationship(id, return_module, return_id)
  +getPortalEmailContents(entityData, password, type)
  +save_related_module(module, crmid, with_module, with_crmids)
  +getListButtons(app_strings)
}

CRMEntity <|-- Contacts

class PearDatabase {
  +getInstance()
  +pquery(sql, params)
}

class LoggerManager {
  +getLogger(name)
}

Contacts --> PearDatabase : uses as db
Contacts --> LoggerManager : uses as log

%% Related module entity classes referenced/instantiated by Contacts methods
class Potentials
class Activity
class Campaigns
class Documents
class Emails
class HelpDesk
class Quotes
class SalesOrder
class PurchaseOrder
class Invoice
class Products
class Users
class Accounts
class Vendors

Contacts ..> Potentials : get_opportunities()
Contacts ..> Activity : get_activities(), get_history()
Contacts ..> HelpDesk : get_tickets()
Contacts ..> Quotes : get_quotes()
Contacts ..> SalesOrder : get_salesorder()
Contacts ..> PurchaseOrder : get_purchase_orders()
Contacts ..> Invoice : get_invoices()
Contacts ..> Products : get_products()
Contacts ..> Emails : get_emails()
Contacts ..> Campaigns : get_campaigns()
Contacts ..> Users : plugin/Outlook helpers
Contacts ..> Accounts : unlinkRelationship()
Contacts ..> Vendors : unlinkRelationship()

%% Procedural scripts (file entrypoints) represented as <<script>>
class ContactsHandler <<script>> {
  +Contacts_sendCustomerPortalLoginDetails(entityData)
}

ContactsHandler ..> Contacts : calls getPortalEmailContents()
ContactsHandler ..> PearDatabase : reads/writes portal info

class Contacts_EditView <<script>>
class Contacts_DetailViewAjax <<script>>
class Contacts_CallRelatedList <<script>>

Contacts_EditView ..> CRMEntity : CRMEntity::getInstance()
Contacts_DetailViewAjax ..> CRMEntity : CRMEntity::getInstance()
Contacts_CallRelatedList ..> CRMEntity : CRMEntity::getInstance()
```

## Notes on relationships

The `Contacts` class inherits all baseline CRUD and relation-management behavior from `CRMEntity` and then specializes it for contact-specific tables, related lists, export behavior, portal login emails, and cleanup/unlinking rules.

In `Contacts.php`, related-module access is implemented primarily through vtiger’s related list pattern: each `get_*` method (for example `get_quotes` or `get_tickets`) builds a module-specific SQL query and delegates rendering to `GetRelatedList(...)` using an instance of the related module’s entity class. This is why the diagram shows `Contacts` with dependency edges to multiple module entity classes.

The `modules/Contacts/ContactsHandler.php` file is not a class; it defines a global function `Contacts_sendCustomerPortalLoginDetails($entityData)` that updates `vtiger_portalinfo` and sends portal login emails. It is shown as `ContactsHandler <<script>>` to make its coupling to `Contacts::getPortalEmailContents(...)` explicit.
