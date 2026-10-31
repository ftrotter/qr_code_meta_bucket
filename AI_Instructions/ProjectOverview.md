# Persistent QR-Linked Archival Storage System

**Software Requirements Specification**
**Status:** Exploratory / requirements gathering  
**Clients:** iPhone and Macintosh applications (Swift / SwiftUI), plus a durable web interface
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
9. **Metadata flexibility:** A bucket may have one primary metadata URL pointing to any Google Drive document, Microsoft 365 document, Etherpad pad, or another URL-addressable resource instead of requiring metadata to use an application-defined schema or file type.
10. **Separation of identity and content:** The durable QR identifier is distinct from its current metadata destination, allowing the destination to change without replacing the physical label.
11. **One bucket per QR code:** Each managed QR-code association starts with its own logical data and metadata bucket. Resources may be linked between buckets, but separate QR identifiers do not silently share one bucket.
12. **App-independent operation:** The native iPhone and Macintosh applications are convenient management clients, not prerequisites for resolving, browsing, contributing to, or administering a managed bucket. A standard camera and web browser must remain a complete access path.
13. **Cross-device administration:** The same account, bucket, upload, provider-integration, mirroring, metadata, link, and Shlink-management capabilities must be available from iPhone, Macintosh, and the responsive web interface, subject to platform capabilities and the user's authorization.
14. **Provider-native access control:** Google Drive, Dropbox, S3, Etherpad, and other underlying systems remain responsible for authenticating viewers and enforcing their own sharing policies. This system links to resources and reports observed accessibility; it does not reproduce or override provider permissions.
15. **Copying over restriction:** Keep application-level access control minimal while maximizing verified replication, provider-to-provider copying, restoration, and migration. Long-term survival and portability take priority over keeping archival copies scarce.

### 1.2 Non-goals

- Confidential storage or private-by-default file sharing.
- End-to-end encryption or digital rights management.
- High-assurance identity proofing.
- Restricting public downloads or redistribution.
- Allowing routine users to overwrite or permanently delete archival objects.

Transport encryption, protection of credentials, and controls against unauthorized modifications are still required.

The non-goal of confidential storage does not make integration credentials public. OAuth refresh tokens, API keys, S3 credentials, and equivalent delegated secrets are operational capabilities that require strong protection because they can create, read, copy, or modify provider resources.

### 1.3 Two modes of operation

The system has two complementary modes:

1. **Application and web-management mode:** An authenticated person uses the native iPhone app, native Macintosh app, or corresponding web interface to create buckets, upload resources, set the primary metadata URL, add ordinary or location-conditioned links, invite contributors, control permissions, select public presentation, and manage the Shlink destination associated with a QR code. The interface shows only buckets and administrative actions available to the authenticated user.
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

The iPhone and Macintosh applications and web interface shall accept:

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

Store the exact original string and a separate canonical lookup value. Define explicit rules for URL normalization, case, encoding, and text-versus-URL namespaces. A cryptographic hash of the canonical identifier may be used as an efficient lookup key, but it is not a replacement for retaining the original value.

### 3.4 Folder-name stability

Every connected storage provider should contain an application-owned root directory or equivalent namespace named `M_meta_B_bucket`. Each managed QR bucket receives a child directory, folder, prefix, or provider-equivalent space whose name is deterministically derived from the QR payload. The same QR payload must produce the same derived name on Google Drive, Dropbox, S3, and future providers.

The exact QR payload and its meaning remain separate from this storage name:

- If the payload is a URL, retain the exact URL as the identifier and continue to treat it as a URL for navigation and resolution. Do not rewrite the URL stored in the QR code.
- If the payload is non-URL text, retain the complete exact text, including whitespace, punctuation, Unicode, and line breaks.
- Never rely on the derived directory name to reconstruct the exact original payload; preserve the original in the database and open manifest.
- A provider directory cannot safely contain a literal arbitrary URL because `/`, `:`, `?`, `#`, and other characters have path or provider semantics. The URL therefore remains unchanged as the identifier while its provider-directory name uses the safe derivation below.

Use the following versioned algorithm, initially `folder_name_v1`:

1. Decode the scanned QR payload as text and retain the exact decoded string as `original_identifier`.
2. For URL detection and application lookup, process the value according to the separately documented URL canonicalization rules. Folder-name derivation always begins with the exact `original_identifier`, not the canonicalized URL.
3. Normalize the naming input to Unicode NFKD and remove combining marks so accented characters have a stable readable ASCII base where possible.
4. Preserve ASCII letters and digits. Replace every other character—including spaces, tabs, line breaks, underscores, punctuation, URL separators, emoji, and non-ASCII characters without an ASCII decomposition—with an underscore (`_`).
5. Collapse each consecutive run of underscores into one underscore, then remove leading and trailing underscores.
6. If no readable characters remain, use `qr` as the readable prefix.
7. The final derived name must be no more than **255 ASCII characters**, making its character and UTF-8 byte lengths identical. It must not be `.` or `..` and must not end in a period or space.
8. If the normalized readable name is at most 255 characters and is not already assigned within the `M_meta_B_bucket` namespace to a different exact payload, use it unchanged.
9. If it exceeds 255 characters or collides with a different exact payload after normalization, calculate lowercase hexadecimal `SHA-256` over the complete, exact UTF-8 bytes of `original_identifier`. Construct the name as `<readable_prefix>_sha256_<64_hex_characters>`, truncating only `readable_prefix` so the complete result is exactly 255 characters or fewer. With the literal `_sha256_` separator and 64-character digest, at most 183 characters remain for the readable prefix.

