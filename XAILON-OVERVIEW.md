# Xailon TokenFactory — DevOps Overview

| | |
|---|---|
| **Document owner** | DevOps Team |
| **Prepared by** | Sidhant Khanna |
| **Platform** | Xailon TokenFactory — AI Gateway Platform |
| **Application repository** | `Infinia-Technology/ai-gateway-platform` |
| **Infrastructure repository** | `Infinia-Technology/azure-infra-terraform` |
| **Cluster / namespace** | `aks-tokenfactory` / `xailon-tf-prod` |
| **Version** | 2.0 |
| **Last updated** | 15 October 2026 |

> This document describes the end-to-end DevOps lifecycle of the Xailon TokenFactory deployment:
> how it is architected, developed, built, deployed, run, secured and maintained. It is written for
> management, security and engineering audiences. It mirrors the repositories' actual state and
> does not describe unimplemented tooling as if it existed.

---

## 1. Introduction

### 1.1 Purpose & Scope

**Xailon and TokenFactory are the same application.** The codebase, the Helm chart, the services
and the APIs are identical; only the environment differs. TokenFactory runs on bare metal
(`prod-k8s`); Xailon runs on Azure (AKS, UAE North). Xailon is operated the same way TokenFactory
is — same branches, same CI, same chart, same release discipline — with the deployment target
changed and the platform dependencies swapped for Azure managed services.

In scope: the application lifecycle from branch to running workload, the CI and CD paths, the
Kubernetes runtime, and how the environment is secured, monitored and maintained.

Out of scope: the detail of the Azure infrastructure itself. That is a separate document
(`azure-infra-terraform`); this one states what the application depends on, not how the VNet is
laid out.

### 1.2 Product Overview

TokenFactory is a sovereign, multi-tenant AI gateway exposing **one OpenAI-compatible endpoint**
over many LLM providers. It provides unified access across modalities, per-request metering in
nano-USD, prepaid credit enforcement, and per-organization identity.

| Plane | Components |
|---|---|
| Data plane | `envoy` Gateway (Envoy AI Gateway) with in-line filters `tf-warden` (authz), `tf-relay` (metering), `tf-shim` (transform), `tf-babel` (Anthropic dialect) |
| Control plane | `tf-auth`, `tf-accounts`, `tf-control`, `tf-ledger`, `tf-ops` |
| Web | `tf-portal-web`, `tf-admin-web`, `tf-site-web`, `tf-docs-web` |
| State | PostgreSQL, Redis, Keycloak, OpenFGA, OpenMeter |

### 1.3 DevOps Lifecycle

```
  branch off develop
        ↓
  PR → automated gate (tests, lint, secret scan, smoke)
        ↓
  owner-approved merge to develop
        ↓
  owner-gated release PR develop → main
        ↓
  build.yml — images to GHCR, Trivy scanned
        ↓
  az acr import — GHCR to ACR, server-side
        ↓
  Argo CD Application — manual sync from main
        ↓
  running in xailon-tf-prod
        ↓
  operate, monitor, maintain
```

Two properties define this lifecycle and are deliberate:

- **Nothing builds on merge.** Images are built by an explicit action, never automatically on a
  push to `develop`.
- **Nothing deploys on its own.** Argo CD is installed and owns the Application, but sync is
  manual. A change reaching `main` does not reach the cluster until someone presses Sync.

---

## 2. Architecture

### 2.1 Product Architecture

Every client request to `api.xailon.ai` passes through the filter chain — authorization, then
metering, then optional transform — before being routed to a provider backend. Usage is extracted
on the request and response path and persisted through the money pipeline to PostgreSQL and
OpenMeter.

```
 client → Cloudflare edge → cloudflared → envoy Gateway
                                            ├─ tf-warden   (ext_authz, fail-closed)
                                            ├─ tf-relay    (ext_proc, metering)
                                            ├─ tf-shim     (special paths)
                                            └─ provider backend
                                                   ↓
                                         tf-ledger → PostgreSQL + OpenMeter
```

### 2.2 Kubernetes Architecture

One namespace holds the application. Supporting concerns live in their own namespaces so they can
be upgraded independently.

| Namespace | Contents |
|---|---|
| `xailon-tf-prod` | the `tokenfactory` Helm release — all `tf-*` services, `envoy`, Keycloak, OpenFGA, MinIO, plus `akv-secret-syncer` |
| `openmeter` | the `openmeter` release — OpenMeter, ClickHouse, Svix |
| `cloudflare` | `cloudflared` tunnel connector and its own secret syncer |
| `nginx-ingress` | F5 NGINX Ingress Controller |
| `argocd` | Argo CD v3.5.4 |

