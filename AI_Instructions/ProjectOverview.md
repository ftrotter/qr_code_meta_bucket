# Persistent QR-Linked Archival Storage System

**Software Requirements Specification**
**Status:** Exploratory / requirements gathering  
**Clients:** iPhone application (Swift / SwiftUI) and durable web interface
**Proposed backend:** Python (FastAPI), PostgreSQL  
**Selected URL shortener:** Shlink (PHP), initially on `ft1.us` or another owner-controlled domain  
**Storage targets:** Google Drive, Dropbox, Amazon S3; extensible to other providers
**Collaborative metadata targets:** Google Drive documents, Microsoft Office documents, Etherpad, and other URL-addressable resources

## 1. Purpose and Mission

Build a QR-code-addressable, public-by-default archival system intended to keep family photos, documents, videos, and other digital materials discoverable and retrievable across generations, ideally for 20 years or longer. A QR code may be physically attached to any object or container and associate it with arbitrary, editable metadata held either by this system or in an external collaborative document.

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
9. **Metadata flexibility:** A bucket may link to any Google Drive document, Microsoft 365 document, Etherpad pad, or another URL-addressable resource instead of requiring all metadata to use an application-defined schema or file type.
10. **Separation of identity and content:** The durable QR identifier is distinct from its current metadata destination, allowing the destination to change without replacing the physical label.
11. **One bucket per QR code:** Each managed QR-code association starts with its own logical data and metadata bucket. Resources may be linked between buckets, but separate QR identifiers do not silently share one bucket.
12. **App-independent operation:** The native app is a convenient management client, not a prerequisite for resolving, browsing, contributing to, or administering a managed bucket. A standard camera and web browser must remain a complete access path.

### 1.2 Non-goals

- Confidential storage or private-by-default file sharing.
- End-to-end encryption or digital rights management.
- High-assurance identity proofing.
- Restricting public downloads or redistribution.
- Allowing routine users to overwrite or permanently delete archival objects.

Transport encryption, protection of credentials, and controls against unauthorized modifications are still required.

### 1.3 Two modes of operation

The system has two complementary modes:

1. **Application and web-management mode:** An authenticated person uses the native app or corresponding web interface to create buckets, upload resources, attach metadata endpoints, invite contributors, control permissions, select public presentation, and manage the Shlink destination associated with a QR code. The interface shows only buckets and administrative actions available to the authenticated user.
2. **Durable QR and short-URL mode:** The physical QR code contains its corresponding managed Shlink short URL and can be scanned by a standard phone camera without the native app. Shlink resolves the short code to the bucket's web interface or another owner-selected destination. Public resources require no application installation. Authentication may be requested by the web interface for uploads, restricted resources, or administrative actions.

The second mode is the continuity foundation for the first. If the native app is discontinued, unavailable for a future operating system, or removed from an app store, the QR code, Shlink URL, web interface, authentication flows, public browsing, uploads, and authorized management must continue to function. Native-app-only data formats or administrative operations are prohibited unless an equivalent documented web or restoration path exists.

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

**Any input may be used as the lookup identifier for a logical bucket of resources. Each managed QR-code association begins with a distinct logical bucket that can hold both data and metadata references for that QR code.**

### 3.2 Resolution workflow

1. Scan or enter the identifier.
2. Preserve the original value and calculate a canonical lookup key.
3. Check whether it belongs to a recognized, supported shortener instance.
4. Look up accessible bucket associations for the identifier.
5. If an accessible bucket exists, show its resources and applicable actions.
6. If none exists, offer an authenticated user the ability to create a new, distinct bucket for that identifier.
7. On subsequent scans or entry of the same identifier, resolve the associated bucket for that user or the relevant public namespace.

A recognized Shlink URL may also be followed to its public landing page. An unknown URL must not be treated as evidence of ownership or administrative control over that URL.

### 3.3 Identifier canonicalization

Store the exact original string and a separate canonical lookup value. Define explicit rules for URL normalization, case, encoding, and text-versus-URL namespaces. A cryptographic hash of the canonical identifier is a proposed efficient lookup key, not a replacement for retaining the original value.

