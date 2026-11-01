# Initial Google Cloud Setup

## 1. Purpose

This runbook describes the one-time setup and repeatable deployment process for
the public Google-hosted development environment. It covers:

- Preparing the local development Macintosh.
- Creating and filling the untracked `.env` file.
- Authenticating to Google Cloud correctly.
- Creating the Google Cloud project resources.
- Delegating the domain to Google Cloud DNS.
- Building and pushing Docker images from the local machine.
- Running Apache/Laravel, PostgreSQL, workers, scheduler, and Shlink on one
  Compute Engine VM.
- Routing separate application, public short-link, and Shlink administration
  hostnames to the same environment VM.
- Enabling HTTPS, verifying the deployment, and configuring backups.

The target architecture favors simple, familiar components over serverless or
high-throughput infrastructure. Normal browser development occurs against this
public instance rather than a local `/etc/hosts` or local-TLS setup.

## 2. Resulting Architecture

The initial shared development environment consists of:

- One Google Cloud project.
- One regional reserved static IPv4 address.
- One small Compute Engine VM with a persistent SSD.
- Docker Compose on the VM.
- Apache/PHP serving Laravel.
- PostgreSQL.
- Laravel queue workers and scheduler.
- An unmodified Shlink server and self-hosted official Shlink web client.
- One Artifact Registry Docker repository.
- One public Cloud DNS managed zone.
- Three A records pointing to the same environment VM address:
  - `APP_HOSTNAME` for Laravel.
  - `PUBLIC_SHORT_DOMAIN` for redirects and permanent QR URLs.
  - `SHLINK_WEB_HOSTNAME` for the Shlink web client and REST API gateway.
- Apache name-based routing and HTTPS for all three hosts.
- Encrypted off-VM backups.

## 3. What Is Automated and What Is Manual

### 3.1 One unavoidable registrar action

The codebase can create a Cloud DNS public zone and print its assigned Google
nameservers. The domain owner must then log in to the domain registrar and
replace the domain's authoritative nameservers with the complete Google-assigned
set.

The Cloud DNS API cannot perform this registrar-side delegation. Google must
first become authoritative for the domain.

### 3.2 Operations intended for codebase automation

After delegation, the planned deployment command will be able to:

- Enable or verify required APIs.
- Create or verify Artifact Registry.
- Reserve or reuse the static IP.
- Create or update firewall rules.
- Create or update the VM and runtime service account.
- Push immutable Docker images.
- Deploy Docker Compose.
- Manage the three environment-specific A records.
- Configure Apache virtual hosts.
- Request and renew TLS certificates.
- Run deployment health checks.

The planned command name is `bin/deploy-gcp`. It is specified in the
development plan but does not exist yet. Until Phase 0 implements it, use the
equivalent `gcloud`, Docker, and SSH commands described below.

## 4. Prerequisites

Before starting, obtain:

- Control of the domain registrar account.
- A Google Cloud account with permission to create a project or administer an
  existing project.
- A billing account attached to the project.
- A development Macintosh with:
  - Git.
  - Docker Desktop or another working Docker Engine.
  - Google Cloud CLI (`gcloud`).
  - `dig` or another DNS query tool.
  - SSH.
- An email address for ACME certificate notices.

Protect the registrar and Google accounts with multi-factor authentication.
Enable domain auto-renewal and registry lock where available.

## 5. Prepare `.env`

The tracked `example.env` contains safe examples. The real `.env` is ignored by
Git and must never be committed.

From the repository root:

```bash
cp example.env .env
chmod 600 .env
```

Open `.env` locally and replace every `replace_with_...` value. Do not put a
service-account JSON private key into `.env`.

### 5.1 Application and host variables

Configure:

```dotenv
APP_ENV=development
APP_DEBUG=false
APP_KEY=
APP_URL=https://${APP_HOSTNAME}
APP_HOSTNAME=your-application-hostname
PUBLIC_SHORT_DOMAIN=your-public-short-link-hostname
SHLINK_DEFAULT_DOMAIN=your-public-short-link-hostname
SHLINK_WEB_HOSTNAME=your-shlink-administration-hostname
SHLINK_SERVER_INTERNAL_URL=http://shlink-server:8080
SHLINK_REST_API_BASE_URL=https://${SHLINK_WEB_HOSTNAME}/rest/v3
```

Generate `APP_KEY` with Laravel after the Laravel container exists:

```bash
php artisan key:generate --show
```

Copy the generated value into `.env`; do not commit it.

All three public hostnames must be distinct. They initially resolve to the same
environment static IP. Permanent QR codes contain `PUBLIC_SHORT_DOMAIN`, not
the Laravel or Shlink-administration hostname.

### 5.2 Database and Shlink secrets

Replace at minimum:

```dotenv
DB_PASSWORD=use-a-long-random-password
SHLINK_API_KEY=use-a-generated-shlink-api-key
SHLINK_DB_PASSWORD=use-a-second-long-random-password
```

Use different credentials for Laravel and Shlink databases if Shlink uses a
separate database or database user.

### 5.3 Google Cloud values

Set:

```dotenv
GCP_PROJECT_ID=your-project-id
GCP_REGION=your-region
GCP_ZONE=your-zone-in-that-region
GCP_ARTIFACT_REPOSITORY=your-docker-repository
GCP_ARTIFACT_IMAGE=your-image-name
GCP_VM_NAME=your-vm-name
GCP_MACHINE_TYPE=e2-small
GCP_STATIC_IP_NAME=your-static-ip-name
GCP_NETWORK_TAG=your-web-network-tag
GCP_RUNTIME_SERVICE_ACCOUNT=runtime-service-account-email
GCP_DNS_ZONE=your-cloud-dns-zone-name
GCP_DNS_DOMAIN=your-domain-with-trailing-dot.
GCP_MANAGE_DNS=true
ACME_EMAIL=your-certificate-contact-email
```

`GCP_REGION` and `GCP_ZONE` must agree. A zone such as `REGION-a` must belong to
the selected region.

### 5.4 Google API key versus deployment identity

`GOOGLE_API_KEY` is optional and only applies to application APIs that
explicitly support API keys. It cannot authenticate Docker pushes, VM creation,
IAM changes, static IP operations, or Cloud DNS changes.

Infrastructure deployment requires an authenticated Google principal through
`gcloud`, service-account impersonation, or—only when unavoidable—a protected
service-account credential file.

## 6. Authenticate the Local Machine

The shell does not automatically read `.env`. Load the simple assignment file
after reviewing it:

```bash
set -a
. ./.env
set +a
```

For an initial interactive setup:

```bash
gcloud auth login
gcloud config set project "$GCP_PROJECT_ID"
gcloud config set compute/region "$GCP_REGION"
gcloud config set compute/zone "$GCP_ZONE"
```

Application Default Credentials are separate from `gcloud` CLI authentication.
Configure them only when local Google client libraries need them:

```bash
gcloud auth application-default login
```

For repeatable deployment under a dedicated identity, prefer service-account
impersonation:

```bash
gcloud config set auth/impersonate_service_account \
  "$GCP_IMPERSONATE_SERVICE_ACCOUNT"
```

To stop impersonating:

```bash
gcloud config unset auth/impersonate_service_account
```

If a key file is unavoidable, keep it under the ignored `.credentials/`
directory and set only its path:

```dotenv
GOOGLE_APPLICATION_CREDENTIALS=.credentials/deployer.json
```

Never commit the file or paste its contents into `.env`.

## 7. Environment Variable Reference

