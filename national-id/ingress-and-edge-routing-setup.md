---
description: >-
  Configurar ingress e encaminhamento na periferia (ingress-nginx + gateways Istio) e
  validate routing using httpbin.
---

# Ingress & Edge Routing Configuração

This página explains how we expose **privado admin ferramentas** (Observation cluster) e **MOSIP plataforma endpoints** (MOSIP cluster) utilizando a layered approach:

***

### 0. Pré-requisitos (from anterior pages)

* Observation cluster is **Ready** (`kubectl get nodes`)
* MOSIP cluster is **Ready** (`kubectl get nodes`)
* DNS plano exists (privado vs público FQDNs)
* WireGuard acesso is working para admins

***

### 1. Porquê do we need an ingress layer

{% hint style="info" %}
**Porquê ingress at all?**\
Ingress disponibiliza a consistent way para route HTTP(S) tráfego into Kubernetes services utilizando hostnames e paths, e makes it easier para standardize TLS e acesso policy.

**References:**

* Kubernetes Ingress concept: https://kubernetes.io/docs/concepts/services-networking/ingress/
{% endhint %}

***

### 2. Observation Cluster Ingress (ingress-nginx)

Este passo instala o controlador `ingress-nginx` no cluster de **Observação** para expor ferramentas internas (por exemplo, Rancher UI) através de DNS privado e política de acesso privada.

{% hint style="info" %}
**Porquê ingress-nginx here?**\
It’s the most widely used Kubernetes ingress controller, com strong community support e clear operacional patterns para privado dashboards.

**References:**

* ingress-nginx (official): https://github.com/kubernetes/ingress-nginx
* Helm chart docs: https://kubernetes.github.io/ingress-nginx/
{% endhint %}

#### 2.1 Instalar ingress-nginx (Observation cluster)

```bash
cd $K8_ROOT/rancher/on-prem
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install   ingress-nginx ingress-nginx/ingress-nginx   --namespace ingress-nginx   --version 4.0.18   --create-namespace   -f ingress-nginx.values.yaml
kubectl get all -n ingress-nginx
```

#### 2.2 Validation checks

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Expected:

* Controller pods are `Running`
* O serviço tem o tipo esperado (ClusterIP/NodePort/LoadBalancer) com base no seu ficheiro de values

***

### 3. MOSIP Cluster Ingress (Istio gateways)

MOSIP routing is controlled through **Istio ingress gateways** (internal e external), e then routed para services utilizando Istio resources (Gateway/VirtualService), while exposure policy is enforced at the MOSIP LB.

{% hint style="info" %}
**Porquê Istio gateways on the MOSIP cluster?**\
Gateways provide explicit control over what is exposed e allow consistent tráfego management patterns (host-based routing, canary, header-based routing), com strong tooling para troubleshooting.

**References:**

* Istio instalar: https://istio.io/latest/docs/setup/install/
* Istio ingress/gateway docs: https://istio.io/latest/docs/tasks/traffic-management/ingress/
{% endhint %}

#### 3.1 Instalar Istio on the MOSIP cluster

```bash
cd $K8_ROOT/mosip/on-prem/istio
./install.sh
kubectl get svc -n istio-system
```

#### 3.2 Validation checks

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

Expected:

* `istiod` is Running
* Ingress gateway services exist (internal e/ou external, depending on your instalar)

***

### 4. Edge LBs (VM-based Nginx) e TLS strategy

We usar VM-based Nginx as a stable, auditable edge para:

* TLS termination
* IP allowlists/exposure policy
* Mapping DNS (FQDNs) para internal gateway services

{% hint style="info" %}
**Porquê VM-based Nginx at the edge?**\
On-prem ambientes often need a predictable “front door” that is independent of the cluster lifecycle. Nginx disponibiliza a proven reverse proxy pattern para TLS termination e routing.

**Reference:** https://docs.nginx.com/
{% endhint %}

> **Note:** Nginx configuration files e certificate management (ACME/PKI) deverá be documented in a dedicated “TLS & Certificates” página.

{% hint style="info" %}
**TLS management reference (optional):**\
If you manage certificates inside Kubernetes, `cert-manager` is the common standard.\
Reference: https://cert-manager.io/docs/
{% endhint %}

***

### 5. Connectivity validação utilizando httpbin

We deploy `httpbin` para confirm that ingress routing works **before** validating MOSIP modules.

{% hint style="info" %}
**Porquê httpbin?**\
It’s a lightweight echo service used para validate DNS → LB → ingress routing, TLS behavior, headers, e path rules without waiting para full application prontidão.
{% endhint %}

#### 5.1 Instalar httpbin (utility)

```bash
cd $K8_ROOT/utils/httpbin
./install.sh
```

#### 5.2 Test routing (replace com your domains)

```bash
curl https://api.<env>.<domain>/httpbin/get?show_env=true
curl https://api-internal.<env>.<domain>/httpbin/get?show_env=true
```

***

### 6. Definition of Done (DoD)

Antes proceeding para “MOSIP plataforma instalação”:

* [ ] `ingress-nginx` is installed e healthy on the Observation cluster
* [ ] Istio is installed e healthy on the MOSIP cluster
* [ ] DNS records resolve correctly para the right LB IPs
* [ ] `httpbin` tests succeed para both público (if applicable) e internal routes
* [ ] Admin endpoints are reachable only via WireGuard / allowlist