### 3.4 Ownership of arbitrary identifiers

Scanning a QR code does **not** grant ownership of the original URL or a globally exclusive claim to the identifier. Multiple users may need separate private associations with an arbitrary identifier. An authoritative public bucket or managed redirect requires an explicit ownership/claim process. This avoids the first scanner of `https://google.com` gaining control of everyone else's archive association.

### 3.5 Physical-label and external-metadata workflow

The system shall support the useful general pattern demonstrated by SmartLabels: attach a globally unique QR label to a physical object or container, scan it to establish an association, describe what the object is or contains, and rescan it later to retrieve that information. Unlike a closed inventory schema, this project shall also allow the associated information to live in an owner-selected collaborative metadata system.

1. Generate, register, or scan a unique QR identifier.
2. Affix the QR label to a box, tool, appliance, room, artwork, document, grave marker, collection, or any other physical or conceptual subject.
3. Create a distinct logical data and metadata bucket for that QR-code association.
4. Choose an internal landing page, an external metadata document, or both.
5. Add any suitable Google Drive document, Microsoft Office document, Etherpad pad, or another HTTPS resource as a metadata endpoint.
6. Record a human-readable title, description, endpoint type, sharing status, and optional notes about the endpoint's schema or purpose.
7. On later scans, show the bucket landing page or redirect to the current primary metadata endpoint according to owner configuration.
8. Allow an authorized owner to replace or supplement the endpoint without changing or reprinting the QR code.

The external resource may be a Google Doc, Google Sheet, Google Slides presentation, PDF, image, form, drawing, or other file held in Google Drive; it does not have to be a spreadsheet. It may likewise be any suitable Microsoft Office document, Etherpad pad, or provider-independent resource. It may contain arbitrary prose, columns, rows, links, formulas, photographs, maintenance records, provenance, inventory, instructions, or other metadata. The application must not require all providers to conform to one fixed metadata schema or document type. It should retain enough local descriptive metadata to identify, validate, export, and recover the association.

## 4. Shlink and Programmable Short URLs

### 4.1 Selected shortener

Use **Shlink**, the self-hosted PHP short-URL service and short-code controller, with an initial domain of `ft1.us` or another owner-controlled short domain. Each managed physical QR code should encode a complete Shlink URL containing its corresponding short code, for example `https://ft1.us/a7K2`. The application/backend should support multiple recognized Shlink instances and custom domains. An independent operator should be able to deploy their own instance.

### 4.2 Mutable destination

The owner of a managed short link shall be able to change its destination without changing its printed QR code:

| Time | Short URL | Destination |
|---|---|---|
| Initially | `https://ft1.us/a7K2` | Public Dropbox landing page |
| Later | `https://ft1.us/a7K2` | Google Drive document containing object metadata |
| Next | `https://ft1.us/a7K2` | Microsoft Office document or Etherpad pad |
| Future | `https://ft1.us/a7K2` | Replacement archive landing page |

Use a **temporary HTTP redirect, normally 302**, rather than permanent 301/308 redirects. Configure appropriate cache controls (preferably `Cache-Control: no-store`) to reduce stale redirects. A 307 redirect is also temporary but method-preserving and is not required for ordinary QR-triggered GET requests.

### 4.3 Link administration

Authorized owners shall be able to create managed links, inspect metadata, change destinations, and review redirect history. Keep Shlink API credentials on the backend; never expose administrator credentials to the iPhone client.

### 4.4 Public landing pages

For multi-file archives or buckets with multiple metadata endpoints, prefer pointing Shlink at a stable public bucket landing page that lists links to underlying Google Drive, Dropbox, S3, Microsoft 365, Etherpad, and other locations. For a simple single-document use case, the owner may choose a direct temporary redirect to the primary metadata endpoint. The Shlink destination can change when the document URL, provider, or landing-page infrastructure changes, without changing the physical QR code.

### 4.5 Durable physical QR codes

