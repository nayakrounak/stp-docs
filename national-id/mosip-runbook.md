# MOSIP Runbook

{% hint style="info" %}
This runbook is the deployment notes for São Tomé and Príncipe's MOSIP environment. It standardizes sectioning, corrects language, adds operational notes, and highlights places where manual edits or environment-specific inputs are required.
{% endhint %}

### Purpose of this runbook

This runbook is intended to support end-to-end deployment of the MOSIP platform on an on-premises Kubernetes environment for São Tomé and Príncipe, covering:

* Bastion and VPN setup
* Observation cluster deployment
* MOSIP cluster deployment
* NGINX and DNS configuration
* External dependency installation
* MOSIP module deployment
* Operational notes for customizations and known deviations

### What is different in this runbook

#### Compared to standard MOSIP documentation

* Uses a step-by-step sequence from infrastructure setup to application deployment
* Captures Ooru-specific fixes and deployment decisions
* Uses verified OS images and practical deployment notes
* Explicitly notes where monitoring can be skipped
* Calls out script edits required for deployments without monitoring

{% hint style="warning" %}
This runbook includes environment-specific practices followed by Ooru. Before using it in production, validate all domains, IPs, certificates, ports, secrets, and image versions against the approved STP environment design.
{% endhint %}

***

### Pre-requisites

#### Base operating system

This guide assumes **Ubuntu 22.04 LTS** across all nodes unless otherwise noted.

{% hint style="info" %}
Keep OS versions consistent across bastion, Kubernetes worker/control plane nodes, and NGINX hosts. Mixed OS versions often create avoidable troubleshooting overhead.
{% endhint %}

#### Hardware requirements

<table><thead><tr><th width="45.0546875">#</th><th width="161.890625">Purpose</th><th width="92.6875" align="right">vCPUs</th><th width="82.85546875" align="right">RAM</th><th width="98.8359375" align="right">Storage</th><th width="102.921875" align="right">Number of VMs</th><th width="122.19921875">HA</th><th width="105.703125">OS Image</th></tr></thead><tbody><tr><td>1</td><td>WireGuard Bastion Host</td><td align="right">2</td><td align="right">4 GB</td><td align="right">8 GB</td><td align="right">1</td><td>Active-Passive</td><td>Ubuntu 22.04</td></tr><tr><td>2</td><td>Observation Cluster Nodes</td><td align="right">2</td><td align="right">8 GB</td><td align="right">32 GB</td><td align="right">2</td><td>2 Nodes</td><td>Ubuntu 22.04</td></tr><tr><td>3</td><td>Observation NGINX Server / Load Balancer</td><td align="right">2</td><td align="right">4 GB</td><td align="right">16 GB</td><td align="right">1</td><td>NGINX+ / LB</td><td>Ubuntu 22.04</td></tr><tr><td>4</td><td>MOSIP Cluster Nodes</td><td align="right">12</td><td align="right">32 GB</td><td align="right">128 GB</td><td align="right">6</td><td>6 Nodes</td><td>Ubuntu 22.04</td></tr><tr><td>5</td><td>MOSIP NGINX Server / Load Balancer</td><td align="right">2</td><td align="right">4 GB</td><td align="right">16 GB</td><td align="right">1</td><td>NGINX+ / LB</td><td>Ubuntu 22.04</td></tr></tbody></table>

{% hint style="warning" %}
The above sizing is suitable for the referenced environment setup. Review the actual STP workload assumptions before using this sizing for production, especially for ABIS, packet processing, storage growth, and high-availability targets.
{% endhint %}

#### DNS requirements

