# Phased Development Plan

## 1. Purpose

This plan turns the project overview into incremental, testable releases. Every
phase ends with a deployable minimum viable product (MVP) that can be
demonstrated and evaluated independently. A phase is complete only when its
automated tests, end-to-end demonstration, documentation, and exit criteria
pass.

The sequence deliberately completes and stabilizes the browser application
before native development begins. The iPhone and macOS applications are built
in Phase 7 against the tested public URLs and versioned Laravel API rather than
becoming prerequisites for archive access or forcing premature API stability.

## 2. Selected Technology Baseline

| Area | Selected approach |
| --- | --- |
| Web application and API | Laravel on a supported PHP release |
| Web server | Apache with the standard PHP Apache container pattern |
| Web interface | Laravel Blade and locally hosted progressive JavaScript |
| Database | PostgreSQL |
| Background work | Laravel queues with a durable production queue backend |
| Scheduled work | Laravel scheduler |
| Short URLs | Unmodified upstream Shlink server |
| Shlink administration | Self-hosted official Shlink web client |
| Native clients | Shared Swift and SwiftUI code for iPhone and macOS |
| Provider integrations | Laravel adapters for Drive, Dropbox, and S3 |
| Shared development host | Small Google Compute Engine VM |
| Deployment | Docker Compose on the Google VM |
| Automated web testing | PHPUnit/Pest, Laravel HTTP tests, and browser tests |
| Automated Apple testing | XCTest and XCUITest |
| API contract | Versioned JSON API documented with OpenAPI |

Blade is preferred for the first browser MVP because it minimizes frontend
complexity and leaves a durable server-rendered interface. Small JavaScript
modules may add geolocation, upload progress, and other progressive behavior.
The implementation may adopt a richer frontend later without changing public
bucket URLs or API contracts.

All JavaScript required by project-owned pages must be built into versioned
assets and hosted with the application or on an owner-controlled static origin.
Public CDNs, package CDNs, tag managers, remote analytics scripts, and embedded
third-party widgets are prohibited runtime dependencies. Performance is not a
reason to introduce an external script host; reproducibility, backup coverage,
and long-term availability take priority.

## 3. Rules Applied to Every Phase

### 3.1 Definition of done

Every phase must provide:

1. A deployable application that preserves every capability completed in
   earlier phases.
2. Database migrations, seed or fixture data, and documented local setup.
3. Automated unit, integration, authorization, and regression tests for the
   phase's behavior.
4. At least one end-to-end happy-path test and tests for important failures.
5. A repeatable manual demonstration script using non-production accounts.
6. Structured, specific error messages and sufficient operational logs without
   credentials or precise viewer locations.
7. Updated OpenAPI documentation, architecture notes, and operator runbooks.
8. Backup and restore verification for data introduced in the phase.
9. Passing static analysis, formatting, dependency-audit, and secret-scanning
   checks in continuous integration.
10. No unresolved critical or high-severity security defects in the delivered
    scope.
11. A production-asset check proving that every script URL is owner-controlled,
    all dependencies are pinned, and no browser code is fetched at runtime from
    a package registry, CDN, tag manager, or remote widget service.

### 3.2 Development and testing environments

Use the public Google instance as the primary integration environment from the
beginning. Do not require a locally hosted Laravel, Shlink, or PostgreSQL stack
for ordinary development.

- **Local Macintosh:** source editing, fast unit tests, static analysis,
  dependency installation, locally hosted asset compilation, and Docker-image
  builds.
- **Continuous integration:** isolated automated tests with deterministic
  fixtures and fake provider adapters. CI must not mutate the shared instance.
- **Google development instance:** public DNS and HTTPS, Apache/Laravel,
  PostgreSQL, Shlink, queue workers, scheduler, browser integration tests,
  OAuth callbacks, and provider sandbox/test accounts.
- **Production:** a separate Google VM and Compose project using the same tested
  images and templates with production environment values. It has separate
  databases, links, credentials, static IP, storage, and backups.

Development and production may share the Google Cloud project, Cloud DNS zone,
and Artifact Registry, but no mutable runtime state. Production deploys the
exact image digest verified in development rather than rebuilding it.

Provider contract tests must run against both a deterministic fake adapter and
the provider's sandbox or dedicated development account. Real personal archives
and production provider credentials must not be used in automated tests.

The shared development instance is Internet-accessible and must use
production-safe operational settings despite its development purpose:

- `APP_DEBUG=false`.
- Strong unique credentials.
- HTTPS after initial certificate bootstrap.
- No sample secrets in deployed configuration.
- Regular backups and tested restoration.
- Test or non-sensitive data until production readiness is declared.
- Authentication and authorization enabled as soon as those features exist.

### 3.3 Architectural boundaries

- Shlink remains an unmodified upstream deployment.
- The official Shlink web client is self-hosted. It is separate from Shlink
  server and consumes the pinned server's upstream `/rest/v3` API.
- Laravel is served by Apache using a conventional PHP/Apache container. Do not
  introduce Nginx or a separate PHP-FPM proxy tier without a demonstrated need.
- Laravel owns public landing pages, application authentication, authorization,
  folder-name derivation, manifests, transfer orchestration, and geolocation
  rules.
- Providers own access control for their underlying documents and objects.
- SwiftUI clients use the same versioned Laravel API as the web interface.
- Long-running uploads, copies, and reconciliation execute as durable,
  idempotent Laravel queue jobs.
- Exact QR payloads and assigned folder names are persisted; adapters never
  derive names independently.
- JavaScript is bundled during a controlled build, deployed with Laravel, and
  included in application backups. Production requests never install or fetch
  executable browser dependencies from third parties.

### 3.4 Frontend asset stability policy

- Commit dependency manifests and lockfiles. Release builds must use frozen or
  immutable lockfile installation and fail on dependency drift.
- Generate content-hashed JavaScript filenames and retain the asset manifest
  needed by Laravel to resolve them.
- Prefer small first-party modules and server-rendered Blade over dependencies
  that are not necessary for the specified behavior.
- Do not use `<script>` tags whose source is a public CDN or third-party host.
- Do not use runtime module imports from remote URLs.
- Do not embed third-party widgets that execute remote code in project pages.
- Use a restrictive Content Security Policy, targeting `script-src 'self'` or
  an owner-controlled equivalent without `unsafe-eval`.
- Ensure public bucket content remains readable and links remain usable when
  JavaScript is unavailable.
- Include compiled assets and their manifest in deployment, backup, restore,
  and disaster-recovery tests.
- External identity and storage providers may serve their own pages after a
  redirect or link traversal; their scripts must not be embedded dependencies
  of the Laravel interface.

### 3.5 Initial hosting and operations policy

The detailed operator procedure is maintained in
[`InitialCloudSetup.md`](../InitialCloudSetup.md).

- Run the shared development system on one small Google Compute Engine VM with
  a persistent SSD and static IP address.
- Run Apache/PHP, Laravel workers, the Laravel scheduler, PostgreSQL, and
  unmodified Shlink through Docker Compose on that VM.
- Use the standard PHP Apache container pattern to keep Laravel deployment
  direct and familiar.
- Serve all project-owned static assets through Apache from the deployed build.
- Use the public Google instance for browser development and integration; do not
  add `/etc/hosts`, local DNS, or locally trusted certificate requirements to
  the normal workflow.
- Keep database and service state on persistent storage and send encrypted
  backups off the VM.
- Maintain at least one off-VM backup; prefer an additional copy outside the
  Google account or provider.
- Test restoration onto a clean replacement VM.
- Do not use serverless services, Kubernetes, a service mesh, or autoscaling
  tiers for the initial deployment.
- Do not introduce Nginx, a CDN, or another reverse-proxy layer merely for
  theoretical performance. Add infrastructure only for a measured requirement.
- Preserve portable container and backup boundaries so a later move to managed
  services or another host does not require redesigning the application.

### 3.6 Environment configuration policy

- Keep deploy-specific values, resource names, domains, and secret references
  in the untracked `.env` file.
- Commit `example.env` with safe example values and descriptions, but no working
  credentials or private keys.
- Keep environment hostnames out of source code and Compose templates. Read the
  Laravel host from `APP_HOSTNAME`, public redirect host from
  `PUBLIC_SHORT_DOMAIN`, and Shlink administration host from
  `SHLINK_WEB_HOSTNAME`.
- Use `DEPLOY_ENVIRONMENT` and `COMPOSE_PROJECT_NAME` to select an isolated
  environment. Never infer production from a branch name or hostname substring.
- Require an explicit environment argument and confirmation for production.
  Print the target VM, IP, hostnames, image digest, databases, and backup
  destination before changing production.
- Deployment scripts may load `.env` locally but must never copy the entire
  local deployment `.env` to the VM. Generate a least-privilege server
  environment file containing only values required by deployed services.
- Treat `.env`, service-account key files, generated SSH material, and local
  deployment state as secrets or machine-local state and exclude them from Git.
