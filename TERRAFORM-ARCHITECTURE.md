# Xailon TokenFactory — Azure Infrastructure (Terraform) Architecture

| | |
|---|---|
| **Document owner** | DevOps Team |
| **Prepared by** | Sidhant Khanna |
| **Platform** | Xailon TokenFactory — Azure (UAE North) |
| **Repository** | `Infinia-Technology/azure-infra-terraform` |
| **Version** | 1.1 |
| **Last updated** | 15 October 2026 |

---

## 1. Introduction

### 1.1 Purpose & Scope

This repository is the **single source of truth for the Azure infrastructure**
beneath the Xailon TokenFactory deployment — network, compute, data services,
identity plumbing and egress control. It stops at the cluster boundary.

**In scope (Terraform-managed):**

- Resource group, virtual network, all eight subnets, route table, NSGs
- AKS cluster (private) and both node pools
- Azure Firewall + policy + the full egress allow-list
- Azure Database for PostgreSQL Flexible Server and its five databases
- Azure Managed Redis
- Azure Container Registry, Key Vault, Event Hubs, two storage accounts
- DNS Private Resolver, private DNS zones, private endpoints
- Point-to-Site VPN gateway, Azure Bastion, jumphost VM
- Two Key Vault secrets

**Out of scope (not in this repository):**

- **Everything inside Kubernetes.** The application is deployed by Helm from
  `Infinia-Technology/ai-gateway-platform` (`helm/tokenfactory`, overlay
  `values-xailon-prod.yaml`). No Kubernetes resource is managed here.
- **Cloudflare configuration** — tunnel, DNS records, Origin certificate.
- **Entra ID role assignments for human operators.** Deliberately excluded:
  each person arrives with their own identity and gets exactly what their own
  Azure RBAC grants.

On Azure the infrastructure is code, and this repository is that code.

### 1.2 Design Principles

Four decisions shape everything below. Each is load-bearing; the consequences
are spelled out under One-Way Doors.

1. **Private by default.** Every PaaS service has
   `public_network_access_enabled = false` and is reached through a private
   endpoint in `snet-pe`. The AKS API server is private. There is no public
   ingress into the VNet at all — inbound traffic arrives through a Cloudflare
   Tunnel initiated *outbound* from a pod.
2. **All egress through the firewall.** AKS, management and PostgreSQL subnets
   route `0.0.0.0/0` to the Azure Firewall private IP.
   Nothing leaves the VNet without matching an
   explicit rule. A missed FQDN fails as an application-level timeout, not as a
   visible network error.
3. **Deny-by-default inside the VNet too.** Every NSG ends with
   `Deny-VNet-Inbound` at priority 4096, so subnet-to-subnet traffic is
   allow-listed rather than assumed.
4. **No ambient credentials on any VM.** The jumphost has a system-assigned
   managed identity solely because the Entra SSH extension requires one, and
   that identity is **deliberately granted no role at all**
   (`jumphost.tf`). Operators authenticate as themselves.

### 1.3 Infrastructure Lifecycle

The lifecycle is: **write → validate → plan → apply → verify → operate**.
Changes are authored as HCL in this repository and reviewed through a pull
request. They are then planned and applied **by hand from the jumphost** —
there is no pipeline and no automated apply. Every apply is followed by
a plan that must report no changes. Environments do not promote: there is
a single production environment.

This document is organised around that lifecycle: the architecture, how
a change is made, checked and deployed, what is deployed, how it is built
and how it is secured, and how it is run, maintained, recovered and released.
It is written for **operators who will modify the environment**, and for
security and management reviewers.

---

## 2. Architecture

### 2.1 Topology

```
                             Internet
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
   pip-bastion              pip-firewall                pip-vgw
        │                        │                        │
┌───────┴────────────────────────┴────────────────────────┴──────────────────┐
│  vnet-tokenfactory          10.40.0.0/16          UAE North                │
│                                                                            │
│  GatewaySubnet       10.40.0.0/27    vgw-tokenfactory  (P2S, 172.16.0.0/24)│
│  AzureBastionSubnet  10.40.0.64/26   bas-tokenfactory                      │
│  AzureFirewallSubnet 10.40.0.128/26  fw-tokenfactory   ◄── 0.0.0.0/0 (UDR) │
│  snet-mgmt           10.40.1.0/27    vm-jumphost  10.40.1.4                │
│  snet-pg             10.40.2.0/28    psql-tokenfactory (delegated)         │
│  snet-pe             10.40.3.0/24    private endpoints × 6                 │
│  snet-dnsr-inbound   10.40.4.0/28    dnsr inbound  10.40.4.4  (delegated)  │
│  snet-aks            10.40.8.0/21    aks-tokenfactory — 6 nodes            │
│                                      pods: 10.244.0.0/16 (CNI Overlay)     │
└────────────────────────────────────────────────────────────────────────────┘
          │                 │               │              │
   pe-kv  pe-acr  pe-eh  pe-redis  pe-blob-archive  pe-blob-tfstate
          │                 │               │              │
   kv-tokenfactory   acrtokenfactory   redis-tokenfactory  storage ×2
                     ehns-tokenfactory
```

### 2.2 Resource Inventory

72 `resource` blocks, 2 `data` sources, 1 `output`, 22 variables. With `for_each`
expansion these become **89 addresses in state**, which
reconciles as:

| | Count |
|---|---|
| `resource` blocks as written | 72 |
| `azurerm_eventhub.this` — `for_each` over 7 event hubs | +6 |
| `azurerm_subnet.this` — `for_each` over 6 subnets | +5 |
| `azurerm_postgresql_flexible_server_database.pg_db` — `for_each` over 5 databases | +4 |
| **Managed resources in Azure** | **87** |
| `data` sources, which `terraform state list` also prints | +2 |
| **Total state entries** | **89** |

The distinction matters when reading the figure elsewhere: 87 things exist in
Azure; the other two are data reads re-evaluated on every plan.

| Service | Resource | Key configuration |
|---|---|---|
| Resource group | `rg-tokenfactory` | UAE North |
| Network | `vnet-tokenfactory` | `10.40.0.0/16`, `dns_servers = [10.40.4.4]` |
| AKS | `aks-tokenfactory` | 1.35.7, **private**, SKU tier Standard, CNI Overlay + Cilium dataplane & policy, `outbound_type = userDefinedRouting`, OIDC issuer + workload identity on, Azure Policy on, image cleaner 168h |
| AKS system pool | `agentpool` | 3 × `Standard_D4as_v5` (4 vCPU/16 GB), zones 1-2-3, `only_critical_addons_enabled` |
| AKS user pool | `userpool` | 3 × `Standard_D8as_v5` (8 vCPU/32 GB), zones 1-2-3 |
| PostgreSQL | `psql-tokenfactory` | v16, `GP_Standard_D8ds_v5`, 128 GB auto-grow, HA **ZoneRedundant** (zone 1 / standby 2), 30-day geo-redundant backup, private only |
| PG databases | 5 | `tokenfactory`, `keycloak`, `openmeter`, `svix`, `openfga` |
| Redis | `redis-tokenfactory` | Azure Managed Redis, `Balanced_B10` (12 GB), HA Enabled, **`clusteringPolicy = NoCluster`**, port **10000**, Plaintext, NoEviction, no persistence |
| Registry | `acrtokenfactory` | Premium, `admin_enabled = false`, private only |
| Key Vault | `kv-tokenfactory` | standard, **RBAC mode**, purge protection **on**, 90-day soft delete, private only |
| Event Hubs | `ehns-tokenfactory` | Standard (Kafka protocol needs Standard+), capacity 1, private only, 7 hubs |
| Storage (archive) | `sttokenfactoryarch001` | LRS, Deny + `AzureServices` bypass, Event Hubs Capture target |
| Storage (state) | `sttokenfactorytfstate001` | **ZRS**, versioning on, Deny + bypass |
| Firewall | `fw-tokenfactory` | `AZFW_VNet` Standard, 1 policy, 1 rule collection group |
| DNS resolver | `dnsr-tokenfactory` | inbound endpoint, **static** `10.40.4.4` |
| VPN | `vgw-tokenfactory` | `VpnGw1AZ`, RouteBased, P2S `172.16.0.0/24`, certificate auth |
| Bastion | `bas-tokenfactory` | Standard, copy-paste + native tunneling enabled |
| Jumphost | `vm-jumphost-tokenfactory` | `Standard_D2as_v5`, Ubuntu 22.04 LTS gen2, 30 GB Standard_LRS, Entra SSH extension |
| Private DNS zones | 6 | vault, acr, postgres, redis, blob, servicebus |
| NSGs | 4 | `nsg-pg`, `nsg-pe`, `nsg-aks`, `nsg-jumphost` |

