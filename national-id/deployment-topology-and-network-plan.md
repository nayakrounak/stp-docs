---
description: >-
  Topologia de implementação, zonas de rede, DNS e fluxos de tráfego entre os clusters de Observação
  and MOSIP clusters.
---

# Implementação Topologia & Rede Plano

Esta página descreve **onde os componentes são implementados, como é assegurado o acesso administrativo e como o DNS e o ingress** encaminham o tráfego entre os clusters Kubernetes de **Observação** e **MOSIP** em São Tomé e Príncipe (STP).

## Relacionadas pages

* Arquitetura: [Arquitetura](architecture.md)
* Plataforma pré-requisitos: [Plataforma Pre-requisites](plataforma-pre-requisites.md)
* Ingress & routing: [Ingress & Edge Routing Configuração](ingress-e-edge-routing-configuração.md)

***

### 1. Porquê this topologia

We usar a **two-cluster model**:

* **Observation Cluster**: hosts operacional tooling (cluster management, ops dashboards, ops IAM).
* **MOSIP Cluster**: hosts MOSIP plataforma workloads e MOSIP-facing endpoints.

{% hint style="info" %}
**Porquê two clusters?**\
Keeping “operações” components separate from “mission” components reduces blast radius, allows tighter rede policy para admin ferramentas, e prevents ops UI upgrades/changes from impacting MOSIP runtime.
{% endhint %}

***

### 2. Logical architecture

#### 2.1 Core components

1. **WireGuard Bastion (Admin VPN)**
   * Provides secure privado acesso from admin machines into privado subnets.
   * Reference: WireGuard Quick Começar — https://www.wireguard.com/quickstart/

{% hint style="info" %}
**Porquê WireGuard?**\
WireGuard is a modern VPN designed para be simpler e easier para auditoria than many alternatives, making it suitable para tightly controlled administrative acesso into privado ambientes.
{% endhint %}

2.  **Observation Cluster (Kubernetes)**

    * **Rancher** (cluster management UI)
    * **Ops IAM** (e.g., Keycloak) para admin authentication (if used)
    * **Privado Ingress** (nginx-ingress) para privado dashboards

    References:

    * RKE1 docs (Rancher) — https://rke.docs.rancher.com/installation
    * ingress-nginx (official GitHub) — https://github.com/kubernetes/ingress-nginx

{% hint style="info" %}
**Porquê Rancher/Observation tooling?**\
A centralized cluster management plane reduces operacional friction (acesso, monitorização, node lifecycle, kubeconfig management) e improves auditability. Keeping it privado reduces exposure.
{% endhint %}

3.  **MOSIP Cluster (Kubernetes)**

    * **Istio gateways** para controlled ingress into MOSIP services
    * **MOSIP Nginx LB / reverse proxy** (VM-based) para stable edge routing e TLS termination
    * **Persistent storage** via NFS (server VM + client provisioner) para stateful workloads where required

    References:

    * Istio: Instalar com istioctl — https://istio.io/latest/docs/setup/install/istioctl/
    * NGINX Documentação — https://docs.nginx.com/
    * NFS subdir external provisioner (kubernetes-sigs) — https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

{% hint style="info" %}
**Porquê Istio gateways?**\
Gateways provide explicit, policy-driven ingress control (routing e isolation between internal e external endpoints) e a clean separation between exposure (Gateways) e routing rules (VirtualServices).
{% endhint %}

{% hint style="info" %}
**Porquê a VM-based Nginx LB in front?**\
In on‑prem ou hybrid setups, a VM-based reverse proxy can be the simplest stable “edge” para manage DNS, TLS certificates, allowlists, e exposure policy in one place, independent of cluster churn.
{% endhint %}

{% hint style="info" %}
**Porquê NFS para PVs?**\
NFS is a pragmatic shared storage backend para on‑prem Kubernetes when you need ReadWriteMany-style storage without a full distributed storage plataforma.
{% endhint %}

***

### 3. Rede zones e acesso policy

#### 3.1 Recomendado zones (conceptual)

* **Admin Zone**: admin laptops / CI runners
* **Bastion Zone**: WireGuard VM (público IP) + admin jump acesso
* **Privado Cluster Zone**: Observation & MOSIP cluster nodes, privado LBs, NFS VM(s)
* **Público Zone**: only selected MOSIP endpoints that deve be internet-accessible

#### 3.2 Baseline acesso rules

* **Privado endpoints** (Rancher, ops IAM, `admin.*`, `api-internal.*`, internal dashboards) are reachable **only via WireGuard** e/ou a strict **admin CIDR allowlist**.
* **Público endpoints** (e.g., `api.<env>.<domain>`, `resident.<env>.<domain>` if in scope) deve be protected com:
  * TLS (strong ciphers)
  * WAF / rate limits (where available)
  * IP allowlisting para partner APIs when feasible

{% hint style="info" %}
**Porquê “VPN first” para admin surfaces?**\
As UIs administrativas expõem operações privilegiadas e caminhos de dados sensíveis. Mantê-las fora da Internet pública reduz significativamente a superfície de ataque e simplifica os controlos de conformidade.
{% endhint %}

***

### 4. Traffic fluxo

#### 4.1 Administrative acesso (privado)

