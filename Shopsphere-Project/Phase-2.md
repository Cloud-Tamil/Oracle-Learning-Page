# Phase 2 (Advanced, Manual) – Building OCI Networking from Scratch

Secure, segmented networking for **ShopSphere**, built **entirely by hand** in the OCI Console. You will create a VCN, three gateways, two route tables, five subnets and five Network Security Groups (NSGs) that later host a load balancer, OKE worker nodes, the OKE API endpoint, the database and admin access.

> **Builds on Phase 1.** All resources go in the compartment **`shopsphere-network`** (created in Phase 1), not a `shopsphere-prod` compartment. Apply the tags from Phase 1 (`shopsphere.environment`, `shopsphere.owner`, `shopsphere.cost-center`) to everything you create.

> **How to use this guide:** Work top to bottom, tick each ✅ **Verify** before moving on, and paste every OCID into your git-ignored notes file (`~/shopsphere-oci/docs/phase2-notes.md`).

---

## Table of Contents

1. [Design and IP Plan](#1-design-and-ip-plan)
2. [Pre-flight Checks](#2-pre-flight-checks)
3. [Create the VCN](#3-create-the-vcn)
4. [Create the Internet Gateway](#4-create-the-internet-gateway)
5. [Create the NAT Gateway](#5-create-the-nat-gateway)
6. [Create the Service Gateway](#6-create-the-service-gateway)
7. [Create the Route Tables](#7-create-the-route-tables)
8. [Create Locked-Down Security Lists](#8-create-locked-down-security-lists)
9. [Create the Subnets](#9-create-the-subnets)
10. [Create the NSGs](#10-create-the-nsgs)
11. [Add NSG Rules](#11-add-nsg-rules)
12. [Admin Access: OCI Bastion Service](#12-admin-access-oci-bastion-service)
13. [Manual Validation](#13-manual-validation)
14. [Pitfalls and Hardening Checklist](#15-pitfalls-and-hardening-checklist)

---

## 1. Design and IP Plan

### 1.1 What you are building

| Component | Count | Names |
|---|---|---|
| VCN | 1 | `shopsphere-vcn` (`10.0.0.0/16`) |
| Gateways | 3 | `shopsphere-igw`, `shopsphere-nat`, `shopsphere-sgw` |
| Route tables | 2 | `shopsphere-public-rt`, `shopsphere-private-rt` |
| Subnets | 5 | 1 public (load balancer) + 4 private (OKE workers, OKE API, database, bastion) |
| NSGs | 5 | One per workload type |

### 1.2 IP plan

| Subnet | CIDR | Type | Route table | NSG |
|---|---|---|---|---|
| `shopsphere-lb-subnet` | `10.0.1.0/24` | Public | `shopsphere-public-rt` | `shopsphere-lb-nsg` |
| `shopsphere-oke-worker-subnet` | `10.0.2.0/24` | Private | `shopsphere-private-rt` | `shopsphere-oke-worker-nsg` |
| `shopsphere-oke-api-subnet` | `10.0.3.0/24` | Private | `shopsphere-private-rt` | `shopsphere-oke-api-nsg` |
| `shopsphere-db-subnet` | `10.0.4.0/24` | Private | `shopsphere-private-rt` | `shopsphere-database-nsg` |
| `shopsphere-bastion-subnet` | `10.0.5.0/24` | Private | `shopsphere-private-rt` | `shopsphere-bastion-nsg` |
| *Reserved: pod subnet* | `10.0.6.0/24` | Private | n/a | Only if you choose VCN-native pod networking |
| *Reserved: future* | `10.0.7.0/24` – `10.0.255.0/24` | n/a | n/a | Staging, production, extra tiers |

> **Why `/16` and `/24`s?** A `/16` gives 65,536 addresses. Each `/24` has 256 addresses, but OCI reserves 3 per subnet, leaving 253 usable. Plan larger subnets for the worker and pod ranges if you expect many nodes.

### 1.3 Mental model

```text
Internet
   │
   ▼
[Internet Gateway] ──► shopsphere-lb-subnet (public, 10.0.1.0/24)
                              │  (LB NSG: 443 from internet)
                              ▼
                     shopsphere-oke-worker-subnet (private, 10.0.2.0/24)
                        │                    │
                        ▼                    ▼
              oke-api-subnet         db-subnet (3306 from workers only)
              (private, 10.0.3.0/24)  (private, 10.0.4.0/24)

Private egress:  private subnets ──► NAT Gateway ──► Internet
OCI services:    private subnets ──► Service Gateway ──► Oracle Services Network (no internet)
Admin access:    OCI Bastion service ──► private targets (no public SSH)
```

### 1.4 Corrections to common explanations

| Common statement | Correct picture |
|---|---|
| "The Internet Gateway is stateless" | The gateway itself has no state setting. Statefulness belongs to **security list and NSG rules**, which are stateful by default |
| "An Internet Gateway makes a subnet public" | You also need a **route rule** to it **and** a **public IP** on the resource, **and** rules that allow the traffic |
| "A bastion subnet hosts the bastion" | The **OCI Bastion service** is a managed service attached to a target subnet. A subnet only hosts a jump VM if you deliberately build one |

---

## 2. Pre-flight Checks

| Check | How |
|---|---|
| Region selector shows your chosen region | Top of the Console |
| You are in `shopsphere-admins` (or `shopsphere-network-admins`) | Phase 1, section 4 |
| Compartment `shopsphere-network` is **Active** | Identity & Security → Compartments |
| Tag namespace `shopsphere` exists | Governance & Administration → Tag Namespaces |
| Notes file ready | `docs/phase2-notes.md` is git-ignored |

> Add `docs/phase2-notes.md` to `.gitignore` if it is not already covered.

✅ **Verify:** In the Networking pages, the compartment dropdown lists `shopsphere-network` and you can open it without an authorization error.

---

## 3. Create the VCN

**What it is:** Your isolated virtual network. Every subnet, gateway and route lives inside it.

### Steps

1. Menu (☰) → **Networking** → **Virtual cloud networks**.
2. Set **Compartment** to `shopsphere-network`.
3. Click **Create VCN** (use the **Create VCN** option, **not** the "VCN wizard", so you control every component).
4. Enter:

| Field | Value |
|---|---|
| Name | `shopsphere-vcn` |
| Compartment | `shopsphere-network` |
| IPv4 CIDR blocks | `10.0.0.0/16` |
| Use DNS hostnames in this VCN | ✅ Enabled |
| DNS label | `shopsphere` |
| IPv6 | Leave disabled |

5. Under **Tagging**, add `shopsphere.environment`, `shopsphere.owner` and `shopsphere.cost-center`.
6. Click **Create VCN**.
7. Copy the VCN OCID into your notes as `VCN_OCID`.

> A new VCN gets a **default route table, default security list and default DHCP options**. Leave the defaults untouched and create your own resources instead. Never use the default route table or security list for real subnets.

✅ **Verify:** VCN state is **Available**, CIDR is `10.0.0.0/16`, DNS domain shows `shopsphere.oraclevcn.com`.

<img width="1911" height="755" alt="image" src="https://github.com/user-attachments/assets/54dface7-9ab1-4b83-9d65-0437e574faf0" />

---

## 4. Create the Internet Gateway

**What it is:** Allows resources with **public IPs** in public subnets to reach and be reached from the internet.

### Steps

1. Open `shopsphere-vcn` → **Gateways** → **Internet gateways** → **Create internet gateway**.
2. Enter:
   - **Name:** `shopsphere-igw`
   - **Compartment:** `shopsphere-network`
   - **Enabled:** ✅ (default)
3. Add tags → **Create internet gateway**.

**Used by:** only the public load balancer subnet, through `shopsphere-public-rt`.

✅ **Verify:** Gateway state is **Available** and **Enabled**.

---

## 5. Create the NAT Gateway

**What it is:** Lets private resources start outbound connections (OS updates, pulling images) without being reachable from the internet. It uses a public IP that it manages for you.

### Steps

1. `shopsphere-vcn` → **Gateways** → **NAT gateways** → **Create NAT gateway**.
2. Enter:
   - **Name:** `shopsphere-nat`
   - **Compartment:** `shopsphere-network`
   - **Public IP address:** Ephemeral (default) *(choose Reserved if you need a stable egress IP to allow-list)*
3. Add tags → **Create NAT gateway**.

> NAT gateways allow **egress only**. There is no inbound path from the internet through them.

✅ **Verify:** State is **Available**, and a public IP is shown.

---

## 6. Create the Service Gateway

**What it is:** Private access to Oracle Services Network (OSN) such as Object Storage and the Container Registry, without crossing the internet or using NAT.

### Steps

1. `shopsphere-vcn` → **Gateways** → **Service gateways** → **Create service gateway**.
2. Enter:
   - **Name:** `shopsphere-sgw`
   - **Compartment:** `shopsphere-network`
   - **Services:** select **All `<region>` Services In Oracle Services Network**
3. Add tags → **Create service gateway**.

> Choose **All services**, not the Object-Storage-only option. OKE worker nodes and nodes pulling from OCI Registry need more than Object Storage. You can only attach one OSN service CIDR label per gateway, so the "All services" option avoids having to redo it.

✅ **Verify:** State is **Available** and the service shows **All `<region>` Services In Oracle Services Network**.

---

## 7. Create the Route Tables

A route table matches destination CIDRs to a **target**. Create two and **associate them to subnets at subnet creation**.

### 7.1 Public route table

1. `shopsphere-vcn` → **Routing** → **Route tables** → **Create route table**.
2. Name: `shopsphere-public-rt`, compartment: `shopsphere-network`.
3. **Add route rule**:

| Field | Value |
|---|---|
| Target type | Internet gateway |
| Destination CIDR block | `0.0.0.0/0` |
| Target internet gateway | `shopsphere-igw` |
| Description | Default route to the internet |

4. Create.

### 7.2 Private route table

1. **Create route table** → Name: `shopsphere-private-rt`.
2. Add **two** route rules:

| # | Target type | Destination | Target | Description |
|---|---|---|---|---|
| 1 | NAT gateway | CIDR block `0.0.0.0/0` | `shopsphere-nat` | Outbound internet |
| 2 | Service gateway | **Service:** `All <region> Services In Oracle Services Network` | `shopsphere-sgw` | Private OCI services |

> For rule 2, choose **Destination Type: Service** in the dialog and pick the service from the dropdown. Do **not** type a CIDR by hand. The more specific OSN route takes priority over `0.0.0.0/0`, so OCI service traffic stays private.

3. Create.

✅ **Verify:** `shopsphere-public-rt` has 1 rule and `shopsphere-private-rt` has 2 rules, and no rule points at a gateway that does not exist.

---

## 8. Create Locked-Down Security Lists

Subnets get a security list by default, and the **default** list allows inbound SSH (22) and ICMP from anywhere. Since you control traffic with NSGs, attach your own **minimal** lists.

### 8.1 Private subnets list

1. `shopsphere-vcn` → **Security** → **Security lists** → **Create security list**.
2. Name: `shopsphere-private-sl`.
3. **Ingress rules** (stateful):

| Source CIDR | Protocol | Type/Code | Description |
|---|---|---|---|
| `10.0.0.0/16` | ICMP | 3, 4 | Path MTU discovery inside VCN |

4. **Egress rules:** leave empty (egress is handled by NSG rules; see the note below).
5. Create.

### 8.2 Public subnet list

1. Create another list named `shopsphere-public-sl`.
2. Ingress: same single ICMP `3, 4` rule from `0.0.0.0/0`.
3. Egress: empty.
4. Create.

> **How rules combine:** For a resource, OCI evaluates the **union** of the subnet's security lists **and** the NSGs on its VNIC. Traffic is allowed if **any** rule allows it. Keeping security lists nearly empty means the NSGs are the only meaningful policy. Do not attach the default list.

✅ **Verify:** Both lists exist with exactly one ingress rule each and nothing that opens 22 or 3306.

---

## 9. Create the Subnets

### Common settings for every subnet

| Setting | Value |
|---|---|
| Subnet type | **Regional** (recommended; OKE works across availability domains) |
| DNS resolution | ✅ Use DNS hostnames in this subnet |
| DHCP options | Default DHCP options for `shopsphere-vcn` |
| Compartment | `shopsphere-network` |
| Tags | The three Phase 1 tags |

### 9.1 Create each subnet

`shopsphere-vcn` → **Subnets** → **Create subnet**:

| Name | CIDR | Access | Route table | Security list | DNS label |
|---|---|---|---|---|---|
| `shopsphere-lb-subnet` | `10.0.1.0/24` | **Public** | `shopsphere-public-rt` | `shopsphere-public-sl` | `lb` |
| `shopsphere-oke-worker-subnet` | `10.0.2.0/24` | **Private** | `shopsphere-private-rt` | `shopsphere-private-sl` | `worker` |
| `shopsphere-oke-api-subnet` | `10.0.3.0/24` | **Private** | `shopsphere-private-rt` | `shopsphere-private-sl` | `api` |
| `shopsphere-db-subnet` | `10.0.4.0/24` | **Private** | `shopsphere-private-rt` | `shopsphere-private-sl` | `db` |
| `shopsphere-bastion-subnet` | `10.0.5.0/24` | **Private** | `shopsphere-private-rt` | `shopsphere-private-sl` | `bastion` |

For each one:

1. Enter name, choose **Regional**, enter the CIDR.
2. Select the **route table** from the table above.
3. Choose **Private subnet** or **Public subnet** as listed. For private subnets this sets *Prohibit public IP addresses on VNICs*, which is the guard rail you want.
4. Under **Security lists**, **remove** the default list and select the matching custom list.
5. Create, then copy the OCID into your notes.

### 9.2 Purpose of each subnet

| Subnet | Purpose |
|---|---|
| LB (public) | Internet-facing load balancer for ShopSphere |
| OKE worker (private) | Kubernetes worker nodes |
| OKE API (private) | Private Kubernetes API endpoint |
| DB (private) | Database, never internet exposed |
| Bastion (private) | Target network for the OCI Bastion service, or a jump VM if you build one |

> ⚠️ **OKE note:** This five-subnet layout is a starting point, not a complete OKE network design. Before creating the cluster, decide:
> - **Pod networking:** flannel (overlay, pods use node IPs) or **VCN-native** pod networking (needs a dedicated pod subnet; use the reserved `10.0.6.0/24` and size it for your pods).
> - **Load balancer subnet:** which subnet Kubernetes `LoadBalancer` services use.
> - **API endpoint:** private (recommended here) or public.

✅ **Verify:** Five subnets, all **Available**, only `shopsphere-lb-subnet` shows **Public**.

---

## 10. Create the NSGs

NSGs are virtual firewalls attached to **VNICs**, so you can write rules between **workloads** rather than CIDR ranges.

### Steps

1. `shopsphere-vcn` → **Security** → **Network security groups** → **Create network security group**.
2. Create five groups in `shopsphere-network`, each with the Phase 1 tags:

| NSG | Attach to (later) |
|---|---|
| `shopsphere-lb-nsg` | Load balancer |
| `shopsphere-oke-worker-nsg` | OKE node pool |
| `shopsphere-oke-api-nsg` | OKE API endpoint |
| `shopsphere-database-nsg` | Database |
| `shopsphere-bastion-nsg` | Bastion host / bastion targets |

NSGs do nothing until you add rules **and** attach them to resources in later phases.

✅ **Verify:** Five NSGs exist, each with 0 rules for now.

---

## 11. Add NSG Rules

### How to add a rule

NSG → **Security rules** → **Add rules**. For each rule choose:

- **Direction:** Ingress or Egress
- **Stateless:** leave **unchecked** (stateful)
- **Source/Destination type:** CIDR, **Network security group**, or **Service**
- Protocol, ports and a description

Using **NSG as source/destination** means `Source type = Network security group` and you pick the other NSG from the same VCN.

> **Order tip:** Create all five NSGs first (section 10). Rules that reference another NSG can only be added once it exists.

### 11.1 `shopsphere-lb-nsg`

| Dir | Source / Destination | Protocol | Port(s) | Description |
|---|---|---|---|---|
| Ingress | CIDR `0.0.0.0/0` | TCP | 443 | HTTPS from internet |
| Ingress | CIDR `0.0.0.0/0` | TCP | 80 | HTTP, only to redirect to HTTPS *(optional)* |
| Egress | NSG `shopsphere-oke-worker-nsg` | TCP | 30000-32767 | Kubernetes NodePort range |
| Egress | NSG `shopsphere-oke-worker-nsg` | TCP | 10256 | kube-proxy health check |

### 11.2 `shopsphere-oke-worker-nsg`

| Dir | Source / Destination | Protocol | Port(s) | Description |
|---|---|---|---|---|
| Ingress | NSG `shopsphere-oke-worker-nsg` | All | All | Pod and node traffic between workers |
| Ingress | NSG `shopsphere-oke-api-nsg` | TCP | 10250 | API server to kubelet |
| Ingress | NSG `shopsphere-lb-nsg` | TCP | 30000-32767 | NodePorts from load balancer |
| Ingress | NSG `shopsphere-lb-nsg` | TCP | 10256 | LB health checks |
| Ingress | NSG `shopsphere-bastion-nsg` | TCP | 22 | Admin SSH to nodes *(only if needed)* |
| Egress | NSG `shopsphere-oke-worker-nsg` | All | All | Pod and node traffic between workers |
| Egress | NSG `shopsphere-oke-api-nsg` | TCP | 6443 | Workers to Kubernetes API |
| Egress | NSG `shopsphere-oke-api-nsg` | TCP | 12250 | Workers to OKE control plane |
| Egress | NSG `shopsphere-database-nsg` | TCP | 3306 | Application to database |
| Egress | Service `All <region> Services In Oracle Services Network` | TCP | 443 | OCI services via service gateway |
| Egress | CIDR `0.0.0.0/0` | TCP | 443 | Images and packages via NAT |
| Egress | CIDR `0.0.0.0/0` | ICMP | 3, 4 | Path MTU discovery |

> If your workloads need other outbound ports (for example SMTP or a payment API), add them explicitly rather than opening all egress.

### 11.3 `shopsphere-oke-api-nsg`

| Dir | Source / Destination | Protocol | Port(s) | Description |
|---|---|---|---|---|
| Ingress | NSG `shopsphere-oke-worker-nsg` | TCP | 6443 | Workers to API |
| Ingress | NSG `shopsphere-oke-worker-nsg` | TCP | 12250 | Workers to control plane |
| Ingress | NSG `shopsphere-bastion-nsg` | TCP | 6443 | Admin `kubectl` via bastion |
| Ingress | NSG `shopsphere-oke-worker-nsg` | ICMP | 3, 4 | Path MTU discovery |
| Egress | NSG `shopsphere-oke-worker-nsg` | TCP | 10250 | API to kubelet |
| Egress | NSG `shopsphere-oke-worker-nsg` | ICMP | 3, 4 | Path MTU discovery |
| Egress | Service `All <region> Services In Oracle Services Network` | TCP | 443 | Control plane to OCI services |

### 11.4 `shopsphere-database-nsg`

| Dir | Source / Destination | Protocol | Port(s) | Description |
|---|---|---|---|---|
| Ingress | NSG `shopsphere-oke-worker-nsg` | TCP | **3306** | Application workloads only |
| Ingress | NSG `shopsphere-bastion-nsg` | TCP | 3306 | Admin access via bastion *(optional)* |
| Egress | Service `All <region> Services In Oracle Services Network` | TCP | 443 | Backups to Object Storage |

> 🚫 **Critical:** Never allow port 3306 from `0.0.0.0/0`. The database accepts connections only from application workloads and, optionally, the bastion.
>
> If you later choose an **Oracle Database** instead of MySQL, the listener port is typically **1521**, so adjust the rule and ports consistently in both the database and worker NSGs.

### 11.5 `shopsphere-bastion-nsg`

| Dir | Source / Destination | Protocol | Port(s) | Description |
|---|---|---|---|---|
| Ingress | CIDR `<your-public-IP>/32` | TCP | 22 | Admin SSH, **only if you build a jump VM** |
| Egress | NSG `shopsphere-oke-api-nsg` | TCP | 6443 | `kubectl` to API |
| Egress | NSG `shopsphere-oke-worker-nsg` | TCP | 22 | SSH to worker nodes |
| Egress | NSG `shopsphere-database-nsg` | TCP | 3306 | Admin database access |
| Egress | Service `All <region> Services In Oracle Services Network` | TCP | 443 | OCI CLI and updates |

> The bastion subnet is private. An SSH ingress rule from your public IP only helps if the bastion host has a path from the internet. With the **OCI Bastion service** (section 12) you do not need that rule at all, so leave it out.

✅ **Verify:** Counts are LB 4, worker 12, API 7, database 3, bastion 4 (3 if you omit the SSH rule), and no `0.0.0.0/0` appears on database, API or bastion rules except the HTTPS/ICMP egress rows noted.

---

## 12. Admin Access: OCI Bastion Service

The managed **Bastion service** gives time-limited SSH or port-forwarding sessions to **private** resources without public IPs or an always-on jump VM. You can create the bastion now and use it once targets exist.

### Steps

1. Menu → **Identity & Security** → **Bastion** → **Create bastion**.
2. Enter:

| Field | Value |
|---|---|
| Name | `shopsphere-bastion` |
| Bastion type | Standard |
| Compartment | `shopsphere-network` |
| Target VCN | `shopsphere-vcn` |
| Target subnet | `shopsphere-bastion-subnet` |
| CIDR block allowlist | `<your-public-IP>/32` |

3. Create. Later you create **sessions** (managed SSH or port forwarding) to specific targets.

> Make sure targets are reachable from the bastion's subnet. The bastion service needs the target's **NSG or security list to allow** inbound from the bastion subnet CIDR (`10.0.5.0/24`) or from `shopsphere-bastion-nsg`. For the private OKE API endpoint, use a **port-forwarding session** to port 6443.

✅ **Verify:** Bastion state is **Active** and the allowlist contains only your IP.

---

## 13. Manual Validation

### 13.1 Console checks

| # | Check | Done |
|---|---|---|
| 1 | VCN **Available**, `10.0.0.0/16`, DNS label `shopsphere` | ☐ |
| 2 | Internet gateway **Enabled**, NAT gateway has a public IP, service gateway shows All Services | ☐ |
| 3 | Public RT has 1 rule; private RT has 2 rules (NAT + Service gateway) | ☐ |
| 4 | Five subnets **Available**; only the LB subnet is **Public** | ☐ |
| 5 | Each subnet uses a custom route table and custom security list (not defaults) | ☐ |
| 6 | Five NSGs with the rule counts from section 11 | ☐ |
| 7 | No `0.0.0.0/0` ingress on database, API or bastion NSGs | ☐ |
| 8 | Port 3306 only from worker (and optional bastion) NSG | ☐ |
| 9 | All resources carry the three Phase 1 tags | ☐ |
| 10 | Bastion **Active** with a single-IP allowlist | ☐ |

### 13.2 Optional CLI checks (type one at a time)

First make sure your Phase 1 session is valid:

```bash
oci session validate --profile SHOPSPHERE --local
export OCI_CLI_PROFILE=SHOPSPHERE
export OCI_CLI_AUTH=security_token
```

```bash
oci network vcn list --compartment-id "<shopsphere-network-OCID>" --query 'data[].{name:"display-name",cidr:"cidr-blocks",state:"lifecycle-state"}' --output table
```

```bash
oci network subnet list --compartment-id "<shopsphere-network-OCID>" --vcn-id "<VCN_OCID>" --query 'data[].{name:"display-name",cidr:"cidr-block",public:(not "prohibit-public-ip-on-vnic")}' --output table
```

```bash
oci network nat-gateway list --compartment-id "<shopsphere-network-OCID>" --vcn-id "<VCN_OCID>" --output table
```

Same pattern for the others:

| Resource | Command |
|---|---|
| Internet gateways | `oci network internet-gateway list` |
| Service gateways | `oci network service-gateway list` |
| Route tables | `oci network route-table list` |
| Security lists | `oci network security-list list` |
| NSGs | `oci network nsg list` |
| NSG rules | `oci network nsg rules list --nsg-id <NSG_OCID>` |

> Each list command needs `--compartment-id` and, for most, `--vcn-id`. Run `--help` if a flag is rejected.

### 13.3 Path testing (full test comes in Phase 3)

Use **Networking → Network command center → Network path analyzer** to check reachability once resources exist, for example:

| Path | Expected |
|---|---|
| Internet to LB on 443 | Reachable (after the LB exists) |
| Worker to database on 3306 | Reachable |
| Internet to database on 3306 | **Not** reachable |
| Worker to Object Storage | Reachable through the service gateway |

---

## 14. Pitfalls and Hardening Checklist

### Pitfalls

1. **Using the default route table or security list.** It tempts you to open things "just to test".
2. **Wrong target compartment.** Resources land in the wrong place and policies stop matching.
3. **NSG rules typed as CIDRs for internal traffic.** Reference NSGs so rules follow workloads.
4. **Forgetting egress rules.** NSGs are explicit about egress too when security lists are empty.
5. **Typing the OSN CIDR by hand.** Choose the **Service** destination type instead.
6. **Choosing "Object Storage only" for the service gateway.** OKE needs more.
7. **Assuming a public subnet is exposed automatically.** It still needs a public IP, a route and rules.
8. **Not planning for pods.** The pod subnet decision must be made **before** creating the cluster.
9. **Overlapping CIDRs with on-prem or other VCNs.** Plan ranges now if you may peer later.
10. **Leaving the bastion allowlist at `0.0.0.0/0`.**

### Hardening checklist

- [ ] Only the load balancer subnet is public
- [ ] Private subnets prohibit public IPs on VNICs
- [ ] Custom, minimal security lists attached; defaults detached
- [ ] Every NSG rule has a description and a documented reason
- [ ] Database accepts traffic only from workers (and optionally bastion)
- [ ] No inbound SSH from the internet anywhere; admin access through OCI Bastion
- [ ] VCN flow logs enabled on sensitive subnets *(Observability phase)*
- [ ] Network changes recorded in `docs/adr/0002-network-design.md`
- [ ] Tags applied to every resource

---

## Phase 2 Completion Target

✅ A working, segmented ShopSphere VCN with:

- Public edge (load balancer) isolated from private workloads
- Private egress through NAT, with no public IPs on workers or the database
- Private Oracle service access through the Service Gateway
- Least-privilege NSG rules, with the database port never exposed to the internet
- Clean routing and DNS, verified through the Console and CLI

*Next: Phase 3, deploying compute / OKE into these subnets and running the full connectivity tests.*
