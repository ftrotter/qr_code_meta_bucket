# Persistent QR-Linked Archival Storage System

**Software Requirements Specification — Draft 0.4**  
**Status:** Exploratory / requirements gathering  
**Primary client:** iPhone application (Swift / SwiftUI)  
**Proposed backend:** Python (FastAPI), PostgreSQL  
**Selected URL shortener:** Shlink (PHP), initially on `ft1.us` or another owner-controlled domain  
**Storage targets:** Google Drive, Dropbox, Amazon S3; extensible to other providers

## 1. Purpose and Mission

Build a QR-code-addressable, public-by-default archival system intended to keep family photos, documents, videos, and other digital materials discoverable and retrievable across generations, ideally for 20 years or longer.

**The primary design objective is preventing loss of data and access, not preventing reading.** This is an archive, not a confidential asset vault. The system should favor redundant ways to log in, public download links, additive contributions, durable identifiers, portability, and multiple storage providers.

Long-term survival is a design goal, not a guarantee that domains, providers, or accounts will exist indefinitely.

### 1.1 Core principles

1. **Stable identifier:** A printed QR code remains useful even when storage locations and destinations change.
2. **Public reading:** Anyone with the public archive link can view and download published content without an account.
3. **Easy contribution:** Anyone invited by email who can prove control of that email address may contribute, irrespective of email provider.
4. **Append, don't overwrite:** New uploads never silently replace previous files.
5. **Multiple paths to recovery:** Support several authentication methods and multiple storage destinations.
6. **Provider independence:** A logical archive must not be tied to one cloud-storage provider.
7. **Open restoration:** Metadata and, ideally, files must be exportable and restorable independently of the original software.
8. **Low friction:** Authentication is for controlling contributions and administrative changes, not for restricting public reading.

### 1.2 Non-goals

- Confidential storage or private-by-default file sharing.
- End-to-end encryption or digital rights management.
- High-assurance identity proofing.
- Restricting public downloads or redistribution.
- Allowing routine users to overwrite or permanently delete archival objects.

Transport encryption, protection of credentials, and controls against unauthorized modifications are still required.

## 2. Roles and Permissions

### 2.1 Anonymous visitor

May scan or open a QR code, follow a public short link, browse a public bucket landing page, and download public resources without authentication or app installation.

### 2.2 Authenticated contributor

May authenticate using an approved method, access buckets for which they have contributor rights, upload files, and add resource links. Cannot overwrite existing objects, remove other contributors, or change Shlink destinations merely by being a contributor.

### 2.3 Bucket owner

May manage bucket metadata and presentation, invite or revoke contributors, configure storage destinations, and manage Shlink redirects for links they control. Routine ownership does not imply permission to silently destroy archival content.

### 2.4 Authorization matrix

| Operation | Anonymous | Contributor | Owner |
|---|---|---|---|
| Browse public bucket | Yes | Yes | Yes |
| Download published files | Yes | Yes | Yes |
| Follow public provider links | Yes | Yes | Yes |
| Upload new files | No | Yes | Yes |
| Add new resource links | No | Yes | Yes |
| Overwrite existing files | No | No | No |
| Permanently delete files in ordinary workflow | No | No | No |
| Invite/revoke contributors | No | No | Yes |
| Edit managed Shlink redirect | No | No | Yes |
| Change bucket presentation | No | No | Yes |

Exceptional takedown or removal procedures may be needed for illegal, abusive, or mistakenly published content; they must be explicitly authorized and carefully handled with replicas and backups.

## 3. QR Codes, Identifiers, and Buckets

### 3.1 Accepted inputs

The iPhone app shall accept:

- QR codes containing a managed Shlink URL, e.g., `https://ft1.us/a7K2`.
- QR codes containing any other URL, e.g., `https://google.com`.
- QR codes containing arbitrary text, e.g., `ABC123`.
- Manually entered URLs or text identifiers.

**Any input may be used as the lookup identifier for a logical bucket of resources.**

### 3.2 Resolution workflow