SHA-256 is selected for deterministic collapsing and collision resistance, not to authenticate the payload or keep it secret. The complete digest must be retained; do not truncate the digest to gain a longer readable prefix. Implementations must not use a language runtime's unstable built-in hash function.

Examples:

| Exact QR payload | Derived provider-space name |
|---|---|
| `ABC123` | `ABC123` |
| `Family Photos: 1980–1989` | `Family_Photos_1980_1989` |
| `https://example.org/a?x=1#top` | `https_example_org_a_x_1_top` |
| Text whose normalized form exceeds 255 characters | First 183 readable characters plus `_sha256_` and the SHA-256 of the complete original text |

The mapping from exact identifier to naming-algorithm version, normalized readable value, final derived name, and full SHA-256 digest shall be stored before creating provider spaces. Provider adapters must use that recorded final name rather than independently reimplementing naming. Once assigned, the derived name is stable and must not change merely because a later software release introduces a new naming algorithm. A deliberate rename requires an audited migration that updates all provider-space mappings without changing the logical bucket or QR identifier.

### 3.5 Ownership of arbitrary identifiers

Scanning a QR code does **not** grant ownership of the original URL or a globally exclusive claim to the identifier. Multiple users may need separate private associations with an arbitrary identifier. An authoritative public bucket or managed redirect requires an explicit ownership/claim process. This avoids the first scanner of `https://google.com` gaining control of everyone else's archive association.

### 3.6 Physical-label and external-metadata workflow

The system shall support the useful general pattern demonstrated by SmartLabels: attach a globally unique QR label to a physical object or container, scan it to establish an association, describe what the object is or contains, and rescan it later to retrieve that information. Unlike a closed inventory schema, this project shall also allow the associated information to live in an owner-selected collaborative metadata system.

1. Generate, register, or scan a unique QR identifier.
2. Affix the QR label to a box, tool, appliance, room, artwork, document, grave marker, collection, or any other physical or conceptual subject.
3. Create a distinct logical data and metadata bucket for that QR-code association.
4. Choose an internal landing page, an external metadata document, or both.
5. Optionally set one suitable Google Drive document, Microsoft Office document, Etherpad pad, or another HTTPS resource as the bucket's primary metadata URL.
6. Record a human-readable title, description, URL type, sharing status, and optional notes about the metadata resource's schema or purpose.
7. On later scans, show the bucket landing page or redirect to the current primary metadata URL according to owner configuration.
8. Allow an authorized owner to replace the metadata URL without changing or reprinting the QR code.

The external resource may be a Google Doc, Google Sheet, Google Slides presentation, PDF, image, form, drawing, or other file held in Google Drive; it does not have to be a spreadsheet. It may likewise be any suitable Microsoft Office document, Etherpad pad, or provider-independent resource. It may contain arbitrary prose, columns, rows, links, formulas, photographs, maintenance records, provenance, inventory, instructions, or other metadata. The application must not require all providers to conform to one fixed metadata schema or document type. It should retain enough local descriptive metadata to identify, validate, export, and recover the association.

## 4. Shlink and Programmable Short URLs

### 4.1 Selected shortener

Use **Shlink**, the self-hosted PHP short-URL service and short-code controller, with an initial domain of `ft1.us` or another owner-controlled short domain. Each managed physical QR code should encode a complete Shlink URL containing its corresponding short code, for example `https://ft1.us/a7K2`. The application/backend should support multiple recognized Shlink instances and custom domains. An independent operator should be able to deploy their own instance.

Integrate with an unmodified upstream Shlink deployment through its documented configuration and REST API. Do not fork, patch, or add project-specific geolocation behavior to the Shlink codebase. Shlink's responsibility ends after resolving the stable short URL to this project's landing-page URL. The landing-page client and application API are responsible for requesting location, evaluating presentation rules, and composing the response.

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

Prefer pointing Shlink at a stable public bucket landing page that composes the bucket's metadata URL, provider-backed data resources, and arbitrary links. This indirection is required for location-conditioned presentation. For a simple bucket with no conditional behavior, the owner may choose a direct temporary redirect to the primary metadata URL. The Shlink destination can change when the document URL, provider, or landing-page infrastructure changes, without changing the physical QR code.

