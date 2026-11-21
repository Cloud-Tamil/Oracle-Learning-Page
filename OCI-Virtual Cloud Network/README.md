# OCI Networking Lab: Build `DevOps-VCN` and Access a Server in It

A step-by-step learning guide for Oracle Cloud Infrastructure (OCI). You will build a Virtual Cloud Network (VCN) with a public subnet, an internet gateway, a route table, a security list and a network security group. Then you will launch a VM inside it and connect over SSH.

---

## 1. What you are building

```
                         Internet
                            |
                      [ DevOps-IGW ]            Internet Gateway
                            |
 +--------------------------+--------------------------------------+
 |  DevOps-VCN   10.0.0.0/16                                       |
 |                                                                 |
 |   +---------------------------------------------------------+   |
 |   |  DevOps-Pub-Sub   10.0.1.0/24   (public subnet)         |   |
 |   |                                                         |   |
 |   |   Route table  : DevOps-RT  (0.0.0.0/0 -> DevOps-IGW)   |   |
 |   |   Security list: DevOps-SL  (subnet-level firewall)     |   |
 |   |                                                         |   |
 |   |   +-------------------------------------------------+   |   |
 |   |   |  VM instance  (public IP)                       |   |   |
 |   |   |  NSG: DevOps-SG  (VNIC-level firewall)          |   |   |
 |   |   +-------------------------------------------------+   |   |
 |   +---------------------------------------------------------+   |
 +-----------------------------------------------------------------+
```

### Resource summary

| # | Resource | Name | Value / purpose |
|---|----------|------|-----------------|
| 1 | VCN | `DevOps-VCN` | `10.0.0.0/16` (65,536 private IPs) |
| 2 | Internet Gateway | `DevOps-IGW` | Two-way door between the VCN and the internet |
| 3 | Route Table | `DevOps-RT` | Sends `0.0.0.0/0` traffic to `DevOps-IGW` |
| 4 | Security List | `DevOps-SL` | Subnet-level allow rules (SSH, HTTP) |
| 5 | Subnet | `DevOps-Pub-Sub` | `10.0.1.0/24` (public) |
| 6 | Network Security Group | `DevOps-SG` | VNIC-level allow rules |
| 7 | Compute instance | your choice | The server you SSH into |

Everything lives in your compartment (in your case `Cloud-Tamil (root)`).

---

## 2. Concepts in plain words

| Term | Think of it as | Key fact |
|------|----------------|----------|
| **Compartment** | A folder for organising and controlling access to resources | The root compartment is the tenancy itself. In real projects, create a child compartment instead. |
| **VCN** | Your private data-centre network in the cloud | You choose the IP range (CIDR). |
| **CIDR** | The size of an IP range | `/16` = 65,536 addresses, `/24` = 256 addresses. |
| **Subnet** | A slice of the VCN | A **public** subnet can give VMs public IPs. A **private** subnet cannot. |
| **Internet Gateway (IGW)** | The front door to the internet | Needed for public IPs to work. It does nothing until a route points to it. |
| **Route Table** | The road map | Decides where traffic goes. No route to the IGW means no internet. |
| **Security List** | A firewall at the **subnet** level | Applies to every VM in the subnet. |
| **NSG** | A firewall at the **VNIC** level | Applies only to the VMs you attach it to. |

### The rule that causes most mistakes

> **Internet access needs all four:** an IGW, a route rule to the IGW, a public IP on the VM, and a security rule that allows the port.
> Miss any one and the connection times out.

Security lists and NSGs are **additive**. Traffic is allowed if **either** one allows it. OCI rules are **allow-only**; there are no deny rules.

**Stateful vs stateless:** Stateful rules (the default) automatically allow the reply traffic. Stateless rules need a matching rule in the other direction.

---

## 3. Prerequisites

- An OCI account (the Free Tier works).
- A modern browser to sign in to the OCI Console.
- Your **public IP address**. Search "what is my IP" and note it. You will use it as `x.x.x.x/32` so only you can SSH in.
- A terminal with an SSH client (Windows 10/11 PowerShell, macOS or Linux terminal).