### 2.3 Private DNS Zones

| Zone | For |
|---|---|
| `privatelink.vaultcore.azure.net` | Key Vault |
| `privatelink.azurecr.io` | ACR |
| `privatelink.postgres.database.azure.com` | PostgreSQL |
| `privatelink.redis.azure.net` | Azure Managed Redis |
| `privatelink.blob.core.windows.net` | both storage accounts |
| `privatelink.servicebus.windows.net` | Event Hubs |

Do not substitute `privatelink.redisenterprise.cache.azure.net` — it looks
correct because it matches the ARM type, but it belongs to the older Azure Cache
for Redis Enterprise product, and the private endpoint would register in a zone
no client queries.

### 2.4 Subnets

| Subnet | Prefix | Usable | Delegation | NSG | UDR |
|---|---|---:|---|---|---|
| `GatewaySubnet` | `10.40.0.0/27` | 27 | — | — (not permitted) | — |
| `AzureBastionSubnet` | `10.40.0.64/26` | 59 | — | — | — |
| `AzureFirewallSubnet` | `10.40.0.128/26` | 59 | — | — | — |
| `snet-mgmt` | `10.40.1.0/27` | 27 | — | `nsg-jumphost` (on NIC) | ✅ |
| `snet-pg` | `10.40.2.0/28` | 11 | `Microsoft.DBforPostgreSQL/flexibleServers` | `nsg-pg` | ✅ |
| `snet-pe` | `10.40.3.0/24` | 251 | — | `nsg-pe` | — |
| `snet-dnsr-inbound` | `10.40.4.0/28` | 11 | `Microsoft.Network/dnsResolvers` | — | — |
| `snet-aks` | `10.40.8.0/21` | 2043 | — | `nsg-aks` | ✅ |

The three delegated/platform subnets must be dedicated — nothing else may live
in them. Azure reserves the first four and last address of every subnet, which is
why the resolver's static IP is `.4`, the first usable address in its `/28`.

### 2.5 Routing

`rt-tokenfactory` sends `0.0.0.0/0` to `VirtualAppliance` at the firewall's
private IP, associated to `snet-aks`, `snet-mgmt` and `snet-pg`. AKS is created
with `outbound_type = userDefinedRouting`, which means **the cluster has no
outbound internet path of its own** — if the route table or a firewall rule is
wrong, the symptom is nodes that never register or images that never pull.

### 2.6 Network Security Groups

All three subnet NSGs follow the same shape: specific allows, then
`Deny-VNet-Inbound` at priority 4096.

| NSG | Allows inbound |
|---|---|
| `nsg-pg` | 5432 from `snet-aks` (100), 5432 from `snet-mgmt` (110) |
| `nsg-pe` | all ports from `snet-aks` (100) and `snet-mgmt` (110) |
| `nsg-aks` | node-to-node from `10.40.8.0/21` **and `10.244.0.0/16`** (100), all from `snet-mgmt` (110), **`AzureLoadBalancer` (120)** |
| `nsg-jumphost` | 22 from `10.40.0.64/26` (AzureBastionSubnet) only |

Two rules on `nsg-aks` are the ones that break things when missing:

- **The pod CIDR in `Allow-Node-To-Node`.** With CNI Overlay, pod traffic
  carries a `10.244.0.0/16` source address that is not part of the VNet, so a
  rule listing only the node subnet silently drops pod-to-pod traffic.
- **`Allow-AzureLoadBalancer` at priority 120.** Azure's default
  `AllowAzureLoadBalancerInBound` sits at 65001, *below* the explicit
  `Deny-VNet-Inbound` at 4096, so the deny wins and every load-balancer health
  probe fails. The explicit rule restores it above the deny.

### 2.7 DNS

```
VPN client ──► vnet.dns_servers = 10.40.4.4 ──► dnsr inbound endpoint
                                                        │
                                          VNet-linked private DNS zones
```

`azurerm_virtual_network_gateway`'s `vpn_client_configuration` block **has no
`dns_servers` argument** (schema-verified), so the VNet's own `dns_servers` is
the only way to give P2S clients private DNS.

That field used to be `168.63.129.16`, Azure's platform resolver, **and that was
a bug**. That address is reachable only from inside an Azure VM, never over a P2S
tunnel. Pushed to a client, every DNS query on the laptop went there and timed
out — presenting as "the internet dies the moment I connect to the VPN".

The resolver's IP is **static by design**. A dynamic one would force
`azurerm_virtual_network.dns_servers` to reference the resolver's output, creating
a `vnet → subnet → resolver → inbound → vnet` cycle. Both sides reading the same
variable breaks the cycle.

### 2.8 Egress Allow-List

`firewall.tf` is the largest file in the repository: one rule collection group,
`aks-bootstrap`, holding 9 application rule collections and 2 network rule
collections.

| Priority | Collection | Source | Purpose |
|---|---|---|---|
| 100 | `aks-bootstrap-apps` | AKS | `AzureKubernetesService` FQDN tag |
| 101 | `aks-bootstrap-network` | AKS | NTP/123 UDP, `AzureCloud.uaenorth` 443+9000 TCP |
| 102 | `jumphost-package-installs` | mgmt | pypi, helm, kubectl, hashicorp, postgres, microsoft |
| 103 | `jumphost-azure-control-plane` | mgmt | `login.microsoftonline.com`, `management.azure.com`, `graph.microsoft.com` |
| 104 | `jumphost-apt-mirror` | mgmt | Ubuntu archive |
| 105 | `jumphost-azure-monitor-agent` | mgmt | AMA control + ingestion |
| 106 | `aks-container-registries` | AKS | public registries incl. ghcr blob backend |
| 107 | `aks-app-egress` | AKS | identity providers, Resend, Stripe |
| 108 | `aks-cloudflare-tunnel` | AKS | **UDP/TCP 7844** to two Cloudflare `/24`s |
| 109 | `aks-cloudflare-api` | AKS | cloudflared self-update, Access JWT |
| 110 | `aks-dockerhub-init-images` | AKS | three hardcoded initContainer images |
| 111 | `github-and-maven` | mgmt + AKS | git, GeoIP release download, Maven Central |

Five of these carry non-obvious reasoning worth repeating:

- **106 is usually unnecessary.** Images are brought in with `az acr import`,
  which runs inside ACR's own service and never traverses this firewall; AKS then
  pulls from ACR over a private endpoint. These rules matter only for a direct
  pull from a public registry.
- **110 exists because three initContainer images are hardcoded in chart
  templates**, bypassing `global.imageRegistry`: `busybox:1.36` (minio),
  `busybox:1.36@sha256:…` (tf-auth wait-for-migrate — digest-pinned, no values
  override at all), `curlimages/curl:8.11.1` (tf-ledger geoip-fetch). A Docker
  Hub pull needs **three** hosts — `registry-1.docker.io` (manifest),
  `auth.docker.io` (token), `production.cloudflare.docker.com` (layer blobs).
  Opening only the first gets the manifest and then stalls in
  `ImagePullBackOff` with no useful error.
- **108 must be a network rule, not an application rule.** `cloudflared`
  connects on port **7844** (QUIC over UDP, HTTP/2 over TCP as fallback), and
  Azure Firewall application rules handle HTTP/HTTPS only. It uses IP ranges
  rather than FQDNs because FQDNs in network rules require DNS proxy on the
  policy, which is off — and enabling it introduces its own failure mode where
  client and firewall resolve different addresses.
- **111 includes `repo1.maven.org` for AKS** because Keycloak's providers
  initContainer pulls the 2FA email authenticator jar from there. Without it the
  jar is missing, Keycloak starts without the provider, and **tf-auth's bootstrap
  fails closed and crashloops**.
- **105's `ods.opinsights` entry is a wildcard** because the ingestion endpoint is
  workspace-specific (the workspace ID is the hostname), so changing workspaces
  will not break the rule.

