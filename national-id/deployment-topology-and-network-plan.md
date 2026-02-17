# Deployment Topology & Network Plan

This page describes **where components are deployed, how we secure administrative access, and how DNS and ingress** routes traffic across the **Observation** and **MOSIP** Kubernetes clusters for São Tomé & Príncipe (STP).

***

### 1. Why this topology

We use a **two-cluster model**:

* **Observation Cluster**: hosts operational tooling (e.g., Rancher UI, ops IAM) used to manage and observe Kubernetes.
* **MOSIP Cluster**: hosts MOSIP platform workloads and MOSIP-facing endpoints.

**Why**: separating operations tooling from the mission workload reduces blast radius (ops tools changes don’t impact MOSIP runtime), simplifies access policy (ops endpoints stay private), and keeps the MOSIP cluster focused on platform runtime + integrations.

***

### 2. High-level architecture

#### 2.1 Components (logical)

* **WireGuard Bastion** (admin VPN): private, authenticated tunnel for admins and deployment automation.
  * **Why**: WireGuard provides a modern, secure VPN tunnel with a small attack surface and is well-suited to tightly controlled administrative access.
  * Reference: WireGuard overview. (See public documentation.)
* **Observation Cluster (Kubernetes via RKE)**:
  * **Rancher** (cluster management UI)
    * **Why**: Rancher simplifies multi-cluster Kubernetes lifecycle management (import/monitor/operate clusters centrally).
    * Reference: Rancher “What is Rancher?” documentation.
  * **Keycloak (ops IAM)** (optional / if used for ops)
    * **Why**: centralizes admin authentication/authorization for ops dashboards and reduces shared credentials.
  * **Nginx (Observation LB / reverse proxy)** for private TLS termination and routing.
    * **Why**: stable, auditable entry point for private admin UIs; supports TLS termination and consistent routing rules.
* **MOSIP Cluster (Kubernetes via RKE)**:
  * **Istio Ingress Gateways** (internal + external)
    * **Why**: The Istio Gateway provides controlled edge ingress to the mesh and allows us to apply consistent policies and routing for MOSIP services.
    * Reference: Istio Gateway / Ingress docs.
  * **Nginx (MOSIP LB / reverse proxy)** in front of Istio
    * **Why**: provides TLS termination strategy, domain-level exposure policy (private vs public), and stable load-balancing in on-prem/cloud environments.
  * **Storage (NFS + client provisioner)** for persistent volumes
    * **Why**: provides a practical shared storage backend for on-prem clusters, enabling PV provisioning for stateful workloads.
  * **MOSIP platform modules & external dependencies** (installed via scripts/Helm).

***

### 3. Network zones and access policy

#### 3.1 Recommended zones (conceptual)

* **Admin Zone**: admin laptops / CI runners
* **Bastion Zone**: WireGuard VM (public IP) + admin jump access
* **Private Cluster Zone**: Observation cluster nodes, MOSIP cluster nodes, Nginx LBs, NFS VM(s)
* **Public Zone**: only selected MOSIP endpoints that must be internet-accessible

#### 3.2 Access rules (baseline)

* **All private dashboards and internal APIs** are accessible **only via WireGuard** (or a strict admin CIDR allowlist).
* **Public endpoints** must be protected using:
  * TLS
  * WAF / rate limiting (where available)
  * IP allowlisting for partner APIs when possible

***

### 4. Traffic flow

#### 4.1 Administrative access flow (private)

```mermaid
flowchart LR
  A[Admin Laptop] -->|WireGuard VPN| B[WireGuard Bastion]
  B -->|Private routing| C[Observation Nginx / LB]
  C --> D[Rancher UI]
  C --> E[Ops IAM / Keycloak]
  B -->|Private routing| F[MOSIP Nginx / LB (Private)]
  F --> G[Istio IngressGateway - internal]
  G --> H[MOSIP Internal Services]
```

**Why this flow**: VPN-first access ensures admin-only surfaces (Rancher, Keycloak, admin portals, internal APIs) are not exposed to the internet.

#### 4.2 Public MOSIP access flow (only if required)

```mermaid
flowchart LR
  U[Internet Users / Partners] -->|HTTPS 443| P[MOSIP Nginx / LB (Public)]
  P --> Q[Istio IngressGateway - external]
  Q --> R[MOSIP Public Services]
```

**Why this flow**: the MOSIP Nginx LB is the single controlled edge; Istio then governs routing within the cluster.

***

### 5. Cluster topology (on-prem reference)

#### 5.1 RKE-based Kubernetes clusters

Both clusters are created using **RKE** (Rancher Kubernetes Engine).

* **Why**: RKE simplifies Kubernetes installation and automation on bare-metal/VMs and is commonly used for on-prem deployments.\
  Reference: RKE overview documentation.

**Observation cluster key commands**

Create the RKE cluster configuration:

```bash
rke config
```