<table><thead><tr><th width="41.95703125" align="right">#</th><th width="266.5625">Domain Name</th><th width="239.453125">Mapping Details</th><th>Purpose</th></tr></thead><tbody><tr><td align="right">1</td><td><code>rancher.xyz.net</code></td><td>Private IP of Observation NGINX / LB</td><td>Rancher dashboard for cluster management</td></tr><tr><td align="right">2</td><td><code>keycloak.xyz.net</code></td><td>Private IP of Observation NGINX / LB</td><td>Keycloak for observation/admin access</td></tr><tr><td align="right">3</td><td><code>sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>Internal landing/index page for environment links</td></tr><tr><td align="right">4</td><td><code>api-internal.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>Internal APIs over WireGuard</td></tr><tr><td align="right">5</td><td><code>api.sandbox.xyz.net</code></td><td>Public IP of MOSIP NGINX / LB</td><td>Public APIs</td></tr><tr><td align="right">6</td><td><code>prereg.sandbox.xyz.net</code></td><td>Public IP of MOSIP NGINX / LB</td><td>Pre-registration portal</td></tr><tr><td align="right">7</td><td><code>activemq.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>ActiveMQ dashboard over WireGuard</td></tr><tr><td align="right">8</td><td><code>kibana.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>Kibana over WireGuard, if installed</td></tr><tr><td align="right">9</td><td><code>regclient.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>Registration client download endpoint</td></tr><tr><td align="right">10</td><td><code>admin.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>MOSIP Admin portal</td></tr><tr><td align="right">11</td><td><code>object-store.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>Object store console, if exposed</td></tr><tr><td align="right">12</td><td><code>kafka.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>Kafka UI</td></tr><tr><td align="right">13</td><td><code>iam.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>MOSIP IAM / Keycloak</td></tr><tr><td align="right">14</td><td><code>postgres.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>PostgreSQL host reference</td></tr><tr><td align="right">15</td><td><code>pmp.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>Partner Management Portal</td></tr><tr><td align="right">16</td><td><code>onboarder.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>Partner onboarding reports</td></tr><tr><td align="right">17</td><td><code>resident.sandbox.xyz.net</code></td><td>Public IP of MOSIP NGINX / LB</td><td>Resident portal</td></tr><tr><td align="right">18</td><td><code>idp.sandbox.xyz.net</code></td><td>Public IP of MOSIP NGINX / LB</td><td>Identity Provider</td></tr><tr><td align="right">19</td><td><code>smtp.sandbox.xyz.net</code></td><td>Private IP of MOSIP NGINX / LB</td><td>Mock SMTP UI</td></tr></tbody></table>

{% hint style="danger" %}
Do not expose internal-only domains publicly. Domains such as `admin`, `iam`, `postgres`, `kafka`, `activemq`, and `smtp` should remain private and ideally be reachable only through WireGuard or another approved secure channel.
{% endhint %}

***

### Install required tools on the bastion host

#### Git

Install Git version `2.25.1` or higher.

```bash
sudo apt update
sudo apt install git -y
git --version
```

#### Environment variable configuration

```bash
echo 'export MOSIP_ROOT=/home/ubuntu' >> ~/.bashrc
echo 'export K8_ROOT=$MOSIP_ROOT/k8s-infra' >> ~/.bashrc
echo 'export INFRA_ROOT=$MOSIP_ROOT/mosip-infra' >> ~/.bashrc
source ~/.bashrc
```

{% hint style="info" %}
These environment variables are referenced throughout the runbook. Ensure they are set for the same user account that will execute the deployment.
{% endhint %}

#### kubectl

Install `kubectl` version `v1.34.1` or another version approved for your Kubernetes distribution.

```bash
curl -LO "https://dl.k8s.io/release/v1.34.1/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

#### istioctl

Install `istioctl` version `1.15.0`.

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.15.0 sh -
cd istio-1.15.0
sudo mv bin/istioctl /usr/local/bin/
istioctl version --remote=false
```

#### RKE

Install `rke` version `v1.3.10`.

```bash
curl -L "https://github.com/rancher/rke/releases/download/v1.3.10/rke_linux-amd64" -o rke
chmod +x rke
sudo mv rke /usr/local/bin/
rke --version
```

#### Helm

