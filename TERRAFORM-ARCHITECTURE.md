# Xailon TokenFactory — Azure Infrastructure (Terraform) Architecture

| | |
|---|---|
| **Document owner** | DevOps Team |
| **Prepared by** | Sidhant Khanna |
| **Platform** | Xailon TokenFactory — Azure (UAE North) |
| **Repository** | `Infinia-Technology/azure-infra-terraform` |
| **Version** | 1.2 |
| **Last updated** | 15 October 2026 |

---

## 1. Introduction

### 1.1 Purpose & Scope

This repository is the **single source of truth for the Azure infrastructure** beneath the Xailon TokenFactory deployment — network, compute, data services, identity plumbing and egress control. It stops at the cluster boundary.

**In scope (Terraform-managed):**

- Resource group, virtual network, eight subnets, route table, NSGs
- Private AKS cluster and both node pools
- Azure Firewall, policy and the full egress allow-list
- PostgreSQL Flexible Server (five databases) and Azure Managed Redis
- Container Registry, Key Vault, Event Hubs, two storage accounts
- DNS Private Resolver, private DNS zones, private endpoints
- Point-to-Site VPN gateway, Bastion, jumphost VM
- Two Key Vault secrets

**Out of scope:**

- **Everything inside Kubernetes** — deployed by Helm from `Infinia-Technology/ai-gateway-platform` (overlay `values-xailon-prod.yaml`).
- **Cloudflare configuration** — tunnel, DNS records, Origin certificate.
- **Entra ID role assignments for human operators** — each person gets what their own Azure RBAC grants.

### 1.2 Design Principles

1. **Private by default.** Every PaaS service has public access disabled and is reached through a private endpoint in `snet-pe`. The AKS API server is private. Inbound application traffic arrives through a Cloudflare Tunnel initiated *outbound* from a pod.
2. **All egress through the firewall.** AKS, management and PostgreSQL subnets route `0.0.0.0/0` to Azure Firewall. A missed FQDN fails as an application timeout, not a visible network error.
3. **Deny-by-default inside the VNet.** Every NSG ends with `Deny-VNet-Inbound` at priority 4096.
4. **No ambient credentials on any VM.** The jumphost's managed identity exists only because the Entra SSH extension requires one, and holds no role. Operators authenticate as themselves.

### 1.3 Infrastructure Lifecycle

**write → validate → plan → apply → verify → operate.** Changes are authored as HCL, reviewed through a pull request, then planned and applied **by hand from the jumphost**. There is no pipeline and no automated apply. Every apply is followed by a plan that must report no changes. There is a single production environment, so nothing is promoted.

This document follows that lifecycle and is written for operators who modify the environment, and for security and management reviewers.

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

72 `resource` blocks expand to 87 managed resources via `for_each` (+6 event hubs, +5 subnets, +4 databases). With 2 `data` sources, `terraform state list` shows **89 entries**.

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

Do not use `privatelink.redisenterprise.cache.azure.net` for Redis — it belongs to the older Azure Cache for Redis Enterprise product, and no client queries it.

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

Delegated and platform subnets must be dedicated. Azure reserves the first four and the last address of every subnet.

### 2.5 Routing

`rt-tokenfactory` sends `0.0.0.0/0` to the firewall's private IP for `snet-aks`, `snet-mgmt` and `snet-pg`. AKS uses `outbound_type = userDefinedRouting`, so the cluster has no outbound path of its own: a wrong route or firewall rule shows up as nodes that never register or images that never pull.

### 2.6 Network Security Groups

| NSG | Allows inbound |
|---|---|
| `nsg-pg` | 5432 from `snet-aks` (100), 5432 from `snet-mgmt` (110) |
| `nsg-pe` | all ports from `snet-aks` (100) and `snet-mgmt` (110) |
| `nsg-aks` | node-to-node from `10.40.8.0/21` **and `10.244.0.0/16`** (100), all from `snet-mgmt` (110), **`AzureLoadBalancer` (120)** |
| `nsg-jumphost` | 22 from `10.40.0.64/26` (AzureBastionSubnet) only |