### 2.3 Infrastructure & Dependencies

The application depends on these managed services. All are reached over private endpoints; none is
publicly addressable.

| Dependency | Azure service | Notes |
|---|---|---|
| Database | PostgreSQL Flexible Server | 5 databases, ZoneRedundant HA, 30-day geo-redundant backup |
| Cache & streams | Azure Managed Redis | port **10000**, `NoCluster` policy (see 11.1) |
| Metering transport | Event Hubs (Kafka protocol) | replaces the bundled Kafka |
| Images | Azure Container Registry | images imported from GHCR |
| Secrets | Azure Key Vault | projected into the cluster by the CSI driver |
| Object storage | Blob (×2) | Event Hubs capture, Terraform state |
| Ingress | Cloudflare Tunnel | **no load balancer, no public IP on any Service** |

The equivalents on TokenFactory are bundled or in-cluster (PostgreSQL, Redis, Kafka, HashiCorp
Vault, MetalLB). The application is indifferent: every one of these is selected by values in the
environment overlay, not by a code change.

---

## 3. Development

### 3.1 Development Environment

| Target | Where | Mutations |
|---|---|---|
| local | kind cluster, `values-tf-local.yaml` | free |
| dev-staging | `saif-dev-no-prod-reach`, namespace `tokenfactory` | free — deploy and test |
| QA (`tf-stage`) | `prod-k8s` | never by hand |
| TokenFactory production | `prod-k8s`, `tokenfactory-prod` | read-only inspection |
| **Xailon production** | `aks-tokenfactory`, `xailon-tf-prod` | **through Helm or Argo only** |

Development and testing happen on local or dev-staging. Xailon has **no staging environment of its
own** — it is a single production environment, and changes are validated on dev-staging and
`tf-stage` before they reach it.

### 3.2 Repository Structure

| Path | Contents |
|---|---|
| `tf-*/` | one directory per service — Go services and React SPAs |
| `helm/tokenfactory/` | the umbrella chart, base `values.yaml` and per-environment overlays |
| `helm/infrastructure/` | dependency charts, including OpenMeter |
| `helm/setup/` | install front door |
| `docs/release/vX.Y.md` | release notes, one file per minor line |
| `docs/adr/` | architecture decision records |
| `.github/workflows/` | the four CI workflows |

Xailon's environment overlay is `helm/tokenfactory/values-xailon-prod.yaml`. It is a **thin**
overlay: it carries only what differs from the chart base, so the diff against TokenFactory's
production overlay is the environment, not the application.

### 3.3 Git / Branching Strategy

| Branch | Role |
|---|---|
| `develop` | integration branch — every change lands here first |
| `main` | **release-only**, never a direct target |
| `stage`, `QA` | promotion branches for the bare-metal environments |
| `feat/…`, `fix/…`, `chore/…` | all work, branched off `develop` |
| `release/X.Y.Z` | owner-gated release cut only |

Rules in force:

- Every change lands through a **pull request against `develop`**, with its own release-note entry
  in `docs/release/vX.Y.md`.
- **Merge method is the owner's call.** Squash for feature, fix and refactor PRs, which are
  self-contained. Regular merge for branch-sync and release PRs (`develop → stage`,
  `develop → main`) so the shared branches keep identical history.
- Merge only after the automated gate passes **and the owner approves**. Never self-merge, never
  admin override.
- **Never bump a version** — chart `version`/`appVersion`, image or release tags — outside an
  owner-approved release cycle.
- `main` has branch protection; direct pushes are rejected.

### 3.4 Coding & Configuration Standards

- Go services are formatted with `gofmt` and linted with `golangci-lint`, both enforced in CI.
- OpenAPI specifications in `docs/api/openapi/` are normative; CI fails if the copies served by
  `tf-docs-web` drift from them.
- Configuration is values-driven. Behaviour differences between environments belong in an overlay,
  never in code.
- `requireImmutableTag: true` — the image tag is explicit, and a dropped tag fails the Helm render
  rather than silently falling back to the chart `appVersion`.

---

## 4. Build & CI

### 4.1 Automated Gate on Every Pull Request

Four workflows, all live:

| Workflow | Trigger | What it does |
|---|---|---|
| `test.yml` — Go Test & Lint | PR to `develop`/`main`, push to `develop` | `go vet` and `go test -race` for every Go module (one matrix job each), `gofmt`, `golangci-lint`, and the OpenAPI drift check |
| `ci-secret-scan.yaml` | push and PR to `develop`, `stage`, `QA`, `main` | `gitleaks` with a pinned, checksum-verified binary |
| `pr-build-smoke.yml` | PR to `develop`/`main` | build smoke check on the changed surface |
| `build.yml` — Build & Push Images | **manual dispatch**, or a `v*` tag push | builds and publishes all service images |

One matrix job per module means a failure names the module that broke rather than failing the
whole run opaquely.

### 4.2 Container Image Build

`build.yml` implements ADR-0008 and has one property worth stating to management: **there is no
build on every merge.** Images are produced by an explicit action, in three modes only — manual
dispatch, a `v*` tag push, or a scheduled run that is currently disabled.

| Aspect | Implementation |
|---|---|
| Parallelism | one matrix job per service, discovered from the `tf-*` Dockerfiles; `fail-fast: false`, so one service failing does not cancel the others |
| Build logic | `deploy/scripts/build-all.sh` — the single source of truth for per-service context, `--provenance=false` and digest recording |
| Destination | `ghcr.io/infinia-technology/ai-gateway-platform/<service>` |
| Tag | `<chart version>-<short sha>`, read from `helm/tokenfactory/Chart.yaml` |
| Scanning | Trivy per image, with an explicit retry so a flaky vulnerability-database download does not fail the build |

### 4.3 Getting Images to Xailon

Xailon pulls from **ACR, not GHCR**, so there is one extra step after the build:

```
az acr import → acrtokenfactory.azurecr.io/infinia-technology/ai-gateway-platform/<service>:<tag>
```

`az acr import` runs **server-side**: ACR pulls from the source registry itself. Nothing passes
through an operator's machine, no Docker daemon is required, digests are preserved, and the Azure
Firewall is not in the path — so sources that are blocked from inside the VNet still import. `scripts/acr-import-images.sh` in the infrastructure repository performs the import for all 13
service images plus the dependency images, skipping anything already present. See Operational
Scripts.

The tag currently deployed is `0.16.2-e1bd218`.

### 4.4 Security & Vulnerability Scanning

| Layer | Tool | Where |
|---|---|---|
| Secrets in source | `gitleaks` | every push and PR |
| Container images | Trivy | in `build.yml`, per image |
| Push protection | GitHub secret scanning | rejects a push containing a recognised credential |

Provider versions and checksums for the infrastructure are pinned in a committed lock file.

---

## 5. Deployment & CD

### 5.1 Deployment Architecture

One Helm release deploys the application:

| Item | Value |
|---|---|
| Chart | `helm/tokenfactory` (umbrella) |
| Overlay | `values-xailon-prod.yaml` |
| Release name | **`tokenfactory`** — mandatory |
| Namespace | `xailon-tf-prod` |

The release name is not a preference. Six subcharts hardcode `<release>-secrets` and ignore
`global.secretsName`, so any other release name cannot find the `tokenfactory-secrets` Secret and
every pod fails to start.

OpenMeter is a **separate release** in its own namespace, upgraded independently. Its release name
must be `openmeter`, or `tf-ledger`'s `OPENMETER_URL` will not match and the provisioning hook
stalls silently.

### 5.2 CD Path — Argo CD

Argo CD v3.5.4 runs **inside** the cluster, reachable at `argo.xailon.ai`. It runs in-cluster
because the AKS API server is private and has no inbound path from outside.

| Setting | Value | Why |
|---|---|---|
| Application | `xailon-tokenfactory` | adopts the existing release |
| Source | `ai-gateway-platform`, `targetRevision: main`, path `helm/tokenfactory` | release branch only |
| Values | `values-xailon-prod.yaml` | the chart's own `values.yaml` is applied as the base automatically |
| Sync | **manual** — no `syncPolicy.automated` | a merge to `main` does not deploy itself |
| Prune | manual, `PruneLast=true` | new resources come up before old ones go |
| Finalizer | **absent** | deleting the Application must not delete the workloads |

Two differences the Application has to carry:

- `releaseName` pinned to `tokenfactory`, for the reason in 5.1.
- `ignoreDifferences` on the `tf-auth-openmeter` Secret token and its checksum annotation. The
  chart generates that token with `lookup` falling back to a random value, and Argo's repo-server
  renders templates without cluster access, so `lookup` is always empty and every refresh produces
  a new token — which would leave the Application permanently OutOfSync and restart `tf-auth` on
  every sync.