### 4.5 Bucket content bundle

Each QR-associated logical bucket should present three distinct kinds of content:

1. **Primary metadata URL:** Zero or one URL pointing to the authoritative free-form description of the subject. It may be a Microsoft Office document, Etherpad pad, Google Drive file such as a Google Sheet, or any other supported HTTPS resource. Replacing it preserves its change history; adding a second primary metadata URL replaces or demotes the previous one rather than creating multiple primaries.
2. **Data resources:** Zero or more integrated files, folders, or storage references backed by Google Drive, Dropbox, Amazon S3, or other storage adapters. These resources and their availability do not change based on the viewer's location.
3. **Display links:** Zero or more arbitrary titled URLs shown on the landing page and in the scanner app. A display link may be unconditional or may have one or more location rules controlling whether it is shown.

The metadata URL and data resources form the stable bucket contents. Initial geolocation support changes only the set of display links and related explanatory text presented to the viewer; it must not silently change, hide, or redirect the primary metadata URL or provider-backed data resources.

### 4.6 Durable physical QR codes

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

Application authentication is separate from provider-resource authentication. Following the primary metadata URL or a provider resource hands the viewer to that provider, which may allow anonymous access, require a particular account, or permit anyone with the link to edit. Those outcomes are valid provider configurations and are not reimplemented by this system.

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

Public access may be implemented with Google Drive's link-sharing permissions, Dropbox shared links, S3 public objects, or an equivalent stable public download mechanism. Provider-specific authentication and sharing policies are authoritative and must be respected. The system must not proxy a private resource to bypass them or treat application login as permission to view an underlying provider asset.

### 7.2 Verification

After publishing a resource, the system should test whether the resource is reachable in an unauthenticated context and report the observed result. It may also report that provider authentication is required. It must not automatically broaden provider permissions merely to make the test pass unless an authorized owner explicitly requests a supported sharing change and confirms its consequences.

### 7.3 Privacy implications

Public links are not confidential. They can be forwarded, indexed, copied, or archived by third parties. The product should clearly communicate this when uploading or publishing. The design does not promise that a public link will remain secret.

### 7.4 Optional geolocation and location-conditioned links

The landing page and native scanner app may optionally tailor display links to the viewer's current location. Location access must be opt-in and must use the platform's normal permission mechanism:

- In a browser, use the standards-based Geolocation API from a secure HTTPS context and request precise coordinates only after explaining why they are useful.
- In the native app, use the operating system's location-permission APIs and request accuracy appropriate to the configured rule.
- Do not require location permission to resolve the QR code, view the primary metadata URL, or access the bucket's provider-backed data resources.
- If permission is denied, dismissed, unavailable, timed out, or insufficiently accurate, render the normal no-location view with unconditional display links. Explain that optional nearby links may be hidden and offer a user-initiated retry.

An owner may configure a display link with one or more named circular geofences, each defined by latitude, longitude, and radius in meters. A location-conditioned link is shown when the reported coordinates and accuracy satisfy a configured rule. Rules may also provide location-specific link titles, descriptions, or related presentation text. For example, a link may appear only when the viewer reports being within the configured radius of Times Square.

Evaluate rules in this project's API or client code after Shlink resolves to the landing page. Shlink must remain unaware of coordinates and geofences. The same rule semantics and no-location behavior must be used by the browser and native app so equivalent coordinates produce equivalent content.

Location-conditioned links are **contextual presentation, not access control or a security boundary**. Browser coordinates, native-device coordinates, API requests, and GPS signals can be spoofed, replayed, altered, or imprecise. Anyone willing to fabricate a matching location may discover the conditioned URL. Do not describe such links as secret, protected, or proof that a person physically visited a place. Truly restricted content must use authentication and authorization independent of geolocation.

Treat precise coordinates as sensitive transient data. By default, evaluate a location request without retaining the viewer's exact coordinates. If analytics or audit retention is later enabled, require a documented purpose, explicit disclosure, a retention period, and collection minimization. Do not place precise coordinates in Shlink URLs, browser history, public manifests, or routine application logs.

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

Allow owners or authorized operators to connect cloud-storage accounts and destinations through provider-appropriate delegated credentials, including OAuth grants, scoped API keys, S3-compatible access keys, service accounts, or equivalent mechanisms. A connection belongs to a user or operator and may be authorized for selected buckets; it does not change logical bucket identity.

Retain integration credentials when persistent automation is requested so background uploads, replication, health checks, restoration, and migration can continue without asking the user to reauthenticate for every object. Store credentials only on the backend in an encrypted secrets store or encrypted database fields backed by an external encryption key. Never include secrets in manifests, logs, URLs, client storage, exports, or QR codes. Use the narrowest practical provider scopes, redact secrets from errors, support credential rotation and revocation, and record which authorized actor connected or used an account.