- The **pod CIDR** must be in `nsg-aks` node-to-node: CNI Overlay pod traffic has a `10.244.0.0/16` source outside the VNet and is silently dropped otherwise.
- **`Allow-AzureLoadBalancer` (120)** must sit above the deny at 4096, or every load-balancer health probe fails.

### 2.7 DNS

VPN client → `vnet.dns_servers = 10.40.4.4` → resolver inbound endpoint → VNet-linked private DNS zones.

- The VPN gateway block has no `dns_servers` argument, so the VNet's own setting is the only way to give P2S clients private DNS.
- It used to point at `168.63.129.16`, which is unreachable over a P2S tunnel; clients lost all DNS on connect.
- The resolver IP is static to avoid a `vnet → subnet → resolver → vnet` dependency cycle.

### 2.8 Egress Allow-List

`firewall.tf` holds one rule collection group (`aks-bootstrap`) with 9 application and 2 network rule collections.

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

- **106** is rarely needed: images are imported with `az acr import` and pulled from ACR over a private endpoint.
- **110** exists because three chart initContainer images bypass `global.imageRegistry`. A Docker Hub pull needs three hosts (`registry-1.docker.io`, `auth.docker.io`, `production.cloudflare.docker.com`); opening fewer stalls in `ImagePullBackOff`.
- **108** must be a network rule: cloudflared uses port 7844 and application rules handle HTTP/HTTPS only.
- **111** includes `repo1.maven.org` because Keycloak's providers initContainer pulls a jar from it; without it tf-auth crashloops.
- **107 is marked unconfirmed in the code.** A missing endpoint is blocked silently; diagnosing it needs firewall deny logs, which require `logging.tf.disabled` to be enabled.

---

## 3. Development

### 3.1 Repository Structure

21 configuration files, flat layout, **no modules** (the environment is a singleton).

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

Also committed: `.gitignore`, `.terraform.lock.hcl` and `tokenfactory-terraform-file-reference.xlsx`.

### 3.2 Comment Style

Comments are **decision records, written in English**: what was tried, what Azure returned, and why the current form is the answer. `redis.tf` is the clearest example.

### 3.3 Git / Branching Strategy

- **Branch protection is on** for `main`: changes go through a pull request and direct pushes are rejected.
- The jumphost is a checkout of `main` (`git pull`); nothing is edited there.
- Reviewer count, commit-message convention and release tagging are not defined.

### 3.4 Coding & Configuration Standards

- One file per service; flat layout, no modules.
- Variables live in `variables.tf`; `terraform.tfvars` holds only a public certificate.
- Parked configuration is renamed `*.tf.disabled`; re-enabling is a `git mv`.
- Provider versions are pinned and the lock file is committed.
- Settings that look redundant are drift suppression — do not clean them up. Settings under One-Way Doors are irreversible and must be called out in the pull request.

---

## 4. Build & CI

### 4.1 Build Process

There is **no build step and no CI pipeline**. The nearest artefact is the saved plan produced on the jumphost and applied unchanged. `terraform plan` needs VNet-internal execution, so a hosted runner cannot run it; static checks could still run on every pull request.

### 4.2 Validation & Testing

Checks are manual: `terraform plan` on unchanged configuration must print **No changes**, HCL parse of all 21 files, and spot-checks of declared versus live attributes.

### 4.3 Security & Vulnerability Scanning

- Repository audited 12 October 2026: no hardcoded secrets, private keys, state files or binaries. `.gitignore` covers `.terraform/`, `*.tfplan`, `*.tfstate*`, keys and certificates, and local scratch files.
- The only IP literals are `168.63.129.16` (Azure platform constant) and Cloudflare's tunnel ranges `198.41.192.0/24` and `198.41.200.0/24`.
- No automated scanner runs against the repository.

### 4.4 Provider Supply Chain

`.terraform.lock.hcl` is committed, so provider versions and checksums are identical on every machine. No container images are built here; images are imported into ACR by a manual script.

---

## 5. Deployment & CD

### 5.1 Terraform Runs on the Jumphost — Only

```
operator ──P2S VPN or Bastion──► vm-jumphost (10.40.1.4) ──► terraform
                                        │
                              az CLI 2.90.0 at /usr/local/bin/az
```