The system shall support generating QR artwork suitable for both ordinary printing and long-term physical preservation. Generate a vector format such as SVG, and optionally a high-resolution raster format, so the same managed Shlink URL can be fabricated as:

- An engraved, etched, stamped, or laser-marked metal plate.
- A ceramic, stone, glass, or other weather-resistant marker.
- A printed label encapsulated in clear resin or another protective material.
- A replaceable paper or polymer label for less permanent uses.

Durable fabrication must preserve the QR quiet zone, square geometry, contrast, and module size. Provide a human-readable short URL and short code beside the QR symbol so the destination can still be entered manually if the symbol is damaged. Favor an appropriate QR error-correction level and test the final fabricated item with multiple ordinary camera applications before deployment. Material selection, engraving depth, pigments, adhesives, ultraviolet exposure, moisture, abrasion, corrosion, thermal cycling, and resin yellowing can all affect longevity and should be covered by fabrication guidance rather than assumed away.

The permanent object stores only the stable managed URL, not provider-specific document URLs, credentials, or a native-app-specific payload. Changing storage providers, metadata documents, or web implementations must not require remaking the metal, resin-encased, or otherwise permanent QR marker.

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

A logical bucket is **not** the same thing as an S3 bucket, Google Drive folder, or Dropbox folder: it is an application-level collection referencing data and metadata resources stored in one or more locations. A distinct logical bucket is created when a managed QR code is first associated with the system. Resources can be cross-linked or copied between buckets when useful, but each QR code retains its own bucket identity, manifest, permissions, and history.

### 9.2 Storage accounts

Allow owners or authorized operators to configure cloud-storage accounts and upload destinations. Determine separately whether contributors upload through owner-controlled storage credentials or their own linked provider accounts; the choice must not affect the logical bucket identity.

### 9.3 Replication

Support multiple independent copies of a resource, optionally across providers. Where multiple providers are configured, prefer redundancy. Record per-provider states such as pending, complete, failed, and unavailable. Never report a resource as fully replicated until all configured replication operations succeed.

### 9.4 Integrity

Calculate SHA-256 or another strong content checksum for uploaded files. Store size, checksum, and replication metadata. Provide integrity verification and flag unavailable or corrupted replicas without deleting their historical records.

### 9.5 Provider failures

If a provider or public URL fails, use an available alternate copy where possible. Do not erase metadata about missing providers or assume that a link's current availability guarantees future preservation.

### 9.6 Collaborative metadata endpoints

A bucket may have zero or more external metadata endpoints. Initial endpoint types should include:

- Any suitable document or file in Google Drive, including Google Docs, Sheets, Slides, Forms, drawings, PDFs, images, and other uploaded formats.
- Microsoft Office documents shared through Microsoft 365, OneDrive, or SharePoint, including Word documents, Excel workbooks, PowerPoint presentations, PDFs, and other supported files.
- Etherpad pads, including independently hosted Etherpad instances.
- Other owner-supplied HTTPS documents or web applications that can hold arbitrary metadata.

An endpoint is a linked, mutable metadata resource, not necessarily an immutable archival file or storage replica. Store its provider type, canonical URL, display title, optional description, primary/secondary role, sharing mode, verification state, creation time, last successful check, and change history. Provider-specific document IDs may be stored when available, but the implementation must retain the usable URL and must not make resolution depend exclusively on a proprietary API.

The endpoint's own access controls remain authoritative. A public bucket must clearly distinguish a publicly readable endpoint from one that requires a provider login or access grant. If the owner intends public access, verify the endpoint from an unauthenticated context where technically possible and report a specific sharing or availability failure. Never infer edit permission from read access, and never expose OAuth tokens or document-editing credentials through the public landing page.

When APIs and permissions permit, the system may offer endpoint previews, search indexing, or structured synchronization. These are enhancements rather than prerequisites: the baseline behavior is durable resolution to the external document. Any cached or indexed content must record its source and synchronization time and must not be represented as current when refresh has failed.

## 10. Native App and Web Interface