1. Scan or enter the identifier.
2. Preserve the original value and calculate a canonical lookup key.
3. Check whether it belongs to a recognized, supported shortener instance.
4. Look up accessible bucket associations for the identifier.
5. If an accessible bucket exists, show its resources and applicable actions.
6. If none exists, offer an authenticated user the ability to create a bucket associated with that identifier.
7. On subsequent scans or entry of the same identifier, resolve the associated bucket for that user or the relevant public namespace.

A recognized Shlink URL may also be followed to its public landing page. An unknown URL must not be treated as evidence of ownership or administrative control over that URL.

### 3.3 Identifier canonicalization

Store the exact original string and a separate canonical lookup value. Define explicit rules for URL normalization, case, encoding, and text-versus-URL namespaces. A cryptographic hash of the canonical identifier is a proposed efficient lookup key, not a replacement for retaining the original value.

### 3.4 Ownership of arbitrary identifiers

Scanning a QR code does **not** grant ownership of the original URL or a globally exclusive claim to the identifier. Multiple users may need separate private associations with an arbitrary identifier. An authoritative public bucket or managed redirect requires an explicit ownership/claim process. This avoids the first scanner of `https://google.com` gaining control of everyone else's archive association.

## 4. Shlink and Programmable Short URLs

### 4.1 Selected shortener

Use **Shlink**, the self-hosted PHP URL shortener, with an initial domain of `ft1.us` or another owner-controlled short domain. The application/backend should support multiple recognized Shlink instances and custom domains. An independent operator should be able to deploy their own instance.

### 4.2 Mutable destination

The owner of a managed short link shall be able to change its destination without changing its printed QR code:

| Time | Short URL | Destination |
|---|---|---|
| Initially | `https://ft1.us/a7K2` | Public Dropbox landing page |
| Later | `https://ft1.us/a7K2` | Google Drive resource |
| Future | `https://ft1.us/a7K2` | Replacement archive landing page |

Use a **temporary HTTP redirect, normally 302**, rather than permanent 301/308 redirects. Configure appropriate cache controls (preferably `Cache-Control: no-store`) to reduce stale redirects. A 307 redirect is also temporary but method-preserving and is not required for ordinary QR-triggered GET requests.

### 4.3 Link administration

Authorized owners shall be able to create managed links, inspect metadata, change destinations, and review redirect history. Keep Shlink API credentials on the backend; never expose administrator credentials to the iPhone client.

### 4.4 Public landing pages

For multi-file archives, prefer pointing Shlink at a stable public bucket landing page that lists links to underlying Google Drive, Dropbox, S3, and other locations. The Shlink destination can change when the landing-page infrastructure changes, without changing the physical QR code.

QR images may be generated independently in Python, ideally as vector SVG for crisp printing and resin-embedded labels.

## 5. Authentication and Account Recovery

### 5.1 Philosophy

Authentication exists to determine who may **add data** or administer an archive. It is not a gate on public reading and is not intended to be high-assurance identity proofing. The system should be generous about legitimate ways to regain access.

### 5.2 Initial login methods

Support:

- Google login via OpenID Connect.
- Microsoft login via OpenID Connect.
- Apple login.
- Email magic links and email one-time passcodes for **any functioning email provider**, including Hotmail/Outlook, Yahoo, iCloud, and custom domains.
- Multiple linked, verified email addresses and authentication methods on one internal account.

Facebook login is explicitly out of scope.

### 5.3 Local device login

Provide a convenient local PIN-based unlock/backup option in the iPhone app. A PIN unlocking an existing device-bound session is not, by itself, a portable recovery mechanism after a phone is lost. Design a separate recoverable credential or recovery process if PIN-based access is intended to survive device loss. Do not make routine sign-in unnecessarily restrictive.

### 5.4 Email address as contributor entitlement

**If someone can currently prove control of an invited email address, that is sufficient to grant the associated contributor access.** The app must not require a Google account or additional identity proof merely because the address is hosted at Yahoo, Hotmail, or another provider.

Verified email claims from federated providers may be used when trustworthy; otherwise use a magic link or one-time code. A verified address maps to an internal user identity and its bucket permissions. Avoid merging two established accounts solely on an unverified email claim.