Working directory: `azureuser@10.40.1.4:~/terraform_tokenfactory`

This is a requirement, not a preference. Four resources make data-plane calls that only succeed from inside the VNet: the `pg_admin_password` and `redis_password` Key Vault secrets (vault has public access off) and the `eh_capture` and `tfstate` storage containers (account `default_action = "Deny"`). From outside they fail with `ForbiddenByConnection` / 403. A storage IP rule does not help; a private endpoint is the only fix. `snet-pe`'s NSG deliberately does not admit the VPN client range, so blob access from a laptop stays closed.

### 5.2 Standard Change Workflow

```bash
# on the jumphost
cd ~/terraform_tokenfactory
git pull                 # the repo is the source; the jumphost is a checkout
terraform init           # only after a provider change
terraform plan  -out=tf.plan
# review every line, then:
terraform apply tf.plan
```

A plan against unchanged configuration must print **`No changes. Your infrastructure matches the configuration.`** Investigate any drift before applying anything else.

### 5.3 Verifying DNS Before an Apply

```bash
getent hosts kv-tokenfactory.vault.azure.net     # expect 10.40.3.x
getent hosts psql-tokenfactory.postgres.database.azure.com
```

Use `getent`, not `dig`: `dig` bypasses the OS resolver order and can report success while the application path fails.

### 5.4 Environment Promotion

| Env | Resource group | Deploy model | Notes |
|---|---|---|---|
| production | `rg-tokenfactory` (UAE North) | Manual `plan` → review → `apply` from the jumphost, after merge to `main` | The only environment |

There is no dev or staging copy, so plan review is the only gate. Globally unique names are hardcoded, so a parallel environment cannot be created without editing them.

### 5.5 Rollback

Terraform has no undo.

1. **Revert the commit** through a pull request, pull it on the jumphost, then plan and apply.
2. **Never apply a stale plan** — if configuration or state changed after `tf.plan` was written, plan again.
3. **State history:** blob versioning recovers earlier state files; it does not revert resources.
4. **Irreversible settings** cannot be rolled back by reverting code.

---

## 6. Compute, Data & Access

### 6.1 PostgreSQL

Single Flexible Server, VNet-injected into `snet-pg`, zone-redundant HA, 128 GB auto-grow storage, 30-day geo-redundant backups, five databases. **128 GB provisions only 500 IOPS** (IOPS scale with storage) — the capacity number most likely to bite under load, and not yet load-tested.

### 6.2 Redis — Why `NoCluster`

- **`clusteringPolicy = NoCluster`** stores data without sharding, so multi-key Lua scripts work unchanged. `EnterpriseCluster` gives one endpoint but still shards keys, and the application's scripts failed with `CROSSSLOT`.
- **Constraints:** ≤ 25 GB only; the policy is fixed at create time and can move `NoCluster → clustered` but not back without deleting the database.
- **Planned move to `EnterpriseCluster`** (decided 12 Oct 2026), strictly in order: ship hash tags in tf-ledger and tf-warden together, verify in production on `NoCluster`, then flip the policy. On the flip, the plan must say `update in-place`; if it says `destroy and then create replacement`, stop.
- **`port = 10000`:** clients connect to the cluster hostname, not the privatelink hostname.
- **`evictionPolicy = NoEviction`:** credit counters and reservation holds must never be evicted; writes fail loudly instead.
- **No `zones` block:** ARM rejects it for this SKU, and zone redundancy is on by default.
- **Sizing:** `Balanced_B10` (12 GB, about 9.6 GB usable) against an application cap of 3 GB. Scaling works in both directions; `Balanced_B5` would still clear the cap.
- **Rotation:** `redis-password` in Key Vault tracks `primaryKey`; rotating in Azure does not update it until the next `terraform apply`.

### 6.3 Event Hubs

Standard tier, capacity 1, private endpoint only, 7-day retention. Seven hubs:

| Hub | Partitions | Capture |
|---|---:|---|
| `om_sys.ingest_events` | 8 | — |
| `om_sys.api_events` | 4 | — |
| `om_sys.balance_worker_events` | 4 | — |
| `om_default_events` | 1 | — |
| `om_sys.balance_worker_dlq` | 1 | ✅ Avro → blob |
| `om_sys.billing_worker_dlq` | 1 | ✅ |
| `om_sys.notification_service_dlq` | 1 | ✅ |

- The three DLQ hubs capture to `sttokenfactoryarch001/eventhub-capture` (Avro, 300 s, 300 MB, skip empty).
- Capture needs `Storage Blob Data Owner` on the archive account for the Terraform principal; `Owner`/`Contributor` do not satisfy the linked-access check.
- The `openmeter-app` authorization rule grants `listen` + `send` only.

### 6.4 Storage Accounts

| Account | Replication | Public access | Purpose |
|---|---|---|---|
| `sttokenfactorytfstate001` | **ZRS** | `true` + Deny rules | Terraform state; versioning on |
| `sttokenfactoryarch001` | LRS | `true` + Deny rules | Event Hubs Capture |

Both use `default_action = "Deny"` with an `AzureServices` bypass: Event Hubs Capture writes through the trusted-service path, while operator and Terraform access uses the private endpoints.

### 6.5 Container Registry

Premium, `admin_enabled = false`, public access off. AKS pulls through `pe-acr-tokenfactory` with an `AcrPull` grant on the kubelet identity. Images are imported by an unversioned script on the jumphost.

### 6.6 Key Vault

RBAC mode, purge protection on, 90-day soft delete, public access off. Terraform manages exactly **two** secrets; everything else is seeded outside it.

| Assignment | Principal | Role |
|---|---|---|
| `kv_admin_self` | the Terraform caller | Key Vault Administrator |
| `aks_kv_secrets_user` | AKS Key Vault Secrets Provider addon identity | Key Vault Secrets User |

Secret rotation is enabled on the addon at a 2-minute interval.

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

- Authentication is local accounts plus Kubernetes RBAC via an exported kubeconfig, not Entra sign-in — the one place the "no ambient credentials" principle is relaxed.
- Auto-upgrade is off; six nodes in total (36 vCPU / 144 GB) across zones 1, 2 and 3.

### 6.8 Jumphost

`Standard_D2as_v5`, Ubuntu 22.04, in `snet-mgmt`; its NSG is on the NIC and allows SSH only from `AzureBastionSubnet`. Two access paths:

1. **Entra ID login** (`AADSSHLoginForLinux`): one Linux account per user. `sudo` needs *Virtual Machine Administrator Login*, non-sudo needs *Virtual Machine User Login*; `Owner` and `Contributor` grant no sign-in.
2. **`azureuser` + SSH key** as break-glass.

The system-assigned managed identity exists only because the extension fails to install without one (exit code 22). It holds **no role** and must never be granted one.

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

Everything beneath the cluster boundary is Terraform in this repository: `azurerm` for almost all resources and `azapi` for Azure Managed Redis. The layout is flat, state is remote, and there is no Terraform Cloud or pipeline wrapper. The application layer (Helm) is a separate repository.

### 7.2 State & Backend

| Setting | Value |
|---|---|
| Storage account | `sttokenfactorytfstate001` (ZRS, versioning on) |
| Container | `tfstate` |
| Key | `tokenfactory.tfstate` |
| Resource group | `rg-tokenfactory` |

- Versioning gives state history without a separate backup job. There is no state locking beyond the default blob lease, so concurrent applies are prevented only by convention.
- **The backend block lives only in `providers.tf`.** Copying a version without it onto the jumphost makes Terraform fall back to local state and puts the 87 live resources at risk. Never overwrite that file without checking the block survived.
- **The state account is in its own state** (`backend.tf`), so a clean rebuild cannot start from `terraform init`. This has never been tested.

### 7.3 Providers & Version Pinning

| Provider | Constraint | Why |
|---|---|---|
| `hashicorp/azurerm` | `~> 3.0` | 87 live resources are planned against 3.x |
| `Azure/azapi` | `~> 2.0` | Azure Managed Redis |
| `hashicorp/random` | `~> 3.0` | PostgreSQL admin password |
| `hashicorp/time` | `~> 0.11` | two propagation waits |

