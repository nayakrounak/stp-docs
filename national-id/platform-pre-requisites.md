---
description: >-
  Minimum prerequisites to prepare infrastructure, access, DNS, and tooling
  antes da implementação do MOSIP.
---

# Plataforma Pre-requisites

Esta secção lista os pré-requisitos mínimos necessários antes de iniciar a implementação do MOSIP para São Tomé e Príncipe (STP). Inclui o **conjunto de ferramentas do bastion host**, o **dimensionamento base das VMs**, o **planeamento de DNS** e a **configuração de acesso seguro (WireGuard + boas práticas de SSH)**.

{% hint style="info" %}
**Antes You Começar**

Fill these values once per ambiente e keep them consistent across all pages:

* **Base domain:** `<domain>` (example: `nid.gov.st`)
* **Environment code:** `<env>` (example: `dev`, `sit`, `uat`, `prod`, `dr`)
* **Observation LB Privado IP:** `<obs_lb_private_ip>`
* **MOSIP LB Privado IP:** `<mosip_lb_private_ip>`
* **MOSIP LB Público IP (if applicable):** `<mosip_lb_public_ip>`
* **WireGuard Público IP:** `<wg_public_ip>`
* **WireGuard Port:** `51820/udp`
* **Admin CIDR allowlist:** `<admin_cidr_list>`
* **SSH key path:** `<ssh_private_key_path>`
* **Container registry:** `<registry_url>`
* **HSM/KMS approach:** `<hsm_or_kms_or_tbd>`
{% endhint %}

***

### 1. Bastion Host Configuração (Tools & Repository)

All implementação activities deverá be executed from a secure **bastion host**.

{% hint style="info" %}
**Porquê a Bastion host?**\
A bastion disponibiliza a single reforçado entry point para administrative acesso, tooling, e auditability, reducing direct exposure of cluster nodes e internal services.
{% endhint %}

#### 1.1 Instalar Git (v2.25.1 ou higher)

