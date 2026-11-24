# Phase 4 (Advanced, Manual) – Create the OKE Kubernetes Cluster

Build a private, production-style **Oracle Kubernetes Engine (OKE)** cluster for **ShopSphere** on the network from Phase 2, administered securely through the bastion from Phase 3. Everything is done **by hand** in the OCI Console and a terminal.

> **Builds on Phases 1–3.**
> - **Compartments:** the cluster and both node pools go in **`shopsphere-platform`**. The VCN and subnets stay in `shopsphere-network`.
> - **Subnets:** API endpoint → `shopsphere-oke-api-subnet`; workers → `shopsphere-oke-worker-subnet`; Kubernetes load balancers → `shopsphere-lb-subnet`.
> - **NSGs:** `shopsphere-oke-api-nsg`, `shopsphere-oke-worker-nsg`, `shopsphere-lb-nsg` (Phase 2).
> - **Admin access:** `shopsphere-bastion` with port forwarding (Phase 3). **SSH key:** `~/.ssh/shopsphere_node.pub` (Phase 3).
> - **Dynamic group:** `shopsphere-oke-nodes` already matches the `shopsphere-platform` compartment (Phase 3, section 9).

> **How to use this guide:** Work top to bottom, tick each ✅ **Verify**, and record OCIDs in your git-ignored notes file (`docs/phase4-notes.md`).

---

## Table of Contents

