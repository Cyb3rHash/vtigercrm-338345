# `packages/vtiger/mandatory/` — mandatory vtiger packages (ZIP)

This folder contains ZIP packages that vtiger considers “mandatory” in many distributions.

Examples in this repo include:
- `Mobile.zip`
- `PBXManager.zip`
- `MailManager.zip`
- `Import.zip`
- `Services.zip`, `ServiceContracts.zip`
- `ModTracker.zip`, `WSAPP.zip`, etc.

Operationally:
- These ZIPs are installed/imported through vtiger’s module import mechanisms (typically via Settings → Module Manager).
- Keep these as **artifacts**; source code for installed modules lives under `modules/`.