{% hint style="info" %}
**Porquê Git?**\
We usar Git para version-control infrastructure scripts, Helm chart values, e implementação configs, enabling repeatable deployments, peer review, e traceability. ([git-scm.com](https://git-scm.com/))
{% endhint %}

```bash
sudo apt update
sudo apt install git
```

#### 1.2 Configure ambiente variables

{% hint style="info" %}
**Porquê ambiente variables?**\
MOSIP instalação scripts e supporting automation typically assume consistent base paths. Standardizing these paths reduces human error e simplifies runbooks e support.
{% endhint %}

```bash
export MOSIP_ROOT=/home/ubuntu
export K8_ROOT=$MOSIP_ROOT/k8s-infra
export INFRA_ROOT=$MOSIP_ROOT/mosip-infra
```

> Dica: Adicione os `export` ao \~/.bashrc se quiser que persistam entre sessões.

#### 1.3 Instalar kubectl

{% hint style="info" %}
**Porquê kubectl?**\
`kubectl` é a ferramenta de linha de comando padrão para interagir com clusters Kubernetes (deployments, logs, troubleshooting). Usamo-la para validar o estado do cluster e operar workloads do MOSIP. ([kubernetes.io](https://kubernetes.io/docs/tasks/tools/))
{% endhint %}

```bash
curl -LO "https://dl.k8s.io/release/v1.34.1/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

#### 1.4 Instalar istioctl (v1.15.0)

{% hint style="info" %}
**Porquê istioctl?**\
When Istio is used para service mesh capabilities (tráfego management, segurança policies, diagnostics), `istioctl` is the recomendado tool para instalação, configuration validação, e troubleshooting. ([istio.io](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/))
{% endhint %}

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.15.0 sh -
cd istio-1.15.0
sudo mv bin/istioctl /usr/local/bin/
istioctl version --remote=false
```

#### 1.5 Instalar RKE (v1.3.10)

{% hint style="warning" %}
**Porquê RKE (e an important note):**\
RKE is a Kubernetes cluster provisioning tool that simplifies on-premises cluster creation. However, RKE1 has reached end of life; new deployments deverá consider RKE2 (ou another supported Kubernetes distribution) unless there is a project constraint that requires remaining on RKE1 para compatibility. ([github.com](https://github.com/rancher/rke))
{% endhint %}

```bash
curl -L "https://github.com/rancher/rke/releases/download/v1.3.10/rke_linux-amd64" -o rke
chmod +x rke
sudo mv rke /usr/local/bin/
rke --version
```

#### 1.6 Instalar Helm (v3.0.0 ou higher) e add Helm repos

{% hint style="info" %}
**Porquê Helm?**\
MOSIP services are packaged as Helm charts para standardize instalação e upgrades on Kubernetes. Helm helps manage complex applications com versionado, repeatable deployments. ([helm.sh](https://helm.sh/docs/))
{% endhint %}

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add mosip https://mosip.github.io/mosip-helm
```

#### 1.7 Instalar Ansible (> 2.12.4)

{% hint style="info" %}
**Porquê Ansible?**\
We usar Ansible para automate repeatable server configuration tasks (e.g., installing Docker, disabling swap, e base node hardening). This reduces manual errors across multiple VMs. ([docs.ansible.com](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html))
{% endhint %}

```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

#### 1.8 Clone MOSIP k8s-infra repo (tag v1.2.0.2)

{% hint style="info" %}
**Porquê MOSIP k8s-infra?**\
The `k8s-infra` repository disponibiliza reference architecture e scripts para deploy Kubernetes-based clusters e supporting infrastructure para MOSIP (e relacionadas DPG components). ([github.com](https://github.com/mosip/k8s-infra))
{% endhint %}

```bash
git clone https://github.com/mosip/k8s-infra -b v1.2.0.2
```

***

### 2. Operating System Baseline

Todas as VMs referidas neste guia assumem o Ubuntu 22.04 como imagem padrão do sistema operativo.

{% hint style="info" %}
**Porquê Ubuntu 22.04 LTS?**\
Ubuntu LTS releases provide long-term maintenance e segurança updates, which is important para stable government infrastructure. ([discourse.ubuntu.com](https://discourse.ubuntu.com/t/jammy-jellyfish-release-notes/24668))
{% endhint %}

***

### 3. Baseline Hardware Requirements (On-Prem Reference)

> Adjust sizing based on expected population, registo concurrency, HA needs, e DR requirements.

{% hint style="info" %}
**Porquê separate Observation vs MOSIP clusters?**\
A separação dos componentes de **Observação/gestão** (por exemplo, gestão do cluster e IAM de operações) do cluster principal de aplicações MOSIP reduz o risco e evita que ferramentas de gestão afetem o desempenho do runtime do MOSIP. As orientações on‑prem do MOSIP também refletem o conceito de cluster de observação para instalar ingress/storage e aplicações de suporte. ([docs.mosip.io](https://docs.mosip.io/1.2.0/setup/deploymentnew/v3-installation/on-prem-installation-guidelines))
{% endhint %}

<table data-header-hidden><thead><tr><th></th><th width="80.0859375"></th><th width="79.921875"></th><th width="86.94921875"></th><th width="87.140625"></th><th width="137.48046875"></th><th></th></tr></thead><tbody><tr><td><strong>Purpose</strong></td><td><strong>vCPU</strong></td><td><strong>RAM</strong></td><td><strong>Storage</strong></td><td><strong># VMs</strong></td><td><strong>HA</strong></td><td><strong>OS</strong></td></tr><tr><td>WireGuard Bastion Host</td><td>2</td><td>4 GB</td><td>8 GB</td><td>1</td><td>Active-Passive</td><td>Ubuntu 22.04</td></tr><tr><td>Observation Cluster nodes</td><td>2</td><td>8 GB</td><td>32 GB</td><td>2</td><td>2</td><td>Ubuntu 22.04</td></tr><tr><td>Observation Nginx / LB</td><td>2</td><td>4 GB</td><td>16 GB</td><td>1</td><td>Nginx+</td><td>Ubuntu 22.04</td></tr><tr><td>MOSIP Cluster nodes</td><td>12</td><td>32 GB</td><td>128 GB</td><td>6</td><td>6</td><td>Ubuntu 22.04</td></tr><tr><td>MOSIP Nginx / LB</td><td>2</td><td>4 GB</td><td>16 GB</td><td>1</td><td>Nginx+</td><td>Ubuntu 22.04</td></tr></tbody></table>

***

### 4. DNS Requirements (Domains & Mappings)

MOSIP deployments require DNS records mapped para:

* **Observation Cluster Nginx/LB** (privado only)
* **MOSIP Cluster Nginx/LB** (privado e público, depending on exposure policy)

{% hint style="info" %}
**Porquê DNS e stable FQDNs?**\
Stable hostnames simplify TLS certificate management, client configuration (e.g., registration clients), e operacional runbooks. MOSIP Helm-based deployments assume consistent endpoint URLs across ambientes. ([docs.mosip.io](https://docs.mosip.io/1.2.0/setup/deploymentnew/getting-started/helm-charts))
{% endhint %}

#### 4.1 DNS Table

<table data-header-hidden><thead><tr><th width="195.19140625"></th><th width="102.19140625"></th><th width="200.1796875"></th><th width="123.08203125"></th><th></th></tr></thead><tbody><tr><td><strong>FQDN Pattern</strong></td><td><strong>Exposure</strong></td><td><strong>Maps To</strong></td><td><strong>Purpose</strong></td><td><strong>Acesso Policy</strong></td></tr><tr><td>rancher.&#x3C;domain></td><td>Privado</td><td>&#x3C;obs_lb_private_ip></td><td>Rancher dashboard</td><td>Only via WireGuard / admin CIDR allowlist</td></tr><tr><td>keycloak.&#x3C;domain></td><td>Privado</td><td>&#x3C;obs_lb_private_ip></td><td>Keycloak admin (ops IAM)</td><td>Only via WireGuard / admin CIDR allowlist</td></tr><tr><td>sandbox.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Internal landing página (non-prod)</td><td>Never expose in UAT/PROD; WireGuard only</td></tr><tr><td>api-internal.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Internal APIs</td><td>WireGuard only</td></tr><tr><td>admin.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Admin portal</td><td>WireGuard only</td></tr><tr><td>iam.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>IAM admin / internal IAM endpoints</td><td>WireGuard only</td></tr><tr><td>regclient.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Registration client download</td><td>WireGuard only</td></tr><tr><td>activemq.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>ActiveMQ UI (if enabled)</td><td>WireGuard only</td></tr><tr><td>kibana.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Kibana UI (optional)</td><td>WireGuard only</td></tr><tr><td>kafka.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Kafka UI (optional)</td><td>WireGuard only</td></tr><tr><td>object-store.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>MinIO console (optional)</td><td>WireGuard only</td></tr><tr><td>postgres.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>DB acesso (usually via port-forward)</td><td>WireGuard only; never público</td></tr><tr><td>pmp.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Partner Mgmt Portal (if enabled)</td><td>WireGuard only</td></tr><tr><td>onboarder.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Partner onboarding reports</td><td>WireGuard only</td></tr><tr><td>smtp.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Mock SMTP UI (if enabled)</td><td>WireGuard only</td></tr><tr><td>api.&#x3C;env>.&#x3C;domain></td><td>Público</td><td>&#x3C;mosip_lb_public_ip></td><td>Público APIs (only what deve be público)</td><td>WAF + allowlists where possible</td></tr><tr><td>prereg.&#x3C;env>.&#x3C;domain></td><td>Público</td><td>&#x3C;mosip_lb_public_ip></td><td>Pre-registration portal (if in scope)</td><td>WAF + rate limits</td></tr><tr><td>resident.&#x3C;env>.&#x3C;domain></td><td>Público</td><td>&#x3C;mosip_lb_public_ip></td><td>Resident portal (if in scope)</td><td>WAF + rate limits</td></tr><tr><td>idp.&#x3C;env>.&#x3C;domain></td><td>Público</td><td>&#x3C;mosip_lb_public_ip></td><td>IDP (if in scope)</td><td>WAF + strict TLS</td></tr></tbody></table>

#### 4.2 Exposure Policy (Recomendado)

{% hint style="info" %}
**Porquê restrict privado endpoints?**\
Admin e internal endpoints expose powerful operações e sensitive data flows. Restricting them para WireGuard/admin allowlists reduces the attack surface e supports auditoria e conformidade requirements.
{% endhint %}

* Privado endpoints deve only be accessible through WireGuard e/ou a tightly controlled admin CIDR allowlist.
* Público endpoints deve be protected com WAF, TLS, rate limiting, e (where feasible) IP allowlisting para partner APIs.

***

### 5. Secure Acesso Baseline (SSH Hygiene + WireGuard)

#### 5.1 SSH privado key permissions

{% hint style="info" %}
**Porquê strict SSH key permissions?**\
Limiting privado key file permissions is a basic segurança control para prevent accidental disclosure of privileged acesso credentials.
{% endhint %}

```bash
sudo chmod 400 ~/.ssh/privkey.pem
```

(Example alternate comando used across steps)

```bash
chmod 400 ~/.ssh/<your private key>
```

#### 5.2 WireGuard pré-requisitos

{% hint style="info" %}
**Porquê WireGuard?**\
WireGuard is a modern VPN designed para be simpler e high-performance. We usar it para provide secure privado acesso para internal dashboards e APIs without exposing them publicly. ([wireguard.com](https://www.wireguard.com/quickstart/))
{% endhint %}

* WireGuard server listens on UDP 51820
* Ensure UDP 51820 is open on the VM e on any external firewall.

***

### 6. WireGuard Bastion Server Configuração (On-Prem)

#### 6.1 SSH into WireGuard VM

```bash
ssh -i <path to .pem> ubuntu@<Wireguard server public ip>
```

#### 6.2 Create config directory

```bash
mkdir -p wireguard/config
```

#### 6.3 Instalar Docker on WireGuard VM (via Ansible)

{% hint style="info" %}
**Porquê automate Docker instalação?**\
Using Ansible para base instalação ensures consistent configuration across nodes e supports repeatability in DR rebuilds.
{% endhint %}

> Execute o playbook do Docker para instalar o Docker e adicionar o utilizador ao grupo Docker.

```bash
ansible-playbook -i hosts.ini docker.yaml
```

#### 6.4 Instalar e start WireGuard server (Docker)

{% hint style="info" %}
**Porquê the linuxserver/wireguard container?**\
It disponibiliza a widely used, repeatable, containerized WireGuard configuração that simplifies implementação e upgrades compared para manual server configuration. ([hub.docker.com](https://hub.docker.com/r/linuxserver/wireguard))
{% endhint %}

```bash
sudo docker run -d   --name=wireguard   --cap-add=NET_ADMIN   --cap-add=SYS_MODULE   -e PUID=1000   -e PGID=1000   -e TZ=Asia/Calcutta   -e PEERS=30   -p 51820:51820/udp   -v /home/ubuntu/wireguard/config:/config   -v /lib/modules:/lib/modules   --sysctl="net.ipv4.conf.all.src_valid_mark=1"   --restart unless-stopped   ghcr.io/linuxserver/wireguard
```

> Note: Increase -e PEERS=30 if more than 30 client configurations are needed.

***

### 7. Observation Kubernetes Cluster – Initial Configuração Comandos

{% hint style="info" %}
**Porquê an Observation cluster?**\
Este cluster aloja tipicamente a gestão da plataforma e serviços de suporte (por exemplo, Rancher, IAM de operações), ajudando a manter as ferramentas operacionais isoladas do runtime do MOSIP. As orientações on‑prem do MOSIP referem a preparação de um cluster Rancher/observação com ingress/storage para aplicações de suporte. ([docs.mosip.io](https://docs.mosip.io/1.2.0/setup/deploymentnew/v3-installation/on-prem-installation-guidelines))
{% endhint %}

#### 7.1 Set up passwordless SSH (optional)

```bash
ssh-keygen -t rsa
```

```bash
ssh-copy-id <remote-user>@<remote-ip>
```

```bash
ssh -i ~/.ssh/<your private key> <remote-user>@<remote-ip>
```

```bash
chmod 400 ~/.ssh/<your private key>
```

#### 7.2 Prepare e execute Ansible playbooks (Observation nodes)

Navigate para the Ansible directory:

```bash
cd $K8_ROOT/rancher/on-prem
```

Copy e update the hosts inventory:

```bash
cp hosts.ini.sample hosts.ini
```

Disable swap on cluster nodes (ignore if already disabled):

```bash
ansible-playbook -i hosts.ini swap.yaml
```

> Atenção: verifique o estado do swap antes de executar:

```bash
swapon --show
```

Instalar o Docker e adicionar o utilizador ao grupo Docker:

```bash
ansible-playbook -i hosts.ini docker.yaml
```

***

### 8. Create RKE Cluster Configuration

{% hint style="info" %}
**Porquê does Rancher / Keycloak appear in the DNS list?**\
Rancher is commonly used para manage Kubernetes clusters e provide a central operações UI. ([ranchermanager.docs.rancher.com](https://ranchermanager.docs.rancher.com/))\
Keycloak disponibiliza IAM capabilities (login, administration, e RBAC integration patterns) e is widely used para manage acesso para administrative services. ([keycloak.org](https://www.keycloak.org/docs/latest/server_admin/index.html))
{% endhint %}

Generate cluster.yml utilizando RKE:

```bash
rke config
```

> The comando will prompt para nodal details, including:

* SSH Privado Key Path: \<path/para/your/ssh/privado/key>

***

### 9. What Must Be Ready Antes “Day 1” Implementação

{% hint style="info" %}
**Porquê do these gates matter?**\
Most MOSIP implementação failures come from missing base dependencies (DNS/TLS, networking, cluster prontidão, e acesso controls). Completing these pré-requisitos reduces rework e stabilizes the ambiente promotion. ([docs.mosip.io](https://docs.mosip.io/1.2.0/readme/overview))
{% endhint %}

* Bastion host has Git, kubectl, istioctl, RKE, Helm, e Ansible installed
* Environment variables exported (MOSIP\_ROOT, K8\_ROOT, INFRA\_ROOT)
* Ubuntu 22.04 images available para all VMs
* VMs provisioned as per base sizing (ou approved updated sizing)
* DNS records planned para observation + MOSIP clusters (público e privado)
* WireGuard UDP 51820 opened (VM + firewall) e SSH key permissions reforçado