---

## 4. Step-by-step: create the network

Open the console menu and go to **Networking → Virtual cloud networks**. Make sure the compartment selector on the left shows your compartment.

### Step 1: Create the VCN

1. Click **Create VCN**. (Choose the plain option, not the "VCN with Internet Connectivity" wizard, so you learn each part.)
2. **Name:** `DevOps-VCN`
3. **Compartment:** `Cloud-Tamil`
4. **IPv4 CIDR Blocks:** `10.0.0.0/16`
5. Leave DNS resolution enabled and click **Create VCN**.

OCI automatically creates three defaults inside the VCN: a **Default Route Table** (0 rules), a **Default Security List** and **DHCP Options**. This is why your screenshot shows "Default Route Table for DevOps-VCN" with 0 rules.

### Step 2: Create the Internet Gateway

1. Open `DevOps-VCN` → **Gateways** tab → **Internet gateways** → **Create Internet Gateway**.
2. **Name:** `DevOps-IGW`
3. Click **Create Internet Gateway**. The state should become **Available**.

### Step 3: Create the Route Table

1. In `DevOps-VCN` → **Routing** tab → **Create Route Table**.
2. **Name:** `DevOps-RT`
3. Click **+ Another Route Rule** and enter:
   - **Target Type:** `Internet Gateway`
   - **Destination CIDR Block:** `0.0.0.0/0`
   - **Target Internet Gateway:** `DevOps-IGW`
4. Click **Create**. The table now shows **1 rule**.

`0.0.0.0/0` means "any address not found inside the VCN". Internet-bound traffic goes to the IGW.

### Step 4: Create the Security List

1. In `DevOps-VCN` → **Security** tab → **Security Lists** → **Create Security List**.
2. **Name:** `DevOps-SL`
3. **Ingress rules** (traffic coming in). Add these:

   | Stateless | Source CIDR | Protocol | Dest. port | Purpose |
   |-----------|-------------|----------|------------|---------|
   | No | `<your-ip>/32` | TCP | `22` | SSH from your PC only |
   | No | `0.0.0.0/0` | TCP | `80` | HTTP (optional, for web test) |
   | No | `0.0.0.0/0` | TCP | `443` | HTTPS (optional) |

4. **Egress rules** (traffic going out). Add one:

   | Stateless | Destination CIDR | Protocol |
   |-----------|------------------|----------|
   | No | `0.0.0.0/0` | All protocols |

5. Click **Create Security List**.

> Avoid `0.0.0.0/0` for port 22 on anything real. Bots scan the internet for open SSH constantly.

### Step 5: Create the Public Subnet

1. In `DevOps-VCN` → **Subnets** tab → **Create Subnet**.
2. **Name:** `DevOps-Pub-Sub`
3. **Subnet type:** Regional
4. **IPv4 CIDR Block:** `10.0.1.0/24`
5. **Route Table:** select **`DevOps-RT`** (this step is easy to miss, and without it the default empty table is used)
6. **Subnet access:** **Public Subnet**
7. **DHCP options:** default
8. **Security Lists:** select **`DevOps-SL`** (you can also keep the default list; both apply)
9. Click **Create Subnet**.

OCI reserves 3 addresses in each subnet, so a `/24` gives 253 usable IPs.

### Step 6: Create the Network Security Group

1. In `DevOps-VCN` → **Security** tab → **Network Security Groups** → **Create Network Security Group**.
2. **Name:** `DevOps-SG` → **Next**.
3. Add a rule:
   - **Direction:** Ingress
   - **Source Type:** CIDR, **Source CIDR:** `<your-ip>/32`
   - **IP Protocol:** TCP, **Destination Port Range:** `22`
4. Add a second rule for HTTP if you want it (source `0.0.0.0/0`, TCP, port `80`).
5. Click **Create**.

An NSG does nothing until you attach it to a VM's VNIC (Step 7).

---

## 5. Step-by-step: launch a VM and access it

### Step 7: Create the compute instance