- Use Laravel configuration accessors rather than calling `env()` outside
  configuration files, so production configuration caching remains reliable.

### 3.7 Google Cloud authentication boundary

`GOOGLE_API_KEY` may be present in `.env` for an application feature that calls
a Google API which explicitly supports API keys. It is **not** sufficient to
push images, create VMs, assign IAM roles, reserve addresses, or update Cloud
DNS. Those deployment operations require an authenticated Google principal.

For interactive local development, use one of these approaches:

1. `gcloud auth login` for the Google Cloud CLI, plus
   `gcloud auth application-default login` only when local libraries require
   Application Default Credentials.
2. Prefer service-account impersonation when deployment should use a dedicated
   identity without downloading a long-lived key.
3. If unattended local automation truly requires a service-account key, store
   it outside Git under `.credentials/`, point
   `GOOGLE_APPLICATION_CREDENTIALS` to it, scope its IAM roles narrowly, and
   rotate it. Never place the JSON key contents in `.env`.

The deployment identity needs only the roles required for its enabled steps,
typically permissions for Artifact Registry writes, Compute Engine
administration or a narrower custom role, service-account use, and Cloud DNS
changes when `GCP_MANAGE_DNS=true`. The VM runtime service account should be a
different, less privileged identity with read access to its Artifact Registry
images and only the runtime APIs it actually uses.

### 3.8 First-time Google Cloud setup

The operator must complete these steps once before automated deployments:

1. Create or select a Google Cloud project with billing enabled.
2. Install the Google Cloud CLI and Docker on the development Macintosh.
3. Authenticate `gcloud`, select `GCP_PROJECT_ID`, and set the configured region
   and zone.
4. Enable Compute Engine, Artifact Registry, Cloud DNS when used, IAM, and any
   other APIs explicitly required by the deployment.
5. Create the Docker-format Artifact Registry repository named by
   `GCP_ARTIFACT_REPOSITORY` in `GCP_REGION`.
6. Create a narrowly scoped deployment service account and a separate VM
   runtime service account; grant only the required roles.
7. Configure Docker authentication for the regional Artifact Registry host by
   running `gcloud auth configure-docker` for that host.
8. Reserve the regional static external IPv4 address named by
   `GCP_STATIC_IP_NAME`.
9. Create firewall rules that allow inbound TCP 80 and 443 to the VM's web tag.
   Restrict SSH through OS Login, IAP, or a documented trusted source rather
   than exposing unrestricted password SSH.
10. Create the public Cloud DNS managed zone named by `GCP_DNS_ZONE` for the
    fully qualified zone in `GCP_DNS_DOMAIN` if it does not already exist.
11. Read the managed zone's assigned authoritative nameservers. At the domain
    registrar, replace the domain's current nameservers with all of the Google
    nameservers exactly as assigned. This registrar delegation is the one
    unavoidable manual DNS step and cannot be performed by the Cloud DNS API
    before Google is authoritative for the domain.
12. Before changing nameservers, reproduce any existing DNS records that must
    survive in the Cloud DNS zone, especially MX, TXT, CAA, verification, and
    existing service records. Nameserver delegation moves authority for the
    entire zone; omitted records stop resolving after propagation.
13. Wait until public NS queries return the assigned Google nameservers. After
    that one-time delegation, set `GCP_MANAGE_DNS=true`; this codebase can then
    manage its application and Shlink records through Cloud DNS.
14. Copy `example.env` to `.env` and fill in local values. Do not commit `.env`.

### 3.9 Automated local-to-VM development workflow

Provide a checked-in deployment command, such as `bin/deploy-gcp`, with
idempotent subcommands. The implementation language may be shell or a small
Laravel command, but it must show commands before executing them, stop on
errors, validate required variables, and never print secrets.

The workflow shall perform these operations:

1. **Preflight:** Load `.env`; validate `gcloud`, Docker, project, region, zone,
   authenticated account, required APIs, IAM access, clean image tag, and DNS
   configuration. Explicitly reject using `GOOGLE_API_KEY` as deployment
   authentication.
2. **Build and test:** Run the complete test suite and build deployable Docker
   images with locally bundled frontend assets. Use an immutable source commit
   or release identifier for `DEPLOY_IMAGE_TAG`; do not deploy an ambiguous
   `latest` tag as the only reference.
3. **Push:** Tag images using the Artifact Registry form
   `REGION-docker.pkg.dev/PROJECT/REPOSITORY/IMAGE:TAG`, authenticate Docker
   through `gcloud`, and push them from the local development machine.