If a provider supports expiring access tokens plus refresh tokens, retain only what is required to obtain future authorized access. If a connection expires or is revoked, mark affected operations as blocked with a provider-specific explanation while preserving manifests, queued work, and historical replica records.

### 9.3 Corresponding provider spaces and upload fan-out

For each logical QR bucket, the owner may select one or more connected providers as replication targets. The system shall create or adopt the application root `M_meta_B_bucket` and the bucket's recorded derived-name child space in each target—for example, a Google Drive folder, Dropbox folder, S3 prefix, or provider-equivalent container—and record the mapping in the bucket manifest. Provider adapters must not invent different names for the same bucket.

When a user uploads one file or a batch such as one hundred images, the system shall:

1. Create or verify the corresponding space on every selected target.
2. Create one immutable logical resource record per input file.
3. Queue an independent copy to every selected provider space.
4. Preserve relative paths and filenames where supported, applying documented collision rules when required.
5. Calculate a source checksum and verify each completed copy by checksum or the strongest trustworthy provider mechanism available.
6. Report per-file and per-provider progress, failures, retries, and final state without representing partial fan-out as complete replication.

Fan-out jobs must be resumable and idempotent. Retrying a completed step must not create unintended duplicate provider objects. A failure at one provider must not roll back successful copies at other providers, and the system must make retrying only missing or failed copies straightforward.

### 9.4 Replication

Support multiple independent copies of a resource across providers and accounts. Where multiple targets are configured, redundancy is the default objective. Record per-provider states such as pending, copying, verifying, complete, failed, blocked-credentials, and unavailable. Never report a resource as fully replicated until all configured replication and verification operations succeed.

### 9.5 Existing-archive import and provider-to-provider mirroring

An owner may designate an existing Google Drive folder, Dropbox folder, S3 prefix, or provider-equivalent space as a source archive. The system shall inventory that source, create or adopt corresponding spaces on selected destination providers, and copy the archive into them without requiring the files to be downloaded manually through the client.

Import and mirroring shall support:

- Recursive discovery with pagination until the complete source selection is inventoried.
- Preservation of relative paths, filenames, sizes, timestamps when supported, media types, checksums, and source identifiers.
- Resumable, rate-limit-aware transfer with bounded concurrency and durable job state.
- Server-side or provider-native copy operations when safe and available; otherwise stream through the backend without persisting unnecessary plaintext temporary copies.
- Idempotent reconciliation that copies missing or changed objects and recognizes already verified replicas.
- Explicit conflict handling without silently overwriting either side.
- Dry-run inventory and estimated object count/bytes before a large migration.
- Per-object verification and a final reconciliation report listing complete, failed, skipped, conflicting, and unverifiable items.

Initial mirroring is copy-oriented and non-destructive. It must not delete source objects or propagate deletions merely because an item is absent from one side. Continuous bidirectional synchronization is not required; a user may rerun reconciliation or schedule one-way mirroring from a designated source of record.

Provider adapters must expose common capabilities for listing, creating spaces, reading, writing, copying, checking metadata, and verifying content while preserving provider-specific limitations. Adding a future provider such as "My Favorite Box" should allow an owner to connect it and mirror a bucket from a declining provider without changing the QR code, logical resource IDs, or bucket manifest.

### 9.6 Integrity

Calculate SHA-256 or another strong content checksum for uploaded files. Store size, checksum, and replication metadata. Provide integrity verification and flag unavailable or corrupted replicas without deleting their historical records.

### 9.7 Provider failures

If a provider or public URL fails, use an available alternate copy where possible. Do not erase metadata about missing providers or assume that a link's current availability guarantees future preservation.

### 9.8 Primary metadata URL

A bucket may have zero or one primary external metadata URL. Supported resource types should include:

- Any suitable document or file in Google Drive, including Google Docs, Sheets, Slides, Forms, drawings, PDFs, images, and other uploaded formats.
- Microsoft Office documents shared through Microsoft 365, OneDrive, or SharePoint, including Word documents, Excel workbooks, PowerPoint presentations, PDFs, and other supported files.
- Etherpad pads, including independently hosted Etherpad instances.
- Other owner-supplied HTTPS documents or web applications that can hold arbitrary metadata.

The metadata URL identifies a linked, mutable metadata resource, not necessarily an immutable archival file or storage replica. Store its provider type, canonical URL, display title, optional description, sharing mode, verification state, creation time, last successful check, and change history. Provider-specific document IDs may be stored when available, but the implementation must retain the usable URL and must not make resolution depend exclusively on a proprietary API.

The metadata resource's own access controls remain authoritative. The system shall accept provider-authenticated documents, anyone-with-the-link resources, and anonymously editable resources such as a public Etherpad. A bucket must clearly distinguish a publicly reachable URL from one that appears to require provider authentication, but this status is informational rather than an application-enforced permission. Never infer edit permission from read access, proxy around provider authentication, or expose OAuth tokens or document-editing credentials through the public landing page.