## 6. Contributor Invitations

1. The owner signs in, for example with Google.
2. The owner enters an invitee's email address, e.g., `sister@hotmail.com` or `brother@yahoo.com`.
3. The system sends a bucket-specific invitation link.
4. The recipient opens the link in the app (when installed) or in a browser.
5. The recipient proves control of the invited address through email verification or a verified federated login.
6. The system grants contributor rights to the designated bucket.
7. The contributor may return, authenticate by an available method, and add new resources.

Owners can inspect pending and accepted invitations, resend or cancel invitations, and revoke contributor permissions. Invitation tokens should be single-use and time-limited; normal ongoing access should not require accepting a new invitation each time.

A contributor may link additional verified login methods to reduce future lockout risk.

## 7. Public-by-Default Reading

### 7.1 Public resources

Buckets and newly uploaded archival resources shall default to publicly readable. Anyone with the archive landing-page link can follow provider links and retrieve published material without authenticating.

Public access may be implemented with Google Drive's link-sharing permissions, Dropbox shared links, S3 public objects, or an equivalent stable public download mechanism. Provider-specific sharing policies must be respected.

### 7.2 Verification

After publishing a resource, the system should verify that the download link works in an unauthenticated context. If public sharing cannot be enabled, show a clear warning rather than pretending the file is publicly accessible.

### 7.3 Privacy implications

Public links are not confidential. They can be forwarded, indexed, copied, or archived by third parties. The product should clearly communicate this when uploading or publishing. The design does not promise that a public link will remain secret.

## 8. Uploads, Filename Collisions, and Immutability

### 8.1 Append-only uploads

A successful new upload creates a new immutable resource record and storage object. Uploads must **never silently overwrite or delete** earlier objects.

### 8.2 Filename collision convention

If a filename already exists in the target bucket or relevant storage namespace, append a **dot-separated, zero-padded numeric suffix immediately before the extension**:

| Upload | Stored filename |
|---|---|
| First | `family-photo.jpg` |
| Second | `family-photo.001.jpg` |
| Third | `family-photo.002.jpg` |
| Fourth | `family-photo.003.jpg` |

Use **dots, not dashes**, for the numeric suffix. Preserve the original submitted filename separately in metadata. For names with multiple extensions, define a consistent suffix-placement rule. Collision allocation must be atomic to prevent simultaneous uploads from choosing the same name.

### 8.3 Immutable identities and history

Every upload receives a unique resource ID independent of its filename. Record uploader, upload time, original filename, stored filename, MIME type, size, checksum, and provider locations. Re-uploading identical bytes may be recognized by checksum, but must not erase the distinct historical upload event.

### 8.4 Changes and removals

Prefer additive metadata events over destructive edits. Ordinary contributors cannot delete or replace resources. Owners may hide content from the public listing as a reversible presentation action, while preserving underlying files where appropriate. Exceptional removal and abuse handling must be designed separately, including behavior for replicated copies.

## 9. Storage Backends and Replication

### 9.1 Providers

Support Google Drive, Dropbox, and Amazon S3 initially, with a pluggable storage-provider interface for future services.

A logical bucket is **not** the same thing as an S3 bucket or a Dropbox folder: it is an application-level collection referencing resources stored in one or more locations.

### 9.2 Storage accounts

Allow owners or authorized operators to configure cloud-storage accounts and upload destinations. Determine separately whether contributors upload through owner-controlled storage credentials or their own linked provider accounts; the choice must not affect the logical bucket identity.

### 9.3 Replication

Support multiple independent copies of a resource, optionally across providers. Where multiple providers are configured, prefer redundancy. Record per-provider states such as pending, complete, failed, and unavailable. Never report a resource as fully replicated until all configured replication operations succeed.

### 9.4 Integrity

Calculate SHA-256 or another strong content checksum for uploaded files. Store size, checksum, and replication metadata. Provide integrity verification and flag unavailable or corrupted replicas without deleting their historical records.

### 9.5 Provider failures

If a provider or public URL fails, use an available alternate copy where possible. Do not erase metadata about missing providers or assume that a link's current availability guarantees future preservation.