1. Go to **Compute → Instances → Create instance**.
2. **Name:** `devops-vm-01`
3. **Image:** Oracle Linux 9 (or Ubuntu 22.04/24.04)
4. **Shape:** a Free Tier one, for example `VM.Standard.E2.1.Micro`, or an Ampere A1 shape if available in your region
5. **Networking** section → **Edit**:
   - **VCN:** `DevOps-VCN`
   - **Subnet:** `DevOps-Pub-Sub`
   - **Public IPv4 address:** **Automatically assign public IPv4 address**
   - Tick **Use network security groups to control traffic** and choose **`DevOps-SG`**
6. **Add SSH keys:** choose **Generate a key pair for me** and **download both keys**. Alternatively, **Upload public key files** if you made your own (see below).
7. Click **Create** and wait for the state to become **Running**.
8. Copy the **Public IP address** from the instance details page.

**Optional: generate your own SSH key (PowerShell or Linux/macOS):**

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/oci_devops
# Upload the public key:  ~/.ssh/oci_devops.pub
```

### Step 8: Connect with SSH

The default username depends on the image:

| Image | Username |
|-------|----------|
| Oracle Linux | `opc` |
| Ubuntu | `ubuntu` |

**Linux / macOS:**

```bash
chmod 400 ~/Downloads/ssh-key-*.key
ssh -i ~/Downloads/ssh-key-*.key opc@<PUBLIC_IP>
```

**Windows PowerShell:**

```powershell
icacls "$HOME\Downloads\ssh-key-xxxx.key" /inheritance:r /grant:r "$($env:USERNAME):(R)"
ssh -i "$HOME\Downloads\ssh-key-xxxx.key" opc@<PUBLIC_IP>
```

Type `yes` at the host-fingerprint prompt. A shell prompt like `[opc@devops-vm-01 ~]$` means success.

### Step 9: Test a web server (optional)

Run these on the VM.

**Oracle Linux:**

```bash
sudo dnf install -y nginx
sudo systemctl enable --now nginx
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

**Ubuntu:**

```bash
sudo apt update && sudo apt install -y nginx
# OCI Ubuntu images ship with restrictive iptables rules; list them first:
sudo iptables -L INPUT -n --line-numbers
# then insert an allow rule above the REJECT line (use that line number):
sudo iptables -I INPUT <LINE_NUMBER> -p tcp --dport 80 -m state --state NEW -j ACCEPT
sudo netfilter-persistent save
```

From your own PC:

```bash
curl http://<PUBLIC_IP>
```

You should see the nginx welcome page. A timeout means one of the four requirements in section 2 is missing. See the troubleshooting table below.

---

## 6. Verify the setup

Check each item in the console:

| Check | Where | Expected |
|-------|-------|----------|
| VCN state | Networking → VCNs | Available, `10.0.0.0/16` |
| IGW state | VCN → Gateways | `DevOps-IGW` Available |
| Route rule | VCN → Routing → `DevOps-RT` | `0.0.0.0/0` → `DevOps-IGW` |
| Subnet route table | VCN → Subnets → `DevOps-Pub-Sub` | Route table = `DevOps-RT` |
| Subnet security list | same page | Includes `DevOps-SL` |
| VM public IP | Compute → Instance | A public IPv4 is shown |
| VM NSG | Instance → VNIC details | `DevOps-SG` attached |

For a deeper check, use **Networking → Network command center → Network Path Analyzer**. Pick source `Internet` and destination your VM, protocol TCP, port 22. It tells you exactly which rule blocks the path.

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| SSH times out | Subnet still uses the default route table | Edit subnet → set `DevOps-RT` |
| SSH times out | No public IP on the VM | Recreate the VNIC setting or attach a reserved public IP |
| SSH times out | Your IP changed, or the rule is not `/32` of your current IP | Update the ingress rule in `DevOps-SL` / `DevOps-SG` |
| `Permission denied (publickey)` | Wrong username or wrong key | Use `opc` (Oracle Linux) or `ubuntu` (Ubuntu) with the matching key |
| `UNPROTECTED PRIVATE KEY FILE` | Key file permissions too open | `chmod 400` (Linux/macOS) or the `icacls` command (Windows) |
| SSH works, web page times out | Port 80 not allowed in the cloud rules or in the OS firewall | Add the rule to `DevOps-SL`/`DevOps-SG` and open the OS firewall |
| Cannot delete the IGW | A route rule still references it | Delete or edit the route table first |

