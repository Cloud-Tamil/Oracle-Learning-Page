# Phase 3 (Advanced, Manual) – Bastion and Secure Administration

Give authorized administrators a **time-limited, audited, IAM-controlled** path to private ShopSphere resources (OKE worker nodes, database, internal VMs) with **no public IPs and no inbound internet ports**. Then give workloads their own identity (instance principals and workload identity) so no long-lived keys live on servers.

Everything is done **by hand** in the OCI Console and a terminal.

> **Builds on Phases 1 and 2.**
> - Compartments: `shopsphere-network` (bastion, network), `shopsphere-security`, `shopsphere-platform`, `shopsphere-app`.
> - Bastion subnet: `shopsphere-bastion-subnet` (`10.0.5.0/24`, private).
> - Groups: `shopsphere-admins`, `shopsphere-developers`, `shopsphere-readonly`. Alert topic: `shopsphere-alerts`.
> - If you already created `shopsphere-bastion` at the end of Phase 2, **do not create it again**; section 4 verifies and tunes it.

> **How to use this guide:** Work top to bottom, tick each ✅ **Verify**, and record OCIDs in your git-ignored notes file (`docs/phase3-notes.md`).

---

## Table of Contents

1. [Objective and Design](#1-objective-and-design)
2. [Pre-flight Checks](#2-pre-flight-checks)
3. [IAM Policies for Bastion Access](#3-iam-policies-for-bastion-access)
4. [Create or Verify the Bastion](#4-create-or-verify-the-bastion)
5. [Generate SSH Keys](#5-generate-ssh-keys)
6. [Create a Temporary Test Target](#6-create-a-temporary-test-target)
7. [Create Sessions](#7-create-sessions)
8. [Connect and Test](#8-connect-and-test)
9. [Dynamic Groups and Service Policies](#9-dynamic-groups-and-service-policies)
10. [Instance Principals](#10-instance-principals)
11. [Workload Identity for OKE Pods](#11-workload-identity-for-oke-pods)
12. [Audit and Alerting on Bastion Use](#12-audit-and-alerting-on-bastion-use)
13. [Validation Checklist](#13-validation-checklist)
14. [Clean Up the Test Target](#14-clean-up-the-test-target)
15. [Troubleshooting](#15-troubleshooting)
16. [Pitfalls and Hardening Checklist](#16-pitfalls-and-hardening-checklist)

---

## 1. Objective and Design

### 1.1 Goal

| Requirement | How this phase meets it |
|---|---|
| No public SSH/RDP/DB exposure | Targets keep private IPs only; no internet ingress rules |
| Auditable | Every session create/delete appears in OCI Audit |
| Time-bound | Sessions expire (short TTL), cannot be reused |
| Least privilege | Only named groups may create sessions; only allow-listed IPs may connect |
| No static credentials on servers | Instance principals and workload identity |

### 1.2 What OCI Bastion is

A **managed** jump service. There is no VM to patch. You attach it to **one target subnet**, and it creates a private endpoint there. Sessions then reach targets over the VCN, subject to your security rules.

| Session type | Use for | Notes |
|---|---|---|
| **Managed SSH** | Linux compute instances | Needs Oracle Cloud Agent with the **Bastion plugin** enabled on the instance |
| **SSH port forwarding** | Any private IP and port (SSH to a node, database port, Kubernetes API 6443) | Works with any TCP service |
| **Dynamic port forwarding (SOCKS5)** | Several targets through one tunnel | Check that your Console offers it |

### 1.3 Flow

```text
Admin workstation (allow-listed public IP, SSH key)
        │  SSH to the bastion endpoint, session-scoped
        ▼
OCI Bastion (private endpoint in shopsphere-bastion-subnet, 10.0.5.0/24)
        │  private VCN path
        ▼
 ┌──────────────┬──────────────┬──────────────┐
 │ OKE worker   │ Database     │ Internal VM  │
 │ (no public   │ (no public   │ (no public   │
 │  IP)         │  IP)         │  IP)         │
 └──────────────┴──────────────┴──────────────┘
```

### 1.4 Corrections to common explanations

| Common statement | Better picture |
|---|---|
| "Sessions last up to 24 hours" | The Bastion service limits session time to a few hours (up to **3 hours** at the time of writing). Check your bastion's **Maximum session TTL** setting |
| "Workload identity maps a pod to a **dynamic group**" | OKE workload identity uses **IAM policies with `request.principal` conditions** on the pod's namespace, service account and cluster. Dynamic groups are for **instances** |
| "Use `resource.type = 'cluster'` to include OKE nodes" | Worker nodes are **compute instances**, so match them with `instance.compartment.id`. The `cluster` rule matches the cluster resource itself |
| "Bastion rules can reference the bastion NSG" | The managed bastion's **private endpoint is not a VNIC you attach an NSG to**. Use the **bastion subnet CIDR** (`10.0.5.0/24`) as the source on target rules |
| "Port 1521 for the database" | Only for Oracle Database. Phase 2 assumed MySQL (3306). Use the port your database actually listens on |

> **Important follow-up to Phase 2:** Phase 2 allowed the bastion through `shopsphere-bastion-nsg`. That works for a self-built jump VM, but for the **managed** Bastion service you must add CIDR-based rules for `10.0.5.0/24` (section 6.2 and 7.4 show where).

---

## 2. Pre-flight Checks

| Check | How |
|---|---|
| You are in `shopsphere-admins` | Domains → your domain → Groups |
| CLI session valid (optional) | `oci session validate --profile SHOPSPHERE --local` |
| Bastion subnet exists and is **Private** | Networking → VCN → Subnets |
| Service gateway route in private RT | Phase 2, section 7.2 |
| Your current public IP known | `curl ifconfig.me` from the admin machine |
| Audit and `shopsphere-alerts` topic exist | Phase 1, section 11 |

Record your public IP in the notes file as `ADMIN_IP`.

✅ **Verify:** You can open the bastion subnet and see it listed as **Private**.

---

## 3. IAM Policies for Bastion Access

### 3.1 Create the policy

1. Menu → **Identity & Security** → **Policies**; set compartment to **root**.
2. **Create policy**:
   - **Name:** `shopsphere-bastion-policies`
   - **Description:** `Who may manage Bastion and create sessions`
   - **Compartment:** root
3. Use the **manual editor** and paste:

```text
Allow group shopsphere-admins to manage bastion in compartment shopsphere-network
Allow group shopsphere-admins to manage bastion-session in compartment shopsphere-network
Allow group shopsphere-admins to read virtual-network-family in compartment shopsphere-network
Allow group shopsphere-admins to read instance-family in compartment shopsphere-app
Allow group shopsphere-admins to read instance-family in compartment shopsphere-platform
Allow group shopsphere-admins to read instance-agent-plugins in compartment shopsphere-app
Allow group shopsphere-admins to read instance-agent-plugins in compartment shopsphere-platform
Allow group shopsphere-admins to read bastion-work-request in compartment shopsphere-network
```

4. Click **Create**.

> **Group syntax:** If your groups live in a non-default identity domain, write `'Domain'/'shopsphere-admins'`.
>
> **Resource types:** `bastion-family` is the aggregate for all Bastion resources. Using `bastion` and `bastion-session` separately keeps the grant explicit. Policy resource names can change, so check the policy builder's autocomplete if a statement is rejected.

### 3.2 Who should **not** get access

| Group | Bastion access |
|---|---|
| `shopsphere-developers` | None by default. Grant only if they truly need node or database access |
| `shopsphere-readonly` | None (read-only audit role) |
| `shopsphere-dba` | Optional: only `manage bastion-session` plus access to the database compartment, if DBAs need port forwarding |

### 3.3 Optional: separate "session user" role

For admins who can create sessions but cannot change the bastion itself:

```text
Allow group shopsphere-session-users to read bastion in compartment shopsphere-network
Allow group shopsphere-session-users to manage bastion-session in compartment shopsphere-network
```

Create the group first (Phase 1 pattern), then add this policy statement.

✅ **Verify:** The policy is **Active** and lists every statement you pasted.

---

## 4. Create or Verify the Bastion

### 4.1 If you did **not** create it in Phase 2

1. Menu → **Identity & Security** → **Bastion** → **Create bastion**.
2. Compartment: `shopsphere-network`.
3. Enter:

| Field | Value |
|---|---|
| Name | `shopsphere-bastion` |
| Target virtual cloud network | `shopsphere-vcn` |
| Target subnet | `shopsphere-bastion-subnet` |
| CIDR block allowlist | `<ADMIN_IP>/32` |
| Maximum session TTL | **1 hour** (lower than the maximum on purpose) |

4. Add the Phase 1 tags → **Create bastion**.
5. Wait until **Active** and copy the OCID as `BASTION_OCID`.

### 4.2 If you already created it in Phase 2

1. Open `shopsphere-bastion`.
2. Check **Target subnet**, **CIDR allowlist** and **Maximum session TTL**.
3. Use **Edit** to set the TTL to 1 hour and remove any broad allowlist entry.

### 4.3 Allowlist rules

| Use | Value |
|---|---|
| Single admin workstation | `<ADMIN_IP>/32` |
| VPN | The VPN egress range |
| Never | `0.0.0.0/0` |

> If your home IP changes, update the allowlist. Do **not** widen it to "fix" a connection problem.

### 4.4 Remember the endpoint

The SSH endpoint has this form (the Console shows the exact command):

```text
host.bastion.<region>.oci.oraclecloud.com
```

✅ **Verify:** Bastion is **Active**, target subnet is `shopsphere-bastion-subnet`, allowlist has only your IP, TTL is 1 hour.

---

## 5. Generate SSH Keys

Use **separate keys** for the bastion session and for the target. Never share keys between admins.

### 5.1 Bastion session key

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/shopsphere_bastion -C "shopsphere-bastion-session"
chmod 600 ~/.ssh/shopsphere_bastion
```

> Bastion sessions have historically required **RSA** public keys. If your Console accepts other types, Ed25519 is fine too; use whatever the **Add SSH key** dialog accepts.

### 5.2 Target key (used when you later create nodes or VMs)

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/shopsphere_node -C "shopsphere-node-access"
chmod 600 ~/.ssh/shopsphere_node
```

### 5.3 Key hygiene

| Rule | Why |
|---|---|
| Passphrase on every private key | Stolen file alone is useless |
| One key per admin | Individual accountability and revocation |
| Store in a password manager or hardware key | Never in Git, chat or tickets |
| Rotate periodically and on staff changes | Limits long-term exposure |
| `~/.ssh` is `700`, private keys `600` | SSH refuses loose permissions |

Show the public keys (you will paste these, never the private files):

```bash
cat ~/.ssh/shopsphere_bastion.pub
cat ~/.ssh/shopsphere_node.pub
```

✅ **Verify:** Both key pairs exist with `600` permissions on the private files.

---

## 6. Create a Temporary Test Target

There are no real nodes yet, so build a small throw-away VM to prove the bastion path. You delete it in section 14.

### 6.1 Create a temporary NSG

1. Networking → `shopsphere-vcn` → **Network security groups** → **Create**.
2. Name: `shopsphere-test-nsg`, compartment `shopsphere-network`, with Phase 1 tags.
3. Add **one** ingress rule:

| Dir | Source | Protocol | Port | Description |
|---|---|---|---|---|
| Ingress | CIDR `10.0.5.0/24` | TCP | 22 | SSH from Bastion subnet (temporary) |

4. Add egress for the Cloud Agent:

| Dir | Destination | Protocol | Port | Description |
|---|---|---|---|---|
| Egress | Service `All <region> Services In Oracle Services Network` | TCP | 443 | Cloud Agent and updates via service gateway |

### 6.2 Allow the bastion subnet in permanent rules (do this when real targets exist)

For **real** targets, add CIDR-based ingress from the bastion subnet:

| NSG | Source | Port | Purpose |
|---|---|---|---|
| `shopsphere-oke-worker-nsg` | CIDR `10.0.5.0/24` | TCP 22 | SSH to nodes via Bastion |
| `shopsphere-oke-api-nsg` | CIDR `10.0.5.0/24` | TCP 6443 | `kubectl` via Bastion port forwarding |
| `shopsphere-database-nsg` | CIDR `10.0.5.0/24` | TCP 3306 *(or your DB port)* | Admin DB access via Bastion |

> Add only what you need, and remove it when you no longer need it.

### 6.3 Create the test instance

1. Menu → **Compute** → **Instances** → **Create instance**.
2. Enter:

| Field | Value |
|---|---|
| Name | `shopsphere-bastion-test` |
| Compartment | `shopsphere-app` |
| Image | Oracle Linux (latest) |
| Shape | Smallest available (a small flexible shape is enough) |
| Virtual cloud network | `shopsphere-vcn` (compartment `shopsphere-network`) |
| Subnet | `shopsphere-oke-worker-subnet` |
| **Public IPv4 address** | **Do not assign** (the subnet is private) |
| Network security group | `shopsphere-test-nsg` |
| SSH keys | **Paste public key** → paste `shopsphere_node.pub` |

3. Under **Oracle Cloud Agent**, make sure the **Bastion** plugin is **enabled**.
4. Add Phase 1 tags (include `shopsphere.environment = dev`).
5. **Create**; wait for **Running**; note the **private IP** as `TARGET_IP`.

> The Bastion plugin can take a few minutes after boot to show **Running**. Check **Instance details → Oracle Cloud Agent**.

✅ **Verify:** Instance is **Running**, has **no public IP**, and the Bastion plugin shows **Running**.

---

## 7. Create Sessions

Open **Identity & Security → Bastion → `shopsphere-bastion` → Sessions → Create session**.

### 7.1 Managed SSH session (to the test VM)

| Field | Value |
|---|---|
| Session type | **Managed SSH session** |
| Session name | `test-managed-ssh` |
| Username | `opc` (Oracle Linux default) |
| Compute instance | `shopsphere-bastion-test` |
| Add SSH key | **Paste public key** → paste `shopsphere_bastion.pub` |
| Session TTL | `60` minutes or less |

Click **Create session** and wait for **Active**.

### 7.2 SSH port forwarding session (to the same VM, port 22)

| Field | Value |
|---|---|
| Session type | **SSH port forwarding session** |
| Session name | `test-port-forward` |
| Connect to the target by using | **IP address** |
| IP address | `TARGET_IP` |
| Port | `22` |
| Add SSH key | `shopsphere_bastion.pub` |
| Session TTL | `60` minutes or less |

### 7.3 Dynamic port forwarding (SOCKS5), optional

Use it when you must reach several private hosts in one sitting. Create it like 7.2, choosing the **dynamic / SOCKS5** type if offered. Keep the TTL short and prefer single-target port forwarding when possible.

### 7.4 Future target examples

| Target | Session type | Target | Port |
|---|---|---|---|
| OKE worker node | Port forwarding | Node private IP | 22 |
| OKE API (private endpoint) | Port forwarding | API endpoint private IP | 6443 |
| Database | Port forwarding | DB private IP | 3306 (or 1521 for Oracle DB) |

### 7.5 Session rules

- **One session per task.** Delete it when finished.
- Session TTL is fixed at creation and **cannot be extended**. Create a new session instead.
- A session key only authenticates to the bastion; the **target** still needs its own authorization (its `authorized_keys` or Managed SSH).

✅ **Verify:** Both sessions show **Active**.

---

## 8. Connect and Test

### 8.1 Copy the exact command

Open the session → **⋮ (Actions)** → **Copy SSH command** (or **View SSH command**). Use that command rather than retyping it, because it contains the right region endpoint and session OCID.

### 8.2 Managed SSH (shape of the command)

```bash
ssh -i ~/.ssh/shopsphere_node \
  -o ProxyCommand='ssh -i ~/.ssh/shopsphere_bastion -W %h:%p -p 22 <SESSION_OCID>@host.bastion.<region>.oci.oraclecloud.com' \
  -p 22 opc@<TARGET_IP>
```

> Use the command the Console shows. Make sure `-i` points to the **bastion session key** in the ProxyCommand and to the **node key** for the final hop.

### 8.3 Port forwarding (two terminals)

**Terminal 1: open the tunnel** (stays running):

```bash
ssh -i ~/.ssh/shopsphere_bastion -N -L 2222:<TARGET_IP>:22 \
  -p 22 <SESSION_OCID>@host.bastion.<region>.oci.oraclecloud.com
```

**Terminal 2: connect through it:**

```bash
ssh -i ~/.ssh/shopsphere_node -p 2222 opc@127.0.0.1
```

### 8.4 Smoke test on the target

```bash
hostname
ip -4 addr show | grep inet
curl -s -m 5 https://ifconfig.me ; echo
```

Expected: the VM's private IP only. The `curl` shows the **NAT gateway's** public IP, proving egress goes through NAT and the VM has no public address of its own.

### 8.5 Negative tests

| Test | How | Expected |
|---|---|---|
| Direct SSH from the internet | `ssh -i ~/.ssh/shopsphere_node opc@<TARGET_IP>` from your workstation | Times out (private IP, no route) |
| Wrong source IP | Try the bastion command from a different network (for example a phone hotspot) | Rejected by the allowlist |
| Expired session | Wait past the TTL, retry the same command | Connection refused or denied |
| Deleted session | Delete the session, retry | Denied |
| Unauthorized user | Sign in as a user **not** in `shopsphere-admins` and open the Bastion page | No create-session permission |

> Hotspot test: remember to switch back and **do not** add the hotspot IP to the allowlist.

✅ **Verify:** The managed SSH and port-forward connections work, and every negative test fails as expected.

---

## 9. Dynamic Groups and Service Policies

**Dynamic groups** let *instances* (not people) receive permissions. Phase 1 created `shopsphere-app-instances`. Now add node and platform groups.

### 9.1 Create `shopsphere-oke-nodes`

1. Menu → **Identity & Security** → **Domains** → your domain → **Dynamic groups** → **Create dynamic group**.
2. **Name:** `shopsphere-oke-nodes`, **Description:** `OKE worker node instances`.
3. Matching rule (replace the OCIDs):

```text
ANY {
  instance.compartment.id = '<SHOPSPHERE_APP_COMPARTMENT_OCID>',
  instance.compartment.id = '<SHOPSPHERE_PLATFORM_COMPARTMENT_OCID>'
}
```

> Include **every compartment where node pools will be created**. A node pool in a compartment that is not in the rule gets no permissions.

### 9.2 Optional: separate group for the cluster resource

If a feature specifically needs the cluster resource as a principal:

```text
ALL {resource.type = 'cluster', resource.compartment.id = '<SHOPSPHERE_PLATFORM_COMPARTMENT_OCID>'}
```

Create it as `shopsphere-oke-clusters` and grant only what that feature documents.

### 9.3 Service access policy

Create a policy `shopsphere-service-access` in the **root** compartment:

```text
Allow dynamic-group shopsphere-oke-nodes to read repos in compartment shopsphere-platform
Allow dynamic-group shopsphere-oke-nodes to read secret-bundles in compartment shopsphere-security
Allow dynamic-group shopsphere-app-instances to read secret-bundles in compartment shopsphere-security
```

> Grant only what each workload needs. Do **not** give nodes `manage` verbs. If you do not use Vault secrets yet, skip the `secret-bundles` lines and add them in the security phase.

✅ **Verify:** Both dynamic groups show the correct rule, and `shopsphere-service-access` is **Active**.

---

## 10. Instance Principals

**What:** An instance calls OCI APIs **as itself**, using a certificate OCI issues to it, so there is **no API key, config file or password** on the machine.

### 10.1 Prerequisites

- The instance is matched by a dynamic group (section 9).
- A policy grants that dynamic group the access it needs.
- The instance can reach the OCI API endpoints (the **service gateway** route and egress rule you built in Phase 2).

### 10.2 Test from the test VM

Through your bastion session, install the CLI (or use the one preinstalled on Oracle Linux images where available) and run:

```bash
oci os ns get --auth instance_principal
```

Expected output: your Object Storage namespace.

To test a negative case:

```bash
oci iam compartment list --auth instance_principal
```

Expected: **authorization error**, since no policy grants it. That is correct behavior and proves least privilege.

> The test VM lives in `shopsphere-app`, so the Phase 1 dynamic group `shopsphere-app-instances` matches it. To make the namespace call succeed, add a **temporary** statement such as `Allow dynamic-group shopsphere-app-instances to read objectstorage-namespaces in tenancy`, test, then **remove it**.

✅ **Verify:** Authorized calls succeed with `--auth instance_principal`, and unauthorized calls fail.

---

## 11. Workload Identity for OKE Pods

**What:** A pod uses its **Kubernetes service account** to get OCI access, with **no keys in the pod**. IAM policies grant permissions based on the pod's namespace, service account and cluster.

### 11.1 Requirements

| Item | Detail |
|---|---|
| Cluster type | **Enhanced** OKE cluster |
| Supported services | Not every OCI service and SDK version supports it. Check Oracle's current docs before relying on it |
| Pod identity | Defined by `namespace` + `service account` + `cluster OCID` |

> You cannot configure this until the cluster exists, so this section is the **design and policy template**. Apply it in the OKE phase.

### 11.2 Policy template

Create a policy in the **root** compartment when the cluster is ready (replace placeholders):

```text
Allow any-user to read secret-bundles in compartment shopsphere-security
  where all {
    request.principal.type = 'workload',
    request.principal.namespace = '<k8s-namespace>',
    request.principal.service_account = '<k8s-service-account>',
    request.principal.cluster_id = '<CLUSTER_OCID>'
  }
```

### 11.3 Steps when the cluster exists

1. Create a namespace and a service account in Kubernetes (for example `shopsphere` / `catalog-sa`).
2. Run the pod using that service account.
3. Add the policy above with the exact namespace, service account and cluster OCID.
4. Use an OCI SDK version that supports workload identity inside the pod.
5. Test: an authorized call succeeds; the same call from a pod with a different service account fails.

> **No dynamic group is involved** for workload identity.

### 11.4 Decision table

| Need | Use |
|---|---|
| VM or OKE **node** needs OCI access | Instance principal + dynamic group |
| A **pod** needs OCI access | Workload identity + `request.principal` policy |
| Human or CI job with no instance | Session token (humans) or a tightly scoped API key (last resort) |

✅ **Verify (design):** The policy template is saved in `docs/` with placeholders, ready for the OKE phase.

---

## 12. Audit and Alerting on Bastion Use

### 12.1 Look at the audit trail

1. Menu → **Observability & Management** → **Audit**.
2. Compartment: `shopsphere-network` (or root, with sub-compartments included).
3. Set the time range to the last hour and click **Search**.
4. Filter or search for events from the **Bastion** service, such as session creation and deletion.
5. Open an event and confirm it records **who**, **from which IP**, and **when**.

### 12.2 Alert on every session creation

1. Menu → **Observability & Management** → **Events Service** → **Rules** → **Create rule**.
2. Enter:

| Field | Value |
|---|---|
| Display name | `shopsphere-bastion-session-created` |
| Compartment | `shopsphere-security` |
| Condition | **Event Type** |
| Service name | **Bastion** |
| Event type | The **Create session** event(s) (begin and end variants may both appear) |
| Action type | **Notifications** |
| Topic | `shopsphere-alerts` (compartment `shopsphere-security`) |

3. Click **Create rule**.
4. Create and delete a short test session, then check your email.

> The Events rule only sees events in its own compartment tree unless created at a higher level. Create it in a compartment that covers the bastion, or at the root if your policy allows.

✅ **Verify:** Audit shows your test session events, and an email arrives when a session is created.

---

## 13. Validation Checklist

| # | Check | Done |
|---|---|---|
| 1 | `shopsphere-bastion-policies` is **Active** | ☐ |
| 2 | Bastion **Active**, in `shopsphere-bastion-subnet`, allowlist is a single IP, TTL 1 hour | ☐ |
| 3 | Test VM has **no public IP**; Bastion plugin **Running** | ☐ |
| 4 | Managed SSH session works end to end | ☐ |
| 5 | Port-forwarding session works end to end | ☐ |
| 6 | Direct SSH from the internet to the private IP **fails** | ☐ |
| 7 | Connection from a non-allowlisted IP **fails** | ☐ |
| 8 | Session stops working after TTL / deletion | ☐ |
| 9 | Non-admin user **cannot** create sessions | ☐ |
| 10 | Audit shows session create/delete events | ☐ |
| 11 | Events rule emails you on session creation | ☐ |
| 12 | Dynamic groups `shopsphere-oke-nodes` and `shopsphere-app-instances` exist with correct rules | ☐ |
| 13 | Instance principal call succeeds (authorized) and fails (unauthorized) | ☐ |
| 14 | Workload identity policy template documented for the OKE phase | ☐ |
| 15 | Temporary test resources are **deleted** (section 14) | ☐ |

---

## 14. Clean Up the Test Target

Do this as soon as validation passes to avoid cost and exposure.

1. **Delete sessions:** Bastion → Sessions → select each → **Delete**.
2. **Terminate the instance:** Compute → Instances → `shopsphere-bastion-test` → **Terminate** (tick **Permanently delete the attached boot volume**).
3. **Delete the temporary NSG:** `shopsphere-test-nsg` (only after the instance is gone).
4. **Remove temporary policy lines** you added for the instance principal test.
5. Remove temporary entries from the local `~/.ssh/known_hosts` for `127.0.0.1:2222` if you used port forwarding.

> Keep the bastion itself, the policies, the dynamic groups and the Events rule. Those are permanent.

✅ **Verify:** No test instance, volume or NSG remains; Cost Analysis shows no new compute cost.

---

## 15. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Cannot see or create bastion | Missing `manage bastion` or wrong compartment | Re-check `shopsphere-bastion-policies` and group membership |
| Session stays **Creating** or fails | Target unreachable or Bastion plugin not running | Check the plugin status; check security rules allow `10.0.5.0/24` |
| Managed SSH option unavailable for the instance | Bastion plugin disabled, agent not running, or no `read instance-agent-plugins` | Enable the plugin; give the policy; wait a few minutes |
| `Permission denied (publickey)` at the target | Wrong key at the final hop or key not on the target | Use the **node key** for the target and the **session key** for the bastion hop |
| Connection to bastion times out | Your IP is not in the allowlist | Update the allowlist with your current `/32` |
| Port-forward tunnel opens but target refuses | Target security rules block `10.0.5.0/24` | Add the CIDR ingress rule (section 6.2) |
| Session works once, then fails | Session expired (TTL) | Create a new session |
| Cloud Agent cannot reach services | Private RT or NSG egress missing for service gateway | Re-check Phase 2 route 2 and the egress Service rule |
| `NotAuthorizedOrNotFound` with instance principals | Instance not matched by dynamic group, or no policy | Check the rule's compartment OCID and the policy |
| Workload identity call denied | Namespace, service account or cluster OCID mismatch | Match them **exactly** in the policy; confirm the cluster is **Enhanced** |
| Events email never arrives | Subscription unconfirmed, wrong topic, or rule compartment too narrow | Confirm the subscription; recheck the rule |

---

## 16. Pitfalls and Hardening Checklist

### Pitfalls

1. **Allowlist set to `0.0.0.0/0`.** It defeats the purpose of the service.
2. **Long session TTL "for convenience".** Keep it short and create new sessions.
3. **Sharing one SSH key between admins.** You lose accountability.
4. **Opening target rules to the world** while debugging. Fix the real issue instead.
5. **Relying on the bastion NSG** for managed Bastion traffic. Use the subnet CIDR.
6. **Forgetting the compartments in dynamic group rules.** Nodes in unlisted compartments get no access.
7. **Using dynamic groups for pods.** Pods use workload identity policies.
8. **Leaving test VMs running.** They cost money and widen exposure.
9. **Keeping static API keys on servers** when instance principals exist.
10. **No alerting.** An unnoticed session is an unaudited session.

### Hardening checklist

- [ ] Bastion allowlist is a single admin `/32` or a VPN range
- [ ] Maximum session TTL set to 1 hour or less
- [ ] Only `shopsphere-admins` (and an optional session-users group) can create sessions
- [ ] Every admin uses their own passphrase-protected key
- [ ] Target security rules allow only `10.0.5.0/24` on the needed port, and only while needed
- [ ] No public IPs and no internet SSH/database ingress anywhere
- [ ] Audit reviewed regularly; Events rule alerts on session creation
- [ ] Instance principals replace API keys on workloads
- [ ] Workload identity policies written per namespace and service account
- [ ] Policies reviewed every quarter; unused dynamic groups removed
- [ ] Decisions recorded in `docs/adr/0003-secure-administration.md`

---

## Phase 3 Completion Target

✅ Private ShopSphere infrastructure is reachable only through an **auditable, time-bound, IAM-controlled bastion**, with:

- No public exposure of worker nodes, databases or internal VMs
- Short-lived sessions tied to allow-listed IPs and individual SSH keys
- Session creation logged in Audit and announced by email
- Instance principals ready for VMs and nodes, and a workload identity policy plan for pods

*Next: Phase 4, deploying compute / OKE into the Phase 2 subnets and putting this access model to real use.*
