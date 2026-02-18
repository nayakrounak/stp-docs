---
description: >-
  Configurar ingress e encaminhamento na periferia (ingress-nginx + gateways
  Istio) e validate routing using httpbin.
---

# Configuração de Ingress e Encaminhamento na Periferia

Esta página explica como expomos **ferramentas administrativas privadas** (cluster de Observação) e **endpoints da plataforma MOSIP** (cluster MOSIP) usando uma abordagem em camadas.

***

### 0. Pré-requisitos (das páginas anteriores)

* O cluster de Observação está **Ready** (`kubectl get nodes`)
* O cluster MOSIP está **Ready** (`kubectl get nodes`)
* Existe um plano de DNS (FQDNs privados vs públicos)
* O acesso via WireGuard está a funcionar para administradores

***

### 1. Porque precisamos de uma camada de ingress

{% hint style="info" %}
**Porquê ingress?**\
O ingress fornece uma forma consistente de encaminhar tráfego HTTP(S) para serviços em Kubernetes usando hostnames e paths, e facilita a padronização de TLS e políticas de acesso.

**Referências:**

* Conceito de Ingress no Kubernetes: [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)
{% endhint %}

***

### 2. Ingress no Cluster de Observação (ingress-nginx)

Este passo instala o controlador `ingress-nginx` no cluster de **Observação** para expor ferramentas internas (por exemplo, Rancher UI) via DNS privado e política de acesso privada.

{% hint style="info" %}
**Porquê ingress-nginx aqui?**\
É o controlador de ingress mais utilizado em Kubernetes, com forte suporte da comunidade e padrões operacionais claros para dashboards privados.

**Referências:**

* ingress-nginx (oficial): [https://github.com/kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)
* Documentação do Helm chart: [https://kubernetes.github.io/ingress-nginx/](https://kubernetes.github.io/ingress-nginx/)
{% endhint %}

#### 2.1 Instalar ingress-nginx (cluster de Observação)

```bash
cd $K8_ROOT/rancher/on-prem
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install   ingress-nginx ingress-nginx/ingress-nginx   --namespace ingress-nginx   --version 4.0.18   --create-namespace   -f ingress-nginx.values.yaml
kubectl get all -n ingress-nginx
```

#### 2.2 Verificações de validação

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Esperado:

* Pods do controlador em estado `Running`
* O Service tem o tipo esperado (ClusterIP/NodePort/LoadBalancer) de acordo com o seu ficheiro de valores

***

### 3. Ingress no Cluster MOSIP (Gateways Istio)

O routing do MOSIP é controlado através de **Istio ingress gateways** (interno e externo) e depois encaminhado para serviços usando recursos do Istio (Gateway/VirtualService), enquanto a política de exposição é aplicada no LB do MOSIP.

{% hint style="info" %}
**Porquê gateways Istio no cluster MOSIP?**\
Os gateways fornecem controlo explícito sobre o que é exposto e permitem padrões consistentes de gestão de tráfego (routing por hostname, canary, routing por headers), com ferramentas fortes para troubleshooting.

**Referências:**

* Instalação do Istio: [https://istio.io/latest/docs/setup/install/](https://istio.io/latest/docs/setup/install/)
* Ingress/Gateway no Istio: [https://istio.io/latest/docs/tasks/traffic-management/ingress/](https://istio.io/latest/docs/tasks/traffic-management/ingress/)
{% endhint %}

#### 3.1 Instalar Istio no cluster MOSIP

```bash
cd $K8_ROOT/mosip/on-prem/istio
./install.sh
kubectl get svc -n istio-system
```

#### 3.2 Verificações de validação

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

Esperado:

* `istiod` em estado `Running`
* Serviços de ingress gateway existentes (interno e/ou externo, conforme a instalação)

***

### 4. LBs de Edge (Nginx em VM) e estratégia de TLS

Usamos Nginx em VM como _edge_ estável e auditável para:

* Terminação de TLS
* Allowlists de IP / política de exposição
* Mapear DNS (FQDNs) para serviços internos de gateway

{% hint style="info" %}
**Porquê Nginx em VM no edge?**\
Em ambientes on‑prem, muitas vezes é necessário um “front door” previsível e independente do ciclo de vida do cluster. O Nginx fornece um padrão comprovado de reverse proxy para terminação TLS e routing.

**Referência:** [https://docs.nginx.com/](https://docs.nginx.com/)
{% endhint %}

> **Nota:** Ficheiros de configuração do Nginx e gestão de certificados (ACME/PKI) devem ser documentados numa página dedicada de “TLS e Certificados”.

{% hint style="info" %}
**Referência opcional para gestão de TLS:**\
Se gerir certificados dentro do Kubernetes, o `cert-manager` é o standard mais comum.\
Referência: [https://cert-manager.io/docs/](https://cert-manager.io/docs/)
{% endhint %}

***

### 5. Validação de conectividade com httpbin

Implementamos `httpbin` para confirmar que o routing de ingress funciona **antes** de validar módulos MOSIP.

{% hint style="info" %}
**Porquê httpbin?**\
É um serviço leve de “echo” usado para validar DNS → LB → ingress routing, comportamento TLS, headers e regras de path sem depender da prontidão completa da aplicação.
{% endhint %}

#### 5.1 Instalar httpbin (utilitário)

```bash
cd $K8_ROOT/utils/httpbin
./install.sh
```

#### 5.2 Testar routing (substitua pelos seus domínios)

```bash
curl https://api.<env>.<domain>/httpbin/get?show_env=true
curl https://api-internal.<env>.<domain>/httpbin/get?show_env=true
```

***

### 6. Definição de Pronto (DoD)

Antes de avançar para “instalação da plataforma MOSIP”:

* [ ] `ingress-nginx` está instalado e saudável no cluster de Observação
* [ ] Istio está instalado e saudável no cluster MOSIP
* [ ] Registos DNS resolvem corretamente para os IPs de LB certos
* [ ] Testes `httpbin` têm sucesso para rotas públicas (se aplicável) e internas
* [ ] Endpoints administrativos são acessíveis apenas via WireGuard / allowlist
