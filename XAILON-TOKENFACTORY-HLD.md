# Xailon TokenFactory — Azure Deployment
## High-Level Design

| | |
|---|---|
| **Document** | High-Level Design |
| **Product** | Xailon TokenFactory — AI Gateway Platform on Azure |
| **Namespace (primary deployment)** | `xailon-tf-prod` |
| **Cluster** | `aks-tokenfactory` (private AKS, UAE North) |
| **Application repository** | `github.com/Infinia-Technology/ai-gateway-platform` |
| **Infrastructure repository** | `github.com/Infinia-Technology/azure-infra-terraform` |
| **Branch** | `main` |
| **Image tag observed in cluster** | `0.16.2-e1bd218` |
| **Author** | Sidhant Khanna |
| **Date** | 2026-10-15 |
| **Status** | Draft |

---

## Table of Contents

1. [Document History](#1-document-history)
2. [Management Summary](#2-management-summary)
3. [Preface](#3-preface)
4. [Requirements](#4-requirements)
5. [Solution Overview](#5-solution-overview)
6. [Architectural Overview](#6-architectural-overview)
7. [Logical Architecture](#7-logical-architecture)
8. [Physical Architecture](#8-physical-architecture)
9. [API Catalog & Security](#9-api-catalog--security)
10. [Encryption Strategy](#10-encryption-strategy)
11. [Recovery](#11-recovery)
12. [Appendices & Reference Materials](#12-appendices--reference-materials)

---

## 1 Document History

### 1.1 Revision History

| Revision | Date | Author | Summary of Change |
|---|---|---|---|
| 0.1 | 2026-10-15 | Sidhant Khanna | Initial HLD for the Xailon Azure deployment, authored from the Terraform configuration, the `values-xailon-prod.yaml` overlay and the live `xailon-tf-prod` deployment. Companion to the platform HLD, which remains the authority on the application architecture itself. |

### 1.2 Document Approvers

| Name | Title |
|---|---|
| (to be confirmed) | (to be confirmed) |

### 1.3 Document Location

This controlled document is stored as `XAILON-TOKENFACTORY-HLD.md`, alongside the
Xailon DevOps Overview and Low-Level Design.

---

## 2 Management Summary

**Purpose.** This document is the executive and architectural blueprint for the
**Xailon** deployment of TokenFactory on Microsoft Azure.

**Xailon and TokenFactory are the same product.** The application code, the Helm
chart, the services and the APIs are identical. What this document describes is a
different *substrate*: TokenFactory runs on bare metal with bundled
dependencies; Xailon runs on Azure in UAE North with managed services.

**Why this deployment exists.** Three requirements drove it:

1. **Data residency** — the platform and all of its data must sit in a UAE
   region under a single subscription.
2. **No public attack surface** — no component may be addressable from the
   internet. The cluster API is private, every data service is private-endpoint
   only, and inbound traffic arrives through an outbound-initiated tunnel.
3. **Managed state** — the operational burden of running PostgreSQL, Redis and
   Kafka in-cluster is replaced by Azure managed services with platform-level
   high availability and backup.

**What was built.** A private AKS cluster, Azure Database for PostgreSQL
Flexible Server with zone-redundant HA, Azure Managed Redis, Event Hubs in place
of Kafka, Azure Container Registry, Key Vault, two storage accounts, Azure
Firewall mediating all egress, a DNS Private Resolver, and point-to-site and
site-to-site VPN gateways — approximately 106 resources, all defined as
Terraform.

**Current status.** The infrastructure and the platform are deployed and
running. The remaining work is configuration and hardening rather than
architecture. The open items are recorded in section 5.4; the most significant
are that observability is deliberately not deployed, and that a rebuild from the
repositories alone has never been tested and is not currently possible.

**A second party.** A partner (ZySec) operates two collector VMs inside this
environment, reached over a site-to-site VPN. They are confined to a dedicated
subnet and can reach nothing else; this is verified by test, not assumed.

---

## 3 Preface

### 3.1 Introduction & Background

TokenFactory is a sovereign, multi-tenant AI Gateway exposing **one
OpenAI-compatible endpoint** across many LLM providers, with per-request
metering in nano-USD, prepaid credit enforcement and per-organization identity.
The platform HLD is the authority on that architecture.

Xailon is that platform delivered on Azure for a distinct customer and brand
(`*.xailon.ai`). Every architectural decision recorded here concerns the
environment, not the product.

### 3.2 Input Sources for this Document

| Source | Contribution |
|---|---|
| `azure-infra-terraform` at `417dc58` and later | the infrastructure as actually declared |
| `helm/tokenfactory/values-xailon-prod.yaml` | the environment overlay and its recorded reasons |
| Live inspection of `aks-tokenfactory` | workload counts, addresses, effective rules |
| `az` control-plane queries | resource SKUs, zones, network configuration |
| Connectivity testing from inside the environment | the isolation claims in section 9 |

### 3.3 Related Documents

| Document | Role |
|---|---|
| TokenFactory HLD | the platform architecture — environment-agnostic, remains authoritative |
| TokenFactory LLD | platform implementation detail |
| `XAILON-OVERVIEW.md` | the end-to-end DevOps lifecycle for this deployment |
| `XAILON-TOKENFACTORY-LLD.md` | implementation detail for this deployment |

### 3.4 Definitions & Glossary

Platform terms are defined in the TokenFactory HLD. Terms specific to this
deployment:

| Term | Meaning |
|---|---|
| AMR | Azure Managed Redis |
| Private endpoint | A private IP inside the VNet representing a PaaS service |
| CSI Secrets Store | The AKS addon projecting Key Vault secrets into pods |
| SecretProviderClass | The manifest declaring which vault secrets the cluster fetches |
| UDR | User-defined routing; forces egress through the firewall |
| One-way door | A setting that cannot be reverted by reverting the code |
| Collector enclave | The isolated subnet holding the partner's two VMs |

### 3.5 Purpose of this Document

To record, at an executive and architect level, what was built on Azure, which
decisions were made, what they cost, and which risks remain open. Implementation
detail belongs in the LLD; lifecycle and process belong in the DevOps Overview.

### 3.6 Scope

**In scope:** the Azure environment, the platform's deployment onto it, the
decisions that differ from the platform baseline, and the partner connectivity.

**Out of scope:** the application architecture, which the platform HLD covers;
and the second party's own environment beyond the address space they present.

---

## 4 Requirements

### 4.1 Current Context

TokenFactory already runs in production on bare metal. Xailon required the same
platform under separate branding, in a UAE region, on infrastructure the
customer's governance would accept. The application was not to be forked: any
environment difference had to be expressible in Helm values, not code.

That constraint held. **No application change was required** for this
deployment; every difference is a value in the overlay or a Terraform resource.

### 4.2 Functional Requirements

Identical to the platform HLD — OpenAI-compatible inference across modalities,
metering, prepaid credit, multi-tenant identity, tenant and operator portals.

The two requirements specific to this deployment:

| # | Requirement |
|---|---|
| F1 | Serve `*.xailon.ai` as a distinct brand and identity domain |
| F2 | Provide a second party with network access to two collector VMs, and to nothing else |

### 4.3 Non-Functional Requirements

| # | Requirement | How it is met |
|---|---|---|
| N1 | All data resident in UAE North | Single region, single subscription, no geo-replication outside the region except PostgreSQL's geo-redundant backup |
| N2 | No public ingress to the environment | No public IP on any cluster service; inbound arrives via an outbound-initiated Cloudflare tunnel |
| N3 | All egress controlled and inspectable | Azure Firewall with an explicit FQDN allow-list; UDR on every workload subnet |
| N4 | Platform-level HA for stateful services | PostgreSQL zone-redundant HA; Redis HA enabled; AKS node pools across three zones |
| N5 | Secrets never in source control | Azure Key Vault, projected by CSI driver; nothing committed |
| N6 | Infrastructure reproducible as code | Terraform in a protected repository |
| N7 | Partner confined to two hosts | Dedicated subnet, deny-both-directions NSG, verified by test |

**N6 is only partially met.** See section 5.4.

### 4.4 Access Locations

| Who | Path |
|---|---|
| End users and API clients | Internet → Cloudflare edge → tunnel → cluster |
| Platform operators | Point-to-site VPN or Azure Bastion → jumphost → cluster API |
| Terraform execution | Jumphost only — the state backend and all data services are private |
| Second party | Site-to-site VPN → collector subnet only |

### 4.5 Other Requirements

Cost is an active constraint and has shaped the design: the observability stack
is implemented but deliberately not deployed, and the DNS and firewall diagnostic
categories are off. This is recorded as a risk rather than presented as a choice
without consequence.

---

## 5 Solution Overview

### 5.1 System Context

```
     Internet                        │  AZURE — UAE North, subscription: Token Factory
                                     │
  ┌──────────────┐                   │   ┌──────────────────────────────────────────┐
  │ API clients  │──── HTTPS ───┐    │   │  vnet-tokenfactory  10.40.0.0/16         │
  │ Portals      │              │    │   │                                          │
  └──────────────┘              ▼    │   │   ┌────────────────────────────────┐     │
                        ┌─────────────────┐  │ aks-tokenfactory  (PRIVATE)    │     │
                        │ Cloudflare edge │  │  xailon-tf-prod   — platform   │     │
                        └────────┬────────┘  │  openmeter        — metering   │     │
                                 │        │  │  cloudflare       — connector  │     │
                        outbound 7844 only│  │  nginx-ingress · argocd        │     │
                                 │        │  └──────────┬─────────────────────┘     │
                                 └───────▶│             │ private endpoints         │
                                          │   ┌─────────▼──────────────────────┐    │
  ┌──────────────┐                        │   │ PostgreSQL · Redis · Key Vault │    │
  │ Operators    │── P2S VPN / Bastion ──▶│   │ ACR · Event Hubs · Blob        │    │
  └──────────────┘                        │   └────────────────────────────────┘    │
                                          │                                          │
  ┌──────────────┐                        │   ┌────────────────────────────────┐    │
  │ ZySec        │── S2S VPN ────────────▶│   │ snet-collector 10.40.5.0/28    │    │
  │ 10.23.3.0/26 │                        │   │ two collector VMs — isolated   │    │
  └──────────────┘                        │   └────────────────────────────────┘    │
                                          │              │                           │
                                          │        Azure Firewall  ── all egress ──▶ │
                                          └──────────────────────────────────────────┘
```

Three entry paths, each authenticated and none of them a public listener on the
cluster: the Cloudflare tunnel for application traffic, the point-to-site VPN or
Bastion for operators, and the site-to-site VPN for the partner. All egress
leaves through the firewall against an explicit allow-list.

### 5.2 Design Constraints & Vendor Recommendations

- **Private cluster.** `enablePrivateCluster = true`; the API server has no
  public FQDN. Every `kubectl` and `terraform` operation runs from the jumphost.
- **`outbound_type = userDefinedRouting`.** The cluster has no outbound path of
  its own; if the route table or a firewall rule is wrong, nodes do not register
  and images do not pull.
- **CNI Overlay with Cilium.** The pod CIDR `10.244.0.0/16` sits outside the VNet
  space and is **immutable after cluster creation**. Pod egress is SNATed to the
  node address, so firewall rules match node IPs.
- **Redis must run `NoCluster`.** See section 5.3.
- **Helm release name must be `tokenfactory`.** Six subcharts hardcode
  `<release>-secrets` and ignore `global.secretsName`.
- **Images must come from ACR.** Infra subcharts read `global.imageRegistry`,
  not `global.image.registry`; both are set.

### 5.3 Architectural Decisions

| ID | Decision | Alternatives considered | Rationale |
|---|---|---|---|
| X1 | **No public load balancer; inbound via Cloudflare Tunnel** | Service type LoadBalancer; Application Gateway | On AKS, `type: LoadBalancer` provisions a public IP and UDR does not prevent it. The environment holds exactly three public IPs — Bastion, Firewall, VPN gateway — all authenticated. A tunnel connector dialling outbound adds no listener. |
| X2 | **Azure Key Vault via the CSI driver, not HashiCorp Vault** | Vault in-cluster, as the platform uses | Removes a stateful component to operate, and aligns secret custody with the subscription's RBAC. Cost: a derived Kubernetes Secret still exists in etcd, because the services read environment variables. |
| X3 | **Managed PostgreSQL, Redis and Event Hubs** | The chart's bundled equivalents | Platform HA, backup and patching. Cost: two behaviours had to be worked around — Redis clustering and a chart defect in `tf-auth`. |
| X4 | **Redis `clusteringPolicy = NoCluster`** | `EnterpriseCluster` | `EnterpriseCluster` presents one endpoint but still shards keys; `tf-ledger`'s multi-key credit scripts then fail with `CROSSSLOT`. Hit in production on 11 Oct 2026 and reverted. Moving to clustered mode requires hash tags shipped and verified first. |
| X5 | **Redis provisioned with `azapi`, not `azurerm`** | Wait for provider support | The `azurerm` provider lacked Azure Managed Redis at the required API version. Two API versions are pinned deliberately: the cluster on `2025-04-01`, the database on `2025-07-01`. |
| X6 | **F5 NGINX Ingress Controller, not community `ingress-nginx`** | The community controller | Every Ingress annotation is `nginx.org/*`; the community controller ignores them **silently**, including the proxy-buffer settings that fix an OIDC callback failure. |
| X7 | **Argo CD in-cluster, manual sync** | External automation, as production uses | The API server is private, so an external controller has no path. Manual sync keeps deployment deliberate. This makes Xailon more GitOps-native than the bare-metal environment, which is upgraded by automation running outside the cluster. |
| X8 | **`tf-ledger` pinned to one replica** | Three, like every other service | It starts five unguarded ticker loops with no leader election. The bug was hit at two replicas and fixed by pinning. Raising it requires leader election first. |
| X9 | **Partner isolated in a dedicated subnet** | Placing the VMs in an existing subnet | An NSG attaches to a subnet; a dedicated one is what makes "these two hosts and nothing else" enforceable, and allows denying outbound to the rest of the VNet. |
| X10 | **Observability deployed as code but not enabled** | Deploy it | Cost. Recorded as a risk, not a neutral choice — see 5.4. |

### 5.4 Statement of Design Assumptions & Risks

| # | Risk | Impact | Current position |
|---|---|---|---|
| R1 | **No observability** | Firewall denies are invisible; a missing egress rule presents as an application timeout with nothing to correlate. Has already cost diagnosis time twice. | Accepted for cost. `logging.tf.disabled` and `grafana.tf.disabled` hold the configuration; re-enabling is a rename and an apply. |
| R2 | **Rebuild from the repositories is not possible and never tested** | Recovery time unknown | Four causes in section 11. Treated as the largest structural gap. |
| R3 | **No infrastructure CI** | A malformed change is caught only at plan time, by a human | `terraform plan` needs VNet-internal execution, so a hosted runner cannot run it. Static checks could run and do not. |
| R4 | **Manual post-apply steps** | Roughly eight to ten steps are scripted but outside both repositories | Scripts are being moved into the infrastructure repository. The Cloudflare configuration has no code form at all. |
| R5 | **Redis API version retirement** | Reported for 31 Oct 2026 for versions older than `2025-07-01`; the cluster is on `2025-04-01` | Not confirmed on an official Microsoft retirement page. The cluster cannot simply be upgraded — a newer version requires a property that cannot be set on an existing resource. Needs a planned migration. |
| R6 | **Secret readable inside the cluster** | The projected Secret is plaintext in etcd, readable by any principal with `get secrets` in the namespace | Inherent to environment-variable consumption. Mitigated by RBAC, not by the secret store. |
| R7 | **OpenFGA runs in memory** | Authorization state does not survive a pod restart | Its database exists and is unused. |
| R8 | **`tf-auth` depends on an overlay workaround** | A subchart defect means `tf-auth` is fatal at startup without an explicit Redis host | Worked around in values; the fix belongs in the chart. |

### 5.5 Architectural Dependencies

| Dependency | Nature |
|---|---|
| Azure UAE North | Region availability for every SKU in use |
| Cloudflare | The only inbound path; hostname routing lives in their dashboard, not in code |
| GitHub Container Registry | Image source, mirrored into ACR |
| Entra ID | Operator identity for VPN, Bastion, jumphost and cluster access |
| ZySec | Site-to-site tunnel; their endpoint configuration is outstanding |

---

## 6 Architectural Overview

### 6.1 Architectural Overview Diagram

```
  ┌───────────────────────────────── xailon-tf-prod ──────────────────────────────────┐
  │                                                                                   │
  │   cloudflared ──▶ nginx-ingress ──▶ tf-portal-web · tf-admin-web · tf-site-web    │
  │       │                                tf-docs-web · tf-auth (+ Keycloak)         │
  │       │                                                                           │
  │       └──────────▶ envoy Gateway (6) ──▶ tf-warden  (ext_authz, fail-closed)      │
  │                          │              tf-relay   (ext_proc, metering)           │
  │                          │              tf-shim    (special paths)                │
  │                          │              tf-babel   (Anthropic dialect)            │
  │                          ▼                                                        │
  │                    provider backends (via Azure Firewall)                         │
  │                                                                                   │
  │   CONTROL PLANE   tf-auth · tf-accounts · tf-control · tf-ledger · tf-ops          │
  │   IN-CLUSTER      keycloak (3) · openfga · minio · akv-secret-syncer               │
  └───────────┬───────────────────────────────────────────────────────────────────────┘
              │ private endpoints
  ┌───────────▼────────────────────────────────────────────────────────────────────┐
  │  PostgreSQL (5 DBs)  ·  Managed Redis  ·  Key Vault  ·  ACR  ·  Event Hubs      │
  └────────────────────────────────────────────────────────────────────────────────┘
              │
  ┌───────────▼──────────────┐     ┌──────────────────────────────────────────────┐
  │ openmeter namespace      │     │ snet-collector — partner enclave             │
  │ OpenMeter · ClickHouse   │     │ 2 collector VMs · load-balancer virtual IP   │
  │ Svix · bundled Redis     │     │ deny in and out, except the partner CIDR     │
  └──────────────────────────┘     └──────────────────────────────────────────────┘
```

### 6.2 Key Concepts

- **The cluster has no front door.** Every inbound path is a connection the
  environment made outbound first.
- **Egress is allow-listed, not filtered.** Anything not named in the firewall
  policy fails, and fails as a timeout.
- **Secrets are custodied externally and consumed internally.** Key Vault is the
  source of truth; the Kubernetes Secret is a derived copy whose existence
  depends on a placeholder workload holding a CSI mount open.
- **The environment overlay is thin.** It carries only what differs, so the
  distance between Xailon and the bare-metal production environment is
  reviewable as a diff.

### 6.3 Architectural Best Practices & Detailed Considerations

- **Deny by default, both directions.** Every NSG ends with a deny; the partner
  enclave also denies outbound to the VNet, which is what prevents a pivot.
- **Pin what must not move.** Provider versions, the chart's image tag
  (`requireImmutableTag`), the Keycloak image digest, and two Redis API versions.
- **Record decisions where they take effect.** The configuration carries the
  reasoning inline, because several settings look redundant and are not —
  removing them reintroduces drift or an outage.
- **Separate the partner from the platform.** The collector VMs run no
  TokenFactory component and share no subnet with one.

### 6.4 Data-Plane Filter Wiring

Unchanged from the platform HLD: `ext_authz` to `tf-warden` fail-closed,
`ext_proc` to `tf-relay` for metering, `tf-shim` on special paths, `tf-babel` as
a dialect proxy registered as its own route. The Xailon difference is only that
the `envoy` Service is ClusterIP and reached by the tunnel connector rather than
by a load balancer.

---

## 7 Logical Architecture

### 7.1 Logical Node Description

| Node | Role | Realisation on Azure |
|---|---|---|
| Gateway | Data-plane entry, routes by model | `envoy`, 6 replicas, ClusterIP |
| Authorization filter | Fail-closed auth, policy, credit | `tf-warden`, 3 replicas |
| Metering filter | Usage extraction, event emission | `tf-relay`, 3 replicas |
| Transform / dialect | Special paths, Anthropic surface | `tf-shim`, `tf-babel`, 3 each |
| Identity boundary | Sessions, SSO, BYOIDC | `tf-auth` + Keycloak StatefulSet (3) |
| Accounts & billing | Orgs, projects, keys, Stripe | `tf-accounts`, 3 replicas |
| Catalog & routing | Providers, models, EAGW reconciliation | `tf-control`, 3 replicas |
| Money pipeline | Usage ledger, wallet, pricing | `tf-ledger`, **1 replica** |
| Operations | Read-only platform status | `tf-ops`, 3 replicas |
| Web surfaces | Tenant, admin, site, docs | 4 deployments |
| System of record | Durable state | Azure PostgreSQL Flexible Server, 5 databases |
| Hot path & streams | Counters, usage stream | Azure Managed Redis |
| Authorization store | Fine-grained authorization | OpenFGA, **in-memory** |
| Metering sink | Usage aggregation | OpenMeter + ClickHouse, own namespace |
| Event transport | Metering transport | Azure Event Hubs, Kafka protocol |
| Secret custody | All credentials | Azure Key Vault + CSI driver + syncer |
| Object storage | Export archive, capture target | MinIO in-cluster; two storage accounts |

---

## 8 Physical Architecture

### 8.1 Cluster Overview

| Property | Value |
|---|---|
| Cluster | `aks-tokenfactory`, Kubernetes 1.35.7 |
| Access | **Private** — no public API endpoint |
| Network | CNI Overlay + Cilium dataplane and policy |
| Outbound | `userDefinedRouting` to Azure Firewall |
| Identity | OIDC issuer and workload identity enabled |
| Policy | Azure Policy enabled; image cleaner at 168 h |

### 8.2 Node Pools

| Pool | Size | Count | Zones | Notes |
|---|---|---|---|---|
| `agentpool` (system) | `Standard_D4as_v5` | 3 | 1, 2, 3 | `only_critical_addons_enabled` |
| `userpool` | `Standard_D8as_v5` | 3 | 1, 2, 3 | application workloads |

### 8.3 Namespaces

| Namespace | Contents |
|---|---|
| `xailon-tf-prod` | the `tokenfactory` release — all `tf-*`, `envoy`, Keycloak, OpenFGA, MinIO, `akv-secret-syncer` |
| `openmeter` | the `openmeter` release — OpenMeter, ClickHouse, Svix |
| `cloudflare` | tunnel connector and its own secret syncer |
| `nginx-ingress` | F5 NGINX Ingress Controller |
| `argocd` | Argo CD v3.5.4 |

### 8.4 Managed Services

| Service | SKU / configuration |
|---|---|
| PostgreSQL | Flexible Server 16, `GP_Standard_D8ds_v5`, 128 GB auto-grow, **ZoneRedundant HA**, 30-day geo-redundant backup |
| Redis | Azure Managed Redis `Balanced_B10` (12 GB), HA enabled, `NoCluster`, port **10000**, plaintext, `NoEviction`, no persistence |
| Event Hubs | Standard tier, capacity 1, 7 hubs, capture to blob on the three DLQs |
| Container Registry | Premium, `admin_enabled = false`, private endpoint |
| Key Vault | Standard, **RBAC mode**, purge protection on, 90-day soft delete |
| Storage | Archive (LRS) and Terraform state (**ZRS**, versioned) |
| Firewall | `AZFW_VNet` Standard, one policy, one rule collection group |
| DNS | Private Resolver inbound endpoint, static address |
| VPN | `VpnGw1AZ` route-based: point-to-site `172.16.0.0/24`, site-to-site to `10.23.3.0/26` |
| Bastion | Standard, native tunneling enabled |
| Jumphost | `Standard_D2as_v5`, Ubuntu 22.04, Entra SSH extension, **no role assignment** |

### 8.5 Network & Address Plan

Eight subnets in `10.40.0.0/16`: gateway, Bastion, firewall, management,
PostgreSQL (delegated), private endpoints, DNS resolver (delegated), AKS nodes,
plus the partner enclave at `10.40.5.0/28`. Pod CIDR `10.244.0.0/16` and service
CIDR `10.0.0.0/16` sit outside the VNet space.

Every workload subnet routes `0.0.0.0/0` to the firewall. NSGs protect the
PostgreSQL, private-endpoint, AKS and collector subnets, with a NIC-level NSG on
the jumphost.

---

## 9 API Catalog & Security

### 9.1 Public Surfaces

| Host | Backend | Path into the cluster |
|---|---|---|
| `api.xailon.ai` | `envoy` Gateway | tunnel → envoy directly |
| `console.xailon.ai` | `tf-portal-web` | tunnel → nginx ingress |
| `admin.xailon.ai` | `tf-admin-web` | tunnel → nginx ingress |
| `auth.xailon.ai` | `tf-auth` + Keycloak | tunnel → nginx ingress |
| `xailon.ai` | `tf-site-web` | tunnel → nginx ingress |
| `argo.xailon.ai` | Argo CD | tunnel → Argo CD |

Hostname-to-service routing is held in the Cloudflare Zero Trust dashboard and
is **not version-controlled**.

### 9.2 API Security Measures

As per the platform HLD — API keys SHA-256 hashed and validated fail-closed,
OIDC with PKCE for portals, OpenFGA for fine-grained authorization, separate
tenant and platform role axes.

Deployment-specific additions: TLS terminates at the Cloudflare edge; no cluster
service is publicly addressable; all egress to providers passes the firewall
allow-list.

### 9.3 Partner Access Control

The second party reaches `10.40.5.4`, `10.40.5.5` and `10.40.5.6` over the
site-to-site tunnel, and nothing else. Enforcement rests on three properties:

1. Azure's `VirtualNetwork` service tag **expands to include address spaces
   declared on a Local Network Gateway**. Every allow rule in this environment
   uses an explicit CIDR and every deny uses the tag — so the expansion makes
   the existing denies cover the partner rather than granting them access.
2. The collector NSG denies outbound to `VirtualNetwork`, preventing a pivot.
3. A deny on the DNS resolver subnet stops the partner enumerating private names.

**Verified by test** on 15 October 2026 from inside a collector VM: PostgreSQL,
Redis, Key Vault, ACR, Event Hubs, the jumphost and an AKS node were all
unreachable; the peer collector was reachable, as intended.

### 9.4 Error Surfacing

Unchanged from the platform HLD. One environment-specific symptom is worth
recording at architecture level: **a missing firewall allow-list entry presents
as an application-level timeout**, not a network error, and with logging
disabled there is nothing to correlate it against.

---

## 10 Encryption Strategy

### 10.1 In Transit

| Path | Protection |
|---|---|
| Client → Cloudflare edge | TLS |
| Cloudflare → cluster | tunnel over TCP 7844, outbound only |
| Tunnel connector → services | plaintext HTTP **inside the VNet** — which is why Argo CD runs `server.insecure` |
| Services → PostgreSQL | TLS, `sslMode: require` |
| Services → Redis | **plaintext**, port 10000 — the application has no TLS client path |
| Services → Key Vault, ACR, Event Hubs | HTTPS over private endpoints |
| Operators → environment | IPsec (P2S) or Bastion over TLS |
| Partner → collectors | IPsec, IKEv2, AES256/SHA256/DHGroup14 |

Redis in plaintext is a conscious acceptance: the traffic never leaves the VNet
and the application cannot speak TLS to it.

### 10.2 At Rest

| Data | Protection |
|---|---|
| PostgreSQL | Azure platform encryption |
| Blob storage | Azure platform encryption; state account versioned and ZRS |
| Key Vault | Azure-managed keys, purge protection, 90-day soft delete |
| Provider credentials in the catalog | AES-256-GCM at the application layer |
| API keys | SHA-256 hashes; plaintext never persisted |
| **Kubernetes Secrets** | **Plaintext in etcd** — no customer-managed key encryption configured |

### 10.3 Key Management

Key Vault is the single custody point. Terraform generates and stores two
secrets; everything else is seeded by operators. The CSI driver projects them
using the AKS addon identity, which holds `Key Vault Secrets User` and nothing
more.

Two private keys are held **outside** both repositories: the VPN root
certificate and the Cloudflare Origin certificate. Their expiry is not tracked
and nothing alerts before they lapse.

---

## 11 Recovery

### 11.1 Backup Position

| Data | Protection | Tested |
|---|---|---|
| PostgreSQL | 30-day geo-redundant backup, point-in-time restore | **No** |
| Terraform state | Blob versioning on a ZRS account | No |
| Event Hubs | 7-day retention plus capture to blob | No |
| Key Vault | Soft delete 90 days, purge protection | n/a |
| Redis | None — rebuildable from PostgreSQL and traffic | n/a |

### 11.2 Disaster Recovery

**A rebuild from the repositories alone is not currently possible, and has never
been tested.** Four causes, each independent:

1. **Key Vault purge protection** holds the vault name for 90 days after
   deletion. The name cannot be reused within that window.
2. **The state storage account is declared in its own state.** A clean
   `terraform init` has nowhere to initialise from without creating the account
   by hand first.
3. **Eight to ten manual post-apply steps** exist as scripts outside both
   repositories.
4. **Two private keys** are not in either repository.

Recovery time should be treated as unmeasured until a rehearsal is performed.

### 11.3 Failure Domains

| Failure | Effect |
|---|---|
| One availability zone | AKS and PostgreSQL survive; Redis HA covers it |
| Redis outage | Traffic stops — `tf-warden` cannot authorize and `tf-relay` cannot meter — but no durable data is lost |
| PostgreSQL failover | Brief interruption across nearly every service; it is the widest dependency |
| Key Vault outage | Running pods unaffected; any pod restarting during it fails to start |
| `akv-secret-syncer` deleted | **Every pod fails on next restart** — the Secret is owner-referenced to its ReplicaSet |
| Cloudflare outage | All inbound access lost; no alternative path exists by design |

---

## 12 Appendices & Reference Materials

### 12.1 Environment Quick Reference

| | |
|---|---|
| Subscription / region | Token Factory / UAE North |
| Resource group | `rg-tokenfactory` |
| Cluster / namespace | `aks-tokenfactory` / `xailon-tf-prod` |
| Helm release | `tokenfactory` (mandatory) |
| Image tag | `0.16.2-e1bd218` |
| State entries | approximately 106 |

### 12.2 Key Decision References

| Decision | Where recorded |
|---|---|
| Redis clustering and the two API versions | `redis.tf` |
| Why Terraform runs only on the jumphost | `XAILON-OVERVIEW.md` |
| `tf-ledger` replica pin | `values-xailon-prod.yaml` |
| `tf-auth` Redis host workaround | `values-xailon-prod.yaml` |
| Partner isolation design | `XAILON-TOKENFACTORY-LLD.md`, section 7.6 |
| Irreversible settings | `XAILON-OVERVIEW.md`, One-Way Doors |

### 12.3 Observed Versions

| Component | Version |
|---|---|
| Kubernetes | 1.35.7 |
| Application | `0.16.2-e1bd218` |
| Argo CD | v3.5.4 |
| NGINX Ingress (F5) | chart 1.3.2 / appVersion 3.6.2 |
| PostgreSQL | 16 |
| Terraform | 1.16.4 |
| `azurerm` / `azapi` providers | `~> 3.0` / `~> 2.0` |

---

**End of Document**