4. **Provision:** Reuse or create the reserved static IP, firewall rule, runtime
   service account, persistent disk, and VM named in `.env`. VM creation shall
   use the configured zone, machine type, supported Debian image family,
   persistent boot-disk size, runtime service account, and web network tag.
5. **Bootstrap:** Install Docker Engine and the Compose plugin if absent, create
   application directories, attach/mount persistent storage, authenticate the
   VM to Artifact Registry through its attached service account, and install no
   application source dependencies at request time.
6. **Configure:** Render a server-only environment file and Compose
   override from explicit templates. Transfer them securely to the VM without
   transferring the local `.env` or deployment credentials.
7. **Deploy:** Pull the immutable image tags, run database migrations with a
   documented rollback/backup checkpoint, start or update Apache/Laravel,
   PostgreSQL, workers, scheduler, and unmodified Shlink, and remove obsolete
   containers only after health checks pass. This is the normal development
   loop, not an occasional production-release operation.
8. **DNS:** Read the environment's reserved IP. When `GCP_MANAGE_DNS=true`, use
   a Cloud DNS transaction to add or replace A records for `APP_HOSTNAME`,
   `PUBLIC_SHORT_DOMAIN`, and `SHLINK_WEB_HOSTNAME`, all pointing to that
   environment's static IP. When DNS is external, print all three records.
9. **Apache routing:** Render three name-based Apache virtual hosts. Route
   `APP_HOSTNAME` to Laravel, `PUBLIC_SHORT_DOMAIN` to Shlink server redirects,
   and `SHLINK_WEB_HOSTNAME` to the official Shlink web client while proxying
   `/rest/...` to Shlink server. Reject unrecognized Host headers.
10. **TLS:** Obtain or renew certificates covering all three configured names
    using the selected Apache-compatible ACME procedure after DNS resolves.
    Certificate state must live on persistent storage and be included in
    restoration plans. Test automatic renewal with `certbot renew --dry-run`.
11. **Verify:** Check authoritative and public DNS, TLS for all three hosts,
    Laravel health/readiness, local static assets, public Shlink resolution,
    Shlink web-client access, `/rest/v3` API access, queue execution, scheduler
    status, and database access.
    Fail the deployment if any required check fails.
12. **Record:** Write non-secret deployment output—project, zone, VM, image
    digest, commit, reserved IP, and timestamp—to ignored `.deployment/` state
    and to the server's audit/deployment log.

The provision and deployment paths must be safe to rerun. Existing resources
with matching configured names shall be updated or verified, not duplicated.
Destructive replacement of a VM, disk, database, static IP, or DNS record must
require an explicit flag and a fresh backup.

### 3.10 DNS behavior

Automatic hostname attachment is possible only when the configured DNS zone is
managed by the authenticated Google Cloud project. Cloud DNS changes shall use
transactional record updates: start a transaction, remove the old A record when
present, add the new A record with the configured TTL, and execute the
transaction. The deployment must verify the resulting authoritative and public
DNS answers before requesting TLS.

For the initial single-VM deployment, create these records from environment
configuration:

- An A record for `APP_HOSTNAME` pointing to the environment's reserved IP.
- An A record for `PUBLIC_SHORT_DOMAIN` pointing to the same reserved IP.
- An A record for `SHLINK_WEB_HOSTNAME` pointing to the same reserved IP.

Do not create a wildcard record unless a later requirement justifies it. Do not
change unrelated zone records. DNS automation must inspect the transaction and
show the proposed additions/removals before execution.

If the domain uses another DNS host or registrar, the deployment cannot update
it merely because a Google API key is present. It shall output the required
records using all three configured hostnames and the reserved IP, then wait for
the operator to update DNS and rerun verification.

### 3.11 What the domain owner must do

The intended steady state is full-zone delegation to Google Cloud DNS. The
domain owner performs these actions:

1. Confirm access to the domain registrar and preserve/export the current zone.
2. Run the codebase's DNS bootstrap command to create or inspect the Cloud DNS
   public zone and print its assigned nameservers.
3. Copy all required existing records into Cloud DNS and verify the proposed
   zone before delegation.
4. At the registrar, replace the authoritative nameserver set with the complete
   Google-assigned set. Do not add the Google nameservers alongside unrelated
   old nameservers.
5. Wait for delegation propagation and verify public NS responses.
6. Rerun deployment. From then on, the codebase manages its two A records and
   can manage later project records through the authenticated Cloud DNS API.

Domain registration, renewal, registrar account security, and the initial
nameserver change remain outside this codebase. The registrar should use strong
account security, renewal reminders, and registry lock where available.