Install Helm `3.x`.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add mosip https://mosip.github.io/mosip-helm
```

#### Ansible

Install the Ansible version `2.12.4` or above.

```bash
sudo apt update
sudo apt install software-properties-common -y
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
ansible --version
```

#### Clone required repositories

Clone the infrastructure repositories using the tag `v1.2.0.2`.

```bash
git clone https://github.com/mosip/k8s-infra -b v1.2.0.2
git clone https://github.com/mosip/mosip-infra -b v1.2.0.2
```

{% hint style="warning" %}
This runbook uses a mix of `v1.2.0.2` and `release-1.2.0.x` branches in specific steps. Do not normalize all steps to a single branch without first validating the deployment scripts.
{% endhint %}

#### OpenSSL requirement for Registration Client

Registration Client requires OpenSSL `1.1.1.x`.

{% hint style="warning" %}
The original notes mention `1.1.1f`, while the installation steps are downloading `1.1.1s`. Keep the version choice consistent with the Registration Client's confirmed compatibility for your environment.
{% endhint %}

Check the current version:

```bash
openssl version
```

If required, remove the existing package:

```bash
sudo apt remove openssl -y
```

Install prerequisites:

```bash
sudo apt install build-essential checkinstall zlib1g-dev -y
cd /usr/local/src/
sudo wget https://www.openssl.org/source/openssl-1.1.1s.tar.gz
sudo tar -xf openssl-1.1.1s.tar.gz
cd openssl-1.1.1s
sudo ./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl shared zlib
sudo make
sudo make test
sudo make install
```

Configure shared libraries:

```bash
echo "/usr/local/ssl/lib" | sudo tee /etc/ld.so.conf.d/openssl-1.1.1s.conf
sudo ldconfig -v
```

Configure binaries:

```bash
sudo mv /usr/bin/c_rehash /usr/bin/c_rehash.backup
sudo mv /usr/bin/openssl /usr/bin/openssl.backup
```

Update the environment path:

```bash
sudo sed -i 's|^PATH=.*|PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/local/ssl/bin"|' /etc/environment
source /etc/environment
openssl version -a
```

***

### On-premises deployment

### 1. WireGuard bastion server setup

The WireGuard bastion server provides a secure private channel to access the MOSIP cluster.

{% hint style="danger" %}
Ensure UDP port `51820` is open on the bastion host security group or firewall before starting the WireGuard setup.
{% endhint %}

#### 1.1 Install Docker on the bastion host

```bash
sudo apt update
sudo apt install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker ubuntu
```

Reconnect to apply group changes:

```bash
logout
ssh ubuntu@<bastion-ip>
```

#### 1.2 Install and start WireGuard

```bash
cd $K8_ROOT/wireguard/
mkdir -p wireguard/config
```

```bash
sudo docker run -d \
  --name=wireguard \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Asia/Calcutta \
  -e PEERS=30 \
  -p 51820:51820/udp \
  -v /home/ubuntu/wireguard/config:/config \
  -v /lib/modules:/lib/modules \
  --sysctl="net.ipv4.conf.all.src_valid_mark=1" \
  --restart unless-stopped \
  ghcr.io/linuxserver/wireguard
```

{% hint style="info" %}
Increase `PEERS=30` if more than 30 client profiles are needed.
{% endhint %}

Use the generated peer configuration from the selected peer directory:

```bash
cd peer1
cat peer1.conf
```

{% hint style="warning" %}
Store generated peer configurations securely. These are effectively access credentials to the internal environment.
{% endhint %}

***

### 2. Observation Kubernetes cluster setup and configuration

#### 2.1 Configure passwordless SSH

Generate keys on the bastion host:

```bash
ssh-keygen
cat ~/.ssh/id_rsa.pub
```

On each observation node, add the bastion public key to `~/.ssh/authorized_keys`.

Validate access:

```bash
ssh ubuntu@<observation-node-ip>
```

{% hint style="info" %}
Repeat the same passwordless SSH setup pattern later for the MOSIP cluster nodes.
{% endhint %}

#### 2.2 Install Docker on observation nodes using Ansible

```bash
cd $K8_ROOT/rancher/on-prem
cp hosts.ini.sample hosts.ini
vi hosts.ini
```

Update each node entry with:

* `ansible_host=<observation-node-ip>`
* `ansible_user=ubuntu`

Run the Docker installation playbook:

```bash
ansible-playbook -i hosts.ini docker.yaml
```

#### 2.3 Generate cluster configuration

```bash
rke config
```

Provide the required node details when prompted.

Update the generated `cluster.yml`:

```bash
vi cluster.yml
```

Set:

```yaml
ingress:
  provider: none
