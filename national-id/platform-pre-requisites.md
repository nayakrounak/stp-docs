---
description: >-
  Minimum prerequisites to prepare infrastructure, access, DNS, and tooling
  before MOSIP deployment.
---

# Platform Pre-requisites

This section lists the minimum prerequisites required before starting the MOSIP deployment for São Tomé & Príncipe (STP). It includes the **bastion host toolchain**, **baseline VM sizing**, **DNS planning**, and **secure access setup (WireGuard + SSH hygiene)**.

{% hint style="info" %}
**Before You Start**

Fill these values once per environment and keep them consistent across all pages:

* **Base domain:** `<domain>` (example: `nid.gov.st`)
* **Environment code:** `<env>` (example: `dev`, `sit`, `uat`, `prod`, `dr`)
* **Observation LB Private IP:** `<obs_lb_private_ip>`
* **MOSIP LB Private IP:** `<mosip_lb_private_ip>`
* **MOSIP LB Public IP (if applicable):** `<mosip_lb_public_ip>`
* **WireGuard Public IP:** `<wg_public_ip>`
* **WireGuard Port:** `51820/udp`
* **Admin CIDR allowlist:** `<admin_cidr_list>`
* **SSH key path:** `<ssh_private_key_path>`
* **Container registry:** `<registry_url>`
* **HSM/KMS approach:** `<hsm_or_kms_or_tbd>`
{% endhint %}

***

### 1. Bastion Host Setup (Tools & Repository)

All deployment activities should be executed from a secure **bastion host**.

{% hint style="info" %}
**Why a Bastion host?**\
A bastion provides a single hardened entry point for administrative access, tooling, and auditability, reducing direct exposure of cluster nodes and internal services.
{% endhint %}

#### 1.1 Install Git (v2.25.1 or higher)