## 4. Phase 0 — Engineering Foundation

### 4.1 Goal

Create a reproducible development and deployment foundation before product
features accumulate.

### 4.2 Scope

- Initialize the Laravel application and PostgreSQL database.
- Create Docker Compose services for Apache/PHP with Laravel, PostgreSQL, a
  queue worker, the scheduler, and unmodified Shlink.
- Establish environment-based configuration without committing secrets.
- Add health, readiness, and version endpoints.
- Establish API versioning under `/api/v1`.
- Add CI for PHP tests, formatting, static analysis, migration checks, and
  dependency and secret audits.
- Add a local JavaScript build with pinned dependencies, content-hashed output,
  and no runtime CDN references.
- Document local setup, test execution, database reset, and backup/restore.
- Document provisioning a small Compute Engine VM, attaching persistent
  storage, configuring the static IP and HTTPS, and restoring a complete stack.
- Add idempotent local deployment commands for preflight, build/push,
  provisioning, configuration, deployment, DNS, TLS, and verification.

### 4.3 MVP at phase end

A developer can clone the repository, run local checks, build and push an
immutable image, deploy it to the Google development VM, and verify the public
Laravel and Shlink hosts over HTTPS.

### 4.4 Required tests

- Laravel application boots with a clean database.
- Apache serves Laravel routes, local static assets, health endpoints, and
  expected Laravel front-controller behavior.
- All migrations apply to an empty database and roll back in development.
- Health and readiness endpoints distinguish application and database health.
- Queue smoke test dispatches and completes a job.
- Laravel can call the Shlink health/API endpoint without modifying Shlink.
- A browser-network test fails if a project page requests a script from any
  non-owner-controlled origin.
- A clean production build succeeds from the lockfile and serves all scripts
  locally after package-network access is disabled.
- Backup and restore smoke test recovers seeded PostgreSQL data.
- A clean replacement-VM exercise restores PostgreSQL, Shlink mappings,
  Laravel configuration, and locally hosted assets from off-VM backups.
- Deployment preflight rejects missing principal authentication even when
  `GOOGLE_API_KEY` is configured.
- Repeating provisioning and deployment with unchanged inputs creates no
  duplicate VM, address, disk, firewall, DNS, or Artifact Registry resources.
- A development deployment pushes an immutable image from the local machine,
  updates the VM, configures or reports DNS, and passes HTTPS health checks.

### 4.5 Exit demonstration

1. Run all local and CI-equivalent checks from a clean development checkout.
2. Build and push an immutable image.
3. Deploy it to the Google development instance.
4. Show Apache/Laravel, PostgreSQL, queue worker, scheduler, and Shlink as
   healthy through the public HTTPS hosts.

## 5. Phase 1 — Browser-First QR Bucket MVP

### 5.1 Goal

Deliver the smallest useful archive: create a bucket from arbitrary QR content
and retrieve its public landing page without a native app.

### 5.2 Scope

- Implement buckets, identifiers, resources, and storage-location records.
- Accept arbitrary URL or text identifiers through the web interface.
- Implement and version `folder_name_v1` exactly as specified.
- Persist the exact QR payload, canonical lookup value, normalized readable
  value, complete SHA-256, and assigned provider-space name.
- Enforce one logical bucket per managed QR association.
- Provide public bucket landing pages at durable Laravel URLs.
- Allow an authenticated development owner to create a bucket and upload files
  into a local filesystem provider adapter.
- Enforce append-only uploads and dot-separated collision filenames.
- List and download public resources without authentication.
- Export an initial JSON bucket manifest.
- Generate SVG and PNG QR artwork containing the Laravel landing-page URL for
  this phase. Shlink-backed artwork arrives in Phase 2.

### 5.3 MVP at phase end

A user can enter or scan an arbitrary identifier in a browser, create its
bucket, upload files, print its QR code, scan it with an ordinary phone camera,
and browse or download the files from the public Laravel landing page.

### 5.4 Required tests

- Golden-vector unit tests for every documented `folder_name_v1` example.
- Property tests confirm derived names use safe ASCII and never exceed 255
  characters.
- Long-payload and normalization-collision tests preserve exact payloads and
  append the complete SHA-256 correctly.
- Repeated derivation is deterministic across PHP and a Swift test fixture.
- Bucket creation is idempotent under concurrent duplicate requests.
- Upload collision tests produce `.001`, `.002`, and later suffixes atomically.
- Anonymous users can browse and download but cannot upload or administer.
- Manifest export can reconstruct identifiers and resource associations.
- Browser end-to-end test creates, scans/opens, uploads, and downloads.