When APIs and permissions permit, the system may offer endpoint previews, search indexing, or structured synchronization. These are enhancements rather than prerequisites: the baseline behavior is durable resolution to the external document. Any cached or indexed content must record its source and synchronization time and must not be represented as current when refresh has failed.

## 10. iPhone, Macintosh, and Web Interfaces

### 10.1 Technology

Implement native iPhone and macOS clients in **Swift using SwiftUI**, sharing application, networking, authentication, data-model, and transfer-management code where practical while providing platform-appropriate interfaces. Use Keychain-backed credential storage and HTTPS backend APIs. No Objective-C application implementation is required by the identified features.

The iPhone client should use native QR scanning (VisionKit or equivalent) and native photo/file pickers. The Macintosh client shall provide the same archive-management capabilities and support QR input through any available built-in or attached camera, QR-image import from a file or the clipboard, manual short-URL or short-code entry, and Continuity Camera where supported. A Mac without a usable camera must still provide complete management, upload, import, mirroring, and browsing functionality.

Implement a responsive web interface against the same backend capabilities. It must work in current standards-based browsers without the native app and must not require a Shlink administrative credential in the browser. Public views should remain usable without authentication; authenticated contribution and management views shall enforce the same authorization rules as the native app.

### 10.2 Main workflow

**Scan / enter → Resolve identifier → Show bucket or offer creation → Browse publicly → Authenticate when contributing → Upload or manage.**

The native iPhone and Macintosh applications and corresponding authenticated web interface shall:

- Scan QR codes where a camera is available, decode imported QR images where supported, and accept manual input.
- Recognize configured Shlink instances.
- Open existing accessible buckets.
- Offer creation of a user-associated bucket for unknown QR/text inputs.
- Display a multi-provider public resource listing.
- Allow invited authenticated users to upload.
- Allow authorized users to connect, inspect, rotate, and revoke provider integrations without exposing retained credentials.
- Allow owners to select replication targets, start batch fan-out uploads, import existing provider archives, and run or resume one-way mirror jobs.
- Allow owners to invite contributors and manage authorized Shlink destinations.
- Allow owners to set, test, replace, and remove the bucket's single primary metadata URL.
- Allow owners to add, order, edit, test, and remove unconditional or location-conditioned display links and geofences.
- Open the primary Google Drive document, Microsoft Office document, Etherpad pad, or equivalent metadata URL from the bucket landing page.
- Let users revisit the same bucket by rescanning or reentering the same identifier.

### 10.3 Camera and browser operation

People without the app shall be able to scan managed QR codes using a standard iPhone or Android camera and reach the bucket's responsive web interface through its Shlink URL. The default web destination should expose the bucket's public presentation and offer sign-in when the visitor has contribution or management rights. It must not be a dead-end marketing page or merely an instruction to install the app.

The web interface may ask the visitor for precise location permission after the base landing page loads. When granted, it shall obtain coordinates and reported accuracy, evaluate the bucket's location rules, and update the displayed links without changing the stable Shlink URL. The native scanner app shall support the same optional behavior using native location services.

Universal Links or Android App Links may route a managed URL into an installed native app, but the same URL must continue to work in a browser. Browser operation is a permanent, first-class continuity mode.

### 10.4 Macintosh operation

A Macintosh user shall be able to use either the native macOS application or a standards-based Mac browser. Both paths shall support browsing, additive uploads, batch uploads, provider connections, replication-target management, archive import and mirroring, metadata and display-link management, manifest export, and authorized Shlink administration. Native iPhone ownership or access must not be required to initialize or administer an archive from a Mac.

The native Macintosh application should support drag-and-drop and multi-file or folder selection for large archival imports. Long-running fan-out and mirroring jobs execute as durable backend jobs, so closing or sleeping the Mac does not discard server-accepted work. The Mac client shall reconnect and display current job state when reopened.

### 10.5 Native-app disappearance

The deployed web system must retain enough functionality to operate the archive if all native clients disappear. At minimum, a browser user with appropriate authorization must be able to browse buckets, follow and download resources, upload additive resources, connect providers, manage replication targets, run and resume import/mirror jobs, manage the primary metadata URL and display links, configure location rules, manage contributors, export manifests, and update authorized Shlink destinations. Operators must be able to restore or migrate the web interface, API, database, and Shlink configuration from documented backups without rebuilding a native application.

## 11. Backend and Data Model

### 11.1 Proposed components

