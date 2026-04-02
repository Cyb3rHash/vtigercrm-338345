# Documents module

The **Documents** module provides document management in vtigerCRM. Internally, documents are implemented as “Notes” records with optional file attachments and folder organization.

This module includes:
- A CRMEntity implementation (`Documents.php`) that manages document metadata and attachment linkage.
- UI action scripts for listing, editing, saving, moving, and downloading documents.

---

## Responsibilities

- Create/update/delete document records (metadata stored in `vtiger_notes`).
- Handle internal file uploads (stored on disk and linked via `vtiger_attachments` + relation tables).
- Handle external document links (URL stored as the “filename” when `filelocationtype = 'E'`).
- Organize documents into folders.
- Provide download endpoints and track download counts.

---

## Key scripts / classes

### Entity class
- `Documents.php`
  - Defines `class Documents extends CRMEntity`
  - Core tables:
    - `$table_name = "vtiger_notes"`
    - `$tab_name = ['vtiger_crmentity', 'vtiger_notes']`
  - Key behaviors:
    - `save_module()` updates file metadata and manages attachments
    - `insertIntoAttachment()` calls `uploadAndSaveFile()` (from CRMEntity) to persist attachment + link it
    - Folder helper logic (default folder creation, validation on restore)
    - Relationship helpers (`insertintonotesrel()`, `unlinkRelationship()`)

### UI / action entrypoints (common)
- `ListView.php`, `DetailView.php`, `EditView.php`
- `Save.php` (save document record)
- `SaveFile.php` (file handling related save action)
- `MoveFile.php` (move a file/document between folders)
- Folder management:
  - `SaveFolder.php`, `DeleteFolder.php`
- Download / email
  - `DownloadFile.php` (download file content)
  - `EmailFile.php` (email document as attachment / link)

### Internal AJAX
- `DocumentsAjax.php`, `DetailViewAjax.php`

---

## Major workflows (grounded in code)

### 1) Creating/updating a document with an internal file upload
Relevant implementation: `Documents::save_module()` in `Documents.php`

1. The document record is saved (CRMEntity base save + module save hook).
2. If the file type indicates **internal** storage (`filelocationtype = 'I'`):
   - Reads uploaded file info from `$_FILES`
   - Sanitizes file name (`sanitizeUploadFileName`) and normalizes whitespace
   - Updates `vtiger_notes` with:
     - `filename`, `filesize`, `filetype`, `filelocationtype`, `filedownloadcount`
3. Persists attachment records by calling:
   - `insertIntoAttachment($id, 'Documents')` → `uploadAndSaveFile(...)`
4. Ensures `vtiger_seattachmentsrel` links the document record to the attachment.

If the document is edited without a new upload, the existing file metadata is loaded from `vtiger_notes` and preserved.

### 2) Creating/updating a document that is an external link
Relevant implementation: `Documents::save_module()` in `Documents.php`

- When `filelocationtype = 'E'`, the “filename” field is treated as a URL.
- The code ensures a protocol prefix exists (defaults to `http://` when no scheme is detected).
- No attachment linkage is created; any existing `vtiger_seattachmentsrel` link is removed for the record.

### 3) Downloading a document file (and incrementing download count)
Relevant implementation: `DownloadFile.php`

1. Accepts `fileid` (attachment id) and `folderid`.
2. Resolves the owning document record via `vtiger_seattachmentsrel`.
3. Loads file metadata from `vtiger_notes` and the storage path from `vtiger_attachments`.
4. Reads the on-disk file (`<attachmentsid>_<filename>`).
5. If file content is available, increments `vtiger_notes.filedownloadcount`.
6. Sends the file with appropriate headers.

---

## Database touchpoints

### Core document storage
- `vtiger_notes`  
  Document/note metadata including:
  - `filename`, `filesize`, `filetype`
  - `filelocationtype` (`I` internal / `E` external)
  - `filedownloadcount`
  - `folderid`
- `vtiger_crmentity`  
  Shared vtiger entity table (ownership, timestamps, deleted flag).

### Attachments and folders
- `vtiger_attachments` (attachment metadata + filesystem path)
- `vtiger_seattachmentsrel` (link between a document record and attachment record)
- `vtiger_attachmentsfolder` (folder definitions / names used in list views)

### Relationships to other CRM records
- `vtiger_senotesrel` (document-to-entity relationship)
- `vtiger_crmentityrel` (generic entity relationship table used by unlink flows)

---

## Related code
- Base attachment handling: `data/CRMEntity.php` (`uploadAndSaveFile`, relationship helpers)
- Shared upload logic: `include/upload_file.php`