```

```yaml
cluster_name: observation-cluster
```

#### 2.4 Bring up the observation cluster

```bash
rke up
```

Successful output should end with:

```
Finished building the Kubernetes cluster successfully
```

#### 2.5 Configure kubeconfig and verify

```bash
mkdir -p $HOME/.kube
cp ~/k8s-infra/rancher/on-prem/kube_config_cluster.yml $HOME/.kube/rancher.conf
export KUBECONFIG="$HOME/.kube/rancher.conf"
kubectl get nodes
```

***

### 3. Observation cluster ingress and storage class setup

#### 3.1 NGINX Ingress Controller

```bash
cd $K8_ROOT/rancher/on-prem
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --version 4.0.18 \
  --create-namespace \
  -f ingress-nginx.values.yaml
```

Verify:

```bash
kubectl get all -n ingress-nginx
```

#### 3.2 Longhorn storage class

```bash
cd $K8_ROOT/longhorn
./pre_install.sh
./install.sh
kubectl get all -n longhorn-system
```

{% hint style="warning" %}
Before installing Longhorn, confirm that node disks, mount points, and firewall rules meet Longhorn prerequisites. Storage configuration issues are a common source of deployment failures.
{% endhint %}

***

### 4. Observation NGINX server setup

#### 4.1 SSL certificate setup

SSH to the observation NGINX VM and install prerequisites:

```bash
sudo apt update -y
sudo apt-get install software-properties-common -y
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt-get update -y
sudo apt-get install python3.8 -y
sudo apt install letsencrypt -y
sudo apt install certbot python3-certbot-nginx -y
```

Generate a wildcard certificate:

```bash
sudo certbot certonly --agree-tos --manual --preferred-challenges=dns -d *.credissure.com
```

{% hint style="warning" %}
Replace `*.credissure.com` with the actual environment domain. This step uses a DNS challenge and requires a temporary TXT record at `_acme-challenge.<domain>`.
{% endhint %}

#### 4.2 Install NGINX

```bash
ssh ubuntu@<observation-nginx-node>
sudo apt update
sudo apt install nginx -y
git clone https://github.com/mosip/k8s-infra -b v1.2.0.2
cd ~/k8s-infra/rancher/on-prem/nginx
sudo ./install.sh
```

The script will prompt for:

* Observation NGINX server IP
* SSL certificate path
* SSL key path
* Cluster node IPs

Check service status:

```bash
sudo systemctl status nginx
```

{% hint style="info" %}
Create DNS records for domains such as `rancher.<domain>` and `keycloak.<domain>` pointing to the Observation NGINX server.
{% endhint %}

***

### 5. Observation cluster applications

#### 5.1 Rancher UI

```bash
cd $K8_ROOT/rancher/rancher-ui
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

Update `rancher-values.yaml` with the correct hostname, for example `rancher.<domain>`, then install:

```bash
helm install rancher rancher-latest/rancher \
  --version 2.6.9 \
  --namespace cattle-system \
  --create-namespace \
  -f rancher-values.yaml
```

Get the bootstrap password:

```bash
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{ .data.bootstrapPassword|base64decode}}{{ "\n" }}'
```

#### 5.2 Keycloak for observation/admin

```bash
cd $K8_ROOT/rancher/keycloak
git checkout release-1.2.0.x
./install.sh <keycloak.domain.name>
git checkout v1.2.0.2
```

{% hint style="warning" %}
This Observation Keycloak is distinct from the MOSIP IAM setup installed later as an external dependency. Keep the purpose of each Keycloak deployment clear in your environment documentation.
{% endhint %}

***

### 6. MOSIP Kubernetes cluster setup

#### 6.1 Passwordless SSH

Repeat the same bastion-to-node SSH setup process used for the observation cluster, but for all MOSIP cluster nodes.