```mermaid
flowchart LR
  A[Admin Laptop] -->|WireGuard VPN| B[WireGuard Bastion]
  B -->|Private routing| C[Observation Nginx / LB]
  C --> D[Rancher UI]
  C --> E[Ops IAM (e.g., Keycloak)]
  B -->|Private routing| F[MOSIP Nginx / LB (Private)]
  F --> G[Istio IngressGateway - internal]
  G --> H[MOSIP Internal Services]
```

#### 4.2 Público MOSIP acesso (only if required)

```mermaid
flowchart LR
  U[Internet Users / Partners] -->|HTTPS 443| P[MOSIP Nginx / LB (Public)]
  P --> Q[Istio IngressGateway - external]
  Q --> R[MOSIP Public Services]
```

***

### 5. Cluster provisioning model (RKE)

We provision both clusters utilizando **RKE**.

Reference: RKE1 docs (Rancher) — https://rke.docs.rancher.com/installation

{% hint style="info" %}
**Porquê RKE?**\
RKE disponibiliza a repeatable approach para bringing up Kubernetes on VMs/bare-metal utilizando Docker as the runtime. It’s often used in on‑prem contexts where kubeadm-managed lifecycle ou managed Kubernetes isn’t available.
{% endhint %}

#### 5.1 Observation cluster bootstrap (comandos)

```bash
rke config
vi cluster.yml
rke up
```

Set kubeconfig:

```bash
cp $HOME/.kube/<cluster_name>_config $HOME/.kube/config
# OR
export KUBECONFIG="$HOME/.kube/<cluster_name>_config"
```

Validar:

```bash
kubectl get nodes
```

Operational files para retain securely:

* `cluster.yml`
* `kube_config_cluster.yml` (ou your generated kubeconfig)
* `cluster.rkestate`

***

### 6. Ingress e edge routing

#### 6.1 Observation cluster ingress (ingress-nginx)

References:

* ingress-nginx (official GitHub) — https://github.com/kubernetes/ingress-nginx
* Ingress examples: https://kubernetes.github.io/ingress-nginx/examples/

{% hint style="info" %}
**Porquê ingress-nginx para Observation?**\
It disponibiliza a standard Kubernetes ingress controller para privado dashboards, allowing consistent routing e TLS policies para ops ferramentas.
{% endhint %}

```bash
cd $K8_ROOT/rancher/on-prem
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install   ingress-nginx ingress-nginx/ingress-nginx   --namespace ingress-nginx   --version 4.0.18   --create-namespace   -f ingress-nginx.values.yaml
kubectl get all -n ingress-nginx
```

#### 6.2 MOSIP cluster ingress (Istio)

Reference: Istio: Instalar com istioctl — https://istio.io/latest/docs/setup/install/istioctl/

{% hint style="info" %}
**Porquê Istio on the MOSIP cluster?**\
It centralizes ingress e service-para-service policy enforcement while giving operators strong troubleshooting e tráfego-management primitives.
{% endhint %}

```bash
cd $K8_ROOT/mosip/on-prem/istio
./install.sh
kubectl get svc -n istio-system
```

***

### 7. Persistent storage (NFS)

References:

* NFS subdir external provisioner (kubernetes-sigs) — https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner
* Helm repo: https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

{% hint style="info" %}
**Porquê NFS + external provisioner?**\
This pattern enables dynamic provisioning of Kubernetes Persistent Volumes utilizando an existing NFS server, reducing manual PV creation overhead.
{% endhint %}

#### 7.1 NFS server (on NFS VM)

```bash
cd $K8_ROOT/mosip/nfs
cp hosts.ini.sample hosts.ini
ssh -i ~/.ssh/nfs-ssh.pem ubuntu@<internal ip of nfs server>
git clone https://github.com/mosip/k8s-infra -b v1.2.0.1
cd /home/ubuntu/k8s-infra/mosip/nfs/
sudo ./install-nfs-server.sh
```

When prompted:

* Environment name: `<envName>`
* NFS path: `/srv/nfs/mosip/<envName>`

#### 7.2 NFS client provisioner (from bastion)

```bash
cd $K8_ROOT/mosip/nfs/
./install-nfs-client-provisioner.sh
```

Verify:

```bash
kubectl -n nfs get deployment.apps/nfs-client-provisioner
kubectl get storageclass
```

***

### 8. DNS e exposure model

In topologia terms:

* **Observation DNS** → points para **Observation Nginx/LB privado IP**
* **MOSIP Privado DNS** → points para **MOSIP Nginx/LB privado IP**
* **MOSIP Público DNS (only if required)** → points para **MOSIP Nginx/LB público IP**

{% hint style="info" %}
**Porquê strict separation of privado vs público FQDNs?**\
It makes exposure policy enforceable, easier para auditoria, e reduces accidental publication of internal dashboards e APIs.
{% endhint %}

***

### 9. Connectivity validação (edge → cluster)

We deploy an `httpbin` utility para validate ingress routing e ambiente headers.

{% hint style="info" %}
**Porquê httpbin?**\
It’s a simple echo service that helps validate routing, TLS termination, headers, e query/path behavior without depending on MOSIP app prontidão.
{% endhint %}

```bash
cd $K8_ROOT/utils/httpbin
./install.sh
```

Test (replace com your FQDNs):

```bash
curl https://api.<env>.<domain>/httpbin/get?show_env=true
curl https://api-internal.<env>.<domain>/httpbin/get?show_env=true
```