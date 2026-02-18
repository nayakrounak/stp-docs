---
description: >-
  Minimum prerequisites to prepare infrastructure, access, DNS, and tooling
  antes da implementação do MOSIP.
---

# Pré-requisitos da Plataforma

Esta secção lista os pré-requisitos mínimos necessários antes de iniciar a disponibilização (deployment) do MOSIP para São Tomé e Príncipe (STP). Inclui o **toolchain da bastion host**, **dimensionamento da base de VMs**, **planeamento de DNS** e **configuração de acesso seguro (WireGuard + higiene de SSH)**.

{% hint style="info" %}
**Antes de começar**

Preencha estes valores uma vez por ambiente e mantenha-os consistentes em todas as páginas:

* **Domínio base:** `<domain>` (exemplo: `nid.gov.st`)
* **Código do ambiente:** `<env>` (exemplo: `dev`, `sit`, `uat`, `prod`, `dr`)
* **IP privado do LB de Observação:** `<obs_lb_private_ip>`
* **IP privado do LB do MOSIP:** `<mosip_lb_private_ip>`
* **IP público do LB do MOSIP (se aplicável):** `<mosip_lb_public_ip>`
* **IP público do WireGuard:** `<wg_public_ip>`
* **Porta do WireGuard:** `51820/udp`
* **Allowlist de CIDR de administração:** `<admin_cidr_list>`
* **Caminho da chave SSH:** `<ssh_private_key_path>`
* **Registry de containers:** `<registry_url>`
* **Abordagem de HSM/KMS:** `<hsm_or_kms_or_tbd>`
{% endhint %}

***

### 1. Configuração da Bastion Host (Ferramentas e Repositórios)

Todas as atividades de deployment devem ser executadas a partir de uma **bastion host** segura.

{% hint style="info" %}
**Porquê uma bastion host?**\
Uma bastion host fornece um único ponto de entrada endurecido para acesso administrativo, ferramentas e auditoria, reduzindo a exposição direta dos nós do cluster e dos serviços internos.
{% endhint %}

#### 1.1 Instalar Git (v2.25.1 ou superior)