Edit the generated `cluster.yml`:

```bash
vi cluster.yml
```

Disable default ingress installation:

```yaml
ingress:
  provider: none
```

Bring up the cluster:

```bash
rke up
```

Set kubeconfig (either option):

```bash
cp $HOME/.kube/<cluster_name>_config $HOME/.kube/config
```

```bash
export KUBECONFIG="$HOME/.kube/<cluster_name>_config"
```

Verify cluster access:

```bash
kubectl get nodes
```

Save these files securely (needed for maintenance/upgrades):

* `cluster.yml`
* `kube_config_cluster.yml`
* `cluster.rkestate`

***

### 6. Ingress and edge routing strategy

#### 6.1 Observation cluster ingress (nginx-ingress)

We deploy `ingress-nginx` in the Observation cluster to expose internal tools behind a consistent ingress layer.

* **Why**: `ingress-nginx` provides a standard Kubernetes ingress controller for private admin UIs and can integrate cleanly with a VM-based reverse proxy/LB.

Commands:

```bash
cd $K8_ROOT/rancher/on-prem
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install \
  ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --version 4.0.18 \
  --create-namespace \
  -f ingress-nginx.values.yaml
kubectl get all -n ingress-nginx
```

#### 6.2 MOSIP cluster ingress (Istio)

We install Istio components (including ingress gateways) on the MOSIP cluster.

* **Why**: Istio gateways provide policy-driven, explicit ingress control and a clean separation between _port exposure_ (Gateway) and _routing rules_ (VirtualServices).

Commands:

```bash
cd $K8_ROOT/mosip/on-prem/istio
./install.sh
kubectl get svc -n istio-system
```

Expected services include (names may vary by installation scripts):

* `istio-ingressgateway` (external)
* `istio-ingressgateway-internal` (internal)
* `istiod`

***

### 7. Persistent storage (NFS)

We use an NFS server VM + NFS client provisioner for dynamic PV provisioning.

* **Why**: NFS is a pragmatic shared storage layer for on-prem Kubernetes deployments and supports multiple pods accessing shared volumes where needed.

#### 7.1 NFS server setup (on NFS VM)

Prepare Ansible inventory:

```bash
cd $K8_ROOT/mosip/nfs
cp hosts.ini.sample hosts.ini
```

SSH and install:

```bash
ssh -i ~/.ssh/nfs-ssh.pem ubuntu@<internal ip of nfs server>
git clone https://github.com/mosip/k8s-infra -b v1.2.0.1
cd /home/ubuntu/k8s-infra/mosip/nfs/
sudo ./install-nfs-server.sh
```

When prompted:

* Environment name: `<envName>`
* NFS path: `/srv/nfs/mosip/<envName>`

#### 7.2 NFS client provisioner (run from bastion)

```bash
cd $K8_ROOT/mosip/nfs/
./install-nfs-client-provisioner.sh
```

Post-checks:

```bash
kubectl -n nfs get deployment.apps/nfs-client-provisioner
kubectl get storageclass
```

***

### 8. DNS and endpoint exposure model

You already defined your **DNS mapping table** on the Pre-requisites page. In topology terms:

* **Observation DNS** → points to **Observation Nginx/LB private IP**
* **MOSIP Private DNS** → points to **MOSIP Nginx/LB private IP**
* **MOSIP Public DNS (only if required)** → points to **MOSIP Nginx/LB public IP**

**Why**: clean separation of admin/private endpoints vs internet-facing endpoints makes it easy to enforce policies and audit exposure.

***

### 9. Nginx LBs (VM-based reverse proxies)

We run VM-based Nginx as the stable edge:

* **Observation Nginx**: terminates TLS for Rancher/Keycloak; maps domains like `rancher.<domain>` and `keycloak.<domain>`
* **MOSIP Nginx**: terminates TLS and routes domains (private + public) into Istio gateways

**Why**: a VM-based LB is often simpler to operate in on-prem environments, supports certificate management workflows, and provides a single, auditable place to enforce exposure policy.

***

### 10. Connectivity validation (Nginx ↔ Istio wiring)

Use the `httpbin` utility to confirm request flow from domains into the cluster and validate header propagation.

Install httpbin:

```bash
cd $K8_ROOT/utils/httpbin
./install.sh
```

Test internal and public paths (replace with your domains):

```bash
curl https://api.<env>.<domain>/httpbin/get?show_env=true
curl https://api-internal.<env>.<domain>/httpbin/get?show_env=true
```

***

### 11. What should be ready before you proceed to the next page

* WireGuard tunnel operational for admins
* Observation cluster created and reachable (`kubectl get nodes`)
* MOSIP cluster created and reachable (`kubectl get nodes`)
* Ingress strategy installed (nginx-ingress for Observation; Istio for MOSIP)
* NFS storage class available (if used)
* DNS records planned and created for private vs public exposure