| Component | Technology |
|---|---|
| iPhone client | Swift / SwiftUI |
| Macintosh client | Native macOS Swift / SwiftUI application |
| Web client | Responsive standards-based web interface, technology TBD |
| API / orchestration | Python / FastAPI |
| Metadata database | PostgreSQL |
| Short links | Shlink (PHP), via REST API |
| Identity | OIDC plus email magic links / codes |
| Storage adapters | Google Drive, Dropbox, S3 |
| Integration secrets | Backend-only encrypted credential store with rotation/revocation support |
| Transfer workers | Resumable fan-out, import, verification, and mirror jobs |
| Primary metadata URLs | Google Drive documents, Microsoft Office documents, Etherpad, generic HTTPS URLs |
| Contextual links | Arbitrary URLs with optional geofence presentation rules |
| Bucket and management pages | Responsive web frontend, TBD |
| Deployment | Docker Compose, proposed |

FastAPI and PostgreSQL are recommended implementation choices, not yet final user-mandated requirements.

### 11.2 Entities

**User:** internal ID, linked identities, verified email addresses, device credentials, recovery methods, status.

**Bucket:** internal ID, owner, title, description, public visibility, stable derived provider-space name, folder-name algorithm version, full identifier SHA-256, creation time.

**Identifier:** exact original string, type, namespace, canonical value/hash, normalized readable folder-name value, bucket association, claim status.

**Resource:** immutable ID, bucket, original/stored filenames, media type, size, checksum, uploader, timestamps, public listing status.

**StorageLocation:** resource ID, provider, provider-specific object reference, public download URL, replica state, verification status.

**ProviderConnection:** internal ID, owning user/operator, provider type, encrypted credential reference, granted scopes/capabilities, status, expiration when known, creation/rotation/revocation timestamps, and last successful use. Never store a secret directly in audit events or exported manifests.

**ProviderSpace:** bucket ID, provider connection ID, root namespace, recorded derived child name, provider-specific folder/prefix/container reference, source/replication-target role, creation/adoption status, and last inventory time.

**TransferJob:** immutable ID, bucket ID, initiating actor, operation type (fan-out, import, mirror, reconcile, or restore), source space when applicable, destination spaces, requested scope, state, progress counters, retry data, creation/start/completion timestamps, and summary.

**TransferItem:** transfer-job ID, logical resource ID or source object reference, source metadata, destination, copy state, attempt count, checksum/verification result, conflict state, and specific failure detail.

**MetadataURL:** bucket ID with a uniqueness constraint, provider/type, canonical URL, provider document ID when available, title, description, sharing mode, verification state, last successful check, and change timestamps.

**DisplayLink:** immutable ID, bucket ID, URL, title, description, display order, enabled state, unconditional/location-conditioned mode, and change timestamps.

**LocationRule:** display-link ID, human-readable place name, latitude, longitude, radius in meters, optional maximum accepted location-accuracy radius, enabled state, and change timestamps.

**ManagedShortLink:** Shlink instance, domain, short code, associated bucket, current destination, authorized owner, redirect history.

**Permission / Invitation:** bucket, email address or user, role, invitation status, verification, grant/revocation timestamps.

**AuditEvent:** actor, action, affected entity, time, previous/new values where relevant.

### 11.3 Security boundaries

The client must not receive Shlink administrative API secrets or long-lived provider credentials. Authorize changes server-side. Protect upload and admin endpoints from unauthorized access, prevent unsafe URL fetching/SSRF, and keep audit trails for administrative modifications. These protections serve data preservation rather than secrecy.

## 12. Long-Term Preservation and Portability

### 12.1 Export

Support export of a bucket manifest in a documented, open format (e.g., JSON), including:

- Stable bucket and identifier values.
- Exact original QR payload, canonical lookup value, folder-name algorithm version, normalized readable value, final derived provider-space name, and full identifier SHA-256.
- Resource IDs and metadata.
- Original and collision-adjusted filenames.
- Checksums and sizes.
- Provider locations and public links.
- Provider-space mappings and non-secret provider identifiers.
- Transfer-job summaries, reconciliation results, and replica verification states.
- Primary metadata URL, type, title, sharing/verification state, and change history.
- Display links and their location rules, including coordinates, radii, ordering, and presentation metadata.
- Upload and redirect history.
- Relevant ownership/contributor metadata, with appropriate care for personal information.

A full archival export should optionally package the actual files, not just external links. Exports must never contain provider passwords, OAuth tokens, API keys, secret keys, or references that can be exchanged directly for credentials.

### 12.2 Manifest redundancy

**Proposed:** Write copies of the archive manifest alongside stored files on multiple configured providers, so the archive can be reconstructed even if the central database disappears. Define versioning and integrity checks for those manifests.

### 12.3 Recovery

Design for the original owner losing an email account, phone, identity provider, cloud-storage subscription, or access to the native app. Public downloads should not depend on the owner's active login while provider links remain operational. Authorized web workflows must remain available without the native app. Independent restoration should be possible without credentials for the original deployment.