| Variable | Purpose |
| --- | --- |
| `APP_NAME` | Human-readable Laravel application name. |
| `APP_ENV` | Laravel environment; initially `development` on the VM. |
| `APP_KEY` | Laravel encryption key generated locally and kept secret. |
| `APP_DEBUG` | Must be `false` in production. |
| `APP_URL` | Complete HTTPS URL for the Laravel application. |
| `APP_HOSTNAME` | DNS hostname and Apache `ServerName` for Laravel. |
| `PUBLIC_SHORT_DOMAIN` | Public redirects and permanent QR URLs. |
| `SHLINK_DEFAULT_DOMAIN` | Value passed to Shlink as `DEFAULT_DOMAIN`. |
| `SHLINK_WEB_HOSTNAME` | Official web client and REST API gateway host. |
| `SHLINK_SERVER_INTERNAL_URL` | Docker-internal Shlink server URL. |
| `SHLINK_REST_API_BASE_URL` | External upstream API base ending `/rest/v3`. |
| `DB_CONNECTION` | Laravel database driver; expected to be `pgsql`. |
| `DB_HOST` | PostgreSQL Compose service hostname. |
| `DB_PORT` | PostgreSQL port inside the Compose network. |
| `DB_DATABASE` | Laravel database name. |
| `DB_USERNAME` | Laravel database user. |
| `DB_PASSWORD` | Long random Laravel database password. |
| `SHLINK_API_KEY` | Secret Shlink API key retained only by the backend. |
| `SHLINK_WEB_CLIENT_IMAGE` | Pinned official web-client image. |
| `SHLINK_SERVER_IMAGE` | Pinned official Shlink server image. |
| `SHLINK_DB_DATABASE` | Environment-specific Shlink database. |
| `SHLINK_DB_USERNAME` | Environment-specific Shlink database user. |
| `SHLINK_DB_PASSWORD` | Environment-specific Shlink DB password. |
| `GOOGLE_API_KEY` | Optional application API key; never deployment identity. |
| `GOOGLE_APPLICATION_CREDENTIALS` | Ignored credential-file path. |
| `GCP_IMPERSONATE_SERVICE_ACCOUNT` | Deployment identity to impersonate. |
| `GCP_PROJECT_ID` | Google Cloud project ID. |
| `GCP_REGION` | Region for Artifact Registry and the static address. |
| `GCP_ZONE` | Compute Engine zone within `GCP_REGION`. |
| `GCP_ARTIFACT_REPOSITORY` | Docker-format Artifact Registry repository. |
| `GCP_ARTIFACT_IMAGE` | Application image name inside the repository. |
| `DEPLOY_IMAGE_TAG` | Immutable Git commit or release identifier. |
| `DEPLOY_ENVIRONMENT` | Explicit `development` or `production` selector. |
| `GCP_VM_NAME` | Compute Engine instance name. |
| `GCP_MACHINE_TYPE` | Initial VM machine type. |
| `GCP_BOOT_DISK_SIZE` | Boot disk size accepted by `gcloud`. |
| `GCP_IMAGE_FAMILY` | Supported Debian image family. |
| `GCP_IMAGE_PROJECT` | Google project that publishes the image family. |
| `GCP_STATIC_IP_NAME` | Regional reserved-address resource name. |
| `GCP_NETWORK_TAG` | VM tag targeted by the HTTP/HTTPS firewall rule. |
| `GCP_RUNTIME_SERVICE_ACCOUNT` | Low-privilege identity attached to the VM. |
| `GCP_MANAGE_DNS` | Whether deployment may update Cloud DNS records. |
| `GCP_DNS_ZONE` | Cloud DNS managed-zone resource name. |
| `GCP_DNS_DOMAIN` | Fully qualified zone name, including trailing period. |
| `GCP_DNS_TTL` | TTL in seconds for project-managed A records. |
| `COMPOSE_PROJECT_NAME` | Isolated environment Compose project name. |
| `DEPLOY_REMOTE_DIRECTORY` | VM deployment directory. |
| `DEPLOY_BACKUP_BUCKET` | Off-VM backup destination identifier. |
| `DEPLOY_SSH_USER` | Optional explicit VM SSH user. |
| `ACME_EMAIL` | Certificate-expiration and ACME contact address. |
| `QUEUE_CONNECTION` | Laravel queue backend; initially `database`. |

After editing `.env`, verify that Git ignores it:

```bash
git check-ignore -v .env
```

## 8. Create or Select the Google Cloud Project

If the project already exists, skip project creation and verify billing.

```bash
gcloud projects describe "$GCP_PROJECT_ID"
```

To create a new project when your account has permission:

```bash
gcloud projects create "$GCP_PROJECT_ID" \
  --name="QR Meta Bucket"
```

Attach billing through the Google Cloud console or the Billing API. Deployment
cannot create billable resources until billing is enabled.

Verify the selected account and project before every infrastructure operation:

```bash
gcloud auth list
gcloud config get-value project
```

## 9. Enable Required APIs

Enable the initial APIs:

```bash
gcloud services enable \
  compute.googleapis.com \
  artifactregistry.googleapis.com \
  dns.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  serviceusage.googleapis.com
```

If backups use Google Cloud Storage, also enable:

```bash
gcloud services enable storage.googleapis.com
```

## 10. Create Deployment and Runtime Identities

Use separate identities:

- **Deployment identity:** creates or updates infrastructure and pushes images.
- **VM runtime identity:** pulls images and accesses only required runtime APIs.