### 5.5 Exit demonstration

Use `Family Photos: 1980–1989` and one payload longer than 255 characters to
create two buckets. Upload colliding filenames, scan each generated QR code
with an ordinary camera, and show the public files, stable names, and manifests.

## 6. Phase 2 — Durable Shlink URLs and Browser Routing MVP

### 6.1 Goal

Replace phase-one Laravel URLs in newly generated QR codes with stable Shlink
URLs while preserving a complete browser-only workflow.

### 6.2 Scope

- Integrate Laravel with the documented Shlink REST API.
- Create one managed short code per managed bucket and retain redirect history.
- Point Shlink to the Laravel bucket landing page with temporary redirects and
  suitable cache controls.
- Generate SVG/PNG QR artwork containing the complete Shlink URL plus
  human-readable URL and short code.
- Add fabrication guidance for metal, weather-resistant media, and resin.
- Preserve the full browser experience when no native application is present.
- Configure and verify separate Apache virtual hosts for the application and
  Shlink using the environment-provided hostnames.

### 6.3 MVP at phase end

A Shlink QR code can be printed, scanned with an ordinary camera, and resolved
through the Shlink host to the public Laravel bucket page. An owner can change
its Laravel destination without reprinting the QR marker.

### 6.4 Required tests

- Shlink adapter contract tests use a real staging Shlink and a fake client.
- Laravel never exposes the Shlink API key to browser or Swift clients.
- Redirect changes retain history and use temporary redirect behavior.
- QR SVG has a quiet zone, configured error correction, and readable caption.
- Browser resolution works without any native application installed.
- Host-routing tests prove the application host serves Laravel and the
  short-link host serves Shlink without cross-routing unknown Host headers.
- End-to-end test changes a short-code destination and reuses the original QR.

### 6.5 Exit demonstration

Print one QR code and open it through an ordinary phone camera and a desktop
browser. Change its destination in Laravel and show that the unchanged printed
marker reaches the new page.

## 7. Phase 3 — Accounts, Collaboration, and Metadata MVP

### 7.1 Goal

Make buckets manageable by real owners and contributors while keeping reading
public and provider access control provider-native.

### 7.2 Scope

- Add Google OpenID Connect and universal email magic-link/one-time-code login.
- Model users, verified emails, owner/contributor permissions, and invitations.
- Add invitation acceptance, revocation, resend, expiration, and audit events.
- Enforce the authorization matrix in Laravel policies and API middleware.
- Support zero or one primary metadata URL with change history and observed
  accessibility status.
- Support ordered unconditional display links.
- Complete owner and contributor workflows in the responsive web interface.
- Add public landing-page presentation for metadata, data resources, and links.
- Ensure Google, Microsoft, Etherpad, and other resources enforce their own
  authentication; Laravel must not proxy around provider permissions.
- Add open JSON manifest export for users, permissions, metadata URL, links,
  identifiers, and resources without secrets.

### 7.3 MVP at phase end

An owner can sign in, create and manage a bucket, invite an arbitrary email
address, set a metadata URL, add display links, and let the invitee upload a new
file. Anonymous visitors can still read public content without an account.

### 7.4 Required tests

- OIDC callback and magic-link tests cover success, replay, expiration, and
  unverified claims.
- Every authorization-matrix cell has an automated policy test.
- Contributors cannot overwrite, delete, change Shlink, or manage invitations.
- Invitation acceptance works for non-Google email addresses.
- A second primary metadata URL atomically replaces the prior primary while
  retaining history.
- Private provider URLs remain private and are not proxied by Laravel.
- Manifest and logs contain no session tokens, magic tokens, or provider
  credentials.
- API contract tests freeze the browser-used behavior for later native clients.

### 7.5 Exit demonstration

Sign in as an owner, invite a non-Google email address, accept in a separate
browser, upload as the contributor, and verify anonymous reading. Link one
public Etherpad and one Google document requiring Google login to demonstrate
provider-native access control.

## 8. Phase 4 — Single-Provider Durable Storage MVP

### 8.1 Goal

Move beyond local storage by delivering one production-quality provider
integration and the credential/job architecture required for later mirroring.

### 8.2 Scope

- Implement encrypted backend credential references, rotation, and revocation.
- Define and test the provider-adapter contract.
- Implement Google Drive as the first complete adapter. The implementation may
  choose S3 first if deployment constraints make it substantially simpler, but
  only one adapter is required to exit this phase.