## 10. iPhone App

### 10.1 Technology

Implement in **Swift using SwiftUI**. Use native QR scanning (VisionKit or equivalent), native photo/file pickers, Keychain-backed credential storage, and HTTPS backend APIs. No Objective-C application implementation is required by the identified features.

### 10.2 Main workflow

**Scan / enter → Resolve identifier → Show bucket or offer creation → Browse publicly → Authenticate when contributing → Upload or manage.**

The app shall:

- Scan QR codes and accept manual input.
- Recognize configured Shlink instances.
- Open existing accessible buckets.
- Offer creation of a user-associated bucket for unknown QR/text inputs.
- Display a multi-provider public resource listing.
- Allow invited authenticated users to upload.
- Allow owners to invite contributors and manage authorized Shlink destinations.
- Let users revisit the same bucket by rescanning or reentering the same identifier.

### 10.3 Browser fallback

People without the app shall still be able to scan managed QR codes using the standard iPhone camera and reach a public web page. Consider iOS Universal Links to route supported URLs into the installed app while preserving browser fallback.

## 11. Backend and Data Model

### 11.1 Proposed components

| Component | Technology |
|---|---|
| iPhone client | Swift / SwiftUI |
| API / orchestration | Python / FastAPI |
| Metadata database | PostgreSQL |
| Short links | Shlink (PHP), via REST API |
| Identity | OIDC plus email magic links / codes |
| Storage adapters | Google Drive, Dropbox, S3 |
| Public bucket pages | Web frontend, TBD |
| Deployment | Docker Compose, proposed |

FastAPI and PostgreSQL are recommended implementation choices, not yet final user-mandated requirements.

### 11.2 Entities

**User:** internal ID, linked identities, verified email addresses, device credentials, recovery methods, status.

**Bucket:** internal ID, owner, title, description, public visibility, creation time.

**Identifier:** original string, type, namespace, canonical value/hash, bucket association, claim status.

**Resource:** immutable ID, bucket, original/stored filenames, media type, size, checksum, uploader, timestamps, public listing status.

**StorageLocation:** resource ID, provider, provider-specific object reference, public download URL, replica state, verification status.

**ManagedShortLink:** Shlink instance, domain, short code, associated bucket, current destination, authorized owner, redirect history.

**Permission / Invitation:** bucket, email address or user, role, invitation status, verification, grant/revocation timestamps.

**AuditEvent:** actor, action, affected entity, time, previous/new values where relevant.

### 11.3 Security boundaries

The client must not receive Shlink administrative API secrets or long-lived provider credentials. Authorize changes server-side. Protect upload and admin endpoints from unauthorized access, prevent unsafe URL fetching/SSRF, and keep audit trails for administrative modifications. These protections serve data preservation rather than secrecy.

## 12. Long-Term Preservation and Portability

### 12.1 Export

Support export of a bucket manifest in a documented, open format (e.g., JSON), including:

- Stable bucket and identifier values.
- Resource IDs and metadata.
- Original and collision-adjusted filenames.
- Checksums and sizes.
- Provider locations and public links.
- Upload and redirect history.
- Relevant ownership/contributor metadata, with appropriate care for personal information.

A full archival export should optionally package the actual files, not just external links.

### 12.2 Manifest redundancy

**Proposed:** Write copies of the archive manifest alongside stored files on multiple configured providers, so the archive can be reconstructed even if the central database disappears. Define versioning and integrity checks for those manifests.

### 12.3 Recovery

Design for the original owner losing an email account, phone, identity provider, or cloud-storage subscription. Public downloads should not depend on the owner's active login while provider links remain operational. Independent restoration should be possible without credentials for the original deployment.

### 12.4 Domain and service continuity

Document DNS/domain renewal, backup and restore, domain transfers, replacement landing pages, and short-link redirect restoration. A QR code pointing at a domain cannot survive loss of that domain without an alternate resolution or recovery mechanism; record this as an explicit preservation risk.

## 13. Independent Self-Hosting