The exact roles should be narrowed as implementation stabilizes. Initial setup
typically needs permissions equivalent to:

- Artifact Registry writer for the deployment identity.
- Artifact Registry reader for the runtime identity.
- Compute instance, disk, address, and firewall administration.
- Cloud DNS administration when this code manages DNS.
- Service-account use (`iam.serviceAccountUser`) on the runtime identity.

Avoid broad project Owner credentials for routine deployment. Prefer a custom
role after the exact command set is known.

Create the runtime account if it does not exist:

```bash
RUNTIME_ACCOUNT_NAME="${GCP_RUNTIME_SERVICE_ACCOUNT%@*}"

gcloud iam service-accounts describe \
  "$GCP_RUNTIME_SERVICE_ACCOUNT" >/dev/null 2>&1 || \
gcloud iam service-accounts create "$RUNTIME_ACCOUNT_NAME" \
  --display-name="QR Meta Bucket VM runtime"
```

Grant Artifact Registry read access:

```bash
gcloud projects add-iam-policy-binding "$GCP_PROJECT_ID" \
  --member="serviceAccount:${GCP_RUNTIME_SERVICE_ACCOUNT}" \
  --role="roles/artifactregistry.reader"
```

Do not grant the VM runtime account permission to create VMs, modify DNS, or
push images.

## 11. Create Artifact Registry

Create the Docker repository once:

```bash
gcloud artifacts repositories describe "$GCP_ARTIFACT_REPOSITORY" \
  --location="$GCP_REGION" >/dev/null 2>&1 || \
gcloud artifacts repositories create "$GCP_ARTIFACT_REPOSITORY" \
  --repository-format=docker \
  --location="$GCP_REGION" \
  --description="QR Meta Bucket production images"
```

Configure local Docker authentication:

```bash
gcloud auth configure-docker \
  "${GCP_REGION}-docker.pkg.dev"
```

The image name is:

```text
REGION-docker.pkg.dev/PROJECT/REPOSITORY/IMAGE:TAG
```

Use a Git commit or release tag. Do not rely on `latest` as the only deployed
reference.

## 12. Reserve the Static IP

Reserve one regional IPv4 address:

```bash
gcloud compute addresses describe "$GCP_STATIC_IP_NAME" \
  --region="$GCP_REGION" >/dev/null 2>&1 || \
gcloud compute addresses create "$GCP_STATIC_IP_NAME" \
  --region="$GCP_REGION"
```

Read it into the shell:

```bash
GCP_STATIC_IP="$(
  gcloud compute addresses describe "$GCP_STATIC_IP_NAME" \
    --region="$GCP_REGION" \
    --format='value(address)'
)"

printf 'Reserved IP: %s\n' "$GCP_STATIC_IP"
```

The reserved address can incur charges when it is not attached to a running
resource. Do not reserve it long before the VM is ready.

## 13. Configure Firewall Access

Create one web firewall rule using the configured network tag:

```bash
FIREWALL_RULE="${GCP_NETWORK_TAG}-allow-web"

gcloud compute firewall-rules describe "$FIREWALL_RULE" \
  >/dev/null 2>&1 || \
gcloud compute firewall-rules create "$FIREWALL_RULE" \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags="$GCP_NETWORK_TAG"
```

Do not enable password-based SSH from the entire internet. Prefer OS Login and
IAP, or restrict SSH to a known source address.

## 14. Create the Cloud DNS Zone

Create a public managed zone if it does not exist:

```bash
gcloud dns managed-zones describe "$GCP_DNS_ZONE" \
  >/dev/null 2>&1 || \
gcloud dns managed-zones create "$GCP_DNS_ZONE" \
  --description="QR Meta Bucket authoritative zone" \
  --dns-name="$GCP_DNS_DOMAIN" \
  --visibility=public
```

Display the Google-assigned authoritative nameservers:

```bash
gcloud dns managed-zones describe "$GCP_DNS_ZONE" \
  --format='yaml(nameServers)'
```

Save this output. These are the nameservers that must be entered at the domain
registrar.

## 15. Preserve Existing DNS Before Delegation

Delegating the entire zone to Google transfers authority for every record, not
only the two application hosts.

Before changing registrar nameservers:

1. Export or record the current DNS zone.
2. Recreate all required records in Cloud DNS.
3. Pay special attention to:
   - MX records.
   - SPF, DKIM, and DMARC TXT records.
   - CAA records.
   - Domain-verification TXT records.
   - Existing A, AAAA, CNAME, and SRV records.
4. Verify the complete proposed Google zone.

Do not proceed if required email or service records are missing.

## 16. Delegate the Domain at the Registrar

This is the required manual step:

1. Log in to the registrar account.
2. Open the domain's nameserver settings.
3. Replace the current authoritative nameservers with **all** nameservers
   assigned by the Cloud DNS zone.
4. Remove unrelated old authoritative nameservers; do not mix providers.
5. Save the change.

Nameserver changes can take time to propagate. Verify public delegation:

```bash
dig NS "$GCP_DNS_DOMAIN"
```

Compare the answer with:

```bash
gcloud dns managed-zones describe "$GCP_DNS_ZONE" \
  --format='value(nameServers)'
```

Continue only after public NS responses identify the Google nameservers.

## 17. Create the Three Environment DNS Records

All three hosts in one environment point to that environment's reserved IP. Use
a Cloud DNS transaction so the update is explicit and atomic.

Inspect existing records first:

```bash
gcloud dns record-sets list \
  --zone="$GCP_DNS_ZONE" \
  --filter="type=A"
```

The future deployment command will remove an old value when necessary. For an
initial setup where neither record exists:

```bash
gcloud dns record-sets transaction start \
  --zone="$GCP_DNS_ZONE"

gcloud dns record-sets transaction add "$GCP_STATIC_IP" \
  --name="${APP_HOSTNAME}." \
  --ttl="$GCP_DNS_TTL" \
  --type=A \
  --zone="$GCP_DNS_ZONE"

gcloud dns record-sets transaction add "$GCP_STATIC_IP" \
  --name="${PUBLIC_SHORT_DOMAIN}." \
  --ttl="$GCP_DNS_TTL" \
  --type=A \
  --zone="$GCP_DNS_ZONE"

gcloud dns record-sets transaction add "$GCP_STATIC_IP" \
  --name="${SHLINK_WEB_HOSTNAME}." \
  --ttl="$GCP_DNS_TTL" \
  --type=A \
  --zone="$GCP_DNS_ZONE"

gcloud dns record-sets transaction describe \
  --zone="$GCP_DNS_ZONE"

gcloud dns record-sets transaction execute \
  --zone="$GCP_DNS_ZONE"
```

If any record already exists, do not run the initial transaction unchanged.
Start a new transaction, remove the exact old A record using its current TTL and
address, add the desired value, inspect the transaction, and execute it.

Verify authoritative and public answers:

```bash
dig +short A "$APP_HOSTNAME"
dig +short A "$PUBLIC_SHORT_DOMAIN"
dig +short A "$SHLINK_WEB_HOSTNAME"
```

All three must return the environment's reserved static IP.

## 18. Create the Compute Engine VM

Create the VM after reserving the address. The `--address` argument requires the
address value rather than its resource name.

```bash
gcloud compute instances describe "$GCP_VM_NAME" \
  --zone="$GCP_ZONE" >/dev/null 2>&1 || \
gcloud compute instances create "$GCP_VM_NAME" \
  --zone="$GCP_ZONE" \
  --machine-type="$GCP_MACHINE_TYPE" \
  --image-family="$GCP_IMAGE_FAMILY" \
  --image-project="$GCP_IMAGE_PROJECT" \
  --boot-disk-size="$GCP_BOOT_DISK_SIZE" \
  --address="$GCP_STATIC_IP" \
  --service-account="$GCP_RUNTIME_SERVICE_ACCOUNT" \
  --scopes=cloud-platform \
  --tags="$GCP_NETWORK_TAG" \
  --metadata=enable-oslogin=TRUE
```

The VM's attached service account permissions—not the broad
`cloud-platform` OAuth scope—must enforce least privilege.

Verify:

```bash
gcloud compute instances describe "$GCP_VM_NAME" \
  --zone="$GCP_ZONE" \
  --format='yaml(status,machineType,networkInterfaces,serviceAccounts)'
```

## 19. Bootstrap Docker on the VM

Use SSH through the selected secure access method:

```bash
gcloud compute ssh "$GCP_VM_NAME" --zone="$GCP_ZONE"
```

Install Docker Engine and the Docker Compose plugin using Docker's supported
Debian instructions. Do not use an unreviewed `curl | sh` command in production.