### 10.1 Technology

Implement the iPhone client in **Swift using SwiftUI**. Use native QR scanning (VisionKit or equivalent), native photo/file pickers, Keychain-backed credential storage, and HTTPS backend APIs. No Objective-C application implementation is required by the identified features.

Implement a responsive web interface against the same backend capabilities. It must work in current standards-based browsers without the native app and must not require a Shlink administrative credential in the browser. Public views should remain usable without authentication; authenticated contribution and management views shall enforce the same authorization rules as the native app.

### 10.2 Main workflow

**Scan / enter → Resolve identifier → Show bucket or offer creation → Browse publicly → Authenticate when contributing → Upload or manage.**

The native app and corresponding authenticated web interface shall:

- Scan QR codes and accept manual input.
- Recognize configured Shlink instances.
- Open existing accessible buckets.
- Offer creation of a user-associated bucket for unknown QR/text inputs.
- Display a multi-provider public resource listing.
- Allow invited authenticated users to upload.
- Allow owners to invite contributors and manage authorized Shlink destinations.
- Allow owners to add, test, prioritize, replace, and remove collaborative metadata endpoints.
- Open a primary Google Drive document, Microsoft Office document, Etherpad pad, or equivalent endpoint, or display all endpoints on the bucket landing page.
- Let users revisit the same bucket by rescanning or reentering the same identifier.

### 10.3 Camera and browser operation

People without the app shall be able to scan managed QR codes using a standard iPhone or Android camera and reach the bucket's responsive web interface through its Shlink URL. The default web destination should expose the bucket's public presentation and offer sign-in when the visitor has contribution or management rights. It must not be a dead-end marketing page or merely an instruction to install the app.

Universal Links or Android App Links may route a managed URL into an installed native app, but the same URL must continue to work in a browser. Browser operation is a permanent, first-class continuity mode.

### 10.4 Native-app disappearance

The deployed web system must retain enough functionality to operate the archive if all native clients disappear. At minimum, a browser user with appropriate authorization must be able to browse buckets, follow and download resources, upload additive resources, manage external metadata endpoints, manage contributors, export manifests, and update authorized Shlink destinations. Operators must be able to restore or migrate the web interface, API, database, and Shlink configuration from documented backups without rebuilding a native application.

## 11. Backend and Data Model

### 11.1 Proposed components

| Component | Technology |
|---|---|
| iPhone client | Swift / SwiftUI |
| Web client | Responsive standards-based web interface, technology TBD |
| API / orchestration | Python / FastAPI |
| Metadata database | PostgreSQL |
| Short links | Shlink (PHP), via REST API |
| Identity | OIDC plus email magic links / codes |
| Storage adapters | Google Drive, Dropbox, S3 |
| Metadata endpoints | Google Drive documents, Microsoft Office documents, Etherpad, generic HTTPS URLs |
| Bucket and management pages | Responsive web frontend, TBD |
| Deployment | Docker Compose, proposed |

FastAPI and PostgreSQL are recommended implementation choices, not yet final user-mandated requirements.

### 11.2 Entities

**User:** internal ID, linked identities, verified email addresses, device credentials, recovery methods, status.

**Bucket:** internal ID, owner, title, description, public visibility, creation time.

**Identifier:** original string, type, namespace, canonical value/hash, bucket association, claim status.

**Resource:** immutable ID, bucket, original/stored filenames, media type, size, checksum, uploader, timestamps, public listing status.

**StorageLocation:** resource ID, provider, provider-specific object reference, public download URL, replica state, verification status.

**MetadataEndpoint:** bucket ID, provider/type, canonical URL, provider document ID when available, title, description, primary/secondary role, sharing mode, verification state, last successful check, and change timestamps.

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
- External metadata endpoint URLs, types, titles, sharing/verification states, and change history.
- Upload and redirect history.
- Relevant ownership/contributor metadata, with appropriate care for personal information.

A full archival export should optionally package the actual files, not just external links.

### 12.2 Manifest redundancy