{% hint style="info" %}
**Porquê Git?**\
Usamos Git para controle de versões de scripts de infraestrutura, valores de Helm charts e configurações de deployment, permitindo deployments repetíveis, revisão por pares e rastreabilidade. ([git-scm.com](https://git-scm.com/))
{% endhint %}

```bash
sudo apt update
sudo apt install git
```

#### 1.2 Configurar variáveis de ambiente

{% hint style="info" %}
**Porquê variáveis de ambiente?**\
Os scripts de instalação do MOSIP e a automação de suporte assumem, em geral, caminhos base consistentes. Padronizar estes caminhos reduz erros humanos e simplifica runbooks e suporte.
{% endhint %}

```bash
export MOSIP_ROOT=/home/ubuntu
export K8_ROOT=$MOSIP_ROOT/k8s-infra
export INFRA_ROOT=$MOSIP_ROOT/mosip-infra
```

> Dica: Adicione os `export` ao `~/.bashrc` se quiser que persistam entre sessões.

#### 1.3 Instalar kubectl

{% hint style="info" %}
**Porquê kubectl?**\
`kubectl` é a ferramenta standard de linha de comandos para interagir com clusters Kubernetes (deployments, logs, troubleshooting). Usamo-la para validar o estado do cluster e operar workloads MOSIP. ([kubernetes.io](https://kubernetes.io/docs/tasks/tools/))
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
Quando o Istio é usado para capacidades de service mesh (gestão de tráfego, políticas de segurança, diagnósticos), o `istioctl` é a ferramenta recomendada para instalação, validação de configuração e troubleshooting. ([istio.io](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/))
{% endhint %}

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.15.0 sh -
cd istio-1.15.0
sudo mv bin/istioctl /usr/local/bin/
istioctl version --remote=false
```

#### 1.5 Instalar RKE (v1.3.10)

{% hint style="warning" %}
**Porquê RKE (e uma nota importante):**\
O RKE é uma ferramenta de provisionamento de clusters Kubernetes que simplifica a criação de clusters on‑premises. No entanto, o RKE1 chegou ao fim de vida (EOL); novos deployments devem considerar RKE2 (ou outra distribuição Kubernetes suportada), a menos que exista uma restrição de projeto que exija permanecer em RKE1 por motivos de compatibilidade. ([github.com](https://github.com/rancher/rke))
{% endhint %}

```bash
curl -L "https://github.com/rancher/rke/releases/download/v1.3.10/rke_linux-amd64" -o rke
chmod +x rke
sudo mv rke /usr/local/bin/
rke --version
```

#### 1.6 Instalar Helm (v3.0.0 ou superior) e adicionar repositórios Helm

{% hint style="info" %}
**Porquê Helm?**\
Os serviços MOSIP são distribuídos como Helm charts para padronizar instalação e upgrades em Kubernetes. O Helm ajuda a gerir aplicações complexas com deployments versionados e repetíveis. ([helm.sh](https://helm.sh/docs/))
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
Usamos Ansible para automatizar tarefas repetíveis de configuração de servidores (por exemplo, instalar Docker, desativar swap e aplicar hardening base aos nós). Isto reduz erros manuais em múltiplas VMs. ([docs.ansible.com](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html))
{% endhint %}

```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

#### 1.8 Clonar o repositório MOSIP k8s-infra (tag v1.2.0.2)

{% hint style="info" %}
**Porquê MOSIP k8s-infra?**\
O repositório `k8s-infra` disponibiliza arquitetura de referência e scripts para implementar clusters Kubernetes e infraestrutura de suporte para o MOSIP (e componentes DPG relacionados). ([github.com](https://github.com/mosip/k8s-infra))
{% endhint %}

```bash
git clone https://github.com/mosip/k8s-infra -b v1.2.0.2
```

***

### 2. Base do Sistema Operativo

Todas as VMs referidas neste guia assumem Ubuntu 22.04 como imagem standard do sistema operativo.

{% hint style="info" %}
**Porquê Ubuntu 22.04 LTS?**\
As versões LTS do Ubuntu oferecem manutenção e atualizações de segurança de longo prazo, o que é importante para infraestrutura governamental estável. ([discourse.ubuntu.com](https://discourse.ubuntu.com/t/jammy-jellyfish-release-notes/24668))
{% endhint %}

***

### 3. Requisitos de Hardware Base (Referência On‑Prem)

> Ajuste o dimensionamento com base na população esperada, concorrência de enrolamento, necessidades de HA e requisitos de DR.

{% hint style="info" %}
**Porquê separar clusters de Observação vs MOSIP?**\
Separar componentes de **observação/gestão** (por exemplo, gestão do cluster e ops IAM) do cluster principal de aplicação MOSIP reduz risco e evita que ferramentas de gestão impactem o desempenho do runtime MOSIP. As orientações on‑prem do MOSIP também refletem o conceito de _observation cluster_ para instalar ingress/storage e aplicações de suporte. ([docs.mosip.io](https://docs.mosip.io/1.2.0/setup/deploymentnew/v3-installation/on-prem-installation-guidelines))
{% endhint %}

<table data-header-hidden><thead><tr><th></th><th width="80.0859375"></th><th width="79.921875"></th><th width="86.94921875"></th><th width="87.140625"></th><th width="137.48046875"></th><th></th></tr></thead><tbody><tr><td><strong>Finalidade</strong></td><td><strong>vCPU</strong></td><td><strong>RAM</strong></td><td><strong>Armazenamento</strong></td><td><strong># VMs</strong></td><td><strong>HA</strong></td><td><strong>SO</strong></td></tr><tr><td>WireGuard Bastion Host</td><td>2</td><td>4 GB</td><td>8 GB</td><td>1</td><td>Ativo‑Passivo</td><td>Ubuntu 22.04</td></tr><tr><td>Nós do cluster de Observação</td><td>2</td><td>8 GB</td><td>32 GB</td><td>2</td><td>2</td><td>Ubuntu 22.04</td></tr><tr><td>Nginx / LB de Observação</td><td>2</td><td>4 GB</td><td>16 GB</td><td>1</td><td>Nginx+</td><td>Ubuntu 22.04</td></tr><tr><td>Nós do cluster MOSIP</td><td>12</td><td>32 GB</td><td>128 GB</td><td>6</td><td>6</td><td>Ubuntu 22.04</td></tr><tr><td>Nginx / LB do MOSIP</td><td>2</td><td>4 GB</td><td>16 GB</td><td>1</td><td>Nginx+</td><td>Ubuntu 22.04</td></tr></tbody></table>

***

### 4. Requisitos de DNS (Domínios e Mapeamentos)

Os deployments do MOSIP requerem registos DNS mapeados para:

* **Nginx/LB do cluster de Observação** (apenas privado)
* **Nginx/LB do cluster MOSIP** (privado e público, dependendo da política de exposição)

{% hint style="info" %}
**Porquê DNS e FQDNs estáveis?**\
Hostnames estáveis simplificam a gestão de certificados TLS, a configuração de clientes (por exemplo, registration clients) e os runbooks operacionais. Os deployments MOSIP baseados em Helm assumem URLs de endpoints consistentes entre ambientes. ([docs.mosip.io](https://docs.mosip.io/1.2.0/setup/deploymentnew/getting-started/helm-charts))
{% endhint %}

#### 4.1 Tabela de DNS

<table data-header-hidden><thead><tr><th width="195.19140625"></th><th width="102.19140625"></th><th width="200.1796875"></th><th width="123.08203125"></th><th></th></tr></thead><tbody><tr><td><strong>Padrão de FQDN</strong></td><td><strong>Exposição</strong></td><td><strong>Mapeia para</strong></td><td><strong>Finalidade</strong></td><td><strong>Política de acesso</strong></td></tr><tr><td>rancher.&#x3C;domain></td><td>Privado</td><td>&#x3C;obs_lb_private_ip></td><td>Dashboard do Rancher</td><td>Apenas via WireGuard / allowlist de CIDR de administração</td></tr><tr><td>keycloak.&#x3C;domain></td><td>Privado</td><td>&#x3C;obs_lb_private_ip></td><td>Admin do Keycloak (ops IAM)</td><td>Apenas via WireGuard / allowlist de CIDR de administração</td></tr><tr><td>sandbox.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Página interna (não‑prod)</td><td>Nunca expor em UAT/PROD; apenas WireGuard</td></tr><tr><td>api-internal.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>APIs internas</td><td>Apenas WireGuard</td></tr><tr><td>admin.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Portal de administração</td><td>Apenas WireGuard</td></tr><tr><td>iam.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Admin IAM / endpoints internos IAM</td><td>Apenas WireGuard</td></tr><tr><td>regclient.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Download do Registration Client</td><td>Apenas WireGuard</td></tr><tr><td>activemq.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>UI ActiveMQ (se ativado)</td><td>Apenas WireGuard</td></tr><tr><td>kibana.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>UI Kibana (opcional)</td><td>Apenas WireGuard</td></tr><tr><td>kafka.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>UI Kafka (opcional)</td><td>Apenas WireGuard</td></tr><tr><td>object-store.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Consola MinIO (opcional)</td><td>Apenas WireGuard</td></tr><tr><td>postgres.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Acesso à BD (tipicamente via port-forward)</td><td>Apenas WireGuard; nunca público</td></tr><tr><td>pmp.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Portal de Gestão de Parceiros (se ativado)</td><td>Apenas WireGuard</td></tr><tr><td>onboarder.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>Relatórios de onboarding de parceiros</td><td>Apenas WireGuard</td></tr><tr><td>smtp.&#x3C;env>.&#x3C;domain></td><td>Privado</td><td>&#x3C;mosip_lb_private_ip></td><td>UI SMTP mock (se ativado)</td><td>Apenas WireGuard</td></tr><tr><td>api.&#x3C;env>.&#x3C;domain></td><td>Público</td><td>&#x3C;mosip_lb_public_ip></td><td>APIs públicas (apenas o que tem de ser público)</td><td>WAF + allowlists quando possível</td></tr><tr><td>prereg.&#x3C;env>.&#x3C;domain></td><td>Público</td><td>&#x3C;mosip_lb_public_ip></td><td>Portal de pré‑registo (se estiver no âmbito)</td><td>WAF + rate limits</td></tr><tr><td>resident.&#x3C;env>.&#x3C;domain></td><td>Público</td><td>&#x3C;mosip_lb_public_ip></td><td>Portal do residente (se estiver no âmbito)</td><td>WAF + rate limits</td></tr><tr><td>idp.&#x3C;env>.&#x3C;domain></td><td>Público</td><td>&#x3C;mosip_lb_public_ip></td><td>IDP (se estiver no âmbito)</td><td>WAF + TLS rigoroso</td></tr></tbody></table>

#### 4.2 Política de exposição (recomendada)

{% hint style="info" %}
**Porquê restringir endpoints privados?**\
Endpoints administrativos e internos expõem operações poderosas e fluxos de dados sensíveis. Restringi-los a WireGuard/allowlists de administração reduz a superfície de ataque e suporta requisitos de auditoria e conformidade.
{% endhint %}

* Endpoints privados devem ser acessíveis apenas via WireGuard e/ou allowlist de CIDR de administração rigorosamente controlada.
* Endpoints públicos devem ser protegidos com WAF, TLS, rate limiting e (sempre que possível) allowlisting de IP para APIs de parceiros.

***

### 5. Base de Acesso Seguro (Higiene de SSH + WireGuard)

#### 5.1 Permissões da chave privada SSH

{% hint style="info" %}
**Porquê permissões SSH rigorosas?**\
Restringir permissões do ficheiro da chave privada é um controlo básico de segurança para prevenir divulgação acidental de credenciais de acesso privilegiado.
{% endhint %}

```bash
sudo chmod 400 ~/.ssh/privkey.pem
```

(Comando alternativo usado noutros passos)

```bash
chmod 400 ~/.ssh/<your private key>
```

#### 5.2 Pré‑requisitos do WireGuard

{% hint style="info" %}
**Porquê WireGuard?**\
O WireGuard é uma VPN moderna, desenhada para ser simples e de alto desempenho. Usamo-lo para fornecer acesso privado seguro a dashboards e APIs internas sem exposição pública. ([wireguard.com](https://www.wireguard.com/quickstart/))
{% endhint %}

* O servidor WireGuard escuta em UDP 51820
* Garanta que UDP 51820 está aberto na VM e em qualquer firewall externo

***

### 6. Configuração do Servidor Bastion WireGuard (On‑Prem)

#### 6.1 Aceder por SSH à VM do WireGuard

```bash
ssh -i <path to .pem> ubuntu@<Wireguard server public ip>
```

#### 6.2 Criar diretoria de configuração

```bash
mkdir -p wireguard/config
```

#### 6.3 Instalar Docker na VM do WireGuard (via Ansible)

{% hint style="info" %}
**Porquê automatizar a instalação do Docker?**\
Usar Ansible para a instalação base garante configuração consistente e apoia repetibilidade em reconstruções de DR.
{% endhint %}

> Execute o playbook de Docker para instalar Docker e adicionar o utilizador ao grupo `docker`.

```bash
ansible-playbook -i hosts.ini docker.yaml
```

#### 6.4 Instalar e iniciar o servidor WireGuard (Docker)

{% hint style="info" %}
**Porquê o container linuxserver/wireguard?**\
Fornece uma configuração WireGuard containerizada, amplamente utilizada e repetível, simplificando deployment e upgrades face a uma configuração manual do servidor. ([hub.docker.com](https://hub.docker.com/r/linuxserver/wireguard))
{% endhint %}

```bash
sudo docker run -d   --name=wireguard   --cap-add=NET_ADMIN   --cap-add=SYS_MODULE   -e PUID=1000   -e PGID=1000   -e TZ=Asia/Calcutta   -e PEERS=30   -p 51820:51820/udp   -v /home/ubuntu/wireguard/config:/config   -v /lib/modules:/lib/modules   --sysctl="net.ipv4.conf.all.src_valid_mark=1"   --restart unless-stopped   ghcr.io/linuxserver/wireguard
```

> Nota: aumente `-e PEERS=30` se forem necessárias mais de 30 configurações de cliente.

***

### 7. Cluster Kubernetes de Observação – Comandos iniciais

{% hint style="info" %}
**Porquê um cluster de Observação?**\
Este cluster aloja, tipicamente, serviços de gestão da plataforma e de suporte (por exemplo, Rancher, ops IAM), ajudando a isolar ferramentas operacionais do runtime do MOSIP. As orientações on‑prem do MOSIP referem a preparação de um _Rancher/observation cluster_ com ingress/storage para aplicações de suporte. ([docs.mosip.io](https://docs.mosip.io/1.2.0/setup/deploymentnew/v3-installation/on-prem-installation-guidelines))
{% endhint %}

#### 7.1 Configurar SSH sem password (opcional)

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

#### 7.2 Preparar e executar playbooks Ansible (nós de Observação)

Navegue para a diretoria do Ansible:

```bash
cd $K8_ROOT/rancher/on-prem
```

Copie e atualize o inventário de hosts:

```bash
cp hosts.ini.sample hosts.ini
```

Desative swap nos nós do cluster (ignore se já estiver desativado):

```bash
ansible-playbook -i hosts.ini swap.yaml
```

> Atenção: verifique o estado de swap antes de executar:

```bash
swapon --show
```

Instale Docker e adicione o utilizador ao grupo `docker`:

```bash
ansible-playbook -i hosts.ini docker.yaml
```

***

### 8. Criar configuração do cluster RKE

{% hint style="info" %}
**Porque aparecem Rancher / Keycloak na lista de DNS?**\
O Rancher é frequentemente usado para gerir clusters Kubernetes e fornecer uma UI central de operações. ([ranchermanager.docs.rancher.com](https://ranchermanager.docs.rancher.com/))\
O Keycloak fornece capacidades de IAM (login, administração e padrões de integração com RBAC) e é amplamente usado para gerir o acesso a serviços administrativos. ([keycloak.org](https://www.keycloak.org/docs/latest/server_admin/index.html))
{% endhint %}

Gere `cluster.yml` com RKE:

```bash
rke config
```

> O comando irá solicitar detalhes dos nós, incluindo:

* SSH Private Key Path: `<path/to/your/ssh/private/key>`

***

### 9. O que tem de estar pronto antes do deployment do “Dia 1”

{% hint style="info" %}
**Porquê estes gates?**\
A maioria das falhas de deployment do MOSIP ocorre por dependências base em falta (DNS/TLS, rede, prontidão do cluster e controlos de acesso). Completar estes pré-requisitos reduz retrabalho e estabiliza a promoção entre ambientes. ([docs.mosip.io](https://docs.mosip.io/1.2.0/readme/overview))
{% endhint %}

* A bastion host tem Git, kubectl, istioctl, RKE, Helm e Ansible instalados
* Variáveis de ambiente exportadas (MOSIP\_ROOT, K8\_ROOT, INFRA\_ROOT)
* Imagens Ubuntu 22.04 disponíveis para todas as VMs
* VMs provisionadas conforme dimensionamento base (ou dimensionamento atualizado aprovado)
* Registos DNS planeados para clusters de observação + MOSIP (público e privado)
* WireGuard UDP 51820 aberto (VM + firewall) e permissões das chaves SSH endurecidas
