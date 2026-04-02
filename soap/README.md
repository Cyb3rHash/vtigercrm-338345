# `soap/` — SOAP services

vtiger includes multiple SOAP endpoints, dispatched by:

- `vtigerservice.php`

## Dispatch model

Clients call:

- `vtigerservice.php?service=<name>`

`vtigerservice.php` then includes the corresponding service implementation under `soap/`. Examples present in this repository include services such as:

- Outlook integration
- Customer Portal
- Webforms
- Firefox toolbar
- Word plugin
- Thunderbird plugin

These services commonly use NuSOAP (see `include/nusoap/`) and implement their own session/auth patterns (often backed by DB table `vtiger_soapservice`).

## Related docs

- `Docs/VTigerCRM-API-Reference.md` (SOAP section)
