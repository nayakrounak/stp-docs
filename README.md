## Table of Content
- [Runbook for Mosip](#runbook-for-mosip)
- [Difference in MOSIP Documentation vs Ooru Runbook for k8s cluster](#2-difference-in-mosip-documentation-vs-ooru-runbook-for-k8s-cluster)
    - [MOSIP Documentation](#mosip-documentation)
    - [Ooru Runbook](#ooru-runbook)
- [Pre-Requisites](#3-pre-requisites)
    - [Hardware requirements](#hardware-requirements)
    - [DNS requirements](#dns-requirements)
    - [Git](#git-version-2251-or-higher)
    - [Environment Variable Configuration](#environment-variable-configuration)
    - [kubectl](#kubectl-version-2124-or-higher)
    - [Istioctl](#istioctl-version-1150)
    - [rke](#rke-version-v1310)
    - [Helm](#helm-installation-any-client-version-above-300)
    - [Ansible](#ansible-version--2124)
    - [Clone Repo](#clone-k8s-infra-repo-with-tag--1202)
- [On-Premises Deployment](#on-premises-deployment)
     - [Wireguard setup](#1-wireguard-bastion-server-setup)
     - [Observation cluster setup](#2-observation-k8s-cluster-setup-and-configuration)
     - [Observation K8s Cluster Ingress, Storageclass setup](#3-observation-k8s-cluster-ingress-storageclass-setup)
     - [Setting up Nginx Server for Observation K8s Cluster](#4-setting-up-nginx-server-for-observation-k8s-cluster)
     - [Observation K8's Cluster Apps Installation](#5-observation-k8s-cluster-apps-installation)
     - [Cluster Global Configmap](#7-mosip-k8-cluster-global-configmap-ingress-and-storage-class-setup)
     - [Import MOSIP Cluster into Rancher UI](#8-import-mosip-cluster-into-rancher-ui)
- [MOSIP External Dependencies](#mosip-external-dependencies-setup)
    - [PostgreSQL](#1-postgresql-installation)
    - [Keycloak](#2-keycloak)
    - [SoftHSM](#3-setup-softhsm)
    - [MinIO](#4-minio-installation)
    - [S3 Credentials](#5-s3-credentials-setup)
    - [ClamAV](#6-clamav-setup)
    - [ActiveMQ](#7-activemq-setup)
    - [Kafka](#8-kafka-setup)
    - [MSG Gateway](#9-msg-gateway)
    - [Captcha](#10-captcha)
    - [Landing page](#11-landing-page-setup)
- [MOSIP Modules Deployment](#11-mosip-modules-deployment)
    - [Conf secrets](#1-conf-secrets)
    - [Config Server](#2-config-server)
    - [Artifactory](#3-artifactory)
    - [Keymanager](#4-keymanager)
    - [WebSub](#5-websub)
    - [Mock-SMTP](#6-mock-smtp)
    - [Kernel](#7-kernel)
    - [Masterdata-loader](#8-masterdata-loader)
    - [Mock-biosdk](#9-mock-biosdk)
    - [Packetmanager](#10-packetmanager)
    - [Datashare](#11-datashare)
    - [Pre-reg](#12-pre-reg)
    - [Idrepo](#13-idrepo)
    - [Partner Management Services](#14-partner-management-services)
    - [Mock ABIS](#15-mock-abis)
    - [Mock-mv](#16-mock-mv)
    - [Registration Processor](#17-registration-processor)
    - [Admin](#18-admin)
    - [ID Authentication](#19-id-authentication)
    - [Print](#20-print)
    - [Partner Onboarder](#21-partner-onboarder)
    - [MOSIP File Server](#22-mosip-file-server)
    - [Resident services](#23-resident-services)
    - [Registration Client](#24-registration-client)

- [API Testrig Setup](#api-testrig-setup)


# Runbook for Mosip

- **Comprehensive Deployment Flow:**  
  Provides a detailed, step-by-step process — from **infrastructure setup** to **module deployment** — including all necessary commands for seamless execution.

- **OS Transparency:**  
  Clearly specifies the **OS image** used for each node, ensuring consistency and ease of replication across environments.

- **Reliable & Verified Images:**  
  Utilizes **active and verified OS images**, guaranteeing smooth, stable, and error-free deployment of all **MOSIP modules**.





## 1) Difference in MOSIP Documentation vs Ooru Runbook for k8s cluster

- ### **MOSIP Documentation**
     - The MOSIP External Module (Postgres) image used is deprecated it needs to be updated in the deployment script.
     - It does not mention that the Monitoring deployment step can be skipped.

- ### **Ooru Runbook**
    - Provides a step-by-step process from infrastructure setup to module deployment, including all necessary commands.

    - Uses active and verified images, ensuring smooth deployment of all MOSIP modules without issues.



## 2) Pre-Requisites 

## Hardware requirements

 *Here, we are referring to Ubuntu OS (22.04) throughout this installation guide.*  


| Sl no. | Purpose | vCPU’s | RAM | Storage (HDD) | Number of VMs | HA | os image
|--------|---------|--------|------|----------------|----------------|-----|----------|
| 1 | Wireguard Bastion Host | 2 | 4 GB | 8 GB | 1 | (Active-Passive) | 22.04 (ubuntu)
| 2 | Observation Cluster nodes | 2 | 8 GB | 32 GB | 2 | 2 | 22.04 (ubuntu)
| 3 | Observation Nginx server (or Load Balancer) | 2 | 4 GB | 16 GB | 1 | Nginx+ | 22.04 (ubuntu)
| 4 | MOSIP Cluster nodes | 12 | 32 GB | 128 GB | 6 | 6 | 22.04 (ubuntu)
| 5 | MOSIP Nginx server (or Load Balancer) | 2 | 4 GB | 16 GB | 1 | Nginx+ | 22.04 (ubuntu)


## DNS requirements

| # | Domain Name | Mapping Details | Purpose |
| :---: | :--- | :--- | :--- |
| 1 | `rancher.xyz.net` | Private IP of Nginx server or load balancer for **Observation cluster** | Rancher dashboard for Kubernetes cluster monitoring and management. |
| 2 | `keycloak.xyz.net` | Private IP of Nginx server for **Observation cluster** | Administrative IAM tool (Keycloak) for Kubernetes administration. |
| 3 | `sandbox.xyx.net` | Private IP of Nginx server for **MOSIP cluster** | Index page for links to different dashboards of MOSIP environment. (Internal reference only, do not expose in production/UAT). |
| 4 | `api-internal.sandbox.xyz.net` | Private IP of Nginx server for **MOSIP cluster** | Internal APIs exposed and accessible privately over the Wireguard channel. |
| 5 | `api.sandbox.xyx.net` | Public IP of Nginx server for **MOSIP cluster** | All publicly usable APIs are exposed using this domain. |
| 6 | `prereg.sandbox.xyz.net` | Public IP of Nginx server for **MOSIP cluster** | MOSIP's pre-registration portal, accessible publicly. |
| 7 | `activemq.sandbox.xyx.net` | Private IP of Nginx server for **MOSIP cluster** | Direct access to the ActiveMQ dashboard, limited to access over Wireguard. |
| 8 | `kibana.sandbox.xyx.net` | Private IP of Nginx server for **MOSIP cluster** | Optional installation. Used to access the Kibana dashboard over Wireguard. |
| 9 | `regclient.sandbox.xyz.net` | Private IP of Nginx server for **MOSIP cluster** | Registration Client download domain, accessible over Wireguard. |
| 10 | `admin.sandbox.xyz.net` | Private IP of Nginx server for **MOSIP cluster** | MOSIP's Admin portal, restricted to access over Wireguard. |
| 11 | `object-store.sandbox.xyx.net` | Private IP of Nginx server for **MOSIP cluster** | Optional. Used to access the object server console (e.g., MinIO Console) over Wireguard. |
| 12 | `kafka.sandbox.xyz.net` | Private IP of Nginx server for **MOSIP cluster** | Access to Kafka UI for administrative needs over Wireguard. |
| 13 | `iam.sandbox.xyz.net` | Private IP of Nginx server for **MOSIP cluster** | Access to the OpenID Connect server (Keycloak) for managing service access over Wireguard. |
| 14 | `postgres.sandbox.xyz.net` | Private IP of Nginx server for **MOSIP cluster** | Points to the PostgreSQL server. Connection is typically via port forwarding over Wireguard. |
| 15 | `pmp.sandbox.xyz.net` | Private IP of Nginx server for **MOSIP cluster** | MOSIP's Partner Management Portal, accessed over Wireguard. |
| 16 | `onboarder.sandbox.xyz.net` | Private IP of Nginx server for **MOSIP cluster** | Accessing reports for MOSIP partner onboarding over Wireguard. |
| 17 | `resident.sandbox.xyz.net` | Public IP of Nginx server for **MOSIP cluster** | Accessing the Resident Portal publicly. |
| 18 | `idp.sandbox.xyz.net` | Public IP of Nginx server for **MOSIP cluster** | Accessing the IDP publicly. |
| 19 | `smtp.sandbox.xyz.net` | Private IP of Nginx server for **MOSIP cluster** | Accessing the mock-SMTP UI over Wireguard. |


- Install all the required tools on bastion host

- ### **Git (version 2.25.1 or higher)**

```
sudo apt update
sudo apt install git
```
- ### **Environment Variable Configuration**

```
echo 'export MOSIP_ROOT=/home/ubuntu' >> ~/.bashrc
echo 'export K8_ROOT=$MOSIP_ROOT/k8s-infra' >> ~/.bashrc
echo 'export INFRA_ROOT=$MOSIP_ROOT/mosip-infra' >> ~/.bashrc
source ~/.bashrc
```

- ### **kubectl (version 2.12.4 or higher)**

```
curl -LO "https://dl.k8s.io/release/v1.34.1/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```
- ### **Istioctl (version: 1.15.0)**

```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.15.0 sh -
cd istio-1.15.0
sudo mv bin/istioctl /usr/local/bin/
istioctl version --remote=false
```
- ### **rke (version v1.3.10)**

```
curl -L "https://github.com/rancher/rke/releases/download/v1.3.10/rke_linux-amd64" -o rke
chmod +x rke
sudo mv rke /usr/local/bin/
rke --version
```

- ### **helm installation (any client version above 3.0.0)**

```
Curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add mosip https://mosip.github.io/mosip-helm
```
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add mosip https://mosip.github.io/mosip-helm
```
- ### **Ansible (version > 2.12.4)**

```
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

- ### **clone k8’s infra repo with tag : 1.2.0.2**
```
git clone https://github.com/mosip/k8s-infra -b v1.2.0.2
git clone https://github.com/mosip/mosip-infra -b v1.2.0.2
```
- ### **Openssl Need openssl version 1.1.1 specifically for regclient installation.**
    - Check the current OpenSSL version:
       ```
       openssl version
       ```
    - If it's not OpenSSL 1.1.1f, remove the existing OpenSSL
      ```
      sudo apt remove openssl 
      ```
    - Manually install OpenSSL 1.1.1f by following commands.
    - Install OpenSSL manually in Ubuntu
      ```
      sudo apt install build-essential checkinstall zlib1g-dev -y
      ```
    - Once you're done with installing prerequisites, change your directory to /usr/local/src/:
      ```
      cd /usr/local/src/
      ```
    - Now, use the wget command to download OpenSSL:
      ```
      sudo wget https://www.openssl.org/source/openssl-1.1.1s.tar.gz
      ```
    - Once you are done with downloading, extract the tar file using the tar utility:
      ```
      sudo tar -xf openssl-1.1.1s.tar.gz
      ```
    - And navigate to the recently extracted tar file:
      ```
      cd openssl-1.1.1s
      ```
    - Now, let's start the installation process by configuration process and it should give you make file:
      ```
      sudo ./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl shared zlib
      ```
    - Now, let's invoke the make command to build OpenSSL:
      ```
      sudo make
      ```
    - To check whether there are no errors in the recent build, use the given command:
      ```
      sudo make test
      ```
    - And if everything goes right, you will get the message "All tests successful". So now, you can proceed with the installation:
      ```
      sudo make install
      ```
    - Configure OpenSSL shared libraries using `vi ` editor and save the using `:wq!`
      ```
      sudo vi /etc/ld.so.conf.d/openssl-1.1.1s.conf
      ```
    - And add the following line to that config file:
      ```
      /usr/local/ssl/lib
      ```
    - Save the config file and reload it to apply recently made changes using the given command:
      ```
      sudo ldconfig -v
      ```
    - Configure OpenSSL Binary
      ```
      sudo mv /usr/bin/c_rehash /usr/bin/c_rehash.backup
      sudo mv /usr/bin/openssl /usr/bin/openssl.backup
      ```
    - Now, open the environment PATH variable using the given command:
      ```
      sudo vi /etc/environment
      ```
    - And add :/usr/local/ssl/bin to the end to add /usr/local/bin/openssl folder to the PATH variable:
      ```
      PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/local/ssl/bin"
      ```
    - Save and exit the editor using `wq!` and now reload the PATH variable using the given command:
      ```
      source /etc/environment
      ```
    - Now, you can check for the installed OpenSSH version:
      ```
      openssl version -a
      ```
      <img width="1254" height="234" alt="openssl-version" src="https://github.com/user-attachments/assets/b88fb1af-54bd-4c19-81ae-87a70e3d41bf" />


## On-Premises Deployment

# 1. Wireguard Bastion Server Setup

The Wireguard bastion server provides a secure, private channel to access the MOSIP cluster nodes. 

- Please ensure that UDP port 51820 is open on the bastion server. If you are managing the security group, kindly allow inbound traffic on UDP port 51820.

### 1.a. Setup Wireguard VM and Bastion Server

- Install Docker on the bastion host using the following command:

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc


sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

```
- Once Docker is installed, please add the ubuntu user to the Docker group to allow non-root access.

```bash
sudo usermod -aG docker ubuntu
```
- After completing the steps, please log out and log back in to the bastion server to apply the changes.

```bash
logout
```
```bash
ssh ubuntu@<ip-address-of-ubuntu>
```

- Navigate to the Wireguard directory:
```bash
cd $K8_ROOT/wireguard/
```
- **Create Config Directory:**
```bash
mkdir -p wireguard/config
```
**Install and Start Wireguard Server (using Docker):**
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
> **Note:**
> * Increase `-e PEERS=30` if more than 30 client configurations are needed.

- Once WireGuard is installed, it will generate 30 peers, which must be used for the MOSIP deployment.
<img width="1298" height="125" alt="wireguard-docker-conatiner" src="https://github.com/user-attachments/assets/ddff7e7e-4fc4-495d-8b0e-19954dbe8246" />
<img width="1298" height="125" alt="peer-wireguard" src="https://github.com/user-attachments/assets/477d7ffb-1a8b-4de7-853d-0c308dbacb33" />

- Get inside your selected peer directory, and add mentioned changes in peer.conf:
```bash
cd peer1
cat peer1.conf
```
- Use the **peer.conf file** to connect to WireGuard.

# 2. Observation K8s Cluster setup and configuration
- Setup passwordless SSH into the cluster nodes via pem keys. (Ignore if VM’s are accessible via pem’s).
1.  **Generate keys on bastion host:**
    ```bash
    ssh-keygen
    ```
    <img width="781" height="417" alt="bastion-ssh-keygen" src="https://github.com/user-attachments/assets/4a231536-3511-45ed-80b0-04739a0ad737" />

2.  **Copy the public keys of bastion host to remote observation node VM’s:**
    ```bash
     cat ~/.ssh/id_rsa.pub
    ```
    <img width="1299" height="155" alt="copy-ssh-pub-key" src="https://github.com/user-attachments/assets/62471c5b-d04f-4b8f-b82f-a42a4c0ffb31" />

3.  **SSH into the observation node to check password-less SSH:**
    ```bash
    ssh ubuntu@<ip-address-of-ubuntu>
     ```
4.  **Generate keys on observation node:**
    ```bash
    ssh-keygen
    ```
5.  **Create a file in .ssh on observation node:**
    ```bash
    vi .ssh/authorized_keys
    ```
6.  **Paste the public key content of the bastion host into the file and use :wq! to save the file:**

   <img width="1304" height="693" alt="paste-pub-key-of-bastion" src="https://github.com/user-attachments/assets/a185d9ad-d8c6-4af5-8a98-f1da8d79f3e6" />

- Install Docker on Observation K8 Cluster node VM’s.

1.  **Navigate to the Ansible directory:**
    ```bash
    cd $K8_ROOT/rancher/on-prem
    ```
2.  **Copy and Update `hosts.ini` `ansible_host=<observation-node-ip>` `ansible_user=ubuntu`:**
    Copy `hosts.ini.sample` to `hosts.ini` and update required details.
    ```bash
    cp hosts.ini.sample hosts.ini
    ```
    <img width="517" height="97" alt="update-ansible-inventory" src="https://github.com/user-attachments/assets/4e3e8ff8-57ce-4340-a2f8-a989859a1fb6" />

#### 3. **To update host.ini use editor:**
- Open the file using the following command:

  ```bash
  vi hosts.ini
  ```
- Press **i** to enter insert mode.
- Update the `hosts.ini` file with the information provided above.
- After updating, press `Esc` to exit insert mode.
- Type `:wq!` and press Enter to save and exit the file.

4. **execute docker.yml to install docker**
```bash
ansible-playbook -i hosts.ini docker.yaml
```

####  Creating RKE Cluster Configuration file
1.  **Generate `cluster.yml` using `rke config`:**
    ```bash
    rke config
    ```
    The command will prompt for nodal details related to the cluster.
    * **Number of Hosts:** `<number of cluster nodes>` (2)
    * **SSH Address of host:** `<node-ip-add>`
    * **SSH User of host:** `<remote-user>` (ubuntu)
    * **Is host (<node1-ip>) a Control Plane host (y/n)? [y]:** `y` (For the first node)
    * **Is host (<node1-ip>) a Worker host (y/n)? [n]:** `y`
    * **Is host (<node1-ip>) an etcd host (y/n)? [n]:** `y` (For the first node)

2.  **Update `cluster.yml`:**
    As a result of `rke config` command, `cluster.yml` file will be generated inside the same directory. Update the below mentioned fields:
    ```bash
    vi cluster.yml
    ```
    * **Remove the default Ingress install:**
        ```yaml
        ingress:
          provider: none
        ```
    * **Update the name of the kubernetes cluster:**
        ```yaml
        cluster_name: observation-cluster
        ```
    - Press **i** to enter insert mode.
    - After updating, press `Esc` to exit insert mode.
    - Type `:wq!` and press Enter to save and exit the file.

## 2.3. Setup up the Cluster

1.  **Bring up the Kubernetes cluster:**
    Once `cluster.yml` is ready, you can bring up the kubernetes cluster using a simple command. 
    ```bash
    rke up
    ```
    Example successful output:
    ```
    INFO[0000] Building Kubernetes cluster
    INFO[0000] [dialer] Setup tunnel for host [10.0.0.1]
    INFO[0000] [network] Deploying port listener containers   
    INFO[0000] [network] Pulling image [alpine:latest] on host [10.0.0.1]
    ...
    INFO[0101] Finished building Kubernetes cluster successfully
    ```
    The last line should read `Finished building Kubernetes cluster successfully` to indicate that your cluster is ready to use.
    
 
## 2.4. Access and Verification



1.  **Access the cluster using kubeconfig file:**

    * **Option A: If the `.kube` folder is not present, create it and then create the `rancher.conf` file inside it**
        ```bash 
       mkdir $HOME/.kube
       touch $HOME/.kube/rancher.conf
         ```
    * **Option B: Copy to default config path**
        ```bash
        cp ~/k8s-infra/rancher/on-prem/kube_config_cluster.yml $HOME/.kube/rancher.conf
        ``` 
    * **Option C: Export KUBECONFIG variable**
        ```bash
        export KUBECONFIG="$HOME/.kube/rancher.conf"
        ```

2.  **Test cluster access:**
    ```bash
    kubectl get nodes
    ```
    <img width="492" height="76" alt="obs-node" src="https://github.com/user-attachments/assets/2df935d5-7641-408a-bde1-9c461f31915c" />

## 3. Observation K8s Cluster Ingress, Storageclass setup

Once the RKE cluster is ready, an Ingress Controller and Storage Class must be configured for other applications.

### 3.a. Nginx Ingress Controller
1.  **Navigate to the installation directory:**
    ```bash
    cd $K8_ROOT/rancher/on-prem
    ```

2.  **Add the Nginx Ingress Helm repository:**
    ```bash
    helm repo add ingress-nginx [https://kubernetes.github.io/ingress-nginx](https://kubernetes.github.io/ingress-nginx)
    ```

3.  **Update the Helm repositories:**
    ```bash
    helm repo update
    ```
4.  **Install the Nginx Ingress Controller using Helm:**
    ```bash
    helm install \                                                                                                             
      ingress-nginx ingress-nginx/ingress-nginx \
      --namespace ingress-nginx \
      --version 4.0.18 \
      --create-namespace  \
      -f ingress-nginx.values.yaml
    ```
5.  **Cross-check the installation:**
    Verify that all necessary components (pods, deployments, etc.) are running in the new namespace.
    ```bash
    kubectl get all -n ingress-nginx
    ```
### 3.b. Storage Classes (longhorn)
1.  **Navigate to the installation directory and install Pre-requisites:**
    ```
      cd $K8_ROOT/longhorn
      ./pre_install.sh
       ```

2.  **Install Longhorn via helm:**
    ```
    ./install.sh
    ```
3.  **Cross-check the installation:**
    ```
    kubectl get all -n longhorn-system
    ```

## 4. Setting up Nginx Server for Observation K8s Cluster

The Nginx server acts as a reverse proxy for the cluster, handling TLS termination and routing traffic.

### 4.a. SSL Certificate Setup for TLS Termination



1.  **SSH into the Nginx Server VM.**
2.  **Install Pre-requisites (on Nginx VM):**
    ```bash
    sudo apt update -y
    sudo apt-get install software-properties-common -y
    sudo add-apt-repository ppa:deadsnakes/ppa
    sudo apt-get update -y
    sudo apt-get install python3.8 -y
    sudo apt install letsencrypt -y
    sudo apt install certbot python3-certbot-nginx -y
    ```
3.  **Generate Wildcard SSL Certificate (using Certbot/Let's Encrypt):**
    ```bash
    sudo certbot certonly --agree-tos --manual --preferred-challenges=dns -d *.credissure.com
    # Replace org.net with your domain.
    ```
    > **Note:** This uses a DNS challenge, requiring you to create a `TXT` record (`_acme-challenge.<your-domain>`) in your DNS service with the string prompted by the script. Verify the entry is active using `host -t TXT _acme-challenge.credissure.com`.


### 4.b. Install Nginx

1.  **Ssh into the observation nginx server**
    ```
     ssh ubuntu@<observation-nginx-node>
    ```

2.  **Install Nginx Software (on Nginx VM):**
    ```bash
    sudo apt update
    sudo apt install nginx -y
    ```
3. **clone the repo:**
    ```
    git clone https://github.com/mosip/k8s-infra -b v1.2.0.2
    ```
    - Nevigate inside the directory
    ```
    cd  ~/k8s-infra/rancher/on-prem/nginx$
    sudo ./install.sh
    ```
    * **Provide Inputs when prompted:**
        * **Rancher nginx ip:** Internal IP of the Nginx server VM.
        * **SSL cert path:** Path to the SSL certificate (e.g., from `/etc/letsencrypt`).
        * **SSL key path:** Path to the SSL key.
        * **Cluster node ip's:** IPs of the RKE cluster nodes.

4.  **Post Installation Check:**
    ```bash
    sudo systemctl status nginx
    ```

5.  **DNS Mapping:** Create DNS records for domains like `rancher.credissure.com` and `keycloak.credissure.com` pointing to the Nginx server's public IP. 

---

## 5. Observation K8's Cluster Apps Installation

### 5.a. Rancher UI

1.  **Install Rancher using Helm:**
    * Navigate to the Rancher directory in bastion host:
        ```bash
        cd $K8_ROOT/rancher/rancher-ui
        ```
    * Add and update the Helm repository:
        ```bash
        helm repo add rancher-latest [https://releases.rancher.com/server-charts/latest](https://releases.rancher.com/server-charts/latest)
        helm repo update
        ```
    * **Install Rancher:** Update the hostname in `rancher-values.yaml` using `vi` editor eg. rancher.credissure.com and execute:
      ```bash
      helm install rancher rancher-latest/rancher \
      --version 2.6.9 \
      --namespace cattle-system \
      --create-namespace \
      -f rancher-values.yaml
      ```
      <img width="496" height="221" alt="update-rancher-vaule yml" src="https://github.com/user-attachments/assets/5449dbc3-cc7d-4e28-b85c-29a5f1514348" />
2.  **Initial Login:**
    * Access the Rancher page via the browser (ensure Wireguard/VPN is connected).
    * Get the bootstrap password:
        ```bash
        kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{ .data.bootstrapPassword|base64decode}}{{ "\n" }}'
        ```

### 5.b.  keycloak setup
1.  **Install Rancher using bash:**
    * Navigate to the keycloak directory in bastion host:
        ```bash
        cd $K8_ROOT/rancher/keycloak
        ```
    * Switch the branch to release-1.2.0.x:
        ```bash
        git checkout release-1.2.0.x
        ```
    * Install keycloak using bash:
        ```bash
        ./install.sh <keycloak.domain.name>
        ```
     <img width="595" height="42" alt="rancherkeycloak-install" src="https://github.com/user-attachments/assets/df9b047b-5c97-4768-a3e5-7fb096f72d14" />

    * Switch the branch to v1.2.0.2:
        ```bash
        git checkout v1.2.0.2
        ```
    
## 6. MOSIP K8s Cluster Setup



### 6.1. Pre-requisites & Initial Checks

- Setup passwordless SSH into the cluster nodes via pem keys. (Ignore if VM’s are accessible via pem’s).
1.  **Generate keys on bastion host:**
    ```bash
    ssh-keygen
    ```
    <img width="781" height="417" alt="bastion-ssh-keygen" src="https://github.com/user-attachments/assets/682bbe93-95ac-4316-a625-6bfc2c070848" />

2.  **Copy the public keys of bastion host to remote observation node VM’s:**
    ```bash
     cat ~/.ssh/id_rsa.pub
    ```
    <img width="1299" height="155" alt="copy-ssh-pub-key" src="https://github.com/user-attachments/assets/fb666054-0a45-4e1a-8e92-3f12bd4f4750" />


3.  **SSH into the observation node to check password-less SSH:**
    ```bash
    ssh ubuntu@<ip-address-of-ubuntu>
     ```
4.  **Generate keys on observation node:**
    ```bash
    ssh-keygen
    ```
5.  **Create a file in .ssh on observation node:**
    ```bash
    vi .ssh/authorized_keys
    ```
6.  **Paste the public key content of the bastion host into the file and use :wq! to save the file:**

    <img width="1304" height="693" alt="paste-pub-key-of-bastion" src="https://github.com/user-attachments/assets/30d02cf6-ccc8-4e71-a408-8776c2084d61" />

### 6.2. Prepare and Execute Ansible Playbooks

1.  **Navigate to MOSIP Ansible directory:** `cd $K8_ROOT/mosip/on-prem`.
2.  **Update `hosts.ini`** (Copy from sample if not already done).
2.  **Copy and Update `hosts.ini` `ansible_host=<observation-node-ip>` `ansible_user=ubuntu`:**
    Copy `hosts.ini.sample` to `hosts.ini` and update required details.
    ```bash
    cp hosts.ini.sample hosts.ini
    ```
    <img width="517" height="97" alt="update-ansible-inventory" src="https://github.com/user-attachments/assets/eb4c1e29-e3bb-4ef4-8906-0c544b089451" />

#### 4. **To update host.ini use editor:**
- Open the file using the following command:

  ```bash
  vi hosts.ini
  ```
- Press **i** to enter insert mode.
- Update the `hosts.ini` file with the information provided above.
- After updating, press `Esc` to exit insert mode.
- Type `:wq!` and press Enter to save and exit the file.
5.  **Execute Playbooks:**
    * **Install Docker:**
        ```bash
        ansible-playbook -i hosts.ini docker.yaml
        ```
### 6.3. Creating and Launching the RKE Cluster

1.  **Generate `cluster.yml`:**
    ```bash
    rke config
    ```
2.  **Update `cluster.yml`:**
    As a result of `rke config` command, `cluster.yml` file will be generated inside the same directory. Update the below mentioned fields:
    ```bash
    vi cluster.yml
    ```
    * **Remove the default Ingress install:**
        ```yaml
        ingress:
          provider: none
        ```
    * **Update the name of the kubernetes cluster:**
        ```yaml
        cluster_name: mosip-cluster
        ```
    - Press **i** to enter insert mode.
    - After updating, press `Esc` to exit insert mode.
    - Type `:wq!` and press Enter to save and exit the file.

3. **Bring up the Kubernetes cluster:**
    Once `cluster.yml` is ready, you can bring up the kubernetes cluster using a simple command. 
    ```bash
    rke up
    ```
    Example successful output:
```
    INFO[0000] Building Kubernetes cluster
    INFO[0000] [dialer] Setup tunnel for host [10.0.0.1]
    INFO[0000] [network] Deploying port listener containers   
    INFO[0000] [network] Pulling image [alpine:latest] on host [10.0.0.1]
    ...
    INFO[0101] Finished building Kubernetes cluster successfully
```
    The last line should read `Finished building Kubernetes cluster successfully` to indicate that your cluster is ready to use.

4. **Access the cluster using kubeconfig file:**

    * **Option A: Create the `default.conf` file inside it**
        ```bash 
       mkdir $HOME/.kube
       touch $HOME/.kube/config
         ```
    * **Option B: Copy to default config path**
        ```bash
        cp $HOME/.kube/kube_config_cluster.yml $HOME/.kube/config
        ``` 
    * **Option C: Export KUBECONFIG variable**
        ```bash
        export KUBECONFIG="$HOME/.kube/config"
        ```
5.  **Test cluster access:**
    ```bash
    kubectl get nodes
    ```
    <img width="612" height="157" alt="mosip-cluster-node" src="https://github.com/user-attachments/assets/adfac243-3a82-4b82-84ac-5d6ef6041d78" />

## 7. MOSIP K8 Cluster Global Configmap, Ingress and Storage Class setup

### 7.a. Global Configmap

This contains necessary common details for applications across the MOSIP cluster namespaces.

1.  **Prepare Configmap:**
    * Navigate to the MOSIP root directory: `cd $K8_ROOT/mosip`.
    * Copy and update domain names in the configuration file:
        ```bash
        cp global_configmap.yaml.sample global_configmap.yaml
        ```
2. **Update domain in global configmap and correct the spelling of `signup` in global configmap:**
    <img width="656" height="539" alt="update-global-config-map" src="https://github.com/user-attachments/assets/ccf2c877-0ac0-43a8-bace-cd0bc0db0d61" />

3.  Open the file using the following command:

    ```bash
    vi global_configmap.yaml
    ```
- Press **i** to enter insert mode.
- Update the `global_configmap.yaml` file with the information provided above.
- After updating, press `Esc` to exit insert mode.
- Type `:wq!` and press Enter to save and exit the file.

4.  **Apply Configmap:**
    ```bash
    kubectl apply -f global_configmap.yaml
    ```
### 7.b. Istio Ingress Setup



1.  **Install Istio:**
    * Navigate to the Istio directory: `cd $K8_ROOT/mosip/on-prem/istio`.
    * Run the installation script:
        ```bash
        ./install.sh
        ```
    > **Note:** This installs all Istio components and Ingress Gateways.
2.  **Check Ingress Gateway Services:**
    ```bash
    kubectl get svc -n istio-system
    ```
    * **Verify:** The response should include `istio-ingressgateway` (external), `istio-ingressgateway-internal` (internal), and `istiod`.

### 7.c. Storage Classes (longhorn)
1.  **Navigate to the installation directory and install Pre-requisites:**
    ```
      cd $K8_ROOT/longhorn
      ./pre_install.sh
       ```

2.  **Install Longhorn via helm:**
    ```
    ./install.sh
    ```
3.  **Cross-check the installation:**
    ```
    kubectl get all -n longhorn-system
    ```
## 8. Import MOSIP Cluster into Rancher UI



1.  **Login to Rancher:** Access the Rancher console using the admin credentials.
2.  **Start Import:** Click **Import Existing** for cluster addition.
3.  **Cluster Details:** Select **Generic** as the cluster type. Fill the **Cluster Name** field with a unique name (e.g., `mosip-sandbox`). Select **Create**.
4.  **Execute Import Command:** Rancher will provide a `kubectl apply` command.
    * **Ensure Kubeconfig is set to the MOSIP Cluster.**
    * Copy and execute the command from your PC:
        ```bash
        kubectl apply -f https://<rancher-host>/v3/import/<cluster-id>.yaml
        ```
5.  **Verification:** Wait for the cluster to be verified and show as **Active** in the Rancher UI.

---

## 9. MOSIP K8 Cluster Nginx Server Setup

### 9.a. SSL Certificates Creation

- You already have the certificate; you can use this one as well.

### 9.b. Nginx Server Setup for MOSIP K8s Cluster

1.  **Ssh into the observation nginx server**
    ```
     ssh ubuntu@<observation-nginx-node>
    ```

2.  **Install Nginx Software (on Nginx VM):**
    ```bash
    sudo apt update
    sudo apt install nginx -y
    ```
3. **clone the repo:**
    ```
    git clone https://github.com/mosip/k8s-infra -b v1.2.0.2
    ```
    - Nevigate inside the directory
    ```
    cd  ~/k8s-infra/mosip/on-prem/nginx
    sudo ./install.sh
    ```
   * **Provide Inputs:** The script will prompt for:
        * MOSIP Nginx server internal IP
        * MOSIP Nginx server public IP
        * Mosip Publicly accessible domains (comma separated)
        * SSL certificate path (`Full chain path`)
        * SSL key path (`Private key path`)
        * Cluster node IPs (comma separated)

4.  **Post Installation Check:**
    ```bash
    sudo systemctl status nginx
    ```
5. **For now we can skip monitoring part.**

# MOSIP External Dependencies setup

#### 1. PostgreSQL Installation

PostgreSQL is required as the primary relational database for the MOSIP services.

### 1.1. Installation

1.  **Navigate to the installation directory:**
    ```bash
    cd $INFRA_ROOT/deployment/v3/external/postgres
    ```
2. **Switch the branch to release-1.2.0.x:**
    ```bash
    git checkout release-1.2.0.x
    ```
3.  **Execute the installation script:**
    ```bash
    ./install.sh
    ```
4.  **Execute the installation script:**
    ```bash
    ./init_db.sh
    ```
#### 2. Keycloak

1.  **Navigate to the installation directory:**
    ```bash
    cd $INFRA_ROOT/deployment/v3/external/iam
    ```
2.  **Execute the installation script:**
    ```bash
    ./install.sh
    ```

3. **Initialize Keycloak:**
   ```bash
    cd $INFRA_ROOT/deployment/v3/external/iam
    ./keycloak_init.sh
    ```
- When we run this it will ask some prompt we have to do according to this

```
keycloak.realms.mosip.realm_config.smtpServer.auth: "false"
keycloak.realms.mosip.realm_config.smtpServer.host: "smtp.gmail.com"
keycloak.realms.mosip.realm_config.smtpServer.port: "465"
keycloak.realms.mosip.realm_config.smtpServer.from: "mosipqa@gmail.com"
keycloak.realms.mosip.realm_config.smtpServer.starttls: "false"
keycloak.realms.mosip.realm_config.smtpServer.ssl: "true"
```
4. **Switch the branch to v1.2.0.2:**
   ```bash
   git checkout v1.2.0.2
    ```

#### 3. Setup SoftHSM

1. **Navigate to the installation directory &Execute the installation script:**
    ```bash
    cd $INFRA_ROOT/deployment/v3/external/hsm/softhsm
    ./install.sh
    ```

- Setup Object store
   - Opt 1 for MinIO

   - Opt 2 for S3 (incase you are not going with MinIO installation and want s3 to be installed)

        Enter the prompte: Opt 1 for MinIO


#### 4. MinIO installation

1. **Switch the branch to release-1.2.0.x:**
    ```bash
    git checkout release-1.2.0.x
    ```
2. **Navigate to the installation directory &Execute the installation script:**

    ```bash
    cd $INFRA_ROOT/deployment/v3/external/object-store/minio
    ./install.sh
    ```
3. **Switch the branch to v1.2.0.2:**
   ```bash
   git checkout v1.2.0.2
    ```

#### 5. S3 Credentials setup

1. **Navigate to the installation directory &Execute the installation script:**
```
cd $INFRA_ROOT/deployment/v3/external/object-store/
./cred.sh
```
- When we run this it will ask some prompt we have to do according to this

```
Please enter the S3 region" REGION : (Leave blank or enter your specific region, if applicable)
S3 Host: eg. http://minio.minio:9000
```
#### 6. ClamAV setup

1. **Navigate to the installation directory &Execute the installation script:**

```
cd $INFRA_ROOT/deployment/v3/external/antivirus/clamav
./install.sh
```

#### 7. ActiveMQ setup
1. **Navigate to the installation directory &Execute the installation script:**
```
cd $INFRA_ROOT/deployment/v3/external/activemq
./install.sh
```

#### 8. Kafka setup
1. **Switch the branch to release-1.2.0.x:**
```bash
git checkout release-1.2.0.x
```

2. **Navigate to the installation directory &Execute the installation script:**
```
cd $INFRA_ROOT/deployment/v3/external/kafka
./install.sh
```
3. **Switch the branch to v1.2.0.2:**
```bash
git checkout v1.2.0.2
```

#### 9. MSG Gateway
1. **Navigate to the installation directory &Execute the installation script:**
```
cd $INFRA_ROOT/deployment/v3/external/msg-gateway
./install.sh
```

- **MOSIP provides mock smtp server which will be installed as part of default installation, opt for Y.**

#### 10. Captcha
1. **Configure reCAPTCHA keys for each domain:**
   - Open the following link to create reCAPTCHA keys:
     https://www.google.com/recaptcha/admin/create
    
   - Create captcha key for prereg domain (note: Replace the domain name of prereg in domain section)
    <img width="1207" height="557" alt="prereg-ggogle-captcha-key" src="https://github.com/user-attachments/assets/735a3678-8667-4027-bd71-6992e07e10ca" />
    
   - Click on submit button it will create site-key and secret key noted down in notepad we have to use while deploying captcha.
   <img width="1259" height="603" alt="captch-keys-for-prereg-domain" src="https://github.com/user-attachments/assets/5ce7d322-e7a9-40e3-bddc-f271c6cba4fc" />

   - Go to the setting and save it.
   - Create one more captch key for resident domain follow the same process that we did for prereg domain.

2. **Navigate to the installation directory &Execute the installation script :**

    - Note: While executing it will ask some promt what is the site key and secret key for prereg domain and resident domain
    ```
    cd $INFRA_ROOT/deployment/v3/external/captcha
    ./install.sh
    ```

#### 11. Landing page setup


1. **Navigate to the installation directory &Execute the installation script:**
```
cd $INFRA_ROOT/deployment/v3/external/landing-page
./install.sh
```

# MOSIP Modules Deployment

   #### 1. Conf secrets

**Navigate to the installation directory &Execute the installation script:**
```
cd $INFRA_ROOT/deployment/v3/mosip/conf-secrets
./install.sh
```

#### 2. Config Server
**Navigate to the installation directory &Execute the installation script:**
```
cd $INFRA_ROOT/deployment/v3/mosip/config-server
./install.sh
```

#### 3. Artifactory
**Navigate to the installation directory &Execute the installation script:**
```
cd $INFRA_ROOT/deployment/v3/mosip/artifactory
./install.sh
```

#### 4. Keymanager

1. **Switch the branch to release-1.2.0.x:**
    ```bash
   git checkout release-1.2.0.x
    ```
2. **Navigate to the installation directory & edit the installation script:**

    ```
    cd $INFRA_ROOT/deployment/v3/mosip/keymanager
    ```
    ```
    vi install.sh
    ```
3. **Add `--set metrics.enabled=false` because we did not deployed monitoring components:**
   <img width="1290" height="623" alt="edit-keycloak" src="https://github.com/user-attachments/assets/ba5c5a89-5163-4156-aa2b-30a8d8833b5b" />

4. **Switch the branch to v1.2.0.2:**
```bash
git checkout v1.2.0.2
```
#### 5. WebSub
**Navigate to the installation directory &Execute the installation script:**
```
cd $INFRA_ROOT/deployment/v3/mosip/websub
./install.sh
```
#### 6. Mock-SMTP
**Navigate to the installation directory &Execute the installation script:**
```
cd $INFRA_ROOT/deployment/v3/mosip/mock-smtp
./install.sh
```

#### 7. Kernel
**Navigate to the installation directory &Execute the installation script:**
- Add ( --set metrics.enabled=false ) in keymanager mosip/keymanager helm chart because we did not install monitoring components.


```
cd $INFRA_ROOT/deployment/v3/mosip/kernel
./install.sh
```

#### 8. Masterdata-loader 
**Navigate to the installation directory &Execute the installation script:**
```
 cd $INFRA_ROOT/deployment/v3/mosip/masterdata-loader
./install.sh
```

#### 9. Mock-biosdk
**Navigate to the installation directory &Execute the installation script:**
```
 cd $INFRA_ROOT/deployment/v3/mosip/biosdk
```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
  <img width="1255" height="588" alt="Mock-biosdk" src="https://github.com/user-attachments/assets/e3881b97-b582-4868-ba71-79dbaaff77e2" />

**Execute the installation script:**
```
./install.sh
```

#### 10. Packetmanager
**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/packetmanager
```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
<img width="1255" height="588" alt="packet-manager" src="https://github.com/user-attachments/assets/dbcf289f-a7dc-4610-8d98-e8082389cc93" />


**Execute the installation script:**
```
./install.sh
```

#### 11. Datashare
**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/datashare
```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
<img width="1255" height="588" alt="datashare" src="https://github.com/user-attachments/assets/4e9c25b1-73c8-438f-995e-20851c1b52df" />


**Execute the installation script:**
```
./install.sh
```
#### 12. Pre-reg

**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/prereg
```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
<img width="1255" height="588" alt="pre-reg" src="https://github.com/user-attachments/assets/ec3e0a32-e1b2-4aed-8619-70a4b9b42e29" />

**Execute the installation script:**
```
./install.sh
```
#### 13. idrepo

**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/idrepo
```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
<img width="1255" height="588" alt="idrepo" src="https://github.com/user-attachments/assets/c0157550-c1d2-4cf7-9f45-8eba2d53cd74" />

**Execute the installation script:**
```
./install.sh
```
#### 14. Partner Management Services

**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/pms
```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
<img width="1255" height="588" alt="pms" src="https://github.com/user-attachments/assets/467bdf8d-33a0-497e-b498-f69d980533c5" />

**Execute the installation script:**
```
./install.sh
```

#### 15. Mock ABIS

**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/mock-abis
```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
<img width="1255" height="588" alt="mock-abis" src="https://github.com/user-attachments/assets/98877b1c-a6dc-4930-a1ed-cfd0d79de156" />

**Execute the installation script:**
```
./install.sh
```

#### 16. Mock-mv

**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/Mock-mv

```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
<img width="1255" height="588" alt="mock-mv" src="https://github.com/user-attachments/assets/6cb09760-83b3-4a7b-ae06-5bdbdb404955" />

**Execute the installation script:**
```
./install.sh
```


#### 17. Registration Processor

**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/regproc

```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
<img width="1255" height="588" alt="regproc" src="https://github.com/user-attachments/assets/06c2a79b-e78a-4c78-978b-61e015bebf82" />

**Execute the installation script:**
```
./install.sh
```
#### 18. Admin

**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/admin

```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
<img width="1255" height="588" alt="admin" src="https://github.com/user-attachments/assets/060b44a2-c695-4185-a608-254cee53ae71" />

**Execute the installation script:**
```
./install.sh
```
#### 19. ID Authentication

**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/ida

```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
<img width="1255" height="588" alt="ida" src="https://github.com/user-attachments/assets/0bf8fbae-be18-495a-b1f5-f858d76ae947" />

**Execute the installation script:**
```
./install.sh
```
#### 20. Print

**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/print

```
**Add `--set metrics.enabled=false`**
```
vi install.sh
```
<img width="1255" height="588" alt="print" src="https://github.com/user-attachments/assets/92a3f8d0-22e0-411b-bf49-62a19ae8b061" />

**Execute the installation script:**
```
./install.sh
```

#### 21. Partner Onboarder
**Navigate to the installation directory &Execute the installation script:**

```
cd $INFRA_ROOT/deployment/v3/mosip/partner-onboarder
./install.sh
```
**It will some promt:**
```
- Do you have public domain & valid SSL? (Y/n) = Y
- Provide onboarder bucket name=  onboarder
- Provide onboarder s3 bucket region= Leave as empty
- Provide S3 URL= http://minio.minio:9000
```

#### 22. MOSIP File Server
**Navigate to the installation directory &Execute the installation script:**

```
cd $INFRA_ROOT/deployment/v3/mosip/mosip-file-server
./install.sh
```
#### 23. Resident

**Navigate to the installation directory**
```
cd $INFRA_ROOT/deployment/v3/mosip/resident

```
**Add `--set metrics.enabled=false`** and correct the jsonpath of the resident.
```
vi install.sh
```
<img width="1255" height="588" alt="resident" src="https://github.com/user-attachments/assets/1de1c481-184a-4433-a2c2-3f44325924f7" />
<img width="1255" height="588" alt="resident-api-correction" src="https://github.com/user-attachments/assets/3a885895-b4b0-490f-bc8e-3a84876ce5a5" />

**Execute the installation script:**
```
./install.sh
```

#### 24. Registration Client
**Navigate to the installation directory , install jq &Execute the installation script:**

```
cd $INFRA_ROOT/deployment/v3/mosip/regclient
sudo apt-get update
sudo apt-get install jq
./install.sh
```