#### 6.2 Install Docker using Ansible

```bash
cd $K8_ROOT/mosip/on-prem
cp hosts.ini.sample hosts.ini
vi hosts.ini
```

Update:

* `ansible_host=<mosip-node-ip>`
* `ansible_user=ubuntu`

Run:

```bash
ansible-playbook -i hosts.ini docker.yaml
```

#### 6.3 Generate and launch RKE cluster

```bash
rke config
vi cluster.yml
```

Set:

```yaml
ingress:
  provider: none
```

```yaml
cluster_name: mosip-cluster
```

Bring up the cluster:

```bash
rke up
```

#### 6.4 Configure kubeconfig

```bash
mkdir -p $HOME/.kube
cp ~/k8s-infra/mosip/on-prem/kube_config_cluster.yml $HOME/.kube/config
export KUBECONFIG="$HOME/.kube/config"
kubectl get nodes
```

{% hint style="info" %}
Make sure your active `KUBECONFIG` points to the MOSIP cluster before applying global configmaps, Istio resources, or Rancher import manifests.
{% endhint %}

***

### 7. MOSIP cluster global configmap, ingress, and storage

#### 7.1 Global configmap

```bash
cd $K8_ROOT/mosip
cp global_configmap.yaml.sample global_configmap.yaml
vi global_configmap.yaml
kubectl apply -f global_configmap.yaml
```

{% hint style="warning" %}
Update all domain references carefully. Also, correct the spelling of `signup` in the global configmap where required, as noted in the original deployment steps.
{% endhint %}

#### 7.2 Istio ingress setup

```bash
cd $K8_ROOT/mosip/on-prem/istio
./install.sh
kubectl get svc -n istio-system
```

Verify the presence of:

* `istio-ingressgateway`
* `istio-ingressgateway-internal`
* `istiod`

#### 7.3 Longhorn storage

```bash
cd $K8_ROOT/longhorn
./pre_install.sh
./install.sh
kubectl get all -n longhorn-system
```

***

### 8. Import the MOSIP cluster into Rancher

1. Log in to Rancher
2. Select **Import Existing**
3. Choose **Generic**
4. Provide a cluster name, for example `mosip-sandbox`
5. Copy the provided import command and run it on the bastion while pointing to the MOSIP cluster kubeconfig

Example:

```bash
kubectl apply -f https://<rancher-host>/v3/import/<cluster-id>.yaml
```

{% hint style="warning" %}
Double-check the active `KUBECONFIG` before running the Rancher import command. Running it against the wrong cluster is a common operator mistake.
{% endhint %}

***

### 9. MOSIP NGINX server setup

#### 9.1 SSL certificates

Reuse the approved certificate if already provisioned for the MOSIP environment.

#### 9.2 Install and configure NGINX

```bash
ssh ubuntu@<mosip-nginx-node>
sudo apt update
sudo apt install nginx -y
git clone https://github.com/mosip/k8s-infra -b v1.2.0.2
cd ~/k8s-infra/mosip/on-prem/nginx
sudo ./install.sh
```

The script prompts for:

* MOSIP NGINX internal IP
* MOSIP NGINX public IP
* Publicly accessible MOSIP domains
* Full chain certificate path
* Private key path
* Cluster node IPs

Verify:

```bash
sudo systemctl status nginx
```

{% hint style="info" %}
For the current setup, monitoring can be skipped. Where module install scripts assume metrics are enabled, the required edits are called out later in this runbook.
{% endhint %}

***

### MOSIP external dependencies setup

{% hint style="warning" %}
Several dependency installation steps require switching temporarily to `release-1.2.0.x` and then switching back to `v1.2.0.2`. Do not miss the switch-back step.
{% endhint %}

#### 1. PostgreSQL

```bash
cd $INFRA_ROOT/deployment/v3/external/postgres
git checkout release-1.2.0.x
./install.sh
./init_db.sh
git checkout v1.2.0.2
```

{% hint style="warning" %}
The original note indicates that the PostgreSQL image referenced in the MOSIP documentation may be deprecated. Validate the image tag in the deployment script before installation.
{% endhint %}