{% hint style="info" %}
**Why Git?**\
We use Git to version-control infrastructure scripts, Helm chart values, and deployment configs, enabling repeatable deployments, peer review, and traceability. ([git-scm.com](https://git-scm.com/))
{% endhint %}

```bash
sudo apt update
sudo apt install git
```

#### 1.2 Configure environment variables

{% hint style="info" %}
**Why environment variables?**\
MOSIP installation scripts and supporting automation typically assume consistent base paths. Standardizing these paths reduces human error and simplifies runbooks and support.
{% endhint %}

```bash
export MOSIP_ROOT=/home/ubuntu
export K8_ROOT=$MOSIP_ROOT/k8s-infra
export INFRA_ROOT=$MOSIP_ROOT/mosip-infra
```

> Tip: Add the exports to \~/.bashrc if you want them to persist across sessions.

#### 1.3 Install kubectl

{% hint style="info" %}
**Why kubectl?**\
`kubectl` is the standard command-line tool to interact with Kubernetes clusters (deployments, logs, troubleshooting). We use it to validate cluster state and operate MOSIP workloads. ([kubernetes.io](https://kubernetes.io/docs/tasks/tools/))
{% endhint %}

```bash
curl -LO "https://dl.k8s.io/release/v1.34.1/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

#### 1.4 Install istioctl (v1.15.0)

{% hint style="info" %}
**Why istioctl?**\
When Istio is used for service mesh capabilities (traffic management, security policies, diagnostics), `istioctl` is the recommended tool for installation, configuration validation, and troubleshooting. ([istio.io](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/))
{% endhint %}

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.15.0 sh -
cd istio-1.15.0
sudo mv bin/istioctl /usr/local/bin/
istioctl version --remote=false
```

#### 1.5 Install RKE (v1.3.10)

{% hint style="warning" %}
**Why RKE (and an important note):**\
RKE is a Kubernetes cluster provisioning tool that simplifies on-premises cluster creation. However, RKE1 has reached end of life; new deployments should consider RKE2 (or another supported Kubernetes distribution) unless there is a project constraint that requires remaining on RKE1 for compatibility. ([github.com](https://github.com/rancher/rke))
{% endhint %}

```bash
curl -L "https://github.com/rancher/rke/releases/download/v1.3.10/rke_linux-amd64" -o rke
chmod +x rke
sudo mv rke /usr/local/bin/
rke --version
```

#### 1.6 Install Helm (v3.0.0 or higher) and add Helm repos

{% hint style="info" %}
**Why Helm?**\
MOSIP services are packaged as Helm charts to standardize installation and upgrades on Kubernetes. Helm helps manage complex applications with versioned, repeatable deployments. ([helm.sh](https://helm.sh/docs/))
{% endhint %}

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add mosip https://mosip.github.io/mosip-helm
```

#### 1.7 Install Ansible (> 2.12.4)

{% hint style="info" %}
**Why Ansible?**\
We use Ansible to automate repeatable server configuration tasks (e.g., installing Docker, disabling swap, and baseline node hardening). This reduces manual errors across multiple VMs. ([docs.ansible.com](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html))
{% endhint %}

```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

#### 1.8 Clone MOSIP k8s-infra repo (tag v1.2.0.2)

{% hint style="info" %}
**Why MOSIP k8s-infra?**\
The `k8s-infra` repository provides reference architecture and scripts to deploy Kubernetes-based clusters and supporting infrastructure for MOSIP (and related DPG components). ([github.com](https://github.com/mosip/k8s-infra))
{% endhint %}

```bash
git clone https://github.com/mosip/k8s-infra -b v1.2.0.2
```

***

### 2. Operating System Baseline

All VMs referenced in this guide assume Ubuntu 22.04 as the standard OS image.

{% hint style="info" %}
**Why Ubuntu 22.04 LTS?**\
Ubuntu LTS releases provide long-term maintenance and security updates, which is important for stable government infrastructure. ([discourse.ubuntu.com](https://discourse.ubuntu.com/t/jammy-jellyfish-release-notes/24668))
{% endhint %}

***

### 3. Baseline Hardware Requirements (On-Prem Reference)

> Adjust sizing based on expected population, enrollment concurrency, HA needs, and DR requirements.

{% hint style="info" %}
**Why separate Observation vs MOSIP clusters?**\
Separating the **Observation/management** components (e.g., cluster management and ops IAM) from the main MOSIP application cluster reduces risk and avoids management tooling impacting MOSIP runtime performance. MOSIP’s on-prem guidance also reflects an observation cluster concept for installing ingress/storage and supporting apps. ([docs.mosip.io](https://docs.mosip.io/1.2.0/setup/deploymentnew/v3-installation/on-prem-installation-guidelines))
{% endhint %}

<table data-header-hidden><thead><tr><th></th><th width="80.0859375"></th><th width="79.921875"></th><th width="86.94921875"></th><th width="87.140625"></th><th width="137.48046875"></th><th></th></tr></thead><tbody><tr><td><strong>Purpose</strong></td><td><strong>vCPU</strong></td><td><strong>RAM</strong></td><td><strong>Storage</strong></td><td><strong># VMs</strong></td><td><strong>HA</strong></td><td><strong>OS</strong></td></tr><tr><td>WireGuard Bastion Host</td><td>2</td><td>4 GB</td><td>8 GB</td><td>1</td><td>Active-Passive</td><td>Ubuntu 22.04</td></tr><tr><td>Observation Cluster nodes</td><td>2</td><td>8 GB</td><td>32 GB</td><td>2</td><td>2</td><td>Ubuntu 22.04</td></tr><tr><td>Observation Nginx / LB</td><td>2</td><td>4 GB</td><td>16 GB</td><td>1</td><td>Nginx+</td><td>Ubuntu 22.04</td></tr><tr><td>MOSIP Cluster nodes</td><td>12</td><td>32 GB</td><td>128 GB</td><td>6</td><td>6</td><td>Ubuntu 22.04</td></tr><tr><td>MOSIP Nginx / LB</td><td>2</td><td>4 GB</td><td>16 GB</td><td>1</td><td>Nginx+</td><td>Ubuntu 22.04</td></tr></tbody></table>

***

### 4. DNS Requirements (Domains & Mappings)

MOSIP deployments require DNS records mapped to:

* **Observation Cluster Nginx/LB** (private only)
* **MOSIP Cluster Nginx/LB** (private and public, depending on exposure policy)

{% hint style="info" %}
**Why DNS and stable FQDNs?**\
Stable hostnames simplify TLS certificate management, client configuration (e.g., registration clients), and operational runbooks. MOSIP Helm-based deployments assume consistent endpoint URLs across environments. ([docs.mosip.io](https://docs.mosip.io/1.2.0/setup/deploymentnew/getting-started/helm-charts))
{% endhint %}

#### 4.1 DNS Table

<table data-header-hidden><thead><tr><th width="195.19140625"></th><th width="102.19140625"></th><th width="200.1796875"></th><th width="123.08203125"></th><th></th></tr></thead><tbody><tr><td><strong>FQDN Pattern</strong></td><td><strong>Exposure</strong></td><td><strong>Maps To</strong></td><td><strong>Purpose</strong></td><td><strong>Access Policy</strong></td></tr><tr><td>rancher.&#x3C;domain></td><td>Private</td><td>&#x3C;obs_lb_private_ip></td><td>Rancher dashboard</td><td>Only via WireGuard / admin CIDR allowlist</td></tr><tr><td>keycloak.&#x3C;domain></td><td>Private</td><td>&#x3C;obs_lb_private_ip></td><td>Keycloak admin (ops IAM)</td><td>Only via WireGuard / admin CIDR allowlist</td></tr><tr><td>sandbox.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>Internal landing page (non-prod)</td><td>Never expose in UAT/PROD; WireGuard only</td></tr><tr><td>api-internal.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>Internal APIs</td><td>WireGuard only</td></tr><tr><td>admin.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>Admin portal</td><td>WireGuard only</td></tr><tr><td>iam.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>IAM admin / internal IAM endpoints</td><td>WireGuard only</td></tr><tr><td>regclient.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>Registration client download</td><td>WireGuard only</td></tr><tr><td>activemq.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>ActiveMQ UI (if enabled)</td><td>WireGuard only</td></tr><tr><td>kibana.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>Kibana UI (optional)</td><td>WireGuard only</td></tr><tr><td>kafka.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>Kafka UI (optional)</td><td>WireGuard only</td></tr><tr><td>object-store.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>MinIO console (optional)</td><td>WireGuard only</td></tr><tr><td>postgres.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>DB access (usually via port-forward)</td><td>WireGuard only; never public</td></tr><tr><td>pmp.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>Partner Mgmt Portal (if enabled)</td><td>WireGuard only</td></tr><tr><td>onboarder.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>Partner onboarding reports</td><td>WireGuard only</td></tr><tr><td>smtp.&#x3C;env>.&#x3C;domain></td><td>Private</td><td>&#x3C;mosip_lb_private_ip></td><td>Mock SMTP UI (if enabled)</td><td>WireGuard only</td></tr><tr><td>api.&#x3C;env>.&#x3C;domain></td><td>Public</td><td>&#x3C;mosip_lb_public_ip></td><td>Public APIs (only what must be public)</td><td>WAF + allowlists where possible</td></tr><tr><td>prereg.&#x3C;env>.&#x3C;domain></td><td>Public</td><td>&#x3C;mosip_lb_public_ip></td><td>Pre-registration portal (if in scope)</td><td>WAF + rate limits</td></tr><tr><td>resident.&#x3C;env>.&#x3C;domain></td><td>Public</td><td>&#x3C;mosip_lb_public_ip></td><td>Resident portal (if in scope)</td><td>WAF + rate limits</td></tr><tr><td>idp.&#x3C;env>.&#x3C;domain></td><td>Public</td><td>&#x3C;mosip_lb_public_ip></td><td>IDP (if in scope)</td><td>WAF + strict TLS</td></tr></tbody></table>

#### 4.2 Exposure Policy (Recommended)

{% hint style="info" %}
**Why restrict private endpoints?**\
Admin and internal endpoints expose powerful operations and sensitive data flows. Restricting them to WireGuard/admin allowlists reduces the attack surface and supports audit and compliance requirements.
{% endhint %}

* Private endpoints must only be accessible through WireGuard and/or a tightly controlled admin CIDR allowlist.
* Public endpoints must be protected with WAF, TLS, rate limiting, and (where feasible) IP allowlisting for partner APIs.

***

### 5. Secure Access Baseline (SSH Hygiene + WireGuard)

#### 5.1 SSH private key permissions

{% hint style="info" %}
**Why strict SSH key permissions?**\
Limiting private key file permissions is a basic security control to prevent accidental disclosure of privileged access credentials.
{% endhint %}

```bash
sudo chmod 400 ~/.ssh/privkey.pem
```

(Example alternate command used across steps)

```bash
chmod 400 ~/.ssh/<your private key>
```

#### 5.2 WireGuard prerequisites

{% hint style="info" %}
**Why WireGuard?**\
WireGuard is a modern VPN designed to be simpler and high-performance. We use it to provide secure private access to internal dashboards and APIs without exposing them publicly. ([wireguard.com](https://www.wireguard.com/quickstart/))
{% endhint %}

* WireGuard server listens on UDP 51820
* Ensure UDP 51820 is open on the VM and on any external firewall.

***

### 6. WireGuard Bastion Server Setup (On-Prem)

#### 6.1 SSH into WireGuard VM

```bash
ssh -i <path to .pem> ubuntu@<Wireguard server public ip>
```

#### 6.2 Create config directory

```bash
mkdir -p wireguard/config
```

#### 6.3 Install Docker on WireGuard VM (via Ansible)

{% hint style="info" %}
**Why automate Docker installation?**\
Using Ansible for baseline installation ensures consistent configuration across nodes and supports repeatability in DR rebuilds.
{% endhint %}

> Execute the Docker playbook to install Docker and add the user to the Docker group.

```bash
ansible-playbook -i hosts.ini docker.yaml
```

#### 6.4 Install and start WireGuard server (Docker)

{% hint style="info" %}
**Why the linuxserver/wireguard container?**\
It provides a widely used, repeatable, containerized WireGuard setup that simplifies deployment and upgrades compared to manual server configuration. ([hub.docker.com](https://hub.docker.com/r/linuxserver/wireguard))
{% endhint %}

```bash
sudo docker run -d   --name=wireguard   --cap-add=NET_ADMIN   --cap-add=SYS_MODULE   -e PUID=1000   -e PGID=1000   -e TZ=Asia/Calcutta   -e PEERS=30   -p 51820:51820/udp   -v /home/ubuntu/wireguard/config:/config   -v /lib/modules:/lib/modules   --sysctl="net.ipv4.conf.all.src_valid_mark=1"   --restart unless-stopped   ghcr.io/linuxserver/wireguard
```

> Note: Increase -e PEERS=30 if more than 30 client configurations are needed.

***

### 7. Observation Kubernetes Cluster – Initial Setup Commands

{% hint style="info" %}
**Why an Observation cluster?**\
This cluster typically hosts platform management and supporting services (e.g., Rancher, ops IAM), helping keep operational tooling isolated from the MOSIP runtime. MOSIP on-prem guidance references preparing a Rancher/observation cluster with ingress/storage for supporting applications. ([docs.mosip.io](https://docs.mosip.io/1.2.0/setup/deploymentnew/v3-installation/on-prem-installation-guidelines))
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

#### 7.2 Prepare and execute Ansible playbooks (Observation nodes)

Navigate to the Ansible directory:

```bash
cd $K8_ROOT/rancher/on-prem
```

Copy and update the hosts inventory:

```bash
cp hosts.ini.sample hosts.ini
```

Disable swap on cluster nodes (ignore if already disabled):

```bash
ansible-playbook -i hosts.ini swap.yaml
```

> Caution: Verify swap status before running:

```bash
swapon --show
```

Install Docker and add the user to the Docker group:

```bash
ansible-playbook -i hosts.ini docker.yaml
```

***

### 8. Create RKE Cluster Configuration

{% hint style="info" %}
**Why does Rancher / Keycloak appear in the DNS list?**\
Rancher is commonly used to manage Kubernetes clusters and provide a central operations UI. ([ranchermanager.docs.rancher.com](https://ranchermanager.docs.rancher.com/))\
Keycloak provides IAM capabilities (login, administration, and RBAC integration patterns) and is widely used to manage access to administrative services. ([keycloak.org](https://www.keycloak.org/docs/latest/server_admin/index.html))
{% endhint %}

Generate cluster.yml using RKE:

```bash
rke config
```

> The command will prompt for nodal details, including:

* SSH Private Key Path: \<path/to/your/ssh/private/key>

***

### 9. What Must Be Ready Before “Day 1” Deployment

{% hint style="info" %}
**Why do these gates matter?**\
Most MOSIP deployment failures come from missing baseline dependencies (DNS/TLS, networking, cluster readiness, and access controls). Completing these prerequisites reduces rework and stabilizes the environment promotion. ([docs.mosip.io](https://docs.mosip.io/1.2.0/readme/overview))
{% endhint %}

* Bastion host has Git, kubectl, istioctl, RKE, Helm, and Ansible installed
* Environment variables exported (MOSIP\_ROOT, K8\_ROOT, INFRA\_ROOT)
* Ubuntu 22.04 images available for all VMs
* VMs provisioned as per baseline sizing (or approved updated sizing)
* DNS records planned for observation + MOSIP clusters (public and private)
* WireGuard UDP 51820 opened (VM + firewall) and SSH key permissions hardened