Create the deployment directory and persistent-data directories:

```bash
sudo mkdir -p "$DEPLOY_REMOTE_DIRECTORY"
sudo mkdir -p /srv/qr-meta-bucket/{postgres,shlink,certbot,backups}
sudo chown -R "$USER":"$USER" "$DEPLOY_REMOTE_DIRECTORY"
sudo chown -R "$USER":"$USER" /srv/qr-meta-bucket
```

The final Compose configuration should mount persistent state from `/srv` or a
separately attached persistent disk, not from an ephemeral container layer.

The Phase 0 deployment script should automate this bootstrap idempotently.

## 20. Build and Push the Docker Image

On the local development machine, run the test suite before building. Local
Docker builds and pushes images; the normal web runtime remains on Google.

Construct the immutable image reference:

```bash
IMAGE_REFERENCE="${GCP_REGION}-docker.pkg.dev/\
${GCP_PROJECT_ID}/${GCP_ARTIFACT_REPOSITORY}/\
${GCP_ARTIFACT_IMAGE}:${DEPLOY_IMAGE_TAG}"
```

The line breaks above are shell continuations; the resulting reference contains
no whitespace or newlines.

Build, tag, and push after the project Dockerfile exists:

```bash
docker build --pull --tag "$IMAGE_REFERENCE" .
docker push "$IMAGE_REFERENCE"
```

Record the pushed digest:

```bash
gcloud artifacts docker images describe "$IMAGE_REFERENCE" \
  --format='value(image_summary.digest)'
```

The shared development deployment should use the immutable tag and record
the digest. Do not deploy only `latest`.

## 21. Configure the VM Deployment

The codebase must eventually provide:

- A server Compose file or override.
- Apache virtual-host templates.
- A server environment template.
- A bootstrap/deploy script.
- Health and readiness endpoints.

Do **not** copy the local `.env` wholesale to the VM. It contains deployment
credentials and workstation-only settings. Generate a server-only environment
file containing the values required by Laravel, PostgreSQL, Shlink, workers,
and the scheduler.

The Apache configuration must define three name-based virtual hosts:

- `APP_HOSTNAME` serves Laravel's `public/` directory.
- `PUBLIC_SHORT_DOMAIN` proxies public redirect requests to Shlink server.
- `SHLINK_WEB_HOSTNAME` serves the official Shlink web client. Its `/rest/...`
  requests proxy to Shlink server; other routes serve the web-client PWA.

Shlink server remains an unmodified upstream container and receives
`SHLINK_DEFAULT_DOMAIN` as its `DEFAULT_DOMAIN`. The official web client is a
separate pinned container. Laravel uses `SHLINK_SERVER_INTERNAL_URL` inside the
Compose network and the Shlink API key. Do not add an `/api/v1` alias: upstream
Shlink REST routes use `/rest/v3` for the selected pinned release.
Requests with unknown Host headers must not route silently to any deployed
service.

## 22. Deploy the Compose Stack

After Compose files and deployment scripts exist, the repeatable deployment
will:

1. Authenticate the VM to Artifact Registry through its attached runtime
   service account.
2. Pull the immutable image reference.
3. Back up PostgreSQL and Shlink state before migrations.
4. Run Laravel migrations.
5. Start or update Apache/Laravel, PostgreSQL, workers, scheduler, and Shlink.
6. Wait for service health checks.
7. Retain the previous image reference for rollback.

A deployment must not delete persistent volumes or replace the VM unless an
explicit destructive flag and a fresh verified backup are present.

## 23. Configure HTTPS

Wait until all three hostnames resolve publicly to the VM and HTTP port 80
responds.

Install Certbot for Apache using the supported Debian/snap instructions, then
request certificates covering all three names. A typical command is:

```bash
sudo certbot --apache \
  --non-interactive \
  --agree-tos \
  --email "$ACME_EMAIL" \
  --redirect \
  -d "$APP_HOSTNAME" \
  -d "$PUBLIC_SHORT_DOMAIN" \
  -d "$SHLINK_WEB_HOSTNAME"
```

Test renewal:

```bash
sudo certbot renew --dry-run
```

Persist `/etc/letsencrypt` or the selected certificate storage and include it
in restoration procedures. Certificate renewal should run through Certbot's
installed systemd timer or cron mechanism.

## 24. Verify the Deployment