**Proposed:** Write copies of the archive manifest alongside stored files on multiple configured providers, so the archive can be reconstructed even if the central database disappears. Define versioning and integrity checks for those manifests.

### 12.3 Recovery

Design for the original owner losing an email account, phone, identity provider, cloud-storage subscription, or access to the native app. Public downloads should not depend on the owner's active login while provider links remain operational. Authorized web workflows must remain available without the native app. Independent restoration should be possible without credentials for the original deployment.

### 12.4 Domain and service continuity

Document DNS/domain renewal, backup and restore, domain transfers, replacement landing pages, and short-link redirect restoration. A QR code pointing at a domain cannot survive loss of that domain without an alternate resolution or recovery mechanism; record this as an explicit preservation risk.

## 13. Independent Self-Hosting

The system should support independent operators deploying their own instance and domain, rather than requiring all archives to be hosted by the original operator.

**Proposed deployment package:** Docker Compose with Shlink, Python API, metadata database, complete responsive web interface, environment-based configuration, storage adapters, backup/export tooling, and operator documentation.

Support a configurable registry/list of recognized Shlink instances in the iPhone application. Define whether cross-instance bucket discovery or exchange is required in a later phase.

## 14. Suggested Delivery Phases

### Phase 1 — Basic archive and linked metadata

SwiftUI scanner; responsive web interface; ordinary-camera QR resolution; arbitrary identifier lookup; Google sign-in plus universal email magic links; bucket creation; public landing page; authenticated web upload and management; generic HTTPS metadata endpoints; direct or landing-page resolution; file upload with dot-separated collision naming; basic contributor invitation; Python API and database.

### Phase 2 — Managed links

Shlink deployment and API integration; editable 302 redirects; managed-link ownership; redirect history; browser-first continuity and optional native-app recognition; vector QR artwork and durable-fabrication guidance.

### Phase 3 — Broader identity, metadata, and storage

Microsoft and Apple sign-in; multiple verified emails per account; local PIN convenience; Google Drive document, Microsoft Office, and Etherpad endpoint adapters; Dropbox/Google Drive/S3 storage adapters; provider public-sharing verification; owner-configured storage; contributor management.

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
16. Creating a managed association for a new QR code creates a distinct logical data and metadata bucket for that QR code.
17. An owner can attach a QR label to a physical object, associate its bucket with any suitable Google Drive document, rescan the label, and open that document or its bucket landing page.
18. An owner can replace a Google Drive document endpoint with a Microsoft Office document or Etherpad pad without reprinting the QR code, while retaining endpoint change history.
19. A bucket can contain multiple arbitrary metadata endpoints without requiring their format or content to follow one application-defined schema.
20. The system clearly identifies whether an external metadata endpoint is public, requires provider authentication, or failed its latest availability check.
21. A manifest export contains enough external-endpoint data to reconstruct the QR-to-metadata associations without the original database.
22. A standard phone camera can scan a managed QR code without the native app and open the corresponding bucket web interface through its Shlink short URL.
23. An authorized person can use only a web browser to upload an additive resource, manage metadata endpoints and contributors, export a manifest, and update an authorized Shlink destination.
24. Removing or discontinuing the native app does not prevent public access, authenticated contribution, authorized management, backup, restoration, or migration.
25. The system can export vector QR artwork with a human-readable Shlink URL and fabrication guidance suitable for metal marking, weather-resistant media, or resin encapsulation.
26. A storage-provider or metadata-destination change does not require remaking a permanent physical QR marker.

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
13. **Default scan behavior:** Should a bucket with one primary metadata endpoint redirect directly to it, or always show the durable landing page first?
14. **Provider integration depth:** Which metadata providers require API-based previews, search, or synchronization beyond durable URL linking?
15. **External edit history:** Should the system archive snapshots or exports of mutable documents and pads, and how should that interact with provider-native version history?

---

**Design test:** Will this decision make it more likely that a person's great-great-grandchildren can discover, retrieve, and understand the archive without depending on one particular account, device, service, or software vendor?
