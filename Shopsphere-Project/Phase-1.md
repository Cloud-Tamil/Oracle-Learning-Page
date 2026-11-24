# Phase 1 – OCI Account and Foundation: Detailed Walkthrough

A step-by-step explanation of everything in Phase 1, with context on why each step matters and how to do it correctly.

---

## Table of Contents

1. [Log in to OCI](#11-log-in-to-oci)
2. [Create the Shopsphere Compartment](#12-create-the-shopsphere-compartment)
3. [Configure IAM](#13-configure-iam-identity-and-access-management)
4. [Enable MFA](#14-enable-mfa-multi-factor-authentication)
5. [Install OCI CLI on Ubuntu](#15-install-oci-cli-on-ubuntu)
6. [Create the Project Directory](#16-create-the-project-directory)
7. [Configure Cost Alerts (Budgets)](#17-configure-cost-alerts-budgets)
8. [Configure Terraform API Authentication](#18-configure-terraform-api-authentication)
9. [Phase 1 Validation](#phase-1-validation)
10. [Deliverable Summary](#deliverable-summary)
11. [Common Pitfalls to Avoid](#common-pitfalls-to-avoid)

---

## 1.1 Log in to OCI

**What it is:** The entry point. You need an Oracle Cloud Infrastructure (OCI) account and a user identity before anything else.

### How to do it

1. Go to <https://cloud.oracle.com> (the OCI Console).
2. Sign in with your cloud account name (tenancy) and your username/password.
3. If prompted, choose your **home region**. This is the region where your tenancy's identity resources live. **It cannot be changed later**, so choose deliberately.
4. You land on the OCI Console dashboard.

### Why record these values

You'll need them repeatedly for CLI and Terraform configuration.

| Value | Where to find it |
|---|---|
| `TENANCY_NAME` | Your cloud account name (shown at login) |
| `TENANCY_OCID` | Profile menu → Tenancy (or Administration → Tenancy Details) |
| `USER_OCID` | Profile menu → My Profile → User Details |
| `HOME_REGION` | Console region selector (e.g., `us-ashburn-1`) |
| `REGION_IDENTIFIER` | The short region key, e.g., `iad`, `phx`, `fra` |

> ⚠️ **Security note:** Keep OCIDs private. Never commit private keys, passwords, or API tokens to Git.

---

## 1.2 Create the Shopsphere Compartment

**What a compartment is:** A logical container for OCI resources (like a folder/project boundary). It is the fundamental unit for access control and cost tracking. Resources in one compartment can be isolated from another via IAM policies.

### How to do it

1. Navigate: **Identity & Security → Identity → Compartments**.
2. Click **Create Compartment**.
3. Fill in:
   - **Name:** `Shopsphere`
   - **Description:** `ShopSphere E-Commerce Production Learning Project`
   - **Parent Compartment:** Root compartment (top-level)
   - **Tags:** optional (skip or add later)
4. Click **Create** and wait until the status is **Active** (usually seconds).
5. **Record the OCID:** Click the compartment name and copy the OCID shown on the details page.

### Local config

```bash
export COMPARTMENT_NAME="Shopsphere"
export SHOPSPHERE_COMPARTMENT_OCID="<paste-ocid-here>"
```

> **Note:** The original document had a typo, `SHOPSPhERE_COMPARTMENT_OCID`. The correct name is `SHOPSPHERE_COMPARTMENT_OCID`.

---

## 1.3 Configure IAM (Identity and Access Management)

**What it is:** OCI's authorization system. You create groups of users and attach policies that grant permissions on compartments.

### Create groups

(if your identity-domain permissions allow)

| Group | Purpose |
|---|---|
| `shopsphere-admins` | Full control |
| `shopsphere-developers` | Limited build/deploy rights |
| `shopsphere-readonly` | View-only |

**Navigation:** Identity & Security → Identity → Domains → *(your domain)* → Groups → Create Group.

### Create policies

**Navigation:** Identity & Security → Identity → Policies → Create Policy.

Example administrator policy:

```text
Allow group shopsphere-admins to manage all-resources in compartment Shopsphere
```

**Why:** Least privilege. `manage all-resources` in the compartment is powerful, so only trusted admins should get it. Developers should get narrower statements (e.g., `manage instances`, `manage object-family`, `use virtual-network-family`).

---

## 1.4 Enable MFA (Multi-Factor Authentication)

**Why:** Passwords alone are not enough. MFA protects your tenancy even if credentials leak.

### How

1. Click your **Profile avatar → My Profile**.
2. Go to the **Security** tab.
3. Enable an MFA method (TOTP authenticator app or hardware key).
4. **Test it:** sign out and sign back in to confirm MFA is enforced.

---

## 1.5 Install OCI CLI on Ubuntu

**What the CLI is:** Oracle's command-line tool for managing OCI resources. It is essential for scripting, Terraform bootstrap, and validation.

### Step 1 – Update and install prerequisites

```bash
# Update the package index and upgrade existing packages
sudo apt update
sudo apt upgrade -y

# Install prerequisites
sudo apt install -y \
  curl unzip git jq \
  python3 python3-pip \
  python3-venv
```

### Step 2 – Install the OCI CLI

Use Oracle's official installer (recommended):

```bash
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"
```

Accept the defaults, then reload your shell (`source ~/.bashrc`) or open a new terminal.

### Step 3 – Verify

```bash
oci --version
```

### Step 4 – Configure the CLI

```bash
oci setup config
```

It will prompt for:

- User OCID
- Tenancy OCID
- Region (e.g., `us-ashburn-1`)
- Whether to generate an API key pair (say **Yes**, the easiest path)

This creates `~/.oci/config` and `~/.oci/oci_api_key.pem` (private key), plus a public key you **must upload** to your user in the Console:

**Profile → My Profile → API Keys → Add API Key → Paste Public Key**

### Step 5 – Test

```bash
oci iam region list
```

---

## 1.6 Create the Project Directory

**Why:** A consistent, predictable layout for Terraform, application code, Kubernetes manifests, and GitOps. Do this now so later phases don't scatter files.

```bash
mkdir -p ~/shopsphere-oci
cd ~/shopsphere-oci

mkdir -p \
  terraform/modules \
  terraform/environments/dev \
  terraform/environments/staging \
  terraform/environments/production \
  application/frontend \
  application/services \
  kubernetes/helm-charts \
  kubernetes/namespaces \
  kubernetes/policies \
  gitops/applications \
  gitops/environments \
  docs \
  scripts
```

### Layout logic

| Path | Purpose |
|---|---|
| `terraform/modules` | Reusable modules (network, compute, etc.) |
| `terraform/environments/{dev,staging,production}` | Per-environment root configs |
| `application/` | Application source |
| `kubernetes/` | Helm charts, namespaces, policies |
| `gitops/` | Argo CD / Flux application definitions |
| `docs/`, `scripts/` | Documentation and helper scripts |

> **Strongly recommended:** Run `git init` here and add a `.gitignore` that excludes `*.pem`, `.oci/`, `.terraform/`, `*.tfstate*`, and `*.tfvars` containing secrets.

---

## 1.7 Configure Cost Alerts (Budgets)

**What it is:** A budget is a **notification threshold, not a hard spending cap**. OCI will email you when spend crosses the percentages you set, but it will not stop resources.

### How

1. Navigate: **Billing & Cost Management → Budgets**.
2. Click **Create Budget**.
3. **Target:** Compartment → `Shopsphere`.
4. Set a monthly amount (e.g., $50 or $100 for a learning project).
5. Add alert rules at **50%, 75%, 90%** of the budget, each with your email.

> 💡 **Reality check:** Budgets alone won't prevent surprise bills. Also enable cost-tracking tags and periodically review **Cost Analysis**. For real protection, pair budgets with automated cleanup scripts (later phase).

---

## 1.8 Configure Terraform API Authentication

**What it is:** Terraform needs to authenticate to OCI. The standard method is API key authentication: a private key stored locally, with the matching public key registered to your OCI user.

### How

If you already ran `oci setup config` (1.5), the key pair exists. Otherwise:

```bash
mkdir -p ~/.oci
chmod 700 ~/.oci
openssl genrsa -out ~/.oci/oci_api_key.pem 2048
chmod 600 ~/.oci/oci_api_key.pem
openssl rsa -pubout -in ~/.oci/oci_api_key.pem -out ~/.oci/oci_api_key_public.pem
```

Register the public key in the Console:

**Profile → My Profile → API Keys → Add API Key → Paste Public Key**

Copy the fingerprint shown and put it in `~/.oci/config`:

```ini
[DEFAULT]
user=ocid1.user.oc1..xxxx
fingerprint=aa:bb:cc:...
tenancy=ocid1.tenancy.oc1..xxxx
region=us-ashburn-1
key_file=~/.oci/oci_api_key.pem
```

> 🔒 **Critical:** The private key (`oci_api_key.pem`) must **never** enter the Git repo. Keep it in `~/.oci` with `chmod 600`. Terraform reads it via `~/.oci/config`.

### Terraform provider block (for later use)

```hcl
provider "oci" {
  tenancy_ocid     = var.tenancy_ocid
  user_ocid        = var.user_ocid
  fingerprint      = var.fingerprint
  private_key_path = "~/.oci/oci_api_key.pem"
  region           = var.region
}
```

---

## Phase 1 Validation

Run:

```bash
oci iam compartment list \
  --compartment-id-in-subtree true \
  --access-level ACCESSIBLE
```

**What this does:**

- `--compartment-id-in-subtree true` includes all compartments under your tenancy root.
- `--access-level ACCESSIBLE` shows only compartments you're allowed to see.

**Expected result:** You see the `Shopsphere` compartment in the JSON output (look for `"name": "Shopsphere"`). If not, check that your IAM policy grants your user access.

### Quick confirm

```bash
oci iam compartment list --all | jq -r '.data[] | select(.name=="Shopsphere") | .id'
```

This should print the compartment OCID.

---

## Deliverable Summary

By the end of Phase 1 you should have:

| Item | Status check |
|---|---|
| OCI tenancy + user | Logged in, OCIDs recorded |
| Shopsphere compartment | Active, OCID recorded |
| IAM groups + policies | Admins/developers/readonly created |
| MFA enabled | Tested login |
| OCI CLI installed + configured | `oci --version` and `oci iam region list` work |
| Project directory tree | `~/shopsphere-oci` with all subfolders |
| Budget alerts | 50/75/90% notifications configured |
| Terraform API auth | Public key registered, `~/.oci/config` in place, private key secured |

---

## Common Pitfalls to Avoid

1. **Wrong home region:** It cannot be changed; verify before creating resources.
2. **Public key not uploaded:** `oci setup config` succeeds but API calls fail with `401`. Always upload the public key to your user.
3. **Private key in Git:** Catastrophic. Add `.gitignore` before the first commit.
4. **Over-permissive policies:** `manage all-resources` for developers is a security anti-pattern. Scope tightly.
5. **Budget mistaken for a cap:** It won't stop spend. Review costs weekly during learning.
6. **Typos in variable names:** e.g., `SHOPSPhERE_...`. Keep them consistent (`SHOPSPHERE_COMPARTMENT_OCID`).