- Create or adopt `M_meta_B_bucket/<derived_name>` provider spaces.
- Add durable `ProviderConnection`, `ProviderSpace`, `TransferJob`, and
  `TransferItem` records.
- Upload single files and large batches through idempotent queue jobs.
- Calculate source SHA-256 and verify destination copies using the strongest
  trustworthy provider mechanism.
- Add progress, retry, blocked-credential, and specific failure states.
- Support browser multi-file upload and reconnection to server-side jobs after
  page closure, browser restart, or network interruption.

### 8.3 MVP at phase end

An owner can connect one real provider, create the stable per-QR provider
space, upload one hundred images from the web interface, close the browser, and
later see every object either verified or reported with a retryable status.

### 8.4 Required tests

- Secrets are encrypted at rest and absent from logs, API responses, manifests,
  queue payloads where avoidable, and client storage.
- Credential rotation and revocation produce expected worker behavior.
- Adapter contract tests cover list, create space, read, write, metadata, and
  verification capabilities.
- Queue retries do not create unintended duplicates.
- Provider-space paths exactly match persisted `folder_name_v1` assignments.
- A 100-file test covers progress, partial failure, retry, and completion.
- Client termination does not cancel accepted backend work.
- Staging test uploads and verifies files in a dedicated provider account.

### 8.5 Exit demonstration

Connect a staging provider account, select 100 images in the web interface,
close it during transfer, and reopen it to show verified completion. Revoke the
provider credential, show a blocked job, reconnect, and resume it.

## 9. Phase 5 — Multi-Provider Replication and Migration MVP

### 9.1 Goal

Deliver the project's central preservation capability: verified copies across
independent providers and migration away from a declining provider.

### 9.2 Scope

- Complete Google Drive, Dropbox, and Amazon S3 adapters.
- Create corresponding stable spaces on every selected target.
- Fan out new uploads to all configured targets.
- Inventory an existing provider folder/prefix recursively with pagination.
- Provide dry-run object/byte estimates and conflict previews.
- Implement resumable one-way import, mirror, and reconciliation jobs.
- Preserve relative paths, filenames, source IDs, sizes, media types,
  timestamps where supported, and checksums.
- Recognize verified existing copies and transfer only missing or changed data.
- Never silently overwrite or propagate deletions.
- Produce per-object and aggregate reconciliation reports.
- Allow a provider to be removed from future targets without erasing history.
- Export manifests and optionally package actual files.

### 9.3 MVP at phase end

An owner can start with an existing Dropbox archive, preview and copy it to
Google Drive and S3, interrupt and resume the process, verify all replicas, and
then remove Dropbox from future targets without changing the QR code or bucket
identity.

### 9.4 Required tests

- Every adapter passes the same provider contract suite.
- Pagination tests prove complete source inventory.
- Fan-out tests cover one success, one transient failure, and one revoked
  credential without rolling back successful copies.
- Reconciliation is idempotent and skips verified replicas.
- Conflict tests never silently overwrite either side.
- Deletion on one provider does not automatically delete another copy.
- Large staged migration can stop and resume from durable checkpoints.
- Final reports reconcile expected counts, bytes, and verification states.
- Restore test rebuilds bucket associations from an exported manifest.

### 9.5 Exit demonstration

Use an existing Dropbox fixture archive containing nested directories and
duplicate filenames. Run a dry run, mirror it to Google Drive and S3, interrupt
the worker, resume it, verify all copies, and rerun reconciliation to show that
no unnecessary duplicates are created.

## 10. Phase 6 — Contextual Links and Preservation Operations MVP

### 10.1 Goal

Complete location-conditioned presentation and production preservation tools
without weakening the stable archive or treating location as security.

### 10.2 Scope

- Add ordered display links with optional circular geofences.
- Request precise browser location only after an explanation and explicit
  permission through the HTTPS Geolocation API.
- Keep metadata URL and provider-backed resources unchanged at every location.
- Provide deterministic no-location behavior and user-initiated retry.
- Avoid retaining precise viewer coordinates by default.
- Warn owners that location can be spoofed and is not access control.
- Add scheduled provider health checks, checksum audits, and reconciliation.
- Write redundant manifests beside provider data where supported.
- Complete backup, restore, domain-transfer, Shlink restoration, and provider
  replacement runbooks.
- Add an adapter-development guide and prove it with a fake fourth provider.
- Perform durable QR fabrication trials using at least one metal/resin sample.

### 10.3 MVP at phase end

The same QR code displays ordinary links everywhere and an additional test link
inside a configured Times Square geofence in the browser. Operators can run
health checks, restore the service from backups, and migrate a fixture archive
to a newly added adapter.