**This is a real difference from TokenFactory.** On `prod-k8s`, `tf-stage` and production are
upgraded by an Argo automation that runs `helm upgrade` from **outside** the cluster — there is no
in-cluster Application or tracking label. Xailon has an actual in-cluster GitOps controller owning
its release, which makes it the more GitOps-native of the two.

### 5.3 Deployment Procedure

1. Merge the owner-gated release PR `develop → main`.
2. Run `build.yml` for that ref; confirm the images and their Trivy results.
3. Import the new tag into ACR.
4. Set `global.image.tag` in `values-xailon-prod.yaml` and merge that change.
5. In Argo CD: **Refresh**, review **App Diff**, then **Sync**.
6. Run the post-deployment checks in 5.5.

Helm from the jumphost remains available for the same operation and is what was used before Argo
CD was installed. Either path produces the same release; the Argo path leaves a reviewable diff.

### 5.4 Environment Promotion

```
 feature branch → develop → stage → QA → TokenFactory production
                     └──────────────────→ main → Xailon production
```

Xailon deploys from `main`, the same branch TokenFactory production uses. It has no promotion
chain of its own, so a change is validated on dev-staging and `tf-stage` before the release PR is
cut.

### 5.5 Rollback & Post-Deployment Validation

**Validation after any sync:**

| Check | Expected |
|---|---|
| Workload health | every Deployment at its configured replica count |
| Keycloak cluster | 3 Infinispan members |
| Migrations | `dirty` false in both tracking tables |
| External reachability | each public host returns its expected status |
| Metering | usage rows advancing in PostgreSQL and OpenMeter |

**Rollback:**

- **Application** — `helm rollback`, or revert the tag in the overlay and sync. Argo CD's history
  makes the previous revision explicit.
- **One caveat specific to this environment:** the chart's checksum mechanism, which normally
  rolls pods when a secret value changes, resolves to an empty string whenever
  `global.secrets.existingSecret` is set — as it is here. A change to the external Secret is
  therefore **not** detected, and the consumers need a manual `rollout restart`.
- **Data** — PostgreSQL point-in-time restore within the 30-day window. Not yet rehearsed.

---

## 6. Kubernetes

### 6.1 Workloads

| Workload | Replicas | Role |
|---|---|---|
| `envoy` | 6 | data-plane gateway |
| `tf-warden`, `tf-relay`, `tf-shim`, `tf-babel` | 3 each | data-plane filters |
| `tf-auth`, `tf-accounts`, `tf-control`, `tf-ops` | 3 each | control plane |
| `tf-ledger` | **1 — pinned** | money pipeline, see below |
| `tf-portal-web`, `tf-admin-web` | 3 each | portals |
| `tf-site-web`, `tf-docs-web` | 1 each | public site and docs |
| `keycloak` (StatefulSet) | 3 | identity |
| `openfga` | 1 | authorization, in-memory datastore |
| `minio` | 1 | export archive |
| `akv-secret-syncer` | 1 | holds the Key Vault CSI mount open |

**`tf-ledger` must stay at one replica.** It starts five unguarded ticker loops and has no leader
election — no `client-go` dependency, no Lease, no RBAC for one. The single cluster-wide guard that
exists covers only one admin-triggered path. The bug was already hit at two replicas and fixed by
pinning it back to one. Raising it requires adding leader election first.

### 6.2 Services & Networking

**Every Service is ClusterIP.** On AKS, `type: LoadBalancer` provisions a public IP, and the
cluster's user-defined routing does not prevent that. Inbound traffic instead arrives through a
Cloudflare Tunnel whose connector dials **outbound only**.

| Host | Path into the cluster |
|---|---|
| `api.xailon.ai` | tunnel → `envoy` directly |
| `console`, `admin`, `auth`, `xailon.ai` | tunnel → F5 NGINX Ingress → service |
| `argo.xailon.ai` | tunnel → Argo CD |

The ingress controller **must** be F5's `nginx.org/ingress-controller`. All the Ingress annotations
are `nginx.org/*`, and the community `ingress-nginx` controller ignores them **silently** —
including the proxy-buffer settings that fix the OIDC callback 502.

Hostname-to-service mapping lives in the Cloudflare Zero Trust dashboard, not in a cluster object,
and is therefore not version-controlled.

### 6.3 ConfigMaps & Secrets