#### 7.3.1 Why Redis Uses `azapi`, Not `azurerm`

- `azurerm_managed_redis` exists only in azurerm v4; upgrading for one resource would put 87 others through v4's breaking changes. When v4 is adopted, the two `azapi_resource` blocks can be replaced with no data migration.
- The `~> 2.0` pin matters: 1.x rejects the `2025-04-01` API version at plan time.
- **Schema validation is not ARM validation.** `azapi` accepted a `zones` block and `NoCluster` at `2025-04-01`; ARM rejected both. A clean plan means well-formed, not accepted.
- **The cluster and database use different API versions, on purpose.** The cluster stays on `2025-04-01`; the database and `listKeys` use `2025-07-01`, the first version where `NoCluster` is valid. Moving the cluster would require `publicNetworkAccess`, which cannot be set on a cluster created under an older API version, so the portal shows Public network access **"Not set"** rather than "Disabled". Traffic still goes through the private endpoint, but this is a known gap.

#### 7.3.2 The Two Deliberate Waits

- `time_sleep.rbac_propagation` — 150 s after the `Storage Blob Data Owner` grant, so the capture linked-access check sees the role.
- `time_sleep.tfstate_pe_dns` — 60 s after the state account's private endpoint, so its DNS record exists before the container is created.

### 7.4 Drift Suppression

Two settings exist only to keep plans clean: `upgrade_settings` on both node pools (`max_surge = "10%"`) and `vm_agent_platform_updates_enabled = true` on the jumphost. Azure sets them itself; removing them shows drift on every plan.

### 7.5 Environment Management

One environment, described by `variables.tf` (22 variables, all defaulted except `p2s_root_cert_data`). No workspaces or overlays; a second copy would need the hardcoded globally unique names changed first.

### 7.6 Configuration Management

- Infrastructure settings are Terraform variables; `kubernetes_version` changes most often.
- Two secrets are generated by Terraform into Key Vault; everything else is seeded outside Terraform.

---

## 8. Security

### 8.1 Access Management

- Operators enter through the P2S VPN or Bastion and land on the jumphost with their own Entra identity; `azureuser` + key is break-glass.
- The jumphost's managed identity holds no role.
- Human Azure RBAC is not managed here. Key Vault uses RBAC mode; AKS is reached through an exported kubeconfig.

### 8.2 Secrets Handling

| Secret | Where it lives |
|---|---|
| `pg-admin-password` | Generated by Terraform → Key Vault (and in state) |
| `redis-password` | Read from ARM `listKeys` → Key Vault (and in state) |
| Everything else | Seeded outside Terraform |

- Secrets reach pods through the **AKS Key Vault Secrets Provider (CSI driver)**: Key Vault → `SecretProviderClass tokenfactory-kv` → `Secret/tokenfactory-secrets` (derived copy) → `secretKeyRef` environment variables.
- The SecretProviderClass is the whole list of what the cluster fetches. A secret in the vault but not named there is never synced and raises no error. The manifest is not version-controlled.
- `Deployment/akv-secret-syncer` exists only to hold the CSI mount open. **Do not delete it** — the Secret has that Deployment's ReplicaSet as its only owner, and deleting it makes every `tf-*` pod fail with `CreateContainerConfigError` on restart. `rollout restart` is safe.
- The vault holds 27 secrets and the Kubernetes Secret 20 keys; the other seven are used elsewhere or by no application.
- Key Vault gives custody and lifecycle, not secrecy from someone already in the cluster: the Kubernetes Secret is plaintext in etcd for anyone with `get secrets`. State holds two plaintext secrets, protected by the private-endpoint-only, versioned state container.

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

Settings that cannot change without destroying and recreating the resource.

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

A `terraform destroy` soft-deletes `kv-tokenfactory` and holds the name for **90 days**, so a rebuild in that window fails on vault creation. This is right for production data safety, and it makes a full destroy-and-recreate impossible within a quarter.

### 8.6 Audit & Compliance