### 24.1 DNS and TLS

```bash
dig +short NS "$GCP_DNS_DOMAIN"
dig +short A "$APP_HOSTNAME"
dig +short A "$PUBLIC_SHORT_DOMAIN"
dig +short A "$SHLINK_WEB_HOSTNAME"
curl --fail --show-error --silent "https://${APP_HOSTNAME}/health"
curl --fail --show-error --silent "https://${PUBLIC_SHORT_DOMAIN}/"
curl --fail --show-error --silent "https://${SHLINK_WEB_HOSTNAME}/"
```

### 24.2 Application behavior

Verify:

- Laravel health and readiness endpoints.
- PostgreSQL connectivity.
- Queue dispatch and completion.
- Scheduler heartbeat.
- All project-owned JavaScript loads from an owner-controlled origin.
- No public CDN JavaScript is requested.
- Shlink creates and resolves a temporary redirect.
- The official Shlink web client loads from `SHLINK_WEB_HOSTNAME` using locally
  hosted assets.
- An authenticated request to `${SHLINK_REST_API_BASE_URL}` reaches the same
  environment's Shlink server and not the other environment.
- The short-link host redirects to a bucket page on the application host.
- Unknown Host headers are rejected or routed to a harmless default response.

### 24.3 Infrastructure state

```bash
gcloud compute instances describe "$GCP_VM_NAME" --zone="$GCP_ZONE"
gcloud compute addresses describe "$GCP_STATIC_IP_NAME" \
  --region="$GCP_REGION"
gcloud artifacts repositories describe "$GCP_ARTIFACT_REPOSITORY" \
  --location="$GCP_REGION"
gcloud dns record-sets list --zone="$GCP_DNS_ZONE"
```

## 25. Configure Backups

At minimum, back up:

- PostgreSQL logical dumps.
- Shlink database and configuration.
- Laravel server configuration, excluding or separately encrypting secrets.
- Bucket manifests.
- Compose files and image digests.
- Locally hosted compiled assets and asset manifest.
- Apache virtual-host configuration.
- Certificate state.

Store encrypted backups off the VM. Prefer a second copy outside the Google
account or provider. Schedule and document restoration tests; a successful
upload is not proof that a backup can be restored.

## 26. Repeatable Future Deployments

After initial setup, a normal deployment should require no registrar changes:

1. Load `.env` locally.
2. Authenticate or impersonate the deployment identity.
3. Run tests and static checks.
4. Build locally bundled frontend assets.
5. Build and push an immutable Docker image.
6. Back up shared development state.
7. Pull and deploy the image on the VM.
8. Run migrations.
9. Verify all three hosts, queues, scheduler, both databases, and Shlink.
10. Record the commit, image digest, VM, IP, and deployment time under the
    ignored `.deployment/` directory and in the server deployment log.

The future convenience command is expected to be:

```bash
bin/deploy-gcp preflight
bin/deploy-gcp deploy
bin/deploy-gcp verify
```

The command must be idempotent. Repeated execution with unchanged configuration
must not create duplicate VMs, addresses, disks, firewall rules, repositories,
or DNS records.

This is the normal web-development feedback loop:

1. Edit source locally.
2. Run focused and full automated checks locally.
3. Build and push a new immutable image.
4. Deploy it to the Google development VM.
5. Run public integration and browser tests.
6. Roll back to the prior image if verification fails.

Do not develop by editing files directly on the VM. Source control and immutable
images remain authoritative.

### 26.1 Create the production environment

Production repeats the same topology with a separate environment file and
separate infrastructure. It may share the Google Cloud project, Cloud DNS zone,
and Artifact Registry, but it must not share mutable runtime state.

Create a production `.env` outside Git by copying `example.env`, then change:

- `DEPLOY_ENVIRONMENT` to `production`.
- `APP_ENV` to `production`.
- `APP_URL`, `APP_HOSTNAME`, `PUBLIC_SHORT_DOMAIN`, and
  `SHLINK_WEB_HOSTNAME` to the production names.
- `SHLINK_DEFAULT_DOMAIN` to the production public short-link domain.
- `SHLINK_REST_API_BASE_URL` to the production Shlink web hostname plus
  `/rest/v3`.
- `GCP_VM_NAME`, `GCP_STATIC_IP_NAME`, `GCP_NETWORK_TAG`,
  `COMPOSE_PROJECT_NAME`, and `DEPLOY_REMOTE_DIRECTORY` to unique production
  values.