#### 2. Keycloak

```bash
cd $INFRA_ROOT/deployment/v3/external/iam
./install.sh
./keycloak_init.sh
```

Use SMTP details as prompted:

```
keycloak.realms.mosip.realm_config.smtpServer.auth: "false"
keycloak.realms.mosip.realm_config.smtpServer.host: "smtp.gmail.com"
keycloak.realms.mosip.realm_config.smtpServer.port: "465"
keycloak.realms.mosip.realm_config.smtpServer.from: "mosipqa@gmail.com"
keycloak.realms.mosip.realm_config.smtpServer.starttls: "false"
keycloak.realms.mosip.realm_config.smtpServer.ssl: "true"
```

#### 3. SoftHSM

```bash
cd $INFRA_ROOT/deployment/v3/external/hsm/softhsm
./install.sh
```

Choose the object store option:

* **Option 1:** MinIO
* **Option 2:** S3

For the referenced flow, use **Option 1 for MinIO**.

#### 4. MinIO

```bash
git checkout release-1.2.0.x
cd $INFRA_ROOT/deployment/v3/external/object-store/minio
./install.sh
git checkout v1.2.0.2
```

#### 5. S3 credentials

```bash
cd $INFRA_ROOT/deployment/v3/external/object-store/
./cred.sh
```

Provide values such as:

```
REGION: leave blank or enter the applicable region
S3 Host: http://minio.minio:9000
```

#### 6. ClamAV

```bash
cd $INFRA_ROOT/deployment/v3/external/antivirus/clamav
./install.sh
```

#### 7. ActiveMQ

```bash
cd $INFRA_ROOT/deployment/v3/external/activemq
./install.sh
```

#### 8. Kafka

```bash
git checkout release-1.2.0.x
cd $INFRA_ROOT/deployment/v3/external/kafka
./install.sh
git checkout v1.2.0.2
```

#### 9. MSG Gateway

```bash
cd $INFRA_ROOT/deployment/v3/external/msg-gateway
./install.sh
```

{% hint style="info" %}
MOSIP provides a mock SMTP server as part of the default installation flow. Choose `Y` when prompted, if you want to use the mock SMTP setup.
{% endhint %}

#### 10. Captcha

Create separate reCAPTCHA credentials for:

* `prereg.<domain>`
* `resident.<domain>`

Then install:

```bash
cd $INFRA_ROOT/deployment/v3/external/captcha
./install.sh
```

{% hint style="warning" %}
During installation, the script will prompt for the site key and secret key for both the preregistration and resident domains. Keep these values ready before execution.
{% endhint %}

#### 11. Landing page

```bash
cd $INFRA_ROOT/deployment/v3/external/landing-page
./install.sh
```

***

### MOSIP modules deployment

{% hint style="info" %}
The following order reflects the current working deployment sequence used in the source notes. Where monitoring is skipped, some scripts need `--set metrics.enabled=false` added manually before execution.
{% endhint %}

#### 1. Conf Secrets

```bash
cd $INFRA_ROOT/deployment/v3/mosip/conf-secrets
./install.sh
```

#### 2. Config Server

```bash
cd $INFRA_ROOT/deployment/v3/mosip/config-server
./install.sh
```

#### 3. Artifactory

```bash
cd $INFRA_ROOT/deployment/v3/mosip/artifactory
./install.sh
```

#### 4. Keymanager

```bash
git checkout release-1.2.0.x
cd $INFRA_ROOT/deployment/v3/mosip/keymanager
vi install.sh
```

Add:

```
--set metrics.enabled=false
```

Then:

```bash
./install.sh
git checkout v1.2.0.2
```

#### 5. WebSub

```bash
cd $INFRA_ROOT/deployment/v3/mosip/websub
./install.sh
```

#### 6. Mock SMTP

```bash
cd $INFRA_ROOT/deployment/v3/mosip/mock-smtp
./install.sh
```

#### 7. Kernel

```bash
cd $INFRA_ROOT/deployment/v3/mosip/kernel
./install.sh
```

{% hint style="warning" %}
The source notes mention adding `--set metrics.enabled=false` for the Keymanager chart because monitoring is not installed. Review any dependent charts that inherit or reference the metrics configuration.
{% endhint %}

#### 8. Masterdata Loader

```bash
cd $INFRA_ROOT/deployment/v3/mosip/masterdata-loader
./install.sh
```

#### 9. Mock BioSDK

```bash
cd $INFRA_ROOT/deployment/v3/mosip/biosdk
vi install.sh
```

Add:

```
--set metrics.enabled=false
```

Then:

```bash
./install.sh
```

#### 10. Packet Manager

```bash
cd $INFRA_ROOT/deployment/v3/mosip/packetmanager
vi install.sh
./install.sh
```

Add:

```
--set metrics.enabled=false
```

#### 11. DataShare

```bash
cd $INFRA_ROOT/deployment/v3/mosip/datashare
vi install.sh
./install.sh
```

Add:

```
--set metrics.enabled=false
```

#### 12. Pre-Registration

```bash
cd $INFRA_ROOT/deployment/v3/mosip/prereg
vi install.sh
./install.sh
```

Add:

```
--set metrics.enabled=false
```

#### 13. ID Repo

```bash
cd $INFRA_ROOT/deployment/v3/mosip/idrepo
vi install.sh
./install.sh
```

Add:

```
--set metrics.enabled=false
```

#### 14. Partner Management Services

```bash
cd $INFRA_ROOT/deployment/v3/mosip/pms
vi install.sh
./install.sh
```

Add:

```
--set metrics.enabled=false
```

#### 15. Mock ABIS

```bash
cd $INFRA_ROOT/deployment/v3/mosip/mock-abis
vi install.sh
./install.sh
```

Add:

```
--set metrics.enabled=false
```

#### 16. Mock MV

```bash
cd $INFRA_ROOT/deployment/v3/mosip/Mock-mv
vi install.sh
./install.sh
```

Add:

```
--set metrics.enabled=false
```

#### 17. Registration Processor

```bash
cd $INFRA_ROOT/deployment/v3/mosip/regproc
vi install.sh
./install.sh
```

Add:

```
--set metrics.enabled=false
```

#### 18. Admin

```bash
cd $INFRA_ROOT/deployment/v3/mosip/admin
vi install.sh
./install.sh
```

Add:

```
--set metrics.enabled=false
```

#### 19. ID Authentication

```bash
cd $INFRA_ROOT/deployment/v3/mosip/ida
vi install.sh
./install.sh
```

Add:

```
--set metrics.enabled=false
```

#### 20. Print

```bash
cd $INFRA_ROOT/deployment/v3/mosip/print
vi install.sh
./install.sh
```

Add:

```
--set metrics.enabled=false
```

#### 21. Partner Onboarder

```bash
cd $INFRA_ROOT/deployment/v3/mosip/partner-onboarder
./install.sh
```

Provide values when prompted:

```
Do you have a public domain & valid SSL? Y
Provide onboarder bucket name: onboarder
Provide onboarder S3 bucket region: leave empty
Provide S3 URL: http://minio.minio:9000
```

#### 22. MOSIP File Server

```bash
cd $INFRA_ROOT/deployment/v3/mosip/mosip-file-server
./install.sh
```

#### 23. Resident Services

```bash
cd $INFRA_ROOT/deployment/v3/mosip/resident
vi install.sh
./install.sh
```

Required changes:

* Add `--set metrics.enabled=false`
* Correct the JSONPath for the resident configuration as noted in the source deployment notes

{% hint style="warning" %}
The resident deployment includes a known manual correction to the JSONPath. Keep this step explicitly documented in your GitBook page with the exact diff or screenshot if the team uses the same branch and scripts.
{% endhint %}

#### 24. Registration Client

```bash
cd $INFRA_ROOT/deployment/v3/mosip/regclient
sudo apt-get update
sudo apt-get install jq -y
./install.sh
```

{% hint style="danger" %}
Registration Client installation is sensitive to OpenSSL compatibility. Confirm the OpenSSL version before starting this step.
{% endhint %}

***