- No audit tooling is deployed: diagnostic settings and Log Analytics are in `logging.tf.disabled`, so firewall deny logs and access logs are not collected.
- Azure's platform Activity Log records control-plane changes by default, but nothing exports or alerts on it.
- State history comes from blob versioning. No compliance framework or attestation is documented.

---

## 9. Monitoring & Operations

### 9.1 Current Monitoring State

Neither observability stack is deployed; both were disabled for **cost**. No alert rules, dashboards or on-call integration exist in this repository, so a blocked egress FQDN surfaces only as an application timeout and nobody is paged.

### 9.2 Re-enabling Observability

```bash
git mv logging.tf.disabled logging.tf      # Log Analytics, diagnostics, AMA
git mv grafana.tf.disabled grafana.tf      # Azure Monitor workspace, Prometheus, Grafana
terraform plan -out=tf.plan
```

`logging.tf` is what makes firewall deny logs visible. The AMA firewall rules (priority 105) are already in place. Enabling logging before a deployment day is strongly recommended.

### 9.3 Health Checks

| Check | How | Expect |
|---|---|---|
| Configuration matches live | `terraform plan` on the jumphost | `No changes. Your infrastructure matches the configuration.` |
| Private DNS | `getent hosts kv-tokenfactory.vault.azure.net` | `10.40.3.x` |
| State inventory | `terraform state list` (count the lines) | 89 entries today |
| Secret reached the cluster | byte count of the key in `tokenfactory-secrets` | non-zero |

### 9.4 Daily Operations: Adding or Rotating a Secret

Adding a secret takes four steps; skipping any of them fails silently. Never edit `tokenfactory-secrets` by hand — it is a derived object and is rewritten from the vault.

1. **Write the value to the vault** from the jumphost: `az keyvault secret set --vault-name kv-tokenfactory --name <secret-name> --file <path>` (a file keeps it out of shell history).
2. **Declare it in `SecretProviderClass tokenfactory-kv`** in two places: `spec.parameters.objects` (what to fetch) and `spec.secretObjects[].data` (what to write into `tokenfactory-secrets`). Back the manifest up first with `kubectl get secretproviderclass tokenfactory-kv -o yaml`.
3. **Wait about 2 minutes and confirm** the key reached `tokenfactory-secrets`. If not, one of the two entries in step 2 is missing or misspelled.
4. **Restart the consumers** with `kubectl rollout restart deploy/<name>`; a running pod keeps its old environment.

Rotating an existing secret skips step 2. For `redis-password` and `pg-admin-password`, rotate from Terraform, not the portal, or the next `apply` overwrites the change. A key in the Secret is still unused unless the chart references it through `global.secrets.*`.

### 9.5 What Terraform Does Not Do

After `terraform apply` the environment is **not ready**. These steps are manual:

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

**This is the largest gap in the repository.** The scripts were backed up and published to a separate repository, but they are not under this repository's version control.

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

Auto-upgrade is off. Change `kubernetes_version` in `variables.tf` (currently `1.35.7`) after checking `az aks get-versions --location uaenorth -o table`. Upgrade the control plane before the node pools, one minor version at a time; `max_surge = "10%"` adds one surge node at a time.

### 10.3 Provider & API-Version Maintenance

- Providers are pinned (`azurerm ~> 3.0`, `azapi ~> 2.0`). Upgrade with `terraform init -upgrade` followed by a full plan with no unexpected changes. Moving to azurerm v4 would also replace the two `azapi_resource` Redis blocks.
- **Action needed — Redis API version.** The Redis cluster is still on `2025-04-01`. Microsoft has announced retirement of Azure Managed Redis API versions older than `2025-07-01` on **31 October 2026**; confirm the notice and plan the cluster's move before then. The `publicNetworkAccess` constraint makes this a planned change, not a version bump.

### 10.4 Certificate Renewal

The VPN root certificate (public half in `p2s_root_cert_data`) and the Cloudflare Origin certificate (applied by hand as a TLS secret) are managed outside this repository. Their expiry dates are not recorded and nothing alerts before they lapse; record both here.

### 10.5 Capacity Management

Capacity changes are variable edits (see Scaling). Redis is larger than the application's 3 GB cap needs; PostgreSQL storage only grows and its IOPS scale with it.

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
| `terraform init` demands `-reconfigure` or `-migrate-state` | Backend block lost from `providers.tf` | Restore the block in `providers.tf` |
| Redis apply: `NoCluster … not supported in API version 2025-04-01` | Database on the older API version | Database uses `2025-07-01` |
| Redis apply: `Specifying zones for SKU … is not supported` | `zones` block present | Remove it |
| Redis `CROSSSLOT` errors from multi-key scripts | `EnterpriseCluster` still shards keys | `NoCluster` |
| Event Hubs capture fails right after apply | RBAC not yet propagated | `rbac_propagation` wait |
| Entra SSH extension fails with exit code 22 | VM has no managed identity | Identity is required, role-less |

### 11.2 Operational Runbooks

Procedures live in this document: Standard Change Workflow, Verifying DNS Before an Apply, Adding or Rotating a Secret, Scaling, Kubernetes Upgrades, Re-enabling Observability, and Redis — Why NoCluster (clustering-policy change). The post-apply scripts are not version-controlled, so they are not runbooks yet.

### 11.3 Incident Management

No incident tooling, on-call rota or alert delivery is defined. Past incidents are recorded as decision comments in the code and under Common Issues.

### 11.4 Backup & Restore

| Data | Protection |
|---|---|
| PostgreSQL | 30-day geo-redundant backup |
| Terraform state | Blob versioning on a ZRS account |
| Event Hubs | 7-day retention, capture to the archive storage account |
| Key Vault | Soft delete and purge protection |
| Post-apply scripts and manifests | A single tarball on a laptop — **not** adequate |

Redis persistence is off. A PostgreSQL restore rehearsal and the application's recovery needs after a cache loss are not documented.

### 11.5 Disaster Recovery & Rebuild From Scratch

The environment is single-region (UAE North). PostgreSQL, Redis and the VPN gateway are zone-redundant, which covers a zone failure but not a regional one. No DR site, RTO or RPO is defined.

**Terraform alone cannot rebuild this environment.** Four blockers:

1. Key Vault purge protection holds the name for 90 days after a destroy.
2. `backend.tf` declares the storage account that holds its own state.
3. The ~8–10 manual steps under What Terraform Does Not Do exist only as unversioned scripts.
4. Two private keys (VPN root, Cloudflare Origin) are not in the repository.

Globally unique names are hardcoded, so a parallel environment needs edits first. **A rebuild has never been tested**; treat the recovery time as unknown.

### 11.6 Rollback Procedures

See Rollback under Deployment & CD. Irreversible settings are listed under One-Way Doors.

---

## 12. Release & Change Management

### 12.1 Release Process

A release is a commit on `main` that has been applied; there are no tags or release branches. Flow: branch → pull request → review of the `plan` → merge → `git pull` on the jumphost → `plan` → `apply` → post-apply validation. The verified commit is recorded in the version history.

### 12.2 Change Management

- Every change goes through a pull request; direct pushes to `main` are rejected.
- Any change touching a One-Way Door must say so in the description.
- Firewall and NSG edits affect every workload; review them against the Network Security Groups and Egress Allow-List sections.
- A `destroy and then create replacement` line on a stateful resource is a **stop**, not a warning.

### 12.3 Post-Apply Validation

1. `terraform plan` prints **No changes. Your infrastructure matches the configuration.**
2. `getent hosts` for Key Vault and PostgreSQL returns `10.40.3.x`.
3. `terraform state list` count matches Accuracy Verification; record any change.
4. If a secret changed, confirm it reached `tokenfactory-secrets` and restart its consumers.
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

Every figure in this document comes from the configuration files or from a command run against the live environment.

### 12.5 Version History

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 13 Oct 2026 | Sidhant Khanna | First issue. Covers `main` at `417dc58`, 25 files on origin, 89 entries in state. |
| 1.1 | 15 Oct 2026 | Sidhant Khanna | Restructured around the infrastructure lifecycle; added missing sections. |
| 1.2 | 15 Oct 2026 | Sidhant Khanna | Condensed each subsection to a short summary; exact values kept in tables. |