1. [Design and Decisions](#1-design-and-decisions)
2. [Pre-flight Checks](#2-pre-flight-checks)
3. [IAM Check](#3-iam-check)
4. [Create the Cluster](#4-create-the-cluster)
5. [Create the System Node Pool](#5-create-the-system-node-pool)
6. [Create the Application Node Pool](#6-create-the-application-node-pool)
7. [Configure Secure Kubernetes Access](#7-configure-secure-kubernetes-access)
8. [Validate the Cluster](#8-validate-the-cluster)
9. [Test Load Balancer Path (Optional)](#9-test-load-balancer-path-optional)
10. [Production Configuration](#10-production-configuration)
11. [Cost Controls](#11-cost-controls)
12. [Validation Checklist](#12-validation-checklist)
13. [Troubleshooting](#13-troubleshooting)
14. [Pitfalls and Hardening Checklist](#14-pitfalls-and-hardening-checklist)
15. [Appendix: VCN-Native Pod Networking](#15-appendix-vcn-native-pod-networking)

---

## 1. Design and Decisions

### 1.1 Topology

```text
shopsphere-oke-cluster  (Enhanced, private API endpoint)
    │
    ├── shopsphere-system-pool   → system add-ons and shared tooling
    │
    └── shopsphere-app-pool      → ShopSphere application pods
```

### 1.2 Why two node pools

| Reason | Benefit |
|---|---|
| Separation of concerns | Add-ons do not compete with application pods |
| Independent scaling | Grow the app pool without touching system nodes |
| Cost control | Keep the system pool small (1–2 nodes) |
| Targeted placement | Labels (and optionally taints) steer pods to the right pool |

### 1.3 Decisions to make **before** you click Create

| Decision | Choice in this guide | Alternatives |
|---|---|---|
| Cluster type | **Enhanced** | Basic (free control plane, fewer features) |
| API endpoint | **Private**, in `shopsphere-oke-api-subnet` | Public (not recommended) |
| Pod networking | **Flannel overlay** (matches the Phase 2 NSG design) | VCN-native pod networking (see appendix 15) |
| Kubernetes version | Newest version the Console offers as supported | Avoid versions close to end of support |
| Worker shape | One architecture for the whole cluster (see 1.5) | Mixed pools need multi-arch images |
| Node count | 1 system + 1 app to start, 2 + 2 for HA | Scale later |

### 1.4 Corrections to common explanations

| Common statement | Better picture |
|---|---|
| "Compartment `shopsphere-prod`" | This project uses `shopsphere-platform` for the cluster, `shopsphere-network` for the VCN |
| "Phase 3 built the VCN" | The VCN came from **Phase 2**; Phase 3 was the bastion |
| "OKE auto-creates NSGs in Custom Create" | In **Custom Create** you choose your own subnets and NSGs. Auto-creation is a **Quick Create** behavior |
| "CoreDNS and kube-proxy run on the system pool" | They run **wherever the scheduler places them**. Pinning needs labels, selectors or taints |
| "Enhanced is free like Basic" | Enhanced clusters carry a **control plane charge per cluster-hour**. Check current pricing against your budget |
| "HPA works out of the box" | HPA needs a **metrics source** such as Metrics Server |
| "Expected `kubectl get nodes` shows ROLES `node`" and a fixed version like `v1.28.x` | Role and version vary. Use the version your cluster actually runs |
| "NetworkPolicy works on any CNI" | Enforcement needs a policy engine. Flannel alone does **not** enforce policies |

### 1.5 Architecture caveat (Free Tier Ampere)

Ampere **A1** shapes are **ARM64**. Every image (apps, sidecars, init containers, add-ons you install) must have a `linux/arm64` variant. x86 shapes (E4 and others) run `linux/amd64`.

Check an image's architectures:

```bash
docker manifest inspect <image>:<tag> | grep -i architecture
```

Build multi-arch images if needed:

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t <registry>/<image>:<tag> --push .
```

> **Do not mix architectures casually.** If one pool is ARM and another x86, every deployment needs a `nodeSelector` on `kubernetes.io/arch` or multi-arch images.

✅ **Verify:** You have written down your decisions (type, pod networking, shape, node counts) in `docs/adr/0004-oke-design.md`.

---

## 2. Pre-flight Checks

| Check | How |
|---|---|
| Phase 2 resources all **Available** | Networking → `shopsphere-vcn` |
| NSG rules from Phase 2 present | Worker 12, API 7, LB 4 rules |
| Bastion CIDR rules added for real targets | Phase 3, section 6.2 (API 6443 and worker 22 from `10.0.5.0/24`) |
| Compartment `shopsphere-platform` **Active** | Identity & Security → Compartments |
| Service limits adequate | Governance → **Limits, Quotas and Usage** (check your chosen shape and OKE cluster count) |
| CLI session valid | `oci session validate --profile SHOPSPHERE --local` |
| SSH key ready | `ls -l ~/.ssh/shopsphere_node.pub` |
| Budget headroom | Phase 1 budget vs expected cluster cost |

✅ **Verify:** Region selector shows your chosen region, you can open `shopsphere-platform`, and limits show room for the shape and nodes you plan.

---

## 3. IAM Check

`shopsphere-admins` already has `manage all-resources` on `Shopsphere` (Phase 1), which covers `shopsphere-platform` and its networking use. **No new admin policy is needed.**

### 3.1 Confirm developer access (optional)

Phase 1 gave `shopsphere-developers`:

```text
Allow group shopsphere-developers to manage cluster-family in compartment shopsphere-platform
Allow group shopsphere-developers to use virtual-network-family in compartment shopsphere-network
```

That is enough for them to **work with** clusters. Whether they may **act inside Kubernetes** is separate (section 10.3, Kubernetes RBAC).

### 3.2 Two layers of access

| Layer | Controlled by | Example |
|---|---|---|
| OCI | IAM policies | Create/delete clusters and node pools, generate kubeconfig |
| Kubernetes | RBAC | `get pods`, `create deployments` |

> The cluster's creator gets cluster-admin. Anyone else needs an explicit RBAC binding (section 10.3).

✅ **Verify:** You can open **Developer Services → Kubernetes Clusters (OKE)** without an authorization error while in `shopsphere-platform`.

---

## 4. Create the Cluster

### 4.1 Open OKE

Menu (☰) → **Developer Services** → **Kubernetes Clusters (OKE)**. (On some Console versions the group is named **Developer Services & AI**; use the search bar if it moved.)

### 4.2 Start the wizard

1. Set **Compartment** to `shopsphere-platform`.
2. Click **Create cluster**.
3. Choose **Custom create** → **Submit/Next**. (Quick create would build its own VCN, which you do not want.)

### 4.3 Cluster basics

| Field | Value |
|---|---|
| Name | `shopsphere-oke-cluster` |
| Compartment | `shopsphere-platform` |
| Kubernetes version | Newest supported version offered |
| Cluster type | **Enhanced** |
| Tags | `shopsphere.environment`, `shopsphere.owner`, `shopsphere.cost-center` |

### 4.4 Network setup

| Field | Value |
|---|---|
| VCN compartment | `shopsphere-network` |
| VCN | `shopsphere-vcn` |
| Kubernetes API endpoint subnet | `shopsphere-oke-api-subnet` |
| **Assign a public IP address to the API endpoint** | **No** (private endpoint) |
| Use network security groups to control traffic | ✅ Yes |
| API endpoint NSG | `shopsphere-oke-api-nsg` |
| Kubernetes service LB subnets | `shopsphere-lb-subnet` |

> If the dialog asks whether to **allow OKE to modify security rules**, choose **not to** where possible. You manage the NSG rules yourself.

### 4.5 Pod networking and add-ons

| Field | Value |
|---|---|
| Pod networking (CNI) | **Flannel overlay** *(or follow appendix 15 for VCN-native)* |
| Kubernetes services CIDR | Default (`10.96.0.0/16`) |
| Pod CIDR | Default (`10.244.0.0/16`) |
| Add-ons | Keep the defaults (CoreDNS, kube-proxy); optionally enable **Kubernetes Metrics Server** if offered |

> These service and pod CIDRs are **virtual** (overlay). They must **not overlap** with `10.0.0.0/16` or with any network you may peer later. The defaults above do not.

### 4.6 Review and create

1. Skip creating a node pool in this wizard if the Console lets you (you create two pools next). If it forces one, create the system pool here using section 5 settings.
2. Review the summary: private endpoint, correct subnets, correct NSG.
3. Click **Create cluster**. Provisioning takes roughly 10–15 minutes.
4. When **Active**, copy the cluster OCID as `CLUSTER_OCID` and the **private endpoint IP** as `API_PRIVATE_IP`.

> The control plane is **managed by Oracle**. You never SSH to it; you only manage node pools and Kubernetes objects.

✅ **Verify:** Cluster state is **Active**, endpoint type is **Private**, API NSG is `shopsphere-oke-api-nsg`, type is **Enhanced**.

---

## 5. Create the System Node Pool

Open the cluster → **Node pools** → **Add node pool**.

### 5.1 Basics

| Field | Value |
|---|---|
| Name | `shopsphere-system-pool` |
| Compartment | `shopsphere-platform` |
| Node type | **Managed** |
| Kubernetes version | Same as the cluster |

### 5.2 Placement and sizing

| Field | Value |
|---|---|
| Virtual cloud network | `shopsphere-vcn` |
| Worker node subnet | `shopsphere-oke-worker-subnet` |
| Availability domains / fault domains | Spread across what your region offers (use **fault domains** in single-AD regions) |
| Number of nodes | **1** to start, **2** for HA |
| Network security group | `shopsphere-oke-worker-nsg` |
| **Assign a public IP to nodes** | **No** |

### 5.3 Shape and image

| Field | Value |
|---|---|
| Shape | Pick per section 1.5, for example `VM.Standard.E4.Flex` (x86) or `VM.Standard.A1.Flex` (ARM) |
| OCPUs / memory | Small for the system pool (for example 1–2 OCPUs, 8–16 GB) |
| Image | Latest **Oracle Linux OKE worker image** that matches the shape architecture and Kubernetes version |
| Boot volume | Default size is fine |
| SSH public key | Paste `shopsphere_node.pub` |

### 5.4 Labels and upgrade behavior

| Field | Value |
|---|---|
| Kubernetes labels (initial) | `shopsphere/pool=system` |
| Cordon and drain | Enabled, with a sensible grace period (for example 60 minutes) |
| Tags | Phase 1 tags (`shopsphere.environment = dev`) |

> **Taints:** If the pool settings offer **Kubernetes taints**, add `shopsphere/pool=system:NoSchedule` so only pods that tolerate it land here. If your Console does not offer taints, do **not** apply them by `kubectl` alone: manual taints are **lost when nodes are replaced**. Use labels plus `nodeSelector` instead.

### 5.5 Create

Click **Add** / **Create** and wait for nodes to become **Active** and, later, **Ready**.

✅ **Verify:** Pool shows the expected node count in **Active**, no node has a public IP.

---

## 6. Create the Application Node Pool

Add a second pool with the same flow:

| Field | Value |
|---|---|
| Name | `shopsphere-app-pool` |
| Compartment | `shopsphere-platform` |
| Subnet | `shopsphere-oke-worker-subnet` |
| NSG | `shopsphere-oke-worker-nsg` |
| Public IP | **No** |
| Nodes | 1 to start, 2 for HA |
| Shape | Same **architecture** as the system pool; size for the workload (for example 2–4 OCPUs, 16–32 GB) |
| Image | Matching OKE worker image |
| SSH key | `shopsphere_node.pub` |
| Labels | `shopsphere/pool=app` |

> Why the same subnet and NSG? Both pools are workers. Separation is done by **pool, label and Kubernetes scheduling**, not by network. Split them into separate subnets only when you need different network rules.

✅ **Verify:** Both pools are **Active**, nodes are in `shopsphere-oke-worker-subnet`, and none has a public IP.

---

## 7. Configure Secure Kubernetes Access

The API endpoint is **private**, so you reach it through the **Bastion** (Phase 3).

### 7.1 Install the tools

The OCI CLI is already installed from Phase 1. Install `kubectl` and Helm:

```bash
sudo snap install kubectl --classic
sudo snap install helm --classic
```

Verify:

```bash
kubectl version --client
helm version
oci --version
```

> `kubectl` should be within one minor version of the cluster. If snap lags behind, install `kubectl` from the Kubernetes release site instead.

### 7.2 Generate the kubeconfig

1. Open the cluster → **Access Cluster**.
2. Choose **Local Access** and copy the displayed command. Its shape is:

```bash
oci ce cluster create-kubeconfig \
  --cluster-id <CLUSTER_OCID> \
  --file $HOME/.kube/config \
  --region <region> \
  --token-version 2.0.0 \
  --kube-endpoint PRIVATE_ENDPOINT
```

3. Keep your Phase 1 session environment exported so the token helper can authenticate:

```bash
export OCI_CLI_PROFILE=SHOPSPHERE
export OCI_CLI_AUTH=security_token
```

4. Protect the file:

```bash
chmod 600 ~/.kube/config
```

> Use the command **the Console shows** rather than retyping it, since it includes the exact endpoint option for your cluster.

### 7.3 Open a Bastion tunnel to the API endpoint

1. Bastion → `shopsphere-bastion` → **Create session**:

| Field | Value |
|---|---|
| Session type | **SSH port forwarding** |
| Name | `oke-api-6443` |
| IP address | `API_PRIVATE_IP` |
| Port | `6443` |
| SSH key | `shopsphere_bastion.pub` |
| TTL | 60 minutes or less |

2. **Terminal 1:** run the copied tunnel command, with local port `6443`:

```bash
ssh -i ~/.ssh/shopsphere_bastion -N -L 6443:<API_PRIVATE_IP>:6443 \
  -p 22 <SESSION_OCID>@host.bastion.<region>.oci.oraclecloud.com
```

3. **Check the API certificate's names** (Terminal 2):

```bash
openssl s_client -connect 127.0.0.1:6443 </dev/null 2>/dev/null | openssl x509 -noout -ext subjectAltName
```

### 7.4 Point kubectl at the tunnel

`kubectl` must connect to `127.0.0.1` while the certificate is checked against a name it contains. Take the cluster name from `kubectl config get-clusters`, then:

```bash
kubectl config set-cluster <cluster-name-from-kubeconfig> \
  --server=https://127.0.0.1:6443 \
  --tls-server-name=<API_PRIVATE_IP>
```

> If you get a certificate error, use a **name or IP that appears in the SAN list** from 7.3 for `--tls-server-name`. Do **not** use `--insecure-skip-tls-verify`.

### 7.5 First contact

```bash
kubectl get nodes
```

Expected shape (versions and names will differ):

```text
NAME          STATUS   ROLES    AGE   VERSION
10.0.2.x      Ready    node     5m    v1.xx.x
```

> `STATUS = Ready` means the kubelet is healthy and the node joined. The count must equal the nodes you created in sections 5 and 6, not a fixed number.

### 7.6 Per-session reminder

Each Bastion session expires. For the next work session: create a new port-forward session, run the new tunnel command, and `kubectl` works again with no config change.

✅ **Verify:** `kubectl get nodes` lists every node as **Ready** through the tunnel.

---

## 8. Validate the Cluster

### 8.1 Core checks

| Command | What it proves |
|---|---|
| `kubectl cluster-info` | You are talking to the right control plane; CoreDNS endpoint is shown |
| `kubectl get namespaces` | API reachable; default namespaces exist (`default`, `kube-system`, `kube-public`, `kube-node-lease`) |
| `kubectl get nodes -o wide` | Nodes **Ready**, private internal IPs, expected OS image and runtime |
| `kubectl get nodes -L kubernetes.io/arch,shopsphere/pool` | Architecture and pool label per node |
| `kubectl get pods -A -o wide` | System pods **Running** (or **Completed** for jobs) |
| `kubectl get --raw='/readyz?verbose'` | API server readiness checks |

> **Healthy baseline:** all nodes `Ready`; system pods (CoreDNS, kube-proxy, CNI) `Running`. Any `CrashLoopBackOff` or `Pending` points to a networking or scheduling problem (section 13).

### 8.2 No public IPs on nodes

```bash
kubectl get nodes -o wide
```

The **EXTERNAL-IP** column should show `<none>`.

### 8.3 DNS check

```bash
kubectl run dns-test --image=busybox:1.36 --restart=Never --rm -it -- nslookup kubernetes.default
```

Expected: the cluster service IP resolves. (If you use ARM nodes, `busybox:1.36` is multi-arch.)

### 8.4 Egress check (through NAT)

```bash
kubectl run egress-test --image=curlimages/curl --restart=Never --rm -it -- curl -s -m 10 https://ifconfig.me
```

Expected: the **NAT gateway's** public IP from Phase 2, not a node IP.

### 8.5 Image pull check

Pull from a public registry (the commands above do this). If you plan to use OCIR, push a small image and pull it from the cluster in the next phase; that validates the service gateway path.

### 8.6 Confirm admin rights

```bash
kubectl auth can-i '*' '*' --all-namespaces
```

Expected: `yes` for the cluster creator.

✅ **Verify:** All commands above succeed and match the expected results.

---

## 9. Test Load Balancer Path (Optional)

This proves the chain **internet → LB → worker NodePort** across the Phase 2 NSG rules. It creates a billable load balancer, so **delete it afterwards**.

### 9.1 Prepare

1. Temporarily add to `shopsphere-lb-nsg`: ingress TCP **80** from `<ADMIN_IP>/32` (narrower than the optional world-open rule in Phase 2).
2. Copy the OCIDs of `shopsphere-lb-subnet` and `shopsphere-lb-nsg`.

### 9.2 Manifest

Save as `lb-test.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lb-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: lb-test
spec:
  replicas: 2
  selector:
    matchLabels: { app: nginx }
  template:
    metadata:
      labels: { app: nginx }
    spec:
      nodeSelector:
        shopsphere/pool: app
      containers:
        - name: nginx
          image: nginx:stable
          ports: [{ containerPort: 80 }]
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits:   { cpu: 200m, memory: 128Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: lb-test
  annotations:
    oci.oraclecloud.com/load-balancer-type: "lb"
    service.beta.kubernetes.io/oci-load-balancer-shape: "flexible"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-min: "10"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-max: "10"
    service.beta.kubernetes.io/oci-load-balancer-subnet1: "<LB_SUBNET_OCID>"
    oci.oraclecloud.com/oci-network-security-groups: "<LB_NSG_OCID>"
    oci.oraclecloud.com/security-rule-management-mode: "None"
spec:
  type: LoadBalancer
  selector: { app: nginx }
  ports:
    - port: 80
      targetPort: 80
```

> `security-rule-management-mode: "None"` tells OKE not to edit security rules, because you manage the NSGs. Annotation names can change, so confirm them in the current OKE load balancer documentation if the service does not provision.

### 9.3 Run

```bash
kubectl apply -f lb-test.yaml
kubectl get svc -n lb-test -w
```

When **EXTERNAL-IP** appears:

```bash
curl -I http://<EXTERNAL_IP>
```

Expected: `HTTP/1.1 200 OK` from nginx.

### 9.4 Clean up

```bash
kubectl delete -f lb-test.yaml
```

Then **remove the temporary port 80 rule** from `shopsphere-lb-nsg` and confirm the load balancer is gone in **Networking → Load balancers**.

✅ **Verify:** The request succeeded, then the LB and temporary rule were removed.

---

## 10. Production Configuration

Do these in a dedicated namespace. Create it first:

```bash
kubectl create namespace shopsphere
kubectl label namespace shopsphere \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted
```

### 10.1 Resource governance

`resources.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: shopsphere-quota
  namespace: shopsphere
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "40"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: shopsphere-defaults
  namespace: shopsphere
spec:
  limits:
    - type: Container
      defaultRequest: { cpu: 100m, memory: 128Mi }
      default:        { cpu: 500m, memory: 512Mi }
```

```bash
kubectl apply -f resources.yaml
```

> Adjust numbers to your node sizes. Requests and limits are required for autoscaling and decide QoS class (Guaranteed, Burstable, BestEffort).

### 10.2 Probes, rolling updates, disruption budgets

Every application Deployment should include:

```yaml
        readinessProbe:
          httpGet: { path: /healthz, port: 8080 }
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet: { path: /healthz, port: 8080 }
          initialDelaySeconds: 15
          periodSeconds: 20
```

```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

| Setting | Purpose |
|---|---|
| Readiness | Removes a pod from Service endpoints until ready, so no traffic reaches warming pods |
| Liveness | Restarts a container that is deadlocked |
| `maxUnavailable: 0` | Zero-downtime image updates |

Pod Disruption Budget (example for a service called `catalog`):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: catalog-pdb
  namespace: shopsphere
spec:
  minAvailable: 1
  selector:
    matchLabels: { app: catalog }
```

> A PDB with `minAvailable: 1` on a **single-replica** deployment blocks node drains. Run at least 2 replicas for services that have a PDB.

### 10.3 Kubernetes RBAC

The creator already has cluster-admin. Grant others least privilege, **bound to IAM user OCIDs** (OKE token authentication identifies users by OCID).

`rbac.yaml` (replace the OCID):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: shopsphere-developer
  namespace: shopsphere
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["pods", "pods/log", "services", "deployments", "replicasets", "jobs", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: shopsphere-developer-binding
  namespace: shopsphere
subjects:
  - kind: User
    name: ocid1.user.oc1..xxxx
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: shopsphere-developer
  apiGroup: rbac.authorization.k8s.io
```

> Check Oracle's current docs on whether your setup supports group-based bindings through identity domains. User OCID bindings always work. Keep `Secret` access out of general developer roles.

Test as the developer (impersonation from your admin session):

```bash
kubectl auth can-i delete pods -n shopsphere --as ocid1.user.oc1..xxxx
kubectl auth can-i get secrets -n kube-system --as ocid1.user.oc1..xxxx
```

Expected: first `yes`, second `no`.

### 10.4 Autoscaling

| Layer | What | How |
|---|---|---|
| Pods (HPA) | Scale replicas by CPU or memory | Needs **Metrics Server** (enable the OKE add-on or install it) |
| Nodes | Add or remove nodes when pods cannot schedule or nodes sit idle | **Cluster Autoscaler** add-on on the cluster; set **min/max** per node pool |

HPA example:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: catalog-hpa
  namespace: shopsphere
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: catalog }
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

Verify Metrics Server first:

```bash
kubectl top nodes
```

> **Cluster Autoscaler:** In the Console open the cluster → **Add-ons** → enable **Cluster Autoscaler**, then set pool **min/max** under each node pool's scaling settings. It needs workload identity and a policy, so follow Oracle's add-on instructions and use the policy pattern from Phase 3, section 11.

### 10.5 Network policies

Pods are open to each other by default.

| Step | Detail |
|---|---|
| 1. Choose an engine | Flannel overlay does **not** enforce `NetworkPolicy` alone. Install a policy engine (for example Calico or Cilium, per Oracle's current guidance for your CNI) |
| 2. Default deny | Apply a deny-all policy per namespace |
| 3. Allow what is needed | Add explicit allows between services |

Default deny:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: shopsphere
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

> **Careful:** Default-deny **egress** also blocks DNS. Add an allow rule for CoreDNS (UDP/TCP 53 to `kube-system`) **before** applying it to workloads, and test in a throw-away namespace first.

### 10.6 Upgrades

| Item | How |
|---|---|
| Control plane | Cluster → **Upgrade** to the next supported minor version (one minor at a time) |
| Node pools | After the control plane, upgrade each pool (new image/version). Cordon and drain protects workloads when PDBs and replicas are set |
| Order | Control plane first, then workers; never skip minor versions |
| Timing | Do it off-peak and after a test run on dev |

✅ **Verify:** Namespace `shopsphere` exists with quota, RBAC test results match, `kubectl top nodes` works (if Metrics Server is enabled).

---

## 11. Cost Controls

| Control | Action |
|---|---|
| Know the fixed cost | An Enhanced control plane bills per hour even when no nodes run |
| Scale idle pools down | Set `shopsphere-app-pool` (and system pool) to **0** nodes when not studying. Pods will be pending until you scale back up |
| Delete test load balancers | Section 9.4 |
| Right-size nodes | Review **Cost Analysis** by `shopsphere.environment` weekly |
| Budget alerts | Phase 1 budget and forecast alert; adjust the amount if the cluster pushes you over |

> To pause: scale both pools to 0. To destroy completely: delete node pools, then the cluster, then any leftover load balancers and boot volumes.

---

## 12. Validation Checklist

| # | Check | Done |
|---|---|---|
| 1 | Cluster **Active**, **Enhanced**, private API endpoint, API NSG attached | ☐ |
| 2 | System and app pools **Active**; nodes in the worker subnet; **no public IPs** | ☐ |
| 3 | Pools use `shopsphere-oke-worker-nsg` and your SSH key | ☐ |
| 4 | Node architecture is consistent (or all images are multi-arch) | ☐ |
| 5 | Bastion tunnel to `API_PRIVATE_IP:6443` works; `kubectl get nodes` shows all **Ready** | ☐ |
| 6 | `kubectl get pods -A` shows system pods **Running** | ☐ |
| 7 | DNS test resolves; egress test shows the NAT gateway IP | ☐ |
| 8 | Direct access to the API endpoint **without** the tunnel fails | ☐ |
| 9 | Optional LB test returned `200 OK`, then LB and temporary rule were removed | ☐ |
| 10 | Namespace `shopsphere` created with Pod Security labels, quota and limits | ☐ |
| 11 | RBAC test: developer can manage workloads but not read secrets cluster-wide | ☐ |
| 12 | Metrics Server works (`kubectl top nodes`) or is planned | ☐ |
| 13 | Cost review done; budget alerts still sensible | ☐ |
| 14 | Design decisions recorded in `docs/adr/0004-oke-design.md` | ☐ |

**Phase success statement:** The Kubernetes cluster is healthy, and your Ubuntu machine administers it only through the private API endpoint using OCI authentication via a time-limited bastion session, with no public exposure of the control plane or nodes.

---

## 13. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Cluster creation fails with authorization error | Missing IAM permission or wrong compartment | Re-check section 3; read the error for the missing resource type |
| Cannot select the VCN or subnets | Wrong VCN compartment in the wizard | Choose `shopsphere-network` for the VCN compartment |
| Nodes stay `NotReady` or never join | Worker ↔ API rules missing (6443, 12250) | Compare worker and API NSG rules with Phase 2, section 11 |
| Node pool stuck **Creating** | Shape limit, capacity, or image/shape mismatch | Check service limits; pick another shape or AD; use a matching architecture image |
| `kubectl` connection refused | Bastion tunnel not running or session expired | Create a new session and rerun the tunnel command |
| `kubectl` certificate error | `--tls-server-name` not in the certificate | Inspect SANs (7.3) and use a name or IP from that list |
| `You must be logged in to the server` | Expired OCI session token or env vars missing | `oci session refresh --profile SHOPSPHERE`; re-export `OCI_CLI_PROFILE` and `OCI_CLI_AUTH` |
| CoreDNS `Pending` | No schedulable node (resources or taints) | Check node capacity and any taints you added |
| Pods `ImagePullBackOff` | No egress, wrong architecture, or private registry auth | Check NAT route and NSG 443 egress; check `kubernetes.io/arch`; use an image pull secret or OCIR access |
| `exec format error` in a pod | Image architecture does not match node (ARM vs x86) | Use a multi-arch image or a matching node pool |
| Pod-to-pod traffic fails between nodes | Worker NSG self rule missing | Add ingress and egress `All` from/to the worker NSG |
| LB created but unreachable | LB NSG ingress missing, or worker NSG missing NodePort/health rules | Check LB NSG ingress; check worker ingress from LB NSG on 30000-32767 and 10256 |
| LB service stuck pending or authorization error | LB subnet or NSG annotation wrong, or cross-compartment policy missing | Re-check OCIDs; follow Oracle's current OKE docs for cross-compartment network policies |
| Drain blocked during upgrade | PDB with a single replica | Run at least 2 replicas, or relax the PDB |
| `kubectl top` fails | Metrics Server missing | Enable or install it |
| Event rule or alerts missing | Out of scope here | See Phase 3, section 12 |

---

## 14. Pitfalls and Hardening Checklist

### Pitfalls

1. **Public API endpoint "for convenience".** It removes your main perimeter. Use the bastion.
2. **Mixing ARM and x86 nodes without planning.** Images fail with `exec format error`.
3. **Pod or service CIDR overlap** with the VCN or future peered networks.
4. **Letting OKE rewrite security rules** while you also manage NSGs. Pick one owner.
5. **Manual `kubectl taint` on nodes.** It disappears when nodes are replaced.
6. **No resource requests.** Autoscaling and scheduling become unpredictable.
7. **Single replica plus a PDB.** Blocks node upgrades.
8. **Default-deny egress without a DNS allow rule.** Breaks name resolution.
9. **Giving developers cluster-admin.** Use namespace roles.
10. **Forgetting the control plane bill** when the cluster is "idle".
11. **Leaving test load balancers and temporary NSG rules** in place.
12. **Skipping minor versions** during upgrades.

### Hardening checklist

- [ ] Private API endpoint; access only through a bastion session
- [ ] Workers have private IPs only; egress through NAT and service gateway
- [ ] NSGs, not security lists, carry the policy; no `0.0.0.0/0` ingress on workers or API
- [ ] Pod Security Admission labels set on application namespaces
- [ ] Resource quotas, limit ranges, requests and limits on every workload
- [ ] RBAC bound to individuals or groups; no shared cluster-admin
- [ ] Network policy engine installed and default-deny tested per namespace
- [ ] Probes, rolling updates and PDBs defined for every service
- [ ] Metrics Server and autoscaling configured with sensible min/max
- [ ] Control plane and node pool upgrade plan documented
- [ ] Audit and Cloud Guard reviewed for the new resources (Phase 1)
- [ ] Decisions recorded in `docs/adr/0004-oke-design.md`

---

## 15. Appendix: VCN-Native Pod Networking

Choose this instead of Flannel when you want pods to receive **real VCN IP addresses** (better integration with NSGs and VCN flow logs, no overlay). The cost is more IP planning and more rules.

### 15.1 When to choose it

| Pick VCN-native if | Pick Flannel if |
|---|---|
| You want NSG rules that target **pods** | You want the simplest setup that matches Phase 2 |
| You need VCN flow logs per pod | Your pod count is small |
| You will use network policy via a supported engine on that CNI | You can live without VCN-level pod visibility |

> Decide **before** creating the cluster. Changing the CNI later means recreating it.

### 15.2 Extra network resources

1. Create subnet `shopsphere-oke-pod-subnet`, CIDR **`10.0.6.0/24`** (the range reserved in Phase 2; widen it if you expect many pods, since **every pod consumes a VCN IP**), **private**, route table `shopsphere-private-rt`, security list `shopsphere-private-sl`.
2. Create NSG `shopsphere-oke-pod-nsg`.

### 15.3 Extra rules (starting point; confirm against Oracle's current OKE networking guide)

| NSG | Dir | Source / Destination | Protocol | Port | Description |
|---|---|---|---|---|---|
| `shopsphere-oke-pod-nsg` | Ingress | NSG `shopsphere-oke-pod-nsg` | All | All | Pod-to-pod |
| `shopsphere-oke-pod-nsg` | Ingress | NSG `shopsphere-oke-worker-nsg` | All | All | Nodes to pods |
| `shopsphere-oke-pod-nsg` | Ingress | NSG `shopsphere-oke-api-nsg` | All | All | API to pods (webhooks, aggregated APIs) |
| `shopsphere-oke-pod-nsg` | Egress | NSG `shopsphere-oke-pod-nsg` | All | All | Pod-to-pod |
| `shopsphere-oke-pod-nsg` | Egress | Service `All <region> Services In Oracle Services Network` | TCP | 443 | OCI services |
| `shopsphere-oke-pod-nsg` | Egress | CIDR `0.0.0.0/0` | TCP | 443 | Images and APIs via NAT |
| `shopsphere-oke-pod-nsg` | Egress | NSG `shopsphere-oke-api-nsg` | TCP | 6443 | Pods to Kubernetes API |
| `shopsphere-oke-pod-nsg` | Egress | NSG `shopsphere-database-nsg` | TCP | 3306 (or your DB port) | Pods to database |
| `shopsphere-oke-worker-nsg` | Ingress | NSG `shopsphere-oke-pod-nsg` | All | All | Pods to nodes |
| `shopsphere-oke-api-nsg` | Ingress | NSG `shopsphere-oke-pod-nsg` | TCP | 6443 | Pods to API |
| `shopsphere-database-nsg` | Ingress | NSG `shopsphere-oke-pod-nsg` | TCP | 3306 (or your DB port) | **Pods** reach the database, so replace the worker rule with this one |

> With VCN-native networking the **database sees pod IPs, not node IPs**. Move the database ingress rule from the worker NSG to the pod NSG, and remove the old rule so only pods can connect.

### 15.4 Wizard changes

| Field | Value |
|---|---|
| Pod networking | **VCN-native pod networking** |
| In each node pool: **Pod communication subnet** | `shopsphere-oke-pod-subnet` |
| In each node pool: **Pod NSG** | `shopsphere-oke-pod-nsg` |
| Max pods per node | Set deliberately; each pod takes a VCN IP |

✅ **Verify:** After creation, `kubectl get pods -A -o wide` shows pod IPs in `10.0.6.0/24`.

---

## Phase 4 Completion Target

✅ A working ShopSphere OKE cluster with:

- An **Enhanced** cluster with a **private** API endpoint on the Phase 2 network
- Two private node pools (system and application) with labels, NSGs and SSH keys
- Admin access only through the **Bastion**, using OCI authentication
- A validated baseline: nodes **Ready**, system pods **Running**, DNS and NAT egress working
- Namespace-level governance: Pod Security labels, quotas, RBAC, probes, PDBs and a plan for autoscaling and network policy

*Next: Phase 5, container registry (OCIR), the database tier, and deploying the first ShopSphere services with Helm.*
