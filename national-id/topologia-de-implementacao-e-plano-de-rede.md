# Topologia de Implementação e Plano de Rede

Esta página descreve **onde os componentes são implementados, como asseguramos o acesso administrativo e como o DNS e o Ingress** encaminham o tráfego entre os clusters Kubernetes de **Observação** e **MOSIP** para São Tomé e Príncipe (STP).

***

#### 1. Porquê esta topologia

Utilizamos um **modelo de dois clusters**:

* **Cluster de Observação**: aloja ferramentas operacionais (gestão do cluster, dashboards operacionais, IAM operacional).
* **Cluster MOSIP**: aloja as cargas de trabalho da plataforma MOSIP e os endpoints expostos pela MOSIP.

{% hint style="info" %}
**Porquê dois clusters?**\
Manter os componentes de “operações” separados dos componentes “de missão” reduz o impacto de falhas (blast radius), permite políticas de rede mais restritivas para as ferramentas de administração e evita que atualizações/alterações de UIs operacionais afetem a execução da MOSIP.
{% endhint %}

***

#### 2. Arquitetura lógica

**2.1 Componentes principais**

1. **Bastion WireGuard (VPN de Administração)**
   * Fornece acesso privado e seguro a partir das máquinas de administração para as sub-redes privadas.
   * Referência: WireGuard Quick Start — [https://www.wireguard.com/quickstart/](https://www.wireguard.com/quickstart/)

{% hint style="info" %}
**Porquê WireGuard?**\
O WireGuard é uma VPN moderna, concebida para ser mais simples e mais fácil de auditar do que muitas alternativas, tornando-a adequada para um acesso administrativo rigorosamente controlado em ambientes privados.
{% endhint %}

2.  **Cluster de Observação (Kubernetes)**

    * **Rancher** (UI de gestão do cluster)
    * **IAM Operacional** (por exemplo, Keycloak) para autenticação de administradores (se aplicável)
    * **Ingress Privado** (ingress-nginx) para dashboards privados

    Referências:

    * Documentação RKE1 (Rancher) — [https://rke.docs.rancher.com/installation](https://rke.docs.rancher.com/installation)
    * ingress-nginx (GitHub oficial) — [https://github.com/kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)

{% hint style="info" %}
**Porquê Rancher/ferramentas de observação?**\
Um plano centralizado de gestão do cluster reduz o esforço operacional (acessos, monitorização, ciclo de vida dos nós, gestão de kubeconfig) e melhora a auditabilidade. Mantê-lo privado reduz a exposição.
{% endhint %}

3.  **Cluster MOSIP (Kubernetes)**

    * **Gateways Istio** para controlo de ingress para serviços MOSIP
    * **LB Nginx / reverse proxy da MOSIP** (baseado em VM) para encaminhamento na periferia e terminação TLS estáveis
    * **Armazenamento persistente** via NFS (VM servidor + provisionador cliente) para workloads com estado, quando necessário

    Referências:

    * Istio: Install with istioctl — [https://istio.io/latest/docs/setup/install/istioctl/](https://istio.io/latest/docs/setup/install/istioctl/)
    * Documentação NGINX — [https://docs.nginx.com/](https://docs.nginx.com/)
    * NFS subdir external provisioner (kubernetes-sigs) — [https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)

{% hint style="info" %}
**Porquê gateways Istio?**\
Os gateways fornecem controlo explícito e orientado por políticas sobre o ingress (encaminhamento e isolamento entre endpoints internos e externos) e uma separação clara entre exposição (Gateways) e regras de encaminhamento (VirtualServices).
{% endhint %}

{% hint style="info" %}
**Porquê um LB Nginx baseado em VM à frente?**\
Em instalações on‑prem ou híbridas, um reverse proxy baseado em VM pode ser a forma mais simples e estável de gerir DNS, certificados TLS, allowlists e a política de exposição num único ponto, independentemente da dinâmica do cluster.
{% endhint %}

{% hint style="info" %}
**Porquê NFS para PVs?**\
O NFS é um backend de armazenamento partilhado pragmático para Kubernetes on‑prem quando é necessário armazenamento do tipo ReadWriteMany sem adotar uma plataforma de armazenamento distribuído completa.
{% endhint %}

***

#### 3. Zonas de rede e política de acesso

**3.1 Zonas recomendadas (conceituais)**

* **Zona de Administração**: portáteis de admin / runners de CI
* **Zona Bastion**: VM WireGuard (IP público) + acesso de salto (jump) de administração
* **Zona de Cluster Privado**: nós dos clusters de Observação e MOSIP, LBs privados, VM(s) de NFS
* **Zona Pública**: apenas endpoints MOSIP selecionados que têm de estar acessíveis a partir da Internet

**3.2 Regras base de acesso**

* **Endpoints privados** (Rancher, IAM operacional, `admin.*`, `api-internal.*`, dashboards internos) são acessíveis **apenas via WireGuard** e/ou através de uma **allowlist CIDR de administração** rigorosa.
* **Endpoints públicos** (por exemplo, `api.<env>.<domain>`, `resident.<env>.<domain>` se estiverem no âmbito) devem ser protegidos com:
  * TLS (cifras fortes)
  * WAF / limites de taxa (quando disponível)
  * allowlisting de IP para APIs de parceiros, quando viável

{% hint style="info" %}
**Porquê “VPN primeiro” para superfícies de administração?**\
As UIs de administração expõem operações privilegiadas e percursos de dados sensíveis. Mantê-las fora da Internet pública reduz significativamente a superfície de ataque e simplifica os controlos de conformidade.
{% endhint %}

***

#### 4. Fluxo de tráfego

**4.1 Acesso administrativo (privado)**

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

**4.2 Acesso público à MOSIP (apenas se necessário)**

```mermaid
flowchart LR
  U[Internet Users / Partners] -->|HTTPS 443| P[MOSIP Nginx / LB (Public)]
  P --> Q[Istio IngressGateway - external]
  Q --> R[MOSIP Public Services]
```

***

#### 5. Modelo de provisionamento do cluster (RKE)

Provisionamos ambos os clusters usando **RKE**.

Referência: Documentação RKE1 (Rancher) — [https://rke.docs.rancher.com/installation](https://rke.docs.rancher.com/installation)

{% hint style="info" %}
**Porquê RKE?**\
O RKE fornece uma abordagem repetível para disponibilizar Kubernetes em VMs/bare-metal utilizando Docker como runtime. É frequentemente usado em contextos on‑prem onde não existe um ciclo de vida gerido por kubeadm ou um Kubernetes gerido.
{% endhint %}

**5.1 Bootstrap do cluster de Observação (comandos)**

```bash
rke config
vi cluster.yml
rke up
```

Definir kubeconfig:

```bash
cp $HOME/.kube/<cluster_name>_config $HOME/.kube/config
# OR
export KUBECONFIG="$HOME/.kube/<cluster_name>_config"
```

Validar:

```bash
kubectl get nodes
```

Ficheiros operacionais a reter de forma segura:

* `cluster.yml`
* `kube_config_cluster.yml` (ou o kubeconfig gerado)
* `cluster.rkestate`

***

#### 6. Ingress e encaminhamento na periferia

**6.1 Ingress do cluster de Observação (ingress-nginx)**

Referências:

* ingress-nginx (GitHub oficial) — [https://github.com/kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)
* Exemplos de ingress: [https://kubernetes.github.io/ingress-nginx/examples/](https://kubernetes.github.io/ingress-nginx/examples/)

{% hint style="info" %}
**Porquê ingress-nginx para Observação?**\
Fornece um controlador ingress padrão para dashboards privados, permitindo encaminhamento consistente e políticas TLS uniformes para as ferramentas operacionais.
{% endhint %}

```bash
cd $K8_ROOT/rancher/on-prem
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install   ingress-nginx ingress-nginx/ingress-nginx   --namespace ingress-nginx   --version 4.0.18   --create-namespace   -f ingress-nginx.values.yaml
kubectl get all -n ingress-nginx
```

**6.2 Ingress do cluster MOSIP (Istio)**

Referência: Istio: Install with istioctl — [https://istio.io/latest/docs/setup/install/istioctl/](https://istio.io/latest/docs/setup/install/istioctl/)

{% hint style="info" %}
**Porquê Istio no cluster MOSIP?**\
Centraliza o ingress e a aplicação de políticas service-to-service, oferecendo aos operadores mecanismos fortes para troubleshooting e gestão de tráfego.
{% endhint %}

```bash
cd $K8_ROOT/mosip/on-prem/istio
./install.sh
kubectl get svc -n istio-system
```

***

#### 7. Armazenamento persistente (NFS)

Referências:

* NFS subdir external provisioner (kubernetes-sigs) — [https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
* Repositório Helm: [https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/](https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/)

{% hint style="info" %}
**Porquê NFS + provisionador externo?**\
Este padrão permite o provisionamento dinâmico de Persistent Volumes no Kubernetes utilizando um servidor NFS existente, reduzindo a sobrecarga de criação manual de PVs.
{% endhint %}

**7.1 Servidor NFS (na VM NFS)**

```bash
cd $K8_ROOT/mosip/nfs
cp hosts.ini.sample hosts.ini
ssh -i ~/.ssh/nfs-ssh.pem ubuntu@<internal ip of nfs server>
git clone https://github.com/mosip/k8s-infra -b v1.2.0.1
cd /home/ubuntu/k8s-infra/mosip/nfs/
sudo ./install-nfs-server.sh
```

Quando solicitado:

* Nome do ambiente: `<envName>`
* Caminho NFS: `/srv/nfs/mosip/<envName>`

**7.2 Provisionador cliente NFS (a partir do bastion)**

```bash
cd $K8_ROOT/mosip/nfs/
./install-nfs-client-provisioner.sh
```

Verificar:

```bash
kubectl -n nfs get deployment.apps/nfs-client-provisioner
kubectl get storageclass
```

***

#### 8. DNS e modelo de exposição

Em termos de topologia:

* **DNS de Observação** → aponta para o **IP privado do Nginx/LB de Observação**
* **DNS Privado da MOSIP** → aponta para o **IP privado do Nginx/LB da MOSIP**
* **DNS Público da MOSIP (apenas se necessário)** → aponta para o **IP público do Nginx/LB da MOSIP**

{% hint style="info" %}
**Porquê uma separação rigorosa entre FQDNs privados e públicos?**\
Torna a política de exposição aplicável e mais fácil de auditar, reduzindo a publicação acidental de dashboards internos e APIs.
{% endhint %}

***

#### 9. Validação de conectividade (periferia → cluster)

Implementamos o utilitário `httpbin` para validar o encaminhamento de ingress e os cabeçalhos de ambiente.

{% hint style="info" %}
**Porquê httpbin?**\
É um serviço simples de “eco” que ajuda a validar encaminhamento, terminação TLS, cabeçalhos e comportamento de query/path sem depender da prontidão das aplicações MOSIP.
{% endhint %}

```bash
cd $K8_ROOT/utils/httpbin
./install.sh
```

Testar (substituir pelos seus FQDNs):

```bash
curl https://api.<env>.<domain>/httpbin/get?show_env=true
curl https://api-internal.<env>.<domain>/httpbin/get?show_env=true
```