---

## 8. Clean up (avoid surprise charges)

Delete in this order, because OCI blocks deleting a resource that something else still uses:

1. **Terminate** the compute instance (tick "permanently delete the attached boot volume").
2. Delete subnet `DevOps-Pub-Sub`.
3. Delete NSG `DevOps-SG`.
4. Delete route table `DevOps-RT` and security list `DevOps-SL` (defaults are removed with the VCN).
5. Delete Internet Gateway `DevOps-IGW`.
6. Delete `DevOps-VCN`.

---

## 9. Same network as Terraform (optional)

This builds steps 1-6 as code. It uses the `oracle/oci` provider. Authenticate with `~/.oci/config` (created by `oci setup config`).

```hcl
terraform {
  required_providers {
    oci = { source = "oracle/oci" }
  }
}

provider "oci" {
  region = var.region
}

variable "region"           { type = string }
variable "compartment_ocid" { type = string }
variable "my_ip_cidr" {
  type        = string
  description = "Your public IP as a /32, e.g. 203.0.113.10/32"
}

resource "oci_core_vcn" "devops" {
  compartment_id = var.compartment_ocid
  display_name   = "DevOps-VCN"
  cidr_blocks    = ["10.0.0.0/16"]
}

resource "oci_core_internet_gateway" "igw" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.devops.id
  display_name   = "DevOps-IGW"
  enabled        = true
}

resource "oci_core_route_table" "rt" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.devops.id
  display_name   = "DevOps-RT"

  route_rules {
    destination       = "0.0.0.0/0"
    destination_type  = "CIDR_BLOCK"
    network_entity_id = oci_core_internet_gateway.igw.id
  }
}

resource "oci_core_security_list" "sl" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.devops.id
  display_name   = "DevOps-SL"

  egress_security_rules {
    destination = "0.0.0.0/0"
    protocol    = "all"
  }

  ingress_security_rules {
    protocol = "6" # TCP
    source   = var.my_ip_cidr
    tcp_options {
      min = 22
      max = 22
    }
  }

  ingress_security_rules {
    protocol = "6"
    source   = "0.0.0.0/0"
    tcp_options {
      min = 80
      max = 80
    }
  }
}

resource "oci_core_network_security_group" "sg" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.devops.id
  display_name   = "DevOps-SG"
}

resource "oci_core_network_security_group_security_rule" "ssh" {
  network_security_group_id = oci_core_network_security_group.sg.id
  direction                = "INGRESS"
  protocol                 = "6"
  source                   = var.my_ip_cidr
  source_type              = "CIDR_BLOCK"

  tcp_options {
    destination_port_range {
      min = 22
      max = 22
    }
  }
}

resource "oci_core_subnet" "pub" {
  compartment_id             = var.compartment_ocid
  vcn_id                     = oci_core_vcn.devops.id
  display_name               = "DevOps-Pub-Sub"
  cidr_block                 = "10.0.1.0/24"
  route_table_id             = oci_core_route_table.rt.id
  security_list_ids          = [oci_core_security_list.sl.id]
  prohibit_public_ip_on_vnic = false
}
```

Run it:

```bash
terraform init
terraform plan  -var="region=<your-region>" -var="compartment_ocid=<ocid>" -var="my_ip_cidr=<ip>/32"
terraform apply -var="region=<your-region>" -var="compartment_ocid=<ocid>" -var="my_ip_cidr=<ip>/32"
```

---

## 10. Key takeaways

1. A VCN is empty and isolated until you add gateways and routes.
2. Internet access = IGW + route rule + public IP + allow rule.
3. Always attach the custom route table and security list to the subnet. Creating them is not enough.
4. Security lists apply per subnet and NSGs per VNIC. Both are allow-only and additive.
5. Restrict SSH to your own IP and delete lab resources when finished.