Application configuration is rendered by Helm from the overlay into ConfigMaps. Secrets come from
Azure Key Vault:

```
Key Vault → SecretProviderClass tokenfactory-kv → Secret tokenfactory-secrets → secretKeyRef
```

Three operational properties:

1. **The SecretProviderClass is the entire list of what the cluster fetches.** A secret in the
   vault but not named there is never synced and raises no error. It must appear in **two** places
   in the manifest — what to fetch, and what to write.
2. **`akv-secret-syncer` must not be deleted.** It exists only to hold the CSI mount open, and the
   Secret is owner-referenced to its ReplicaSet. Deleting the Deployment garbage-collects the
   Secret and every pod then fails on its next restart. `rollout restart` is safe.
3. **Rotation refreshes the Secret, not the pods.** A 2-minute reconciler updates the Secret on its
   own; environment variables are read once at process start, so consumers still need a restart.

The manifest is being moved into the infrastructure repository's `scripts/` folder. Until that
lands it exists only on the jumphost, which makes it the largest single-copy risk in the
environment.

### 6.4 Storage

| Claim | Use |
|---|---|
| MinIO | export archive |
| Keycloak | per-replica state |

PostgreSQL and Redis are external, so neither holds a PersistentVolume here. Redis has no
persistence and is rebuildable from PostgreSQL plus traffic.

### 6.5 RBAC & Resource Management

- Operators reach the cluster through the VPN or Bastion and then the jumphost; the AKS API server
  has no public endpoint.
- The jumphost's managed identity exists only because the Entra SSH extension requires one and is
  deliberately granted **no role at all**.
- Replica counts and resource requests are set per service in the overlay. There is no
  HorizontalPodAutoscaler; scaling is a values change followed by an upgrade.
- A NetworkPolicy on the OpenMeter API admits only the application namespace. If that value is
  wrong, `tf-ledger` is blocked and it presents as "OpenMeter unreachable".

---

## 7. Infrastructure

### 7.1 Where the Code Lives

| Repository | Contents |
|---|---|
| **https://github.com/Infinia-Technology/azure-infra-terraform** | The Azure infrastructure as Terraform — resource group, VNet and subnets, AKS, Azure Firewall and the egress allow-list, PostgreSQL, Redis, Key Vault, ACR, Event Hubs, storage, private DNS, VPN, Bastion and the jumphost. Also the home of the operational scripts (see below). |
| **https://github.com/Infinia-Technology/ai-gateway-platform** | The application, the Helm umbrella chart and the Xailon overlay. |

Branch protection is on for `main` in both repositories; every change goes through a pull request.

**There is no infrastructure CI.** `terraform plan` requires execution from inside the VNet, so a
hosted runner cannot run it. Plans and applies are performed by hand from the jumphost, and every
apply is followed by a plan that must report no changes.

**Some settings cannot be reverted** by reverting the code — Key Vault purge protection, the
PostgreSQL HA mode and the storage size among them. Any pull request touching one must say so.

### 7.2 Operational Scripts

Roughly eight to ten post-apply steps are not performed by Terraform or Helm: Key Vault seeding,
the SecretProviderClass and its syncer, PostgreSQL roles and `public` schema ownership, the ACR
image import, the ingress controller, the Cloudflare tunnel, OpenMeter, the TLS secret, and the
Cloudflare DNS and tunnel configuration.

Scripts exist for all of these. Their home is:

```
https://github.com/Infinia-Technology/azure-infra-terraform  →  scripts/
```

> **Status:** the scripts currently live at `~/xalion` on the jumphost and are being moved into
> that `scripts/` folder. Until that push lands, the jumphost copy is the working one. Treat the
> repository path as the location to pull from once it is populated.

They run **from the jumphost**, because Key Vault, PostgreSQL, ACR and the AKS API server are all
private-endpoint only and unreachable from anywhere else.

| To do this | Run |
|---|---|
| Import images from GHCR into ACR | `scripts/acr-import-images.sh` — `TAG=<tag> ./acr-import-images.sh --check` first, then without `--check`; `--infra` for the dependency images, `--all` for both |
| Seed Key Vault | `scripts/kv-seed-secrets.sh` |
| Build or repair `tokenfactory-secrets` | `scripts/deploy-secrets.sh` — `--check` reports, `--partial` builds from only the keys that exist |
| Create or re-sync PostgreSQL roles | `scripts/pg-create-roles.sh` — `--check` first |
| Unstick a dirty migration | `scripts/fix-tf-migrations.sh` (tokenfactory) or `scripts/fix-openmeter-migration.sh` |
| See the real migration error | `scripts/catch-migration-error.sh` |
| Deploy or re-deploy OpenMeter | `scripts/deploy-openmeter-azure.sh` — `--check`, or `DRY_RUN=1` |
| Create the Argo CD Application | `scripts/argocd-create-xailon-app.sh` |
| Uninstall both releases in the right order | `scripts/uninstall-all.sh` |

Most accept `--check`, which reports state and changes nothing. Use it first. A one-line reference
for every script is in `scripts/SCRIPTS.txt`.

The Cloudflare side has no script — hostname-to-service routing lives in the Zero Trust dashboard
and is not version-controlled anywhere.

### 7.3 Connecting to the Azure Services

Everything below runs **from the jumphost**. Reach the jumphost over the Point-to-Site VPN, or
through Azure Bastion:

```bash
az network bastion ssh -n bas-tokenfactory -g rg-tokenfactory \
  --target-resource-id $(az vm show -g rg-tokenfactory -n vm-jumphost-tokenfactory --query id -o tsv) \
  --auth-type AAD
```

Then, once on the jumphost:

| Service | Command |
|---|---|
| **Azure CLI** | `az login` then `az account set --subscription <id>` |
| **AKS** | `az aks get-credentials -g rg-tokenfactory -n aks-tokenfactory` then `kubectl -n xailon-tf-prod get pods` |
| **Key Vault** | `az keyvault secret list --vault-name kv-tokenfactory -o table` |
| **Key Vault — one value** | `az keyvault secret show --vault-name kv-tokenfactory -n <name> --query value -o tsv` |
| **PostgreSQL — application role** | `PGPASSWORD=$(az keyvault secret show --vault-name kv-tokenfactory -n db-password --query value -o tsv) psql -h psql-tokenfactory.postgres.database.azure.com -U tokenfactory -d tokenfactory --set=sslmode=require` |
| **PostgreSQL — admin** | same, with `-n pg-admin-password` and `-U pgadmintf -d postgres` |
| **Redis** | `redis-cli -h redis-tokenfactory.uaenorth.redis.azure.net -p 10000 -a $(az keyvault secret show --vault-name kv-tokenfactory -n redis-password --query value -o tsv) PING` |
| **ACR — list** | `az acr repository list -n acrtokenfactory -o table` |
| **ACR — tags for one image** | `az acr repository show-tags -n acrtokenfactory --repository infinia-technology/ai-gateway-platform/tf-auth -o table` |
| **Event Hubs** | `az eventhubs eventhub list -g rg-tokenfactory --namespace-name ehns-tokenfactory -o table` |

Four things to know before using these:

- **Port 10000 for Redis, not 6379**, and **no `--tls`**: Azure Managed Redis is configured
  `Plaintext` here because the application has no TLS client path.
- **`sslmode=require` is mandatory** for PostgreSQL on Flexible Server.
- **Verify DNS with `getent hosts`, not `dig`.** `dig` bypasses the OS resolver order and can
  report success while the application path fails. Every one of these hostnames must resolve to a
  private address; a public address means the private DNS zone or the resolver is wrong.
- **Do not use `az aks get-credentials --admin`.** That is the cluster-local admin credential,
  which bypasses every role assignment. Use the plain form so access is governed by your own
  identity.

---

## 8. Security

### 8.1 Access Management

| Layer | Mechanism |
|---|---|
| Data-plane API keys | SHA-256 hash validation, fail-closed in `tf-warden` |
| Portals | OIDC Authorization Code with PKCE via Keycloak, BYOIDC per organization |
| Fine-grained authorization | OpenFGA |
| Role model | separate tenant and platform role axes |
| Cluster | private API server; VPN or Bastion, then the jumphost |

### 8.2 Secrets & Certificates

- Azure Key Vault in **RBAC mode**, purge protection on, 90-day soft delete, private endpoint
  only. Nothing is committed to git.
- API keys are stored as SHA-256 hashes; provider credentials are encrypted AES-256-GCM at rest.
- TLS terminates at the **Cloudflare edge**; the tunnel-to-cluster hop is plaintext inside the
  VNet, which is why Argo CD runs with `server.insecure: true`.
- Honest limit: Key Vault gives custody and lifecycle, not secrecy from inside the cluster. The
  projected Secret is plaintext in etcd and readable by any principal with `get secrets` in the
  namespace. Restricting that is a Kubernetes RBAC question, not a secret-store question.

### 8.3 Container & Image Security

Images are built from pinned bases, scanned with Trivy in CI, and served from a private registry
behind a private endpoint. Tags are explicit and immutable by chart policy.

### 8.4 Network Security

No public ingress into the VNet at all. Every PaaS dependency is private-endpoint only. Pod egress
leaves through Azure Firewall against an explicit allow-list, and every NSG ends with a deny rule
so subnet-to-subnet traffic is allow-listed rather than assumed.

The practical consequence for engineers: **a missing egress rule presents as an application-level
timeout, not a network error.** The Google SSO 502 traced to one missing hostname in that
allow-list, not to a bad secret.

### 8.5 Audit & Compliance

- Multi-tenant isolation through per-organization scoping and OpenFGA; an auditable money ledger;
  immutable usage events.
- **Gap:** cluster audit logging is not enabled, so a secret read in the namespace leaves no
  queryable trace. Enabling the admin-only audit category is the cheapest useful fix.

---

## 9. Monitoring & Operations

### 9.1 Current State

**Neither observability stack is deployed.** Azure Monitor diagnostic settings and Managed Grafana
exist in the infrastructure configuration but are parked, disabled for cost. There are no alert
rules, no dashboards and no on-call integration.

What this costs operationally:

- **Firewall denies are invisible**, which is exactly why a missing egress rule is hard to
  diagnose.
- No historical metrics, so capacity decisions are made without trend data.
- No audit trail for secret access.

Re-enabling is a file rename and an apply. If only part is restored, the admin-only audit category
is the most useful half.

### 9.2 What Monitoring Does Exist

| Source | Signal |
|---|---|
| `tf-ops` | read-only service status and logs through the admin portal |
| Service probes | `/healthz`, `/readyz`, `/version` on every service |
| Argo CD | sync and health status per resource |
| `kubectl` from the jumphost | the operator's primary tool today |

### 9.3 Health Checks

| Check | Expected |
|---|---|
| Private DNS resolution of Key Vault and PostgreSQL | a private address — use `getent hosts`, not `dig` |
| Infrastructure drift | `terraform plan` reports no changes |
| Secret synced | non-zero byte count for the key in `tokenfactory-secrets` |
| Keycloak | 3 Infinispan members |
| Migrations | `dirty` false in both tracking tables |

### 9.4 Daily Operations

Routine work is: inspect through `tf-ops` or `kubectl`, deploy through Argo CD, change
configuration through the overlay and a PR.

**Everything goes through Helm.** A manual `kubectl scale` or `patch` is silently reverted by the
next upgrade — this has already happened twice, to `tf-ledger` and `envoy` replica counts.

---

## 10. Maintenance & Upgrades

### 10.1 Application Upgrades

Import the tag to ACR, set it in the overlay, merge, sync. The chart's `requireImmutableTag` makes
a dropped tag fail the render rather than quietly deploying the wrong version.

### 10.2 Kubernetes & Node Upgrades

The cluster version is a Terraform variable. Node images do not update themselves; upgrades are
deliberate actions planned and applied from the jumphost.

### 10.3 Dependency & Security Patches

Go module and npm updates follow the normal branch-and-PR flow, gated by the same tests, lint and
secret scan. Image rebuilds are explicit runs of `build.yml`.

### 10.4 Capacity

Replica counts and resource requests are values changes. Two standing constraints: `tf-ledger`
stays at one replica until leader election exists, and PostgreSQL storage only grows — its IOPS
scale with the allocated size, and the current allocation has not been load-tested.

---

## 11. Troubleshooting & Recovery

### 11.1 Common Issues

| Symptom | Cause | Action |
|---|---|---|
| Application times out calling an external service | FQDN missing from the firewall allow-list | add the rule; the deny is not logged today |
| `CROSSSLOT` from multi-key Redis scripts | the Redis clustering policy shards keys | the policy must stay `NoCluster`; changing it requires hash tags shipped first |
| Pods stuck in `CreateContainerConfigError` | the Secret is missing a key, or the syncer was deleted | check the SecretProviderClass names it in both places |
| Secret is in the vault but the pod does not see it | one of the two manifest entries is missing, or pods were not restarted | add the entry, then restart the consumers |
| Every pod logs `Dirty database version` | a migration died mid-run and left the flag set | **do not clear the flag** — see 11.2 |
| `tf-auth` fatal at startup | the explicit Redis host override is missing from the overlay | restore it; the subchart reads the wrong value in external mode |
| Name resolves but the application cannot reach it | `dig` bypasses the OS resolver order | verify with `getent hosts` |

### 11.2 Migrations — the one trap worth reading twice

The `tokenfactory` database has **two** independent tracking tables: one for `tf-accounts` and
`tf-auth`, one for `tf-ledger`. Every replica runs migrations at startup, and the migration tool
marks `dirty` **before** applying SQL. If a pod dies mid-migration the flag stays set, every other
pod exits on sight of it, and the real error is never logged.

**Clearing the dirty flag is the wrong recovery.** It makes the tool treat the migration as applied
without running it, silently skipping schema — which is how twelve migrations were skipped once and
tables that should have existed did not. The correct recovery is to force the version back by one
and re-run. `scripts/fix-tf-migrations.sh` serialises this safely; `scripts/catch-migration-error.sh`
recovers the error the dirty flag is hiding.

### 11.3 Backup & Restore

| Data | Protection |
|---|---|
| PostgreSQL | 30-day geo-redundant automated backup, point-in-time restore |
| Terraform state | blob versioning on a zone-redundant account |
| Event Hubs | 7-day retention plus capture to blob |
| Key Vault | soft delete 90 days, purge protection on |
| Redis | none — rebuildable from PostgreSQL and traffic |

**A restore has never been rehearsed.** Recovery times are unmeasured.

### 11.4 Disaster Recovery

A full rebuild from the repositories alone is **not currently possible and has never been tested**.
Four reasons: Key Vault purge protection holds the vault name for 90 days; the Terraform state
account is declared in its own state; the eight-to-ten manual post-apply steps are not in either
repository; and two private keys are held outside both.

### 11.5 Incident Management

No on-call rota, alerting pipeline or incident tooling is defined. Escalation today is through the
operator who holds jumphost access. Incidents so far are recorded as decision comments in the code
and in the issue tables above.

---

## 12. Release & Change Management

### 12.1 Release Process

A release is an owner-gated, version-titled PR from `develop` to `main`.

1. The owner approves a `chore/release-X.Y.Z` branch to start a cycle.
2. Release notes are finalized in `docs/release/vX.Y.md` — one file per minor line, newest section
   first, never one file per patch.
3. The heading becomes `## X.Y.Z — released <date>`, and the `develop → main` PR is cut with a
   version title and merged with a **regular** merge so both branches keep identical history.
4. `build.yml` is run for that ref; images and scan results are confirmed.
5. The tag is imported to ACR, set in the overlay, and synced through Argo CD.

Versions are **never** bumped outside this cycle.

### 12.2 Change Management

- Every change goes through a PR; direct pushes to `main` are rejected in both repositories.
- Each PR carries its own release-note entry, so the notes are never reconstructed afterwards.
- A change touching an irreversible infrastructure setting must say so in the description.
- Firewall and NSG edits affect every workload and are reviewed against the allow-list, not just
  the diff.
- On an infrastructure plan, a `destroy and then create replacement` line on a stateful resource is
  a **stop**, not a warning.

### 12.3 Current Status & Open Items

The environment is deployed and running. It is **not yet serving production traffic**. The
remaining work is configuration and hardening, not architecture.

**Go-live blockers:**

| # | Item |
|---|---|
| 1 | **Model catalog is empty** — zero providers, models, bindings and credentials. The gateway cannot route a request until it is populated. |
| 2 | **Stripe is not configured** — the secret key is absent and the webhook secret holds an inert placeholder, so checkout, top-up and webhooks are all inactive. The publishable key also still points at the previous environment's account. |
| 3 | **Apex domain not reaching the cluster** — `xailon.ai` serves a parked page from outside while the service serves the real one. A Cloudflare routing problem, not a cluster problem. |

**Hardening backlog:**

| Item | Why it matters |
|---|---|
| Enable the admin audit log category | No record of secret access today |
| Finish moving the SecretProviderClass and the post-apply scripts into the infrastructure repository | Single copies on one machine until then |
| Fix the `tf-auth` Redis host defect in the subchart | Removes an overlay workaround |
| Add leader election to `tf-ledger` | Unblocks replicas above one |
| Move OpenFGA onto its database | Authorization state does not survive a restart |
| Rehearse a PostgreSQL restore | Recovery time is unmeasured |
| Resolve the Redis API-version retirement | Reported for 31 October 2026; confirm through Azure Service Health |
| Restore observability | Every operational gap above traces back to this |

---

**End of document**