- Laravel and Shlink database names, users, passwords, API keys, application
  keys, backup destinations, and provider credentials to distinct production
  values.

Production must have its own:

- Compute Engine VM and reserved static IP.
- Docker Compose project and persistent directories or disk.
- Laravel database.
- Shlink database and API keys.
- Shlink links and visit history.
- TLS state.
- Backup destination and restoration test.
- Deployment-state directory or environment-specific state file.

Deploy the exact image digest already verified in development. Do not rebuild a
different image for production. The deployment command must require an explicit
environment selector and confirmation before production changes. It must print
the selected project, VM, static IP, three hostnames, image digest, database
names, and backup destination before proceeding.

Never clone the development Shlink database into production as a routine
promotion mechanism. Development and production links are intentionally
different namespaces. If selected links or archive metadata must move between
environments, use an explicit, reviewed, audited export/import operation.

## 27. Troubleshooting

### 27.1 API key is present but `gcloud` fails

An API key is not deployment authentication. Run:

```bash
gcloud auth list
gcloud config get-value project
```

Authenticate or configure impersonation, then retry.

### 27.2 DNS records exist in Google but do not resolve

Check delegation:

```bash
dig NS "$GCP_DNS_DOMAIN"
```

If the response does not match the Cloud DNS zone's assigned nameservers, fix
the registrar nameserver configuration and wait for propagation.

### 27.3 Existing email stops working after delegation

The old zone's MX or TXT records were probably not copied. Restore the required
records in Cloud DNS from the pre-delegation export.

### 27.4 Certbot validation fails

Confirm all three A records return the environment VM's reserved IP and that TCP
80 is publicly reachable. Certbot's normal Apache HTTP validation expects an
online HTTP site.

### 27.5 VM cannot pull from Artifact Registry

Confirm the attached runtime service account has Artifact Registry reader
permission and the VM is using that account. Do not place deployment writer
credentials on the VM.

### 27.6 Apache serves the wrong application

Inspect enabled virtual hosts and verify `ServerName` values match
`APP_HOSTNAME`, `PUBLIC_SHORT_DOMAIN`, and `SHLINK_WEB_HOSTNAME`. Ensure the
default virtual host does not expose any environment service for unknown hosts.

## 28. Initial Setup Completion Checklist

- [ ] Google Cloud project exists and billing is enabled.
- [ ] Local `gcloud` authentication works.
- [ ] `.env` exists locally, is mode `600`, and is ignored by Git.
- [ ] No private-key contents are stored in `.env`.
- [ ] Required APIs are enabled.
- [ ] Deployment and runtime identities are separate.
- [ ] Artifact Registry exists and local Docker authentication works.
- [ ] Static IPv4 address is reserved.
- [ ] Firewall permits HTTP and HTTPS.
- [ ] Cloud DNS public zone exists.
- [ ] Existing DNS records were copied before delegation.
- [ ] Registrar nameservers match the Google-assigned set.
- [ ] All three configured A records return the environment's reserved IP.
- [ ] Compute Engine VM uses the runtime service account.
- [ ] Docker and Compose are installed on the VM.
- [ ] Persistent directories or disk mounts exist.
- [ ] Immutable application image is pushed.
- [ ] Apache routes all three configured hosts correctly.
- [ ] HTTPS works for all three hosts and renewal dry-run passes.
- [ ] Laravel, PostgreSQL, queue, scheduler, and Shlink health checks pass.
- [ ] Off-VM backups run successfully.
- [ ] A clean restoration test is scheduled or completed.

## 29. References

- [Google Cloud authentication use cases](https://docs.cloud.google.com/docs/authentication/use-cases)
- [Push and pull Artifact Registry images](https://docs.cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling)
- [Create a Compute Engine instance](https://docs.cloud.google.com/compute/docs/instances/create-start-instance)
- [Configure a static external IP](https://docs.cloud.google.com/compute/docs/ip-addresses/configure-static-external-ip-address)
- [Set up Cloud DNS for a domain](https://docs.cloud.google.com/dns/docs/set-up-dns-records-domain-name)
- [Manage Cloud DNS records](https://docs.cloud.google.com/dns/docs/records)
- [Certbot with Apache](https://certbot.eff.org/instructions?ws=apache&os=snap)