> **`aks-app-egress` (107) is explicitly marked unconfirmed in the code.** Its
> FQDN list was derived from which services production uses, not from the Xailon
> application's own configuration. A missing endpoint is blocked **silently** and
> surfaces as an application timeout. The deny logs that would diagnose it come
> from `AzureDiagnostics | where Category == "AZFWApplicationRule"` — which
> **requires `logging.tf.disabled` to be enabled first**.

---

## 3. Development

### 3.1 Repository Structure

21 configuration files, flat layout — **no modules**. The environment is a
singleton, so the indirection would buy nothing.

| File | Lines | Contents |
|---|---:|---|
| `main.tf` | 8 | resource group |
| `providers.tf` | 35 | **backend block** + provider version pins |
| `backend.tf` | 59 | state storage account, container, its private endpoint |
| `variables.tf` | 113 | 22 variables, all defaulted except `p2s_root_cert_data` |
| `terraform.tfvars` | — | `p2s_root_cert_data` only (a **public** X.509 cert) |
| `network.tf` | 50 | VNet, 6 `for_each` subnets, delegated `snet-pg` |
| `nsg.tf` | 198 | 3 NSGs + subnet associations |
| `route_table.tf` | 31 | UDR to firewall, 3 associations |
| `firewall.tf` | 468 | firewall, policy, **the entire egress allow-list** |
| `dns_resolver.tf` | 65 | delegated subnet, resolver, inbound endpoint |
| `aks.tf` | 99 | cluster, user node pool, `AcrPull` grant |
| `acr.tf` | 48 | registry + private endpoint |
| `keyvault.tf` | 74 | vault + PE + 2 role assignments |
| `postgresql.tf` | 75 | server, 5 databases, admin password → Key Vault |
| `redis.tf` | 329 | `azapi` cluster + database, key read, PE, password → Key Vault |
| `eventhub.tf` | 173 | namespace, 7 hubs, capture storage, RBAC, PE, auth rule |
| `storage_pe.tf` | 57 | blob zone + archive account private endpoint |
| `jumphost.tf` | 116 | NIC, NSG, VM, Entra SSH extension |
| `bastion.tf` | 31 | public IP + bastion host |
| `vpn_gateway.tf` | 45 | public IP + gateway + P2S config |
| `logging.tf.disabled` | 364 | Log Analytics, diagnostic settings, AMA — **off** |
| `grafana.tf.disabled` | 134 | Azure Monitor workspace, Managed Prometheus, Managed Grafana — **off** |

Also committed: `.gitignore`, `.terraform.lock.hcl` (provider checksum lock) and
`tokenfactory-terraform-file-reference.xlsx`.

### 3.2 Comment Style

Comments in this repository carry **decision records, not descriptions**. The
pattern is: what was tried, what Azure returned verbatim, and why the current
form is the answer. `redis.tf` is the clearest example — the `NoCluster` block
records why the policy was chosen, the ordered migration plan, and the
`terraform plan` output to check for before flipping the policy.

This is deliberate. Several settings in this configuration are indistinguishable
from arbitrary choices until you know what the alternative did, and the comment
is the only place that knowledge lives.

### 3.3 Git / Branching Strategy

**Branch protection is on** for `azure-infra-terraform`: changes must go through
a pull request. Direct pushes to `main` are rejected.

- **`main` is the source of truth.** The jumphost is a checkout of it
  (`git pull`); nothing is edited on the jumphost.
- **Every change is a pull request** against `main`. The `plan` output for the
  change is the artefact a reviewer needs.
- Required-reviewer count, commit-message convention and release tagging are not
  documented here and no tags are cut.

### 3.4 Coding & Configuration Standards

- **Flat layout, no modules** — one file per service. The environment is a
  singleton.
- **Comments are decision records, written in English**.
- **Variables** live in `variables.tf`, all defaulted except `p2s_root_cert_data`.
  `terraform.tfvars` holds only that value, and it is a public certificate.
- **Parked configuration** is renamed `*.tf.disabled`: Terraform ignores it, the
  history is kept, and re-enabling is a `git mv`.
- **Provider versions are pinned** and the lock file is committed.
- **Settings that look redundant are drift suppression** and must not be cleaned
  up. **The settings listed under One-Way Doors are irreversible** and need an explicit
  call-out in the pull request.

---

## 4. Build & CI

### 4.1 Build Process

There is **no build step and no CI pipeline** for this repository. Terraform is
interpreted; the nearest thing to a build artefact is the saved plan
produced on the jumphost and applied unchanged.

This is a gap, not an omission. `terraform plan` needs VNet-internal execution,
so a hosted runner cannot run it. Static checks (formatting, linting,
secret scanning) do not need network access and could run on every pull request.

### 4.2 Validation & Testing

Checks performed today are manual:

- `terraform plan` against the unchanged configuration must print **No changes**.
- HCL parse of all 21 files, including both `.disabled` files.
- Spot-checks of declared versus live attributes.

### 4.3 Security & Vulnerability Scanning

Audited 12 October 2026, clean on all counts: no hardcoded secrets, no private
keys, no state files, no backups or binaries. `.gitignore` covers `.terraform/`,
`*.tfplan`, `*.tfstate*`, `*.pem`, `*.p12`, `*.csr`, `*.key`, `*.srl`,
`cert-base64.txt`, `jumphost_azure_login.txt` and local scratch patterns.

The only IP literals remaining in the configuration are `168.63.129.16` (an Azure
platform constant) and Cloudflare's published tunnel ranges `198.41.192.0/24`
and `198.41.200.0/24`. An operator's personal public IP in a commented
`ip_rules` line was removed.

No automated scanner runs against the repository. **Recommended:** secret
scanning on push, and a policy scanner (`checkov` or `trivy config`) on pull
requests, which would also catch a public-access regression.

### 4.4 Provider Supply Chain

`.terraform.lock.hcl` is committed, so provider versions and checksums are
identical on every machine. No container images are built from this
repository; images are imported into ACR by a manual script.

---

## 5. Deployment & CD

### 5.1 Terraform Runs on the Jumphost — Only

```
operator ──P2S VPN or Bastion──► vm-jumphost (10.40.1.4) ──► terraform
                                        │
                              az CLI 2.90.0 at /usr/local/bin/az
```

Working directory: `azureuser@10.40.1.4:~/terraform_tokenfactory`

This is **not a preference, it is a requirement.** The configuration makes
data-plane calls that only succeed from inside the VNet:

| Resource | Why it needs VNet-internal execution |
|---|---|
| `azurerm_key_vault_secret.pg_admin_password` | Key Vault data plane; the vault has `public_network_access_enabled = false` |
| `azurerm_key_vault_secret.redis_password` | same |
| `azurerm_storage_container.eh_capture` | blob data plane on an account with `default_action = "Deny"` |
| `azurerm_storage_container.tfstate` | same, plus the backend itself reads through `pe-blob-tfstate` |

From outside, these fail with `ForbiddenByConnection` / 403. A storage **IP rule
does not help**: Azure Storage IP rules only match requests from the public
internet, and same-region VNet traffic travels the Azure backbone and never
matches one. A private endpoint is the only correct answer
(see the header comment in `storage_pe.tf`).

Note that `snet-pe`'s NSG admits `snet-aks` and `snet-mgmt` only — **not** the
VPN client range `172.16.0.0/24`. Blob access from a laptop stays closed even on
the VPN. That is intentional.

### 5.2 Standard Change Workflow

```bash
# on the jumphost
cd ~/terraform_tokenfactory
git pull                 # the repo is the source; the jumphost is a checkout
terraform init            # only after a provider change
terraform plan  -out=tf.plan
# review every line, then:
terraform apply tf.plan
```

A plan against the unchanged configuration must print
**`No changes. Your infrastructure matches the configuration.`** Any drift is
investigated before anything else is applied.

### 5.3 Verifying DNS Before an Apply

Private DNS resolution over the VPN is the most common cause of a confusing
failure. Check it first:

```bash
getent hosts kv-tokenfactory.vault.azure.net     # expect 10.40.3.x
getent hosts psql-tokenfactory.postgres.database.azure.com
```

Use `getent` / `getaddrinfo`, **not `dig`**. `dig` queries a DNS server directly
and bypasses the resolver order the OS actually uses, so it can report success
while the application path still fails — and vice versa.

