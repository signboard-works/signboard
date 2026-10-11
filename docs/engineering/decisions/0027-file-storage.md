# 0027. File storage

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Sign shops work with files: customers' artwork, proofs and their revisions, photos of sites and vehicles, logos, supplier documents, and the quote and invoice PDFs Signboard itself produces. Customers upload artwork through the customer portal, so uploads come from outside the business.

Many modules need files ([ADR 0026](0026-modular-monolith.md)). If each stored them its own way, access rules, scanning, retention and storage paths would differ by module. Storage paths must carry the tenant ([ADR 0004](0004-multi-tenant.md)).

## Decision

### One files component for every module

- File storage is a single component in the `system` module ([ADR 0026](0026-modular-monolith.md)), at `/api/v1/system/files` ([ADR 0020](0020-url-structure.md)). Other modules hold file IDs and use its interface. Nothing else touches storage directly.
- **Every file is an object** (`FIL-…`, [ADR 0017](0017-object-ids.md)). Its metadata is in the database: tenant, the object it is attached to, file name, content type, size, SHA-256 checksum, uploader, audience and scan status.
- **Its content is in object storage**, under a key made only of the tenant and the file ID (`{tenantId}/{fileId}`). File names never appear in storage keys.

### Storage

- Behind an interface, using the S3 API.
- **Hosted service:** Amazon S3 in the Sydney region (`ap-southeast-2`), encrypted at rest. Files untouched for some months move automatically to cheaper storage tiers (for example S3 Intelligent-Tiering).
- **Self-hosted:** any S3-compatible store, or a local filesystem for the simplest installations.

### Uploading and downloading

- **Maximum file size: 50 MB.** A larger file is refused with a clear message giving the limit. The limit is configuration, so it can be raised without a code change.
- **Uploads go straight to storage.** The API checks the actor's permission, creates the file record and returns a short-lived signed upload URL. The browser uploads to it, and the API confirms the upload against the expected size and checksum.
- **Downloads** are a permission check by the API followed by a redirect to a signed download URL that expires within minutes. A signed URL works for anyone holding it until it expires, so expiry is kept short.

### Access

- A file's access follows the object it is attached to: an actor can read a file if they can read that object ([ADR 0022](0022-authorisation-and-roles.md)).
- Each file also has an **audience**: **internal**, or **shared with the customer**. A proof is shared; source artwork and a supplier's invoice stay internal. Customer users only ever see shared files.

### Integrity

- **Files never change.** A new version is a new file, so a proof's revisions are all kept. Files are soft-deleted like everything else ([ADR 0018](0018-soft-delete.md)).
- **Sent documents are kept exactly.** The PDF of a quote, proof or invoice is stored as a file at the moment it is sent ([ADR 0023](0023-email-sending.md)), so what the customer received is never regenerated from data that may since have changed.
- **Uploads are scanned for malware** before they can be downloaded: by GuardDuty on the hosted service ([ADR 0030](0030-hosting-aws.md)) and by a ClamAV container when self-hosted, behind one scanning interface. The background worker records the result. A file that fails is quarantined, and the uploader and the business are told.
- Uploads, deletions, and downloads by operations users are audited ([ADR 0019](0019-audit-trail.md)).

### Deferred

- Previews and thumbnails of design files (PDF, AI, EPS, TIFF). At first, files show as icons with their details and can be downloaded. Rendering them is the kind of heavy processing that may justify a separate service ([ADR 0026](0026-modular-monolith.md)).

## Options considered

1. **One files component, S3-compatible storage, direct signed uploads and downloads**: one set of rules for every module; large transfers never pass through the API; works with any S3-compatible store. **Chosen.**
2. **Files stored in PostgreSQL**: one place for everything, inside transactions; database size, backups and memory suffer quickly with artwork.
3. **Uploads and downloads streamed through the API**: simpler access control; ties up API capacity on every transfer.
4. **Each module storing its own files**: no shared component; inconsistent access, scanning and retention.

## Consequences

- The 50 MB limit will be too small for some print-ready files (for example a full vehicle wrap). The limit is reviewed with the design partner shop.
- Scanning delays a new upload's availability by seconds to minutes, and the app shows that state.
- Storage costs grow with every proof revision and sent document, and are watched once real volumes are known.
- Deleting a business's data when it leaves includes its files ([ADR 0004](0004-multi-tenant.md)).