### 10.4 Required tests

- Browser geofence tests use reusable coordinate fixtures intended for later
  Swift contract tests.
- Permission denied, timeout, inaccurate location, matching, and nonmatching
  cases render the required content.
- Metadata URL and provider resources remain invariant across all locations.
- API tests demonstrate spoofed coordinates can reveal a conditioned link and
  confirm that no security claim depends on geolocation.
- Precise viewer coordinates are absent from routine logs, manifests, URLs, and
  browser history.
- Geolocation and all other interactive browser behavior use locally hosted
  scripts and continue to work when public CDN domains are blocked.
- Scheduled integrity checks detect a missing or altered replica.
- Disaster-recovery exercise restores PostgreSQL, Laravel, Shlink redirects,
  manifests, and public landing pages in a clean environment.
- Fake fourth-provider adapter receives a verified mirror without changing the
  QR code or logical resource IDs.
- Fabricated QR sample scans with multiple ordinary camera applications.

### 10.5 Exit demonstration

Show no-location, matching-location, and nonmatching-location views for one QR
code. Then simulate an unavailable provider, restore an alternate replica,
recover the stack from backups, and mirror the archive into the fourth-provider
adapter while the original QR code continues to work.

## 11. Phase 7 — iPhone and macOS Native Applications MVP

### 11.1 Goal

Build native Apple clients only after the browser application, domain model,
authentication, provider workflows, and versioned API have passed their MVPs.

### 11.2 Scope

- Create a shared Swift package for API models, authentication, bucket logic,
  folder-name golden vectors, geofence rules, and transfer-job status.
- Build SwiftUI iPhone and macOS applications with platform-appropriate views.
- Support the same owner, contributor, metadata, display-link, provider,
  upload, mirroring, manifest, and Shlink workflows available on the web.
- On iPhone, scan QR codes through native camera APIs and use native file/photo
  pickers.
- On macOS, support available cameras, QR-image and clipboard import, manual
  URL/short-code entry, drag-and-drop, and folder selection.
- Implement native geolocation with the same rule fixtures and no-location
  behavior as the browser.
- Store device credentials in Keychain and never embed deployment/provider
  administrative secrets.
- Reconnect to durable backend jobs after suspension, termination, sleep, or
  network interruption.
- Preserve browser operation as the complete continuity path.

### 11.3 MVP at phase end

An owner can manage the same production bucket from the browser, iPhone, and
Macintosh applications. An ordinary camera still reaches the browser when no
app is installed, and all three clients show equivalent authorized resources,
location-conditioned links, and transfer status.

### 11.4 Required tests

- XCTest verifies API decoding, authentication state, folder-name vectors,
  geofence fixtures, and transfer-state handling.
- XCUITest covers iPhone scanning and core owner/contributor workflows.
- macOS UI tests cover image/clipboard/manual QR input, drag-and-drop, and large
  transfer reconnection.
- Contract tests compare browser and Swift results for the same account,
  bucket, coordinates, permissions, and transfer job.
- Keychain and logging tests confirm secrets and tokens are not exposed.
- Compatibility tests verify a QR URL works in browser-only, iPhone, and macOS
  paths without changing the printed code.

### 11.5 Exit demonstration

Create and configure a bucket in the web interface, scan it on iPhone, import
the QR image on macOS, upload files from both native clients, observe one
durable transfer from all clients, and uninstall the apps to prove the same QR
still opens the complete browser experience.

## 12. Deferred Work After Phase 7

The following should not block the phased MVPs unless an earlier implementation
decision makes them necessary:

- Continuous bidirectional provider synchronization.
- Polygon, altitude, route, or beacon location rules.
- Facebook login.
- Confidential-by-default archives, end-to-end encryption, or DRM.
- Native Android application.
- Automated destructive deletion propagation.
- Provider-content indexing or previews beyond durable URL linking.
- Cross-instance bucket discovery.

## 13. Release and Feedback Cadence

At the end of each phase:

1. Deploy the MVP to staging.
2. Run the automated suite and documented exit demonstration.
3. Conduct a short usability test with at least one owner and one reader; add a
   contributor when the phase supports collaboration.
4. Record defects, operational observations, and architecture decisions in
   GitHub.
5. Fix release-blocking defects before starting the next phase.
6. Tag the passing commit so GitHub history identifies the exact phase MVP.

Phases are ordered by dependency, not by fixed calendar duration. Estimates
should be made after Phase 0 establishes team size, deployment environment,
provider-account availability, and Apple signing constraints.