### 5.4 Environment Promotion

| Env | Resource group | Deploy model | Notes |
|---|---|---|---|
| production | `rg-tokenfactory` (UAE North) | Manual `plan` → review → `apply` from the jumphost, after merge to `main` | The only environment |

There is **no dev or staging copy** of this infrastructure, so there is no
promotion path: a change is applied straight to production, and plan review is
the only gate. Globally unique names are hardcoded, so a parallel environment
cannot be stood up without editing them. Application environments are
deployed by Helm and are out of scope.

### 5.5 Rollback

Terraform has no undo. In order of preference:

1. **Revert the commit** through a pull request, pull it on the jumphost, then
   `plan` and `apply`. This works for any in-place change.
2. **Never apply a stale plan.** If the configuration or the state changed after
   `tf.plan` was written, discard it and plan again.
3. **State history.** Blob versioning on the state account keeps earlier state
   versions. That recovers the *state file*; it does not revert resources.
4. **Irreversible settings** cannot be rolled back by reverting the code —
   the resource is destroyed and recreated, or the change is refused.

---

## 6. Compute, Data & Access

### 6.1 PostgreSQL

Single Flexible Server, VNet-injected into the delegated `snet-pg`, with
zone-redundant HA (primary zone 1, standby zone 2), 128 GB auto-grow storage and
30-day geo-redundant backups. Five databases are created by `for_each`.

> **Storage is 128 GB, which provisions 500 IOPS.** Azure couples IOPS to
> allocated storage on Flexible Server. This is the single capacity number most
> likely to bite under load and has not been validated against a load test.

### 6.2 Redis — Why `NoCluster`

This is the most consequential single setting in the repository.

`clusteringPolicy = "NoCluster"` stores data on each node without sharding, so
**no hash slots exist** and multi-key Lua scripts work with the application's
keys unchanged.

`EnterpriseCluster` — the obvious-looking choice — gives a single *endpoint* but
does **not** collapse the key space; it is still sharded. Enterprise policy
permits only a short list of commands across slots, which excludes the scripted
multi-key operations the application relies on.

**Constraints accepted with `NoCluster`:** available at ≤ 25 GB only; the policy
is fixed at create time and can move `NoCluster → clustered` but **not the
reverse** without deleting the database; and it is less performant, because Redis
Enterprise multithreads only when the cache is sharded.

**Planned migration (decided 12 October 2026).** Move to `EnterpriseCluster` once
hash tags ship. The order is not negotiable:

1. Ship hash tags in **tf-ledger and tf-warden together**, with the data
   migration for `tf:credit:used:<org>` (`seed.go:54`).
2. Verify in production on this `NoCluster` instance.
3. **Only then** flip `clusteringPolicy`.

Flipping before the application change breaks the credit scripts immediately.
On the flip, confirm `terraform plan` says **`update in-place`** — if it says
`destroy and then create replacement`, **stop**: the database would be deleted.

Two more settings are deliberate:

- **`port = 10000`**, not 6379 — Azure Managed Redis's standalone endpoint port.
  Clients connect to the cluster hostname, **not** the privatelink hostname,
  which Microsoft's own troubleshooting guide warns against putting in a
  connection string.
- **`evictionPolicy = "NoEviction"`** matches the bundled subchart's deliberate
  `--maxmemory-policy noeviction`. The credit-wallet counters carry no TTL and
  must not be evicted; `VolatileLRU` would evict TTL'd keys first, which here
  means the reservation holds (`tf:{token,credit,projcap}:resv:*`) — a silently
  dropped hold reopens the in-flight overspend window. `NoEviction` fails writes
  loudly instead, and both tf-ledger and tf-warden already have fail-closed
  paths for Redis errors.

**No `zones` block — do not add one back.** ARM rejects it:
`400: Specifying zones for SKU 'Balanced_B20' is not supported. Zone redundancy
is enabled by default for this SKU in regions with zones.` Redundancy is not lost
by omitting it. This is the **opposite** of every other zonal resource here
(public IPs, the VM, disks), where `zones` is required.

**Sizing and scaling.** `Balanced_B10` is 12 GB; Azure reserves ~20 % for
non-cache operations, so usable memory is ~9.6 GB against an application cap of
`maxmemory 3gb`. Scaling works in **both** directions — the authority is
`listSkusForScaling` asked about this specific cluster, which returns **37
options**, from `B0` (0.5 GB) up to 1920 GB, of which 12 are ≤ 25 GB and
therefore keep `NoCluster` available. (Microsoft's architecture page says scaling
down is unsupported and its how-to-scale page says otherwise; the ARM API
settles it.) Right-sizing to `Balanced_B5` (6 GB → ~4.8 GB usable) would clear
the 3 GB cap and cut cost; `B3` would fall below it. `B10` is kept only because
it is running and nothing is under pressure.

Access keys are read with `sensitive_response_export_values`, not the plain
variant, so they land in `sensitive_output` and are redacted in plan/apply and
`terraform output`. They are in state either way.

> **Rotation:** `redis-password` in Key Vault tracks `primaryKey`. Rotating the
> cache key in Azure does **not** update the secret — a `terraform apply`
> re-reads `listKeys` and pushes the new value, after which the CSI syncer
> propagates it.

### 6.3 Event Hubs

Standard tier (Kafka protocol requires Standard or above), capacity 1, private
endpoint only. Seven hubs, 7-day retention (the Standard maximum):

| Hub | Partitions | Capture |
|---|---:|---|
| `om_sys.ingest_events` | 8 | — |
| `om_sys.api_events` | 4 | — |
| `om_sys.balance_worker_events` | 4 | — |
| `om_default_events` | 1 | — |
| `om_sys.balance_worker_dlq` | 1 | ✅ Avro → blob |
| `om_sys.billing_worker_dlq` | 1 | ✅ |
| `om_sys.notification_service_dlq` | 1 | ✅ |

The three DLQ hubs capture to `sttokenfactoryarch001/eventhub-capture`, Avro,
300 s interval, 300 MB size limit, skipping empty archives.

**Capture required a data-plane RBAC grant on the Terraform principal.** Wiring
a hub to a storage account triggers a "linked access check": the *caller* must
hold blob write on that account. `Owner`/`Contributor` **do not satisfy it** —
they are control-plane roles carrying no blob permission. All three DLQ hubs
returned:

```
400: Linked access check failed for capture storage destination …
     doesn't have the required data plane write permissions
```

The fix is `Storage Blob Data Owner` on the archive account, plus
`time_sleep.rbac_propagation`. A managed identity was deliberately not
used — it is only required when the storage account is in a different
subscription, and `identity[0].principal_id` does not resolve at plan time
because the list is still empty in state.

A subtlety that caused confusion during the build: `azurerm_storage_container` is
created with the **account key** (shared access keys are still enabled), while
the linked-access check looks at **RBAC** — which is why the container existed
while the hubs were still failing.

The namespace authorization rule `openmeter-app` grants `listen` + `send`, not
`manage`.

### 6.4 Storage Accounts

| Account | Replication | Public access | Purpose |
|---|---|---|---|
| `sttokenfactorytfstate001` | **ZRS** | `true` + Deny rules | Terraform state; versioning on |
| `sttokenfactoryarch001` | LRS | `true` + Deny rules | Event Hubs Capture |

Both set `public_network_access_enabled = true` with
`network_rules.default_action = "Deny"` and `bypass = ["AzureServices"]`. That
combination is not a contradiction: the account endpoint exists publicly but
rejects everything except Azure trusted services, which is **how Event Hubs
Capture writes** — it uses the trusted-service path, not a private endpoint.
Operator and Terraform access comes through the private endpoints instead.

### 6.5 Container Registry

Premium, `admin_enabled = false`, public access off. AKS pulls through
`pe-acr-tokenfactory` using an `AcrPull` grant on the kubelet identity, so no
image pull secret is needed for ACR images.

Images are imported by a script on the jumphost, which is not version-controlled.

### 6.6 Key Vault

Standard SKU in **RBAC mode** (`enable_rbac_authorization = true`), not access
policies. Purge protection on, 90-day soft delete, public access off,
`network_acls` Deny with `AzureServices` bypass.

Two role assignments:

| Assignment | Principal | Role |
|---|---|---|
| `kv_admin_self` | the Terraform caller | Key Vault Administrator |
| `aks_kv_secrets_user` | AKS Key Vault Secrets Provider addon identity | Key Vault Secrets User |

The addon identity grant is what lets the CSI driver project secrets into pods.
It is `Secrets User` rather than an access policy precisely because the vault is
in RBAC mode. Secret rotation is enabled on the addon with a 2-minute interval.

Terraform manages exactly **two** secrets. Everything else in the vault is seeded
outside Terraform.

### 6.7 AKS

| Setting | Value | Reversible? |
|---|---|---|
| `private_cluster_enabled` | `true` | **No** |
| `network_plugin` / mode | `azure` / `overlay` | **No** |
| `network_data_plane` / `network_policy` | `cilium` / `cilium` | **No** |
| `pod_cidr` | `10.244.0.0/16` | **No** |
| `outbound_type` | `userDefinedRouting` | **No** |
| `kubernetes_version` | `1.35.7` | Yes (upgrade) |
| `sku_tier` | `Standard` | Yes |
| `oidc_issuer_enabled`, `workload_identity_enabled` | `true` | Yes |
| `azure_policy_enabled` | `true` | Yes |
| `image_cleaner_enabled` / interval | `true` / 168 h | Yes |

**Authentication is local accounts + Kubernetes RBAC.** The Azure AD RBAC block
is deliberately not used — access is by exported kubeconfig, not Entra sign-in.
This is the one place where the "no ambient credentials" principle is relaxed,
and it is a known difference from the jumphost's Entra-based SSH.

Auto-upgrade is **commented out** in `aks.tf`, so the cluster and node
images do not update themselves. Upgrades are deliberate actions.

Six nodes total: 3 × D4as_v5 system (critical addons only) + 3 × D8as_v5 user =
36 vCPU / 144 GB, all spread across zones 1, 2 and 3.

### 6.8 Jumphost

`Standard_D2as_v5`, Ubuntu 22.04 LTS gen2, 30 GB Standard_LRS, in `snet-mgmt`.
Its NSG is attached to the **NIC**, not the subnet, and permits SSH only from
`AzureBastionSubnet`.

It carries two independent access paths, on purpose:

1. **Entra ID login** via the `AADSSHLoginForLinux` extension. Each Entra user
   gets their own Linux account named after their UPN, with its own home
   directory, its own `~/.azure` token cache, and their real name in `auth.log`.
   This is what ends the shared-`azureuser` problem. `sudo` comes from the
   **Virtual Machine Administrator Login** role and non-sudo access from
   **Virtual Machine User Login** — note that `Owner` and `Contributor` grant
   **no** VM sign-in at all, because those roles carry no sign-in `dataActions`.
2. **`azureuser` + SSH key** (`~/.ssh/id_rsa_jumphost`) as the break-glass
   route. The extension does not remove local accounts.

The VM's system-assigned managed identity exists **only** because the extension
fails to install without one (exit code 22). It holds **no role assignment**, and
must never be granted one. A managed identity with no roles cannot open any
Azure resource; the extension uses it purely as a device identity.

### 6.9 Bastion & VPN

| | |
|---|---|
| `bas-tokenfactory` | Standard SKU, copy-paste and native tunneling enabled |
| `vgw-tokenfactory` | `VpnGw1AZ` (zone-redundant), RouteBased, no BGP, not active-active |
| P2S client pool | `172.16.0.0/24` |
| P2S auth | Root certificate |

---

## 7. Infrastructure

### 7.1 Infrastructure as Code

Everything beneath the cluster boundary is Terraform in this repository: the
`azurerm` provider for almost all resources, and `azapi` for Azure Managed
Redis. The configuration is flat and has no modules. State is remote. There is
no Terraform Cloud, Atlantis or pipeline wrapper; execution is
described under Deployment & CD. The application layer (Helm) is a different repository.

### 7.2 State & Backend

State is held in the **`azurerm` remote backend**, declared in `providers.tf`:

| Setting | Value |
|---|---|
| Storage account | `sttokenfactorytfstate001` (ZRS, versioning on) |
| Container | `tfstate` |
| Key | `tokenfactory.tfstate` |
| Resource group | `rg-tokenfactory` |

Blob versioning is on, which gives state history without a separate backup job.
There is **no `use_azuread_auth` and no state locking configuration** beyond the
backend's default blob lease — concurrent applies from two machines are not
prevented by anything other than convention.

#### 7.2.1 The Backend Block Lives in Exactly One Place

The `backend` block is inside `providers.tf`. Overwriting that file with a copy
that lacks the block — for example by `scp`-ing from a workstation checkout where
it was removed — makes Terraform fall back to local state. The next
`terraform init` then demands `-reconfigure` or `-migrate-state`, and a wrong
answer there puts all 87 live resources at risk of being treated as new.

**Rule: `providers.tf` is never copied over the jumphost's copy without checking
the backend block survived.**

#### 7.2.2 Chicken-and-Egg: the State Account Is in Its Own State

`backend.tf` declares `azurerm_storage_account.tfstate` — the account that holds
the state file that tracks it. This works for the live environment (it was
created locally, then migrated) but it means a **clean rebuild cannot start from
`terraform init`**: the backend it points at does not exist yet.

A rebuild requires either bootstrapping with local state and migrating, or
creating the account by hand first. **This has never been tested**.

### 7.3 Providers & Version Pinning

| Provider | Constraint | Why |
|---|---|---|
| `hashicorp/azurerm` | `~> 3.0` | 87 live resources are planned against 3.x |
| `Azure/azapi` | `~> 2.0` | Azure Managed Redis — see Why Redis Uses azapi |
| `hashicorp/random` | `~> 3.0` | PostgreSQL admin password |
| `hashicorp/time` | `~> 0.11` | two propagation waits — see The Two Deliberate Waits |

`.terraform.lock.hcl` is committed, so provider versions and checksums are
reproducible across machines.

#### 7.3.1 Why Redis Uses `azapi`, Not `azurerm`

`azurerm_managed_redis` exists **only in azurerm v4.x**. Upgrading the provider
for one resource would have put 87 unrelated resources through v4's breaking
changes, so Redis is declared through `azapi`, which calls the ARM REST API
directly and is version-independent (see the header comment in `redis.tf`).

The `~> 2.0` pin on `azapi` matters: `azapi` embeds a copy of the ARM schemas and
validates `type` locally. The final 1.x release stops at
`Microsoft.Cache/redisEnterprise@2024-06-01-preview`, so a `~> 1.0` pin rejects
the `2025-04-01` GA version at plan time even though ARM accepts it. Dropping to
a 2024 API version instead was ruled out — the `Balanced_*` SKU families postdate
those versions.

When azurerm v4 is adopted (planned separately), the two `azapi_resource` blocks
can be replaced by a native `azurerm_managed_redis` — same Azure resource, same
resource IDs, no data migration.

> **Schema validation is not ARM validation.** `azapi`'s embedded schema accepted
> a `zones` block on the Redis cluster and ARM rejected the PUT with
> `400: Specifying zones for SKU 'Balanced_B20' is not supported`. It likewise
> accepted `clusteringPolicy = "NoCluster"` at `2025-04-01`, which ARM also
> rejected. Treat a clean plan as *well-formed*, not *accepted*.

**The Redis cluster and its database are on different API versions, on purpose.**
The cluster stays on `2025-04-01` and the database (and `listKeys`) uses
`2025-07-01` (`redis_api_version` and `redis_db_api_version` in `redis.tf`).
`NoCluster` is not a valid `clusteringPolicy` before `2025-07-01`; the first
apply failed with `400: NoCluster clustering policy is not supported in API
version 2025-04-01`. Moving the cluster itself to `2025-07-01` is not free:
`properties.publicNetworkAccess` cannot be set on a cluster created under an
older API version. The consequence is that the portal shows **Public network
access = "Not set"** rather than "Disabled" — every other private resource here
sets it explicitly. Traffic still reaches Redis through the private endpoint,
but "not set" is not the same guarantee as "disabled".

#### 7.3.2 The Two Deliberate Waits

Terraform's dependency graph tracks resource creation, not Azure's internal
propagation. Two `time_sleep` resources paper over that:

- `time_sleep.rbac_propagation` — 150 s after the `Storage Blob Data Owner`
  grant. Azure RBAC takes 30 s to a few minutes to apply; without the wait the
  first apply hits the capture linked-access check before the role is live.
- `time_sleep.tfstate_pe_dns` — 60 s after the state account's private endpoint,
  before creating the container, so the private DNS record exists first.

### 7.4 Drift Suppression

Three settings exist purely to keep plans clean, because Azure sets them itself
and the provider defaults differ:

- `upgrade_settings` on both node pools (`max_surge = "10%"`, both timeouts 0) —
  omitting it showed a `- upgrade_settings` removal on every plan.
- `vm_agent_platform_updates_enabled = true` on the jumphost — Azure enabled it,
  the provider defaults to `false`, so every plan showed a `true → false` drift.

These are documented so nobody "cleans them up".

### 7.5 Environment Management

There is one environment. It is described by `variables.tf` (22 variables, all
defaulted except `p2s_root_cert_data`) and there are no workspaces and no
per-environment overlays. Standing up a second copy means editing the hardcoded
globally unique names first.

### 7.6 Configuration Management

- **Infrastructure settings** are Terraform variables with defaults;
  `kubernetes_version` is the one most often changed.
- **Secrets are not configuration here.** Two are generated by Terraform and
  stored in Key Vault; everything else is seeded outside Terraform.
- **Adding or rotating a secret** is a manual, multi-step procedure.

---

## 8. Security

### 8.1 Access Management

- **Operators** enter through the P2S VPN or Bastion and land on the jumphost.
  Each uses their own Entra identity; `azureuser` + key is the
  break-glass path.
- **The jumphost's managed identity holds no role** and must never be granted one.

- **Human Azure RBAC is not managed here.** Each person gets what their own
  assignments grant.
- **Key Vault** uses RBAC mode, not access policies. **AKS** is reached
  through an exported kubeconfig, not Entra sign-in.

### 8.2 Secrets Handling

| Secret | Where it lives |
|---|---|
| `pg-admin-password` | Generated by Terraform → Key Vault (and in state) |
| `redis-password` | Read from ARM `listKeys` → Key Vault (and in state) |
| Everything else | Seeded outside Terraform |

Secrets reach pods through the **AKS Key Vault Secrets Provider (CSI driver)**.
The vault is the source of truth; the Kubernetes Secret is a derived copy.

```
  Terraform apply            operator, az CLI
  (2 secrets)                (everything else)
        │                           │
        └─────────────┬─────────────┘
                      ▼
          ┌───────────────────────────┐
          │   kv-tokenfactory         │   source of truth
          │   Azure Key Vault, RBAC   │   private endpoint 10.40.3.6
          └───────────┬───────────────┘
                      │  addon identity holds Key Vault Secrets User
                      ▼
          ┌───────────────────────────┐
          │ SecretProviderClass       │   declares WHICH secrets to fetch
          │ tokenfactory-kv           │     objects:       what to read
          │                           │     secretObjects: what to write
          └───────────┬───────────────┘
                      │  mounted by Deployment/akv-secret-syncer
                      │  written by the rotation reconciler, ~2 min
                      ▼
          ┌───────────────────────────┐
          │ Secret/tokenfactory-      │   derived copy, lives in etcd
          │   secrets (xailon-tf-prod)│   owned by the syncer's ReplicaSet
          └───────────┬───────────────┘
                      │  secretKeyRef
                      ▼
          ┌───────────────────────────┐
          │ tf-* containers           │   environment variables
          └───────────────────────────┘
```

**Why a Kubernetes Secret has to exist at all.** The services read their
credentials from environment variables, and `secretKeyRef` is the only way to
populate those — it can point at a Secret and nothing else. A pod cannot read
Key Vault directly. So the CSI driver's `secretObjects` feature writes a real
Secret, and the chart is pointed at it by
`global.secrets.existingSecret: "tokenfactory-secrets"`. The alternative the
driver offers — a mount-only tmpfs volume, with no Secret object — is stronger
but unavailable here, because it requires every service to read files instead
of environment variables.

**The SecretProviderClass is the whole list of what the cluster knows to ask
for.** The CSI driver does not scan the vault. A secret added to Key Vault but
not named in that manifest is never fetched, never appears in
`tokenfactory-secrets`, and raises no error — it fails silently. That manifest
is one of the artifacts that is not version-controlled, which makes it
the single most load-bearing unversioned file in the environment.

**Nothing mounts that volume except a placeholder.** The driver only materialises
a Secret while some pod has the SecretProviderClass volume mounted, and the
`tf-*` services consume through `secretKeyRef` rather than a volume. The mount is
therefore held open by **`Deployment/akv-secret-syncer`**, a single-replica pod
that exists for no other purpose. The name is misleading: it does not
reconcile anything, it holds a mount.

> `tokenfactory-secrets` carries an `ownerReference` to that Deployment's
> ReplicaSet, and as of 14 October 2026 there is exactly **one** owner. Kubernetes
> garbage-collects an object once all its owners are gone, so deleting
> `akv-secret-syncer` — or scaling it to zero and pruning its ReplicaSet — deletes
> the Secret, after which every `tf-*` pod fails on its next restart with
> `CreateContainerConfigError`. The cause would not resemble the trigger.
> `rollout restart` is safe, because the new ReplicaSet becomes an owner before
> the old pod terminates. `delete` is not. A second `akv-secret-syncer` with the
> same property runs in the `cloudflare` namespace.

**Writes are asynchronous, reads are not.** Because secret rotation is enabled on
the addon at a 2-minute interval, a newly declared object appears in
`tokenfactory-secrets` on its own, without a restart — verified on 14 October
2026, when a test secret landed roughly a minute after the SecretProviderClass
was edited. What a restart is still required for is the **consumer**: a container
reads its environment once at process start, so a running pod keeps the old value
until it is rolled.

**The two sets are not identical.** The vault holds 27 secrets; the Kubernetes
Secret holds 20 keys. The seven not synced are
`clickhouse-app-password`, `eventhub-connection-string`, `openmeter-db-password`,
`svix-db-password`, `ghcr-pat`, `ghcr-user` and `pg-admin-password` — OpenMeter
runs as a separate release in its own namespace with its own Secret, images come
from ACR rather than ghcr.io, and the Postgres admin password is an operator
credential no application needs.

**What this design does and does not protect.** Key Vault gives custody and
lifecycle: nothing is committed to git, rotation has one authoritative point,
and Terraform can generate a password no human ever reads. What it cannot do is
hide a value from someone already inside the cluster — the copy in
`tokenfactory-secrets` is plaintext in etcd and readable by any principal
holding `get secrets` in the namespace, and a cluster administrator can read a
process environment regardless. That exposure is a Kubernetes RBAC question, not
a secret-store question, and no secret backend changes it.

State contains two plaintext secrets. The compensating controls are: the state
container is private-endpoint-only, `default_action = "Deny"`, versioned, and
reachable only from `snet-aks` / `snet-mgmt`.

### 8.3 Network Security & Closed Surfaces

| Surface | State |
|---|---|
| AKS API server | Private cluster — no public endpoint |
| PostgreSQL | `public_network_access_enabled = false`, VNet-injected |
| Key Vault | Public access off, RBAC mode, Deny ACLs |
| ACR | Public access off, `admin_enabled = false` |
| Event Hubs | Public access off |
| Redis | Private endpoint only |
| Storage × 2 | Deny by default, `AzureServices` bypass only |
| Inbound to VNet | **Nothing.** Bastion and the VPN gateway are the only entry points; application traffic arrives via an outbound-initiated Cloudflare Tunnel |
| Outbound from VNet | Allow-list only, through Azure Firewall |

### 8.4 One-Way Doors

Settings that cannot be changed without destroying and recreating the resource.
Read this section before editing anything in it.

