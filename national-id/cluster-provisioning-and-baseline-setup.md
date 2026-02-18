---
description: >-
  Provision Observation and MOSIP Kubernetes clusters using RKE with a hardened
  baseline.
---

# Cluster Provisioning & Baseline Configuração

Esta página descreve os **passos de provisionamento ponta‑a‑ponta** para colocar em funcionamento o **cluster Kubernetes de Observação** e o **cluster Kubernetes do MOSIP** em São Tomé e Príncipe (STP), utilizando **RKE**, com uma base reforçada (swap desativado, Docker instalado, boas práticas de SSH) e **gates de validação** claros antes de avançar para ingress / Istio / instalações via Helm.

***

### Antes You Começar

{% hint style="info" %}
**Porquê this página matters**

A maioria das falhas de implementação ocorre porque os nós Kubernetes não estão configurados de forma consistente (swap ativo, definições Docker/kernel incorretas, problemas de acesso SSH). Esta página garante que ambos os clusters são construídos da mesma forma em todas as ocasiões — especialmente importante para **reconstruções de DR**.
{% endhint %}

Define these values (once per ambiente):

* **Environment code:** `<env>` (dev/sit/uat/prod/dr)
* **Bastion host:** `<bastion_ip>`
* **Observation nodes:** `<obs_node_ips>`
* **MOSIP nodes:** `<mosip_node_ips>`
* **SSH user:** `<ssh_user>` (example: `ubuntu`)
* **SSH key path:** `<ssh_private_key_path>`

***

### 1. Prepare Bastion Host (Operator Machine)

You deverá execute cluster provisioning **from the bastion host** (recomendado).

#### 1.1 Set ambiente variables

```bash
export MOSIP_ROOT=/home/ubuntu
export K8_ROOT=$MOSIP_ROOT/k8s-infra
export INFRA_ROOT=$MOSIP_ROOT/mosip-infra
```

{% hint style="info" %}
**Porquê ambiente variables?**\
They standardize paths across pages/runbooks so operators don’t “cd into the wrong folder” e scripts work consistently.
{% endhint %}

#### 1.2 Confirm required ferramentas are installed

At minimum:

* Git
* Ansible
* RKE
* kubectl
* Helm

{% hint style="info" %}
**Trusted references**

* kubectl: https://kubernetes.io/docs/tasks/tools/
* RKE (RKE1): https://rke.docs.rancher.com/
* Ansible: https://docs.ansible.com/ansible/latest/installation\_guide/intro\_installation.html
* Helm: https://helm.sh/docs/intro/install/
{% endhint %}

***

### 2. SSH Hygiene & Passwordless Acesso (Optional but Recomendado)

#### 2.1 Generate SSH key (if needed)

```bash
ssh-keygen -t rsa
```

#### 2.2 Copy público key para all nodes

```bash
ssh-copy-id <remote-user>@<remote-ip>
```

#### 2.3 Verify SSH connectivity

```bash
ssh -i ~/.ssh/<your private key> <remote-user>@<remote-ip>
```

#### 2.4 Ensure privado key permissions

```bash
chmod 400 ~/.ssh/<your private key>
```

{% hint style="info" %}
**Porquê passwordless SSH?**\
RKE e Ansible require consistent acesso para nodes. Passwordless SSH reduces errors during provisioning e supports automated, CI-based rebuilds.
{% endhint %}

***

### 3. Prepare Nodes com Ansible (Baseline)

Aplicamos primeiro a mesma base aos nós de Observação e, em seguida, repetimos o processo para os **nós do MOSIP**.

#### 3.1 Go para the Ansible directory (Observation)

```bash
cd $K8_ROOT/rancher/on-prem
```

#### 3.2 Create an inventory file

```bash
cp hosts.ini.sample hosts.ini
```

Update `hosts.ini` com your node IPs/users.

#### 3.3 Check swap status (before disabling)

```bash
swapon --show
```

#### 3.4 Disable swap on all cluster nodes

```bash
ansible-playbook -i hosts.ini swap.yaml
```

{% hint style="info" %}
**Porquê disable swap?**\
Kubernetes requires that swap be disabled (ou explicitly configured) because it can cause unpredictable memory behavior para pods e scheduling.

Reference: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin
{% endhint %}

#### 3.5 Instalar Docker + add user para Docker group

```bash
ansible-playbook -i hosts.ini docker.yaml
```

{% hint style="info" %}
**Porquê Docker base?**\
RKE1 commonly uses Docker as the container runtime. A consistent Docker configuração across nodes prevents runtime issues during cluster bring-up. Reference: https://docs.docker.com/engine/install/
{% endhint %}

***

### 4. Create the Observation Kubernetes Cluster (RKE)

#### 4.1 Generate cluster.yml interactively

```bash
rke config
```

You will be prompted para node details e SSH key path.

#### 4.2 Edit cluster.yml (if needed)

```bash
vi cluster.yml
```

If you want para disable default ingress instalação (common when you manage ingress separately):

```yaml
ingress:
  provider: none
```

#### 4.3 Bring up the cluster

```bash
rke up
```

#### 4.4 Set kubeconfig

Option A (copy para default):

```bash
cp $HOME/.kube/<cluster_name>_config $HOME/.kube/config
```

Option B (export explicitly):

```bash
export KUBECONFIG="$HOME/.kube/<cluster_name>_config"
```

#### 4.5 Validar cluster health

```bash
kubectl get nodes
```

{% hint style="info" %}
**Porquê these validação steps?**\
If nodes aren’t `Ready` now, everything later (ingress, Istio, MOSIP Helm installs) becomes unstable e hard para troubleshoot.
{% endhint %}

#### 4.6 Store RKE artifacts securely

Keep these files in a secure location (required para upgrades/node replacement/rebuilds):

* `cluster.yml`
* `cluster.rkestate`
* `kube_config_cluster.yml` (ou the generated kubeconfig)

***

### 5. Create the MOSIP Kubernetes Cluster (RKE)

Repita exatamente o mesmo fluxo para o cluster MOSIP.

#### 5.1 Prepare inventory & base (MOSIP nodes)

(If you maintain separate inventories para the MOSIP cluster)

```bash
cd $K8_ROOT/rancher/on-prem
cp hosts.ini.sample hosts.ini
ansible-playbook -i hosts.ini swap.yaml
ansible-playbook -i hosts.ini docker.yaml
```

#### 5.2 Generate e bring up the MOSIP cluster

```bash
rke config
vi cluster.yml
rke up
```

#### 5.3 Set kubeconfig e validate

```bash
cp $HOME/.kube/<cluster_name>_config $HOME/.kube/config
kubectl get nodes
```

***

### 6. Definition of Done (DoD) — Cluster Provisioning

Antes proceeding para **Ingress & Istio / Plataforma Instalação**, confirm:

* [ ] Consegue fazer SSH a partir do bastion para todos os nós utilizando a chave prevista
* [ ] Swap is disabled on all nodes (`swapon --show` returns nothing)
* [ ] Docker is installed, e the destina-se user can run Docker (no sudo issues)
* [ ] `rke up` completed successfully (no failed hosts)
* [ ] `kubectl get nodes` shows all nodes in `Ready` state
* [ ] RKE artifacts are stored securely (`cluster.yml`, `cluster.rkestate`, kubeconfig)