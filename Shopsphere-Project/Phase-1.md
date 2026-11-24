# Phase 1 (Advanced, Manual) – OCI Account and Foundation

The advanced Phase 1 for **ShopSphere**, done **entirely by hand** through the OCI Console and a terminal. No automation scripts, loops or CI. Every step is a click path or a single command you type yourself, followed by a check that it worked.

> **How to use this guide:** Work top to bottom. Tick each ✅ **Verify** box before moving on. Keep a notes file (`~/shopsphere-oci/docs/phase1-notes.md`, git-ignored) to paste OCIDs as you create things.

---

## Table of Contents

1. [Target Design](#1-target-design)
2. [Log In and Record Identifiers](#2-log-in-and-record-identifiers)
3. [Create the Compartments](#3-create-the-compartments)
4. [Create IAM Groups](#4-create-iam-groups)
5. [Create IAM Policies](#5-create-iam-policies)
6. [Create a Dynamic Group](#6-create-a-dynamic-group)
7. [Enable and Enforce MFA](#7-enable-and-enforce-mfa)
8. [Install and Configure the OCI CLI](#8-install-and-configure-the-oci-cli)
9. [Create Tags](#9-create-tags)
10. [Create the Budget and Alerts](#10-create-the-budget-and-alerts)
11. [Security Baseline: Audit, Cloud Guard, Notifications](#11-security-baseline-audit-cloud-guard-notifications)
12. [Troubleshooting](#15-troubleshooting)
13. [Pitfalls and Hardening Checklist](#16-pitfalls-and-hardening-checklist)

---

## 1. Target Design

```text
<tenancy root>
└── Shopsphere                  # project boundary (budget, tags, admin policy)
    ├── shopsphere-network      # VCNs, subnets, gateways
    ├── shopsphere-security     # Vault, keys, notifications, state bucket
    ├── shopsphere-platform     # OKE, registries, shared tooling
    ├── shopsphere-app          # compute, containers, load balancers
    └── shopsphere-data         # databases, object storage, backups
```

| Choice | Reason |
|---|---|
| Child compartments per concern | Separate access and blast radius for network, data and app |
| Policies inherited from `Shopsphere` | A statement on the parent covers all children |
| One budget on the parent | Rolls up the whole project's cost |
| Environments handled by tags and Terraform state | Keeps the tree small for a learning project |

---

## 2. Log In and Record Identifiers

### Steps

1. Open <https://cloud.oracle.com>.
2. Enter your **cloud account name (tenancy)**, then sign in with your username and password.
3. If asked, choose your **home region**. It **cannot be changed later**.
4. Open the **Profile** menu (top right) and collect the values below.

### Record these in your notes file

| Value | Where to find it |
|---|---|
| Tenancy name | Shown at login |
| `TENANCY_OCID` | Profile menu → **Tenancy** → copy OCID |
| `USER_OCID` | Profile menu → **My profile** → copy OCID |
| Home region | Region selector at top (e.g. `us-ashburn-1`) |
| Region key | Tenancy details page (e.g. `IAD`) |
| Object Storage namespace | Profile menu → **Tenancy** → *Object storage namespace* |

> ⚠️ Keep OCIDs private. Never commit private keys, passwords or tokens to Git.

✅ **Verify:** All six values are saved in your notes file.

---

## 3. Create the Compartments

IAM, tags and budgets are **home-region** resources. Make sure the region selector shows your home region.

### 3.1 Create the parent compartment

1. Menu (☰) → **Identity & Security** → **Compartments**.
2. Click **Create compartment**.
3. Enter:
   - **Name:** `Shopsphere`
   - **Description:** `ShopSphere E-Commerce Production Learning Project`
   - **Parent compartment:** root (your tenancy)
4. Click **Create compartment**.
5. Wait for **Active**, open it, and copy the OCID as `SHOPSPHERE_COMPARTMENT_OCID`.
<img width="1897" height="799" alt="Picture1" src="https://github.com/user-attachments/assets/353adbda-d1f3-4479-845d-5f338032d790" />

### 3.2 Create the five child compartments

Open `Shopsphere` in the compartment list, then click **Create compartment** inside it (parent must show `Shopsphere`). Create each one:

| Name | Description |
|---|---|
| `shopsphere-network` | ShopSphere network resources |
| `shopsphere-security` | ShopSphere security resources |
| `shopsphere-platform` | ShopSphere platform resources |
| `shopsphere-app` | ShopSphere application resources |
| `shopsphere-data` | ShopSphere data resources |

Copy each OCID into your notes file (`SHOPSPHERE_NETWORK_COMPARTMENT_OCID`, and so on).

> Compartment deletion is slow and needs an empty compartment, so double-check names before creating.

✅ **Verify:** Compartments page shows `Shopsphere` with five **Active** children.
<img width="1902" height="798" alt="image" src="https://github.com/user-attachments/assets/3e5ef275-e896-4a91-9d06-47dffa4bbae6" />

---

## 4. Create IAM Groups

### 4.1 Find your identity domain

Menu → **Identity & Security** → **Domains**. Note whether you use the **Default** domain or another one. This affects policy syntax in section 5.

### 4.2 Create the groups

1. Click your domain → **User management** → **Groups** → **Create group**.
2. Create these five groups (name and description are the same pattern):

| Group | Purpose |
|---|---|
| `shopsphere-admins` | Manage everything in the project |
| `shopsphere-network-admins` | Own VCNs and gateways |
| `shopsphere-developers` | Build and deploy workloads |
| `shopsphere-dba` | Databases and backups |
| `shopsphere-readonly` | View only |

### 4.3 Add yourself to the admins group

1. Open `shopsphere-admins` → **Users** tab → **Assign user to group**.
2. Select your user → **Assign**.

> **Best practice:** Treat the tenancy `Administrators` login as a **break-glass** account. Do daily work as a user in `shopsphere-admins`.

✅ **Verify:** Each group is listed, and your user appears under `shopsphere-admins`.

---

## 5. Create IAM Policies

1. Menu → **Identity & Security** → **Policies**.
2. Set the compartment filter to the **root** (tenancy) so the policy is attached there.
3. Click **Create policy**.
4. Enter:
   - **Name:** `shopsphere-core-policies`
   - **Description:** `ShopSphere role-based access policies`
   - **Compartment:** root
5. Turn on **Show manual editor** and paste:

```text
Allow group shopsphere-admins to manage all-resources in compartment Shopsphere

Allow group shopsphere-network-admins to manage virtual-network-family in compartment shopsphere-network
Allow group shopsphere-network-admins to read all-resources in compartment Shopsphere

Allow group shopsphere-developers to manage instance-family in compartment shopsphere-app
Allow group shopsphere-developers to manage load-balancers in compartment shopsphere-app
Allow group shopsphere-developers to manage cluster-family in compartment shopsphere-platform
Allow group shopsphere-developers to manage repos in compartment shopsphere-platform
Allow group shopsphere-developers to use virtual-network-family in compartment shopsphere-network
Allow group shopsphere-developers to read all-resources in compartment Shopsphere

Allow group shopsphere-dba to manage database-family in compartment shopsphere-data
Allow group shopsphere-dba to manage object-family in compartment shopsphere-data
Allow group shopsphere-dba to use virtual-network-family in compartment shopsphere-network

Allow group shopsphere-readonly to read all-resources in compartment Shopsphere
```

6. Click **Create**.

> **Non-default identity domain?** Prefix each group, for example `Allow group 'MyDomain'/'shopsphere-admins' to ...`.

> OCI IAM is **deny by default** and has no explicit deny. Never grant `manage all-resources` outside the admin group.

✅ **Verify:** The policy shows **Active** with all statements and no syntax warnings.

---

## 6. Create a Dynamic Group

Workloads (compute, OKE nodes) should authenticate with **instance principals**, not stored keys. Create the group now; attach policies in later phases.

1. Menu → **Identity & Security** → **Domains** → your domain → **Dynamic groups** → **Create dynamic group**.
2. Enter:
   - **Name:** `shopsphere-app-instances`
   - **Description:** `Compute instances in the app compartment`
3. Matching rule (replace the OCID with `SHOPSPHERE_APP_COMPARTMENT_OCID`):

```text
ALL {instance.compartment.id = 'ocid1.compartment.oc1..xxxx'}
```

4. Click **Create**.

Example policy for a later phase:

```text
Allow dynamic-group shopsphere-app-instances to read secret-bundles in compartment shopsphere-security
```

✅ **Verify:** The dynamic group exists and the rule shows your app compartment OCID.

---

## 7. Enable and Enforce MFA

### 7.1 Enable MFA for your own user

1. Profile avatar → **My profile** → **Security**.
2. Under **Multi-factor authentication**, click **Enable** and choose an authenticator app (TOTP) or a FIDO2 / passkey device.
3. Scan the QR code and enter the verification code.
4. Save the **backup/recovery** option offline.
5. Sign out, sign in again and confirm the MFA prompt appears.

### 7.2 Enforce MFA for all users

1. Menu → **Identity & Security** → **Domains** → your domain → **Security** → **Sign-on policies**.
2. Edit the **Default Sign-On Policy** (or create a new one) and add a rule that requires MFA for **all** users, not only when risky.
3. Make sure administrators are **not** exempted.
4. Under **MFA settings**, enable at least **two** methods so a lost device does not lock you out.

> MFA protects the **console**. API keys bypass it, which is why section 8 uses short-lived **session tokens**.

✅ **Verify:** Fresh sign-in forces a second factor; two methods are registered.

---

## 8. Install and Configure the OCI CLI

### 8.1 Install prerequisites (Ubuntu)

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl unzip git jq python3 python3-pip python3-venv
```

### 8.2 Install the CLI

```bash
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"
```

Accept the defaults, then open a new terminal (or `source ~/.bashrc`).

```bash
oci --version
```

> 🔎 On shared or sensitive machines, download and read the install script before running it.

### 8.3 Preferred login: session token

```bash
oci session authenticate --region <home-region> --profile-name SHOPSPHERE
```

A browser opens. Sign in (with MFA) and approve. Then tell the CLI to use it:

```bash
export OCI_CLI_PROFILE=SHOPSPHERE
export OCI_CLI_AUTH=security_token
oci iam region list
```

Session tokens expire (about an hour). Refresh with:

```bash
oci session refresh --profile SHOPSPHERE
```

### 8.4 Fallback: API key profile

Use this only where browser login is not possible.

```bash
oci setup config
```

Answer the prompts: user OCID, tenancy OCID, region, and **Yes** to generate a key pair. Then:

1. Console → Profile → **My profile** → **API keys** → **Add API key** → **Paste a public key**.
2. Paste the contents of `~/.oci/oci_api_key_public.pem`.
3. Compare the fingerprint shown with the one in `~/.oci/config`.

Fix file permissions:

```bash
oci setup repair-file-permissions --file ~/.oci/config
oci setup repair-file-permissions --file ~/.oci/oci_api_key.pem
```

Test:

```bash
oci iam region list --auth api_key
```

### 8.5 Config file shape

```ini
[DEFAULT]
user=ocid1.user.oc1..xxxx
fingerprint=aa:bb:cc:...
tenancy=ocid1.tenancy.oc1..xxxx
region=us-ashburn-1
key_file=/home/<you>/.oci/oci_api_key.pem
```

> 🔒 The private key (`oci_api_key.pem`) never goes into Git. Keep it in `~/.oci` with `chmod 600`.

✅ **Verify:** `oci --version` works and `oci iam region list` returns data.

---

## 9. Create Tags

Create tags **before** any resources exist so everything can be tagged from day one.

### 9.1 Tag namespace

1. Menu → **Governance & Administration** → **Tag Namespaces**.
2. Select compartment `Shopsphere` → **Create namespace definition**.
3. **Name:** `shopsphere`, **Description:** `ShopSphere governance tags` → **Create**.

### 9.2 Tag keys

Open the `shopsphere` namespace → **Create tag key definition** for each:

| Key | Description | Settings |
|---|---|---|
| `environment` | dev, staging or production | Cost-tracking **on**; value type **A list of values**: `dev`, `staging`, `production` |
| `owner` | Responsible person or team | Cost-tracking **on** |
| `cost-center` | Cost allocation code | Cost-tracking **on** |

### 9.3 Tag defaults (optional but recommended)

1. Open the `Shopsphere` compartment details → **Tag defaults** → **Assign tag defaults**.
2. Choose `shopsphere.owner` (and `shopsphere.cost-center`) and set a default value, optionally marking it **User-provided** so it is required.

✅ **Verify:** The namespace shows three keys with cost-tracking enabled.

---

## 10. Create the Budget and Alerts

> A budget **alerts**. It never stops resources.

### 10.1 Budget

1. Menu → **Billing & Cost Management** → **Budgets** → **Create budget**.
2. Enter:
   - **Name:** `shopsphere-monthly`
   - **Description:** `Monthly spend alerts for ShopSphere`
   - **Budget scope:** **Compartment** → `Shopsphere`
   - **Reset period:** Monthly (day 1)
   - **Budget amount:** for example `50` (or your currency equivalent)
3. Click **Next**.

### 10.2 Alert rules

Create four rules, each with your email as recipient:

| Rule type | Threshold | Purpose |
|---|---|---|
| Actual spend | 50% | Early notice |
| Actual spend | 75% | Review resources |
| Actual spend | 90% | Act now |
| **Forecast** spend | 100% | Fires **before** you overspend |

Click **Create**.

### 10.3 Weekly manual routine

| Habit | When |
|---|---|
| Open **Cost Analysis**, group by tag `shopsphere.environment` | Weekly |
| Delete idle load balancers, boot volumes, reserved IPs | Weekly |
| Stop or terminate dev compute you are not using | Daily |
| Confirm alert recipient emails are valid | Monthly |

> ⚠️ Cost data can lag by hours up to a day, so a runaway resource can spend a lot before the first alert arrives.

✅ **Verify:** The budget lists four alert rules and the target shows `Shopsphere`.

---

## 11. Security Baseline: Audit, Cloud Guard, Notifications

### 11.1 Audit (always on)

1. Menu → **Observability & Management** → **Audit**.
2. Pick compartment `Shopsphere`, set a recent time range and click **Search**.
3. Confirm you can see your own recent actions (for example the compartment creations).

Default retention is 365 days.

### 11.2 Cloud Guard

1. Menu → **Identity & Security** → **Cloud Guard** → **Enable Cloud Guard**.
2. Choose your **home region** as the reporting region.
3. Accept the policy creation prompt, then set the **root** compartment as the target so it covers `Shopsphere`.
4. Keep the Oracle-managed detector recipes enabled to start.

### 11.3 Notifications topic

1. Menu → **Developer Services** → **Application Integration** → **Notifications** → **Create topic**.
2. Compartment: `shopsphere-security`; **Name:** `shopsphere-alerts`.
3. Open the topic → **Create subscription** → protocol **Email** → your address.
4. Open the confirmation email and click **Confirm subscription**. Without this nothing is delivered.

### 11.4 Later phases

| Capability | Phase |
|---|---|
| Vault and KMS keys | Security phase |
| Security Zones | After networking |
| Bastion service instead of public SSH | Network phase |
| Logging and VCN flow logs | Observability phase |

### 11.5 Console checks

| # | Check | Done |
|---|---|---|
| 1 | Tenancy, user, region, namespace recorded | ☐ |
| 2 | `Shopsphere` and five child compartments are **Active** | ☐ |
| 3 | Five groups exist; you are in `shopsphere-admins` | ☐ |
| 4 | `shopsphere-core-policies` is **Active** | ☐ |
| 5 | Dynamic group `shopsphere-app-instances` exists | ☐ |
| 6 | MFA enforced, two methods registered, sign-in tested | ☐ |
| 7 | CLI session token works (`oci iam region list`) | ☐ |
| 8 | Tag namespace with three cost-tracking keys | ☐ |
| 9 | Budget with four alert rules | ☐ |
| 10 | Audit shows events; Cloud Guard enabled | ☐ |
| 11 | Notification subscription **Confirmed** | ☐ |

✅ **Verify:** Audit shows your events, Cloud Guard is **Enabled**, and the subscription status is **Confirmed**.

---

## 12. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `401 NotAuthenticated` | Public key not uploaded, wrong fingerprint or expired session | Re-upload the key, compare fingerprints, run `oci session refresh` |
| `404 NotAuthorizedOrNotFound` | Missing policy, wrong compartment or region | Check policy statements and group membership. OCI uses one error for both cases |
| Cannot create IAM, tag or budget resources | Not in home region | Switch the region selector to your home region |
| Groups not recognised in a policy | Groups live in a non-default domain | Use `'Domain'/'group'` syntax |
| No budget emails | Wrong recipient or spam filter | Re-check recipients and spam folder |

---

## 13. Pitfalls and Hardening Checklist

### Pitfalls

1. **Wrong home region.** Cannot be changed. Verify before building anything.
2. **Daily use of the tenancy admin.** Keep it for emergencies only.
3. **Long-lived API keys on laptops.** Prefer session tokens; rotate API keys regularly if you keep them.
4. **Private keys or state in Git.** State can hold secrets in plain text.
5. **Over-broad policies.** Scope to compartments and resource families.
6. **Budget treated as a cap.** It only alerts.
7. **Untagged resources.** Cost reports become hard to read.
8. **Single MFA device with no backup.** Register a second method.
9. **Typos in names.** Keep variable names consistent (`SHOPSPHERE_COMPARTMENT_OCID`).

### Hardening checklist

- [ ] MFA enforced through a sign-on policy for every user
- [ ] Break-glass admin credentials stored offline
- [ ] CLI uses session tokens; unneeded API keys deleted
- [ ] Policies reviewed every quarter
- [ ] Cloud Guard active with notifications going to a monitored mailbox
- [ ] State bucket private, versioned and restricted to admins
- [ ] Tag defaults applied; Cost Analysis grouped by tag
---

*End of Phase 1 (Advanced, Manual). Next: Phase 2, Networking (VCN, subnets, gateways, security lists/NSGs), built step by step in `shopsphere-network`.*
