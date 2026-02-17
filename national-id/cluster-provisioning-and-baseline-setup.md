# Cluster Provisioning & Baseline Setup

This page covers the **end-to-end provisioning steps** to bring up the **Observation Kubernetes cluster** and the **MOSIP Kubernetes cluster** for São Tomé & Príncipe (STP), using **RKE**, with a hardened baseline (swap off, Docker installed, SSH hygiene), and clear **validation gates** before moving to ingress / Istio / MOSIP Helm installs.

***

### Before You Start

{% hint style="info" %}
#### Why this page matters

Most deployment failures occur because Kubernetes nodes are not configured consistently (swap enabled, incorrect Docker/kernel settings, SSH access issues). This page ensures both clusters are built the same way every time—especially important for **DR rebuilds**.
{% endhint %}

Define these values (once per environment):

* **Environment code:** `<env>` (dev/sit/uat/prod/dr)
* **Bastion host:** `<bastion_ip>`
* **Observation nodes:** `<obs_node_ips>`
* **MOSIP nodes:** `<mosip_node_ips>`
* **SSH user:** `<ssh_user>` (example: `ubuntu`)
* **SSH key path:** `<ssh_private_key_path>`

***

### 1. Prepare Bastion Host (Operator Machine)

You should execute cluster provisioning **from the bastion host** (recommended).

#### 1.1 Set environment variables

```bash
export MOSIP_ROOT=/home/ubuntu
export K8_ROOT=$MOSIP_ROOT/k8s-infra
export INFRA_ROOT=$MOSIP_ROOT/mosip-infra
```

{% hint style="info" %}
**Why environment variables?**\
They standardize paths across pages/runbooks so operators don’t “cd into the wrong folder” and scripts work consistently.
{% endhint %}

#### 1.2 Confirm required tools are installed

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

### 2. SSH Hygiene & Passwordless Access (Optional but Recommended)

#### 2.1 Generate SSH key (if needed)

```bash
ssh-keygen -t rsa
```

#### 2.2 Copy public key to all nodes

```bash
ssh-copy-id <remote-user>@<remote-ip>
```

#### 2.3 Verify SSH connectivity

```bash
ssh -i ~/.ssh/<your private key> <remote-user>@<remote-ip>
```

#### 2.4 Ensure private key permissions

```bash
chmod 400 ~/.ssh/<your private key>
```

{% hint style="info" %}
**Why passwordless SSH?**\
RKE and Ansible require consistent access to nodes. Passwordless SSH reduces errors during provisioning and supports automated, CI-based rebuilds.
{% endhint %}

***

### 3. Prepare Nodes with Ansible (Baseline)

We first apply the same baseline to Observation nodes, then repeat for **MOSIP nodes**.

#### 3.1 Go to the Ansible directory (Observation)

```bash
cd $K8_ROOT/rancher/on-prem
```

#### 3.2 Create an inventory file

```bash
cp hosts.ini.sample hosts.ini
```

Update `hosts.ini` with your node IPs/users.

#### 3.3 Check swap status (before disabling)

```bash
swapon --show
```

#### 3.4 Disable swap on all cluster nodes

```bash
ansible-playbook -i hosts.ini swap.yaml
```

{% hint style="info" %}
**Why disable swap?**\
Kubernetes requires that swap be disabled (or explicitly configured) because it can cause unpredictable memory behavior for pods and scheduling.&#x20;

Reference: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin
{% endhint %}

#### 3.5 Install Docker + add user to Docker group

```bash
ansible-playbook -i hosts.ini docker.yaml
```

{% hint style="info" %}
**Why Docker baseline?**\
RKE1 commonly uses Docker as the container runtime. A consistent Docker setup across nodes prevents runtime issues during cluster bring-up. Reference: https://docs.docker.com/engine/install/
{% endhint %}

***

### 4. Create the Observation Kubernetes Cluster (RKE)

#### 4.1 Generate cluster.yml interactively

```bash
rke config
```

You will be prompted for node details and SSH key path.

#### 4.2 Edit cluster.yml (if needed)

```bash
vi cluster.yml
```

If you want to disable default ingress installation (common when you manage ingress separately):

```yaml
ingress:
  provider: none
```

#### 4.3 Bring up the cluster

```bash
rke up
```

#### 4.4 Set kubeconfig

Option A (copy to default):

```bash
cp $HOME/.kube/<cluster_name>_config $HOME/.kube/config
```

Option B (export explicitly):

```bash
export KUBECONFIG="$HOME/.kube/<cluster_name>_config"
```

#### 4.5 Validate cluster health

```bash
kubectl get nodes
```

{% hint style="info" %}
**Why these validation steps?**\
If nodes aren’t `Ready` now, everything later (ingress, Istio, MOSIP Helm installs) becomes unstable and hard to troubleshoot.
{% endhint %}

#### 4.6 Store RKE artifacts securely

Keep these files in a secure location (required for upgrades/node replacement/rebuilds):

* `cluster.yml`
* `cluster.rkestate`
* `kube_config_cluster.yml` (or the generated kubeconfig)

***

### 5. Create the MOSIP Kubernetes Cluster (RKE)

Repeat the exact same flow for the MOSIP cluster.

#### 5.1 Prepare inventory & baseline (MOSIP nodes)

(If you maintain separate inventories for the MOSIP cluster)

```bash
cd $K8_ROOT/rancher/on-prem
cp hosts.ini.sample hosts.ini
ansible-playbook -i hosts.ini swap.yaml
ansible-playbook -i hosts.ini docker.yaml
```

#### 5.2 Generate and bring up the MOSIP cluster

```bash
rke config
vi cluster.yml
rke up
```

#### 5.3 Set kubeconfig and validate

```bash
cp $HOME/.kube/<cluster_name>_config $HOME/.kube/config
kubectl get nodes
```

***

### 6. Definition of Done (DoD) — Cluster Provisioning

Before proceeding to **Ingress & Istio / Platform Installation**, confirm:

* [ ] You can SSH from the bastion to every node using the intended key
* [ ] Swap is disabled on all nodes (`swapon --show` returns nothing)
* [ ] Docker is installed, and the intended user can run Docker (no sudo issues)
* [ ] `rke up` completed successfully (no failed hosts)
* [ ] `kubectl get nodes` shows all nodes in `Ready` state
* [ ] RKE artifacts are stored securely (`cluster.yml`, `cluster.rkestate`, kubeconfig)