| Setting | File | Consequence of changing |
|---|---|---|
| `private_cluster_enabled` | `aks.tf` | Cluster replacement |
| `network_plugin_mode = overlay` | `aks.tf` | Cluster replacement |
| `network_data_plane = cilium` | `aks.tf` | Cluster replacement |
| `pod_cidr` | `aks.tf` | Cluster replacement |
| `outbound_type` | `aks.tf` | Cluster replacement |
| PostgreSQL HA mode | `postgresql.tf` | Cannot be disabled once zone-redundant |
| Redis `highAvailability` | `redis.tf` | Not reversible; zone redundancy depends on it |
| Redis `clusteringPolicy` | `redis.tf` | `NoCluster → clustered` allowed; **reverse requires deleting the database** |
| Subnet delegations | `network.tf`, `dns_resolver.tf` | Subnet must be empty to change |
| Key Vault `purge_protection_enabled` | `keyvault.tf` | **Cannot be disabled, ever** |
| Globally unique names | ACR, both storage accounts, Key Vault | Name is taken for 90 days after a delete |

### 8.5 Key Vault Purge Protection

`purge_protection_enabled = true` cannot be turned off once set. A `terraform
destroy` therefore **soft-deletes** `kv-tokenfactory` and holds the name for
**90 days**, during which the vault cannot be purged and the name cannot be
reused. A rebuild in that window fails on vault creation.

This is correct for production data safety and it makes a full
destroy-and-recreate cycle impossible within a quarter. Plan around it; do not
discover it during an incident.

### 8.6 Audit & Compliance

No audit tooling is deployed from this repository today.

- Diagnostic settings and Log Analytics live in `logging.tf.disabled`, so
  firewall deny logs and resource access logs are **not collected**.
- Azure's platform Activity Log records control-plane changes without any
  configuration here, but nothing exports or alerts on it.
- State history is available through blob versioning.
- No compliance framework or attestation is documented for this environment.

---

## 9. Monitoring & Operations

### 9.1 Current Monitoring State

Neither observability stack is deployed. Both were disabled for **cost**.
There are no alert rules, dashboards or on-call integration defined in
this repository. The practical effect: a blocked egress FQDN surfaces only as an
application timeout, and nobody is paged.

### 9.2 Re-enabling Observability

```bash
git mv logging.tf.disabled logging.tf      # Log Analytics, diagnostics, AMA
git mv grafana.tf.disabled grafana.tf      # Azure Monitor workspace, Prometheus, Grafana
terraform plan -out=tf.plan
```

Both were disabled for **cost**, not because they do not work. Note that
`logging.tf` is what makes firewall deny logs visible — without it, blocked
egress is invisible. The firewall rules for AMA (priority 105) are
already in place, so enabling logging does not need a firewall change.

For a deployment day, enabling logging first is strongly recommended.

### 9.3 Health Checks

| Check | How | Expect |
|---|---|---|
| Configuration matches live | `terraform plan` on the jumphost | `No changes. Your infrastructure matches the configuration.` |
| Private DNS | `getent hosts kv-tokenfactory.vault.azure.net` | `10.40.3.x` |
| State inventory | `terraform state list` (count the lines) | 89 entries today |
| Secret reached the cluster | byte count of the key in `tokenfactory-secrets` | non-zero |

### 9.4 Daily Operations: Adding or Rotating a Secret

Adding a secret to Key Vault is **step one of three**. On its own it has no
effect on the cluster, and skipping the remaining steps fails silently rather
than loudly. Never edit `tokenfactory-secrets` by hand — it is a derived object,
and the CSI driver rewrites it from the vault on the next pod mount, so a
hand-added key survives until the next rollout and then disappears with no event
to explain it.

**Step 1 — write the value to the vault.** From the jumphost, where
`kv-tokenfactory.vault.azure.net` resolves to the private endpoint:

```bash
az keyvault secret set --vault-name kv-tokenfactory \
  --name <secret-name> --file /path/to/value --output none
```

Reading from a file rather than `--value` keeps the secret out of shell history.
Confirm it landed without printing it:

```bash
az keyvault secret show --vault-name kv-tokenfactory --name <secret-name> \
  --query "{name:name, len:length(value), updated:attributes.updated}" -o yaml
```

**Step 2 — declare it in the SecretProviderClass.** Two entries are required,
and omitting either one is the usual cause of a secret that is "in the vault but
not working":

| Block | Purpose | Symptom if missing |
|---|---|---|
| `spec.parameters.objects` | what the driver fetches from the vault | never fetched at all |
| `spec.secretObjects[].data` | what is written into `tokenfactory-secrets` | fetched to the tmpfs mount, but `secretKeyRef` still fails |

```yaml
spec:
  parameters:
    objects: |
      array:
        - |
          objectName: <secret-name>
          objectType: secret
  secretObjects:
    - secretName: tokenfactory-secrets
      type: Opaque
      data:
        - objectName: <secret-name>
          key: <secret-name>
```

The SecretProviderClass is named **`tokenfactory-kv`**. Take a copy before
editing, because the manifest is not in any repository:

```bash
kubectl -n xailon-tf-prod get secretproviderclass tokenfactory-kv -o yaml \
  > spc-backup-$(date +%Y%m%d-%H%M).yaml
```

Two indentation levels apply, and they differ. `objects:` is a literal block
scalar, so its contents are an unparsed string the API server does not validate:
`- |` sits at 8 spaces and `objectName:` at 10. `secretObjects:` is real YAML:
`- key:` at 4 spaces and `objectName:` at 6. A wrong indent inside the block
scalar is accepted at apply time and fails later at mount time.

**Step 3 — wait, then confirm the Secret.** No restart is needed for this part.
The rotation reconciler polls at a 2-minute interval and writes the new key on
its own:

```bash
kubectl -n xailon-tf-prod get secret tokenfactory-secrets \
  -o jsonpath='{.data.<secret-name>}' | wc -c
```

A non-zero byte count confirms the key reached the Secret. If it is still zero
after three minutes, the `secretObjects` entry from step 2 is missing or
misspelled — and note that the two halves fail differently: an `objects` entry
without its `secretObjects` pair fetches the value to the tmpfs mount and writes
nothing, with no error logged anywhere.

**Step 4 — restart the consumers.** A container reads its environment once at
process start, so a running pod keeps the old value regardless of what the Secret
now holds:

```bash
kubectl -n xailon-tf-prod rollout restart deploy/<name>
```

**Rotating an existing secret is different from adding one.** The key is already
declared, so step 2 is unnecessary and the reconciler updates the Secret in place
with no action at all — but the running pods keep the old value in their
environment until they are restarted, so step 4 still applies. For `redis-password` and
`pg-admin-password` the rotation must originate from Terraform, not the portal:
both are declared resources, and the next `apply` overwrites a manually changed
value.

**Getting the value into the application is a separate question.** A key present
in `tokenfactory-secrets` is still unread unless a container references it. If
the secret maps to an existing chart key under `global.secrets.*`, set that key
in the overlay and upgrade through Helm. If it does not, the chart has no
`secretKeyRef` for it and the value sits unused — that requires a chart change,
not a values change.

### 9.5 What Terraform Does Not Do

After `terraform apply` completes, the environment is **not ready**. These steps
are manual:

| Step | Script / artifact |
|---|---|
| Create per-application PostgreSQL roles | `pg-create-roles.sh` |
| Seed Key Vault secrets beyond the two Terraform manages | `kv-seed-secrets.sh` |
| Create the CSI SecretProviderClass | `secretproviderclass-tokenfactory.yaml` |
| Import container images into ACR | `acr-import-images.sh` |
| Install the nginx ingress controller | F5 `nginx.org` chart, custom values |
| Deploy OpenMeter | `deploy-openmeter-azure.sh` (wraps the repo script, adds `--post-renderer` and `--wait --timeout 20m`) |
| Deploy cloudflared | `cloudflared.yaml` |
| Create the TLS secret | Cloudflare Origin certificate, applied by hand |
| Configure Cloudflare DNS and the tunnel | Cloudflare dashboard |

**This is the largest gap in the repository.** A backup of these scripts exists
(30 files, integrity verified) and they have since been published to a separate
repository, but neither is this repository's version control.

---

## 10. Maintenance & Upgrades

### 10.1 Scaling

| Component | How | Notes |
|---|---|---|
| AKS node count | `node_count` on either pool | In-place |
| AKS node size | `vm_size` | Node pool replacement |
| PostgreSQL compute | `pg_sku_name` | Brief failover; HA keeps it available |
| PostgreSQL storage | `pg_storage_mb` | Grow only — never shrinks. **IOPS scale with it** |
| Redis | `sku.name` | 37 options via `listSkusForScaling`; ~10 min, hostname and keys unchanged, IP may change. Past 25 GB requires leaving `NoCluster` |
| Event Hubs | `capacity` on the namespace | Throughput units |
| Firewall | — | Standard tier, no scaling knob |

### 10.2 Kubernetes Upgrades

Auto-upgrade is off. Upgrading is a deliberate change to `kubernetes_version`
(`variables.tf`, currently `1.35.7`). Check what the region offers first:

```bash
az aks get-versions --location uaenorth -o table
```

Both node pools carry `max_surge = "10%"`, so a node pool upgrade adds one
surge node at a time. The control plane must be upgraded before the node pools,
and AKS permits only one minor version at a step.

### 10.3 Provider & API-Version Maintenance

- **Providers** are pinned `azurerm ~> 3.0` and `azapi ~> 2.0` with a committed
  lock file. Upgrading is a deliberate `terraform init -upgrade`
  followed by a full `plan` that must show no unexpected changes. The planned
  move to azurerm v4 would also replace the two `azapi_resource` Redis blocks.

- **Redis API version — action needed.** The Redis *cluster* is still on
  `2025-04-01`. Microsoft has announced the retirement of Azure Managed
  Redis API versions older than `2025-07-01` on **31 October 2026**; confirm the
  current notice and plan the cluster's move before that date. The
  `publicNetworkAccess` constraint above is what makes this a planned change
  rather than a version bump.

### 10.4 Certificate Renewal

Two certificates are managed outside this repository: the **VPN root
certificate** (its public half is `p2s_root_cert_data`) and the **Cloudflare
Origin certificate** (applied by hand as a TLS secret). Their expiry dates
are not recorded here, and nothing alerts before they lapse. Record both dates
in this section.

### 10.5 Capacity Management

Capacity is changed by editing variables. Redis is larger than the
application's 3 GB cap needs and can be right-sized. PostgreSQL storage
only grows, and its IOPS scale with it.

---

## 11. Troubleshooting & Recovery

### 11.1 Common Issues

| Symptom | Cause | Fix |
|---|---|---|
| `ForbiddenByConnection` / 403 on Key Vault or blob | Run from outside the VNet | Run on the jumphost |
| Name resolves wrongly, or `dig` disagrees with the app | `dig` bypasses the OS resolver order | Use `getent` |
| Application times out calling an external service | FQDN missing from the firewall allow-list | Add a rule; enable logging to see the deny |
| Secret is in the vault but the pod does not see it | SecretProviderClass missing a half, or pods not restarted | See Adding or Rotating a Secret |
| Every plan shows `upgrade_settings` removal or a `true → false` VM agent drift | Drift-suppression setting removed | Restore it |
| `terraform init` demands `-reconfigure` or `-migrate-state` | Backend block lost from `providers.tf` | See The Backend Block Lives in Exactly One Place |
| Redis apply: `NoCluster … not supported in API version 2025-04-01` | Database on the older API version | Database uses `2025-07-01` |
| Redis apply: `Specifying zones for SKU … is not supported` | `zones` block present | Remove it |
| Redis `CROSSSLOT` errors from multi-key scripts | `EnterpriseCluster` still shards keys | `NoCluster` |
| Event Hubs capture fails right after apply | RBAC not yet propagated | `rbac_propagation` wait |
| Entra SSH extension fails with exit code 22 | VM has no managed identity | Identity is required, role-less |

### 11.2 Operational Runbooks

| Task | Section |
|---|---|
| Make a change | Standard Change Workflow |
| Verify DNS before an apply | Verifying DNS Before an Apply |
| Add or rotate a secret | Daily Operations: Adding or Rotating a Secret |
| Scale a component | Scaling |
| Upgrade Kubernetes | Kubernetes Upgrades |
| Re-enable logging and Grafana | Re-enabling Observability |
| Change the Redis clustering policy | Redis — Why NoCluster |

The post-apply scripts are not version-controlled and so are not
runbooks yet.

### 11.3 Incident Management

No incident tooling, on-call rota or alert delivery is defined here.
Incidents so far are recorded as decision comments in the code and under Common Issues.

### 11.4 Backup & Restore

| Data | Protection |
|---|---|
| PostgreSQL | 30-day geo-redundant backup |
| Terraform state | Blob versioning on a ZRS account |
| Event Hubs | 7-day retention, capture to the archive storage account |
| Key Vault | Soft delete and purge protection |
| Post-apply scripts and manifests | A single tarball on a laptop  — **not** adequate |

Redis persistence is not enabled in `redis.tf`. What the application needs to
recover after a cache loss, and whether any restore procedure has been rehearsed
for PostgreSQL, are not documented here.

### 11.5 Disaster Recovery & Rebuild From Scratch

The environment is single-region (UAE North). PostgreSQL, Redis and the VPN
gateway are zone-redundant, which covers a zone failure but not a regional one.
No DR site, RTO or RPO is defined.

**Terraform alone cannot rebuild this environment.** Four blockers:

1. **Key Vault purge protection** holds the name for 90 days after a destroy.

2. **`backend.tf` declares the storage account that holds its own state** — a
   clean `terraform init` has nowhere to initialise from.
3. **The ~8–10 manual steps described under What Terraform Does Not Do** exist
   only as unversioned scripts.
4. **Two private keys are not in the repository** — the VPN root key and the
   Cloudflare Origin key.

Globally unique names (`acrtokenfactory`, both storage accounts,
`kv-tokenfactory`) are also hardcoded rather than suffixed, so a parallel
environment cannot be stood up without editing them.

**A rebuild has never been tested.** Treat the recovery time as unknown until it
has been.

### 11.6 Rollback Procedures

See Rollback, under Deployment & CD. Irreversible settings are listed under One-Way Doors.

---

## 12. Release & Change Management

### 12.1 Release Process

A "release" of this infrastructure is a commit on `main` that has been applied.
There are no version tags or release branches. The flow is: branch → pull request
→ review of the `plan` → merge → `git pull` on the jumphost → `plan` → `apply`
→ post-apply validation. The commit that was verified is recorded in the
version history.

### 12.2 Change Management

- Every change goes through a pull request; direct pushes to `main` are rejected.

- **Attach the `plan` output to the pull request** (*Recommended*) so the reviewer
  sees what will happen, not just the diff.
- Any change touching a setting listed under One-Way Doors must say so in the
  description, because
  it destroys the resource or cannot be reverted.
- Firewall and NSG edits affect every workload's egress and ingress; review them
  against the Network Security Groups and Egress Allow-List sections.
- A `destroy and then create replacement` line in a plan on a stateful resource
  is a **stop**, not a warning.

### 12.3 Post-Apply Validation

1. `terraform plan` prints **No changes. Your infrastructure matches the
   configuration.**
2. `getent hosts` for Key Vault and PostgreSQL returns `10.40.3.x` addresses.

3. `terraform state list` count matches the figure under Accuracy
   Verification; record any change.
4. If a secret changed, confirm it reached `tokenfactory-secrets` and restart
   its consumers.
5. Update the version history.

### 12.4 Accuracy Verification

| Check | Result | When |
|---|---|---|
| `terraform plan` against unchanged config | **No changes. Your infrastructure matches the configuration.** | 12 Oct 2026 |
| `terraform state list` | 89 entries — 87 managed + 2 data | 12 Oct 2026 |
| HCL parse, all 21 files incl. both `.disabled` | Pass | 13 Oct 2026 |
| Declared vs live spot-checks (11) | All matched — PG SKU and HA mode, AKS version and private flag, ACR tier and public access, Key Vault RBAC and private access, Redis `NoCluster` and SKU, VNet space, 8 subnets, 7 event hubs | 12 Oct 2026 |
| Secret / key / state scan of the repository | Clean | 12 Oct 2026 |
| Block counts in the resource inventory | 72 `resource`, 2 `data`, 1 `output`, 22 `variable` | 13 Oct 2026 |

Every figure in this document comes from the configuration files or from a
command run against the live environment. Nothing in it is inferred.

### 12.5 Version History

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 13 Oct 2026 | Sidhant Khanna | First issue. Covers `main` at `417dc58`, 24 files on origin, 89 entries in state. |
| 1.1 | 15 Oct 2026 | Sidhant Khanna | Restructured around the infrastructure lifecycle; added missing sections. |