The system should support independent operators deploying their own instance and domain, rather than requiring all archives to be hosted by the original operator.

**Proposed deployment package:** Docker Compose with Shlink, Python API, metadata database, web landing page, environment-based configuration, storage adapters, backup/export tooling, and operator documentation.

Support a configurable registry/list of recognized Shlink instances in the iPhone application. Define whether cross-instance bucket discovery or exchange is required in a later phase.

## 14. Suggested Delivery Phases

### Phase 1 — Basic archive

SwiftUI scanner; arbitrary identifier lookup; Google sign-in plus universal email magic links; bucket creation; public landing page; file upload with dot-separated collision naming; basic contributor invitation; Python API and database.

### Phase 2 — Managed links

Shlink deployment and API integration; editable 302 redirects; managed-link ownership; redirect history; browser fallback and app recognition.

### Phase 3 — Broader identity and storage

Microsoft and Apple sign-in; multiple verified emails per account; local PIN convenience; Dropbox/Google Drive/S3 adapters; provider public-sharing verification; owner-configured storage; contributor management.

### Phase 4 — Preservation and independent operation

Replication across providers; checksum auditing; open manifests and full exports; recovery/restore tooling; manifest copies alongside files; documented self-hosting and migration.

The phases are proposed implementation sequencing, not final scope commitments.

## 15. Acceptance Criteria

1. An unauthenticated person can scan a managed QR code, reach the public bucket page, and download published files.
2. A user can scan `ABC123`, create a bucket, upload a photograph, and later rescan `ABC123` to find the same associated content.
3. A user can scan an unrelated QR code containing `https://google.com` and create a personal association without gaining control of that URL or other users' associations.
4. A Google-authenticated owner can invite `sister@hotmail.com` and `brother@yahoo.com`.
5. Both invitees can verify their email addresses via magic link/code and upload without creating Google accounts.
6. Neither contributor can overwrite files, delete existing objects, or edit the owner's Shlink destination.
7. Repeated uploads of `family-photo.jpg` result in `family-photo.jpg`, `family-photo.001.jpg`, and `family-photo.002.jpg`, with all originals preserved.
8. A managed Shlink URL can change destinations while the printed QR code continues to resolve through a temporary redirect.
9. Public resource links work without authentication, or the system clearly reports a sharing failure.
10. Uploads and replicas have recorded checksums and truthful per-provider completion states.
11. When one replica becomes unavailable, an alternate available replica remains discoverable.
12. A bucket manifest can be exported in an open format and used to reconstruct resource associations.
13. A contributor can return using an authorized verified email login after their original sign-in provider changes.
14. Owners can revoke contributor upload rights without disabling anonymous reading.
15. An independent operator can configure their own domain and deployment without using the original service's production credentials.

## 16. Open Decisions

1. **Identifier collisions:** How should globally public buckets versus personal bucket associations be selected for arbitrary QR values?
2. **Public claim verification:** What evidence establishes the authoritative owner of a managed QR code or public bucket?
3. **Storage credentials:** Do contributors upload into owner-controlled storage or their own connected accounts?
4. **Default replication:** Is one provider sufficient for initial upload, or should two independent copies be required before declaring archival success?
5. **Account recovery:** What portable recovery method complements email login and local device PIN?
6. **Owner succession:** How are archives transferred or administered when an owner dies or disappears?
7. **Public landing page:** Should every managed QR always resolve to a bucket landing page, or can owners choose direct provider redirects?
8. **Legal removal:** How should exceptional takedowns propagate to replicas and archival manifests?
9. **Cross-instance discovery:** Can separate self-hosted deployments discover buckets for the same identifier?
10. **Manifest format:** What schema and packaging format best support independent recovery over decades?
11. **Filename corner cases:** Define extensionless filenames, multi-part extensions, Unicode normalization, and concurrent collisions.
12. **Link rot monitoring:** Should the system periodically test public provider URLs and report or repair broken links?

---

**Design test:** Will this decision make it more likely that a person's great-great-grandchildren can discover, retrieve, and understand the archive without depending on one particular account, device, service, or software vendor?

**End of Draft 0.4**