When a provider is declining, unaffordable, or scheduled for closure, an owner should be able to connect a replacement provider, run a full reconciliation from an available source replica, verify the new copy, and then remove the old provider from future replication targets. Historical records of the old replica remain in the manifest even after credentials expire or the service disappears.

### 12.4 Domain and service continuity

Document DNS/domain renewal, backup and restore, domain transfers, replacement landing pages, and short-link redirect restoration. A QR code pointing at a domain cannot survive loss of that domain without an alternate resolution or recovery mechanism; record this as an explicit preservation risk.

## 13. Independent Self-Hosting

The system should support independent operators deploying their own instance and domain, rather than requiring all archives to be hosted by the original operator.

**Proposed deployment package:** Docker Compose with Shlink, Python API, metadata database, complete responsive web interface, environment-based configuration, storage adapters, backup/export tooling, and operator documentation.

Support a configurable registry/list of recognized Shlink instances in the iPhone and Macintosh applications. Define whether cross-instance bucket discovery or exchange is required in a later phase.

## 14. Suggested Delivery Phases

### Phase 1 — Basic archive and linked metadata

Shared Swift/SwiftUI iPhone and Macintosh clients; iPhone camera scanning; Macintosh camera scanning, QR-image import, and manual entry; responsive web interface; ordinary-camera QR resolution; arbitrary identifier lookup; Google sign-in plus universal email magic links; bucket creation; public landing page; authenticated web upload and management; one generic HTTPS metadata URL; arbitrary display links; direct or landing-page resolution; file upload with dot-separated collision naming; basic contributor invitation; Python API and database.

### Phase 2 — Managed links

Shlink deployment and API integration; editable 302 redirects; managed-link ownership; redirect history; browser-first continuity and optional native-app recognition; vector QR artwork and durable-fabrication guidance.

### Phase 3 — Broader identity, metadata, and storage

Microsoft and Apple sign-in; multiple verified emails per account; local PIN convenience; Google Drive document, Microsoft Office, and Etherpad metadata adapters; protected retained provider credentials; Dropbox/Google Drive/S3 storage adapters; corresponding provider spaces; batch upload fan-out; existing-archive import and one-way mirroring; optional browser/native geolocation; geofenced display-link rules; provider accessibility reporting; owner-configured storage; contributor management.

### Phase 4 — Preservation and independent operation

Additional provider adapters; resumable cross-provider migration and reconciliation; checksum auditing; open manifests and full exports; recovery/restore tooling; manifest copies alongside files; documented self-hosting and migration.

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
18. An owner can replace a Google Drive metadata URL with a Microsoft Office document or Etherpad pad without reprinting the QR code, while retaining URL change history.
19. A bucket enforces zero or one primary metadata URL without requiring its format or content to follow one application-defined schema.
20. The system clearly identifies whether the external metadata URL is public, requires provider authentication, or failed its latest availability check.
21. A manifest export contains enough metadata-URL, data-resource, display-link, and location-rule data to reconstruct the QR associations without the original database.
22. A standard phone camera can scan a managed QR code without the native app and open the corresponding bucket web interface through its Shlink short URL.
23. An authorized person can use only a web browser to upload an additive resource, manage the primary metadata URL, display links, location rules, and contributors, export a manifest, and update an authorized Shlink destination.
24. Removing or discontinuing the native app does not prevent public access, authenticated contribution, authorized management, backup, restoration, or migration.
25. The system can export vector QR artwork with a human-readable Shlink URL and fabrication guidance suitable for metal marking, weather-resistant media, or resin encapsulation.
26. A storage-provider or metadata-destination change does not require remaking a permanent physical QR marker.
27. The project works with an unmodified upstream Shlink deployment; all geolocation requests and rule evaluation occur in this project's web, app, or API layers.
28. With location permission denied or unavailable, a visitor sees the primary metadata URL, all provider-backed data resources, and unconditional display links, plus a clear indication that optional nearby links may be hidden.
29. With coordinates matching a configured Times Square geofence, the browser and native app show the same conditioned display link; nonmatching coordinates do not show it.
30. Changing reported coordinates never changes or hides the bucket's primary metadata URL or provider-backed data resources.
31. The interface warns owners that location-conditioned links can be discovered by spoofing coordinates and are not access control.
32. Precise viewer coordinates are not stored by default or exposed in Shlink URLs, public manifests, browser history, or routine logs.
33. Opening a private Google metadata document delegates authentication and authorization to Google; this system neither grants access nor proxies around Google's decision.
34. Public, anyone-with-the-link editable, provider-login-required, and anonymously editable metadata URLs are all valid, and the landing page accurately reports observed accessibility without imposing a second application permission layer.
35. An owner can connect Google Drive, Dropbox, and S3 using delegated credentials that remain available to backend workers across sessions but never appear in clients, logs, manifests, exports, or QR codes.
36. Uploading a batch of one hundred images to a bucket configured for Google Drive, Dropbox, and S3 creates or adopts a corresponding space in each provider, copies every image to all three, verifies each copy, and reports partial failures truthfully.
37. Re-running or resuming an interrupted fan-out job does not create unintended duplicates and retries only incomplete, failed, or unverifiable copies.
38. An owner can select an existing Dropbox archive as a source, inventory it recursively, dry-run the migration, and mirror it into corresponding Google Drive and S3 spaces while preserving relative paths and recording checksums and source identifiers.
39. Reconciliation recognizes already verified replicas, identifies missing or changed objects, reports conflicts, and never silently overwrites or deletes source or destination data.
40. After adding a new provider adapter, an owner can mirror a bucket from an available old-provider replica to the new provider without changing the QR code, bucket identity, logical resource IDs, or manifest history.
41. Revoking or expiring a provider credential blocks affected transfers with a specific status while preserving queued work, manifests, and completed replica records for later reconnection.
42. A Macintosh user without an iPhone can install the native macOS application or use a Mac browser to create and administer a bucket, upload files, connect providers, run or resume mirroring, export a manifest, and manage an authorized Shlink destination.
43. The Macintosh application can resolve a managed QR identifier through an available camera, an imported QR image, or manual short-URL/short-code entry; lack of a camera does not prevent any non-scanning workflow.
44. A Macintosh user can drag and drop or select a large set of files or folders for upload, close the client after the backend accepts the job, and later reopen it to see accurate fan-out and verification progress.
45. `ABC123` produces the provider path `M_meta_B_bucket/ABC123` consistently across Google Drive, Dropbox, S3, and repeated runs.
46. The non-URL payload `Family Photos: 1980–1989` retains its exact original value while producing the derived child name `Family_Photos_1980_1989`.
47. The URL `https://example.org/a?x=1#top` remains unchanged as the navigable identifier while using `https_example_org_a_x_1_top` as its safe provider-space child name.
48. A multi-page text QR payload whose normalized value exceeds 255 characters produces a name no longer than 255 ASCII characters containing a readable prefix, `_sha256_`, and the complete lowercase SHA-256 digest of the exact original UTF-8 payload.
49. Two different short payloads that normalize to the same readable name receive distinct hash-suffixed provider-space names without changing either exact identifier.
50. The assigned derived name and naming-algorithm version survive export, restoration, software upgrades, provider fan-out, and provider-to-provider migration without being recalculated differently.

## 16. Open Decisions

1. **Identifier collisions:** How should globally public buckets versus personal bucket associations be selected for arbitrary QR values?
2. **Public claim verification:** What evidence establishes the authoritative owner of a managed QR code or public bucket?
3. **Contributor storage authority:** Which roles may use an owner's provider connections, and when may contributors connect separate accounts for a bucket?
4. **Replication policy:** How many targets should a new bucket recommend or require, and which failure states permit calling an upload operationally complete rather than fully replicated?
5. **Credential storage:** Which secrets manager or envelope-encryption design should back self-hosted and hosted deployments?
6. **Account recovery:** What portable recovery method complements email login and local device PIN?
7. **Owner succession:** How are archives transferred or administered when an owner dies or disappears?
8. **Public landing page:** Should every managed QR always resolve to a bucket landing page, or can owners choose direct provider redirects?
9. **Legal removal:** How should exceptional takedowns propagate to replicas and archival manifests?
10. **Cross-instance discovery:** Can separate self-hosted deployments discover buckets for the same identifier?
11. **Manifest format:** What schema and packaging format best support independent recovery over decades?
12. **Filename corner cases:** Define extensionless filenames, multi-part extensions, Unicode normalization, and concurrent collisions.
13. **Link rot monitoring:** Should the system periodically test public provider URLs and report or repair broken links?
14. **Default scan behavior:** Should a bucket with one primary metadata URL redirect directly to it, or always show the durable landing page first?
15. **Provider integration depth:** Which metadata providers require API-based previews, search, or synchronization beyond durable URL linking?
16. **External edit history:** Should the system archive snapshots or exports of mutable documents and pads, and how should that interact with provider-native version history?
17. **Geofence geometry:** Are circles sufficient for the first release, or should later versions support polygons, altitude, or routes?
18. **Location accuracy:** What default radius and maximum accepted device-accuracy value balance reliable matching with GPS limitations?
19. **Conditional presentation scope:** Should future releases allow location to alter explanatory text or ordering beyond display-link visibility while preserving location-independent metadata and data resources?
20. **Mirror scheduling:** Which providers and deployment sizes require scheduled reconciliation, webhooks, or incremental change feeds rather than user-initiated scans?
21. **URL canonicalization:** Which URL normalization rules identify equivalent lookup values without changing the exact URL or its folder-name derivation input?

---

**Design test:** Will this decision make it more likely that a person's great-great-grandchildren can discover, retrieve, and understand the archive without depending on one particular account, device, service, or software vendor?
