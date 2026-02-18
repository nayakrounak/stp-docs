---
description: >-
  Configure ingress and edge routing (ingress-nginx + Istio gateways) and
  validate routing using httpbin.
---

# Configuração de Ingress e Encaminhamento na Periferia

Esta página explains how we expose **privado admin ferramentas** (Observation cluster) and **MOSIP platform endpoints** (MOSIP cluster) using a layered approach:

***

### 0. Pré-requisitos (from previous pages)

* Observation cluster is **Ready** (`kubectl get nodes`)
* MOSIP cluster is **Ready** (`kubectl get nodes`)
* DNS plan exists (privado vs público FQDNs)
* WireGuard access is working for admins

***

### 1. Why do we need an ingress layer

{% hint style="info" %}
**Why ingress at all?**\
Ingress provides a consistent way to route HTTP(S) traffic into Kubernetes services using hostnames and paths, and makes it easier to standardize TLS and access policy.

**References:**

* Kubernetes Ingress concept: https://kubernetes.io/docs/concepts/services-networking/ingress/
{% endhint %}

***

### 2. Observation Cluster Ingress (ingress-nginx)

This step installs the `ingress-nginx` controller on the **Observation** cluster to expose internal tooling (e.g., Rancher UI) via privado DNS and privado access policy.

{% hint style="info" %}
**Why ingress-nginx here?**\
It’s the most widely used Kubernetes ingress controller, with strong community support and clear operational patterns for privado dashboards.

**References:**

* ingress-nginx (official): https://github.com/kubernetes/ingress-nginx
* Helm chart docs: https://kubernetes.github.io/ingress-nginx/
{% endhint %}

#### 2.1 Install ingress-nginx (Observation cluster)

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

Expected:

* Controller pods are `Running`
* Service has the expected type (ClusterIP/NodePort/LoadBalancer) based on your values file

***

### 3. MOSIP Cluster Ingress (Istio gateways)

MOSIP routing is controlled through **Istio ingress gateways** (internal and external), and then routed to services using Istio resources (Gateway/VirtualService), while exposure policy is enforced at the MOSIP LB.

{% hint style="info" %}
**Why Istio gateways on the MOSIP cluster?**\
Gateways provide explicit control over what is exposed and allow consistent traffic management patterns (host-based routing, podeary, header-based routing), with strong tooling for troubleshooting.

**References:**

* Istio install: https://istio.io/latest/docs/setup/install/
* Istio ingress/gateway docs: https://istio.io/latest/docs/tasks/traffic-management/ingress/
{% endhint %}

#### 3.1 Install Istio on the MOSIP cluster

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

Expected:

* `istiod` is Running
* Ingress gateway services exist (internal and/or external, depending on your install)

***

### 4. Edge LBs (VM-based Nginx) and TLS strategy

We use VM-based Nginx as a stable, auditable edge for:

* TLS termination
* IP allowlists/exposure policy
* Mapping DNS (FQDNs) to internal gateway services

{% hint style="info" %}
**Why VM-based Nginx at the edge?**\
On-prem ambientes often need a predictable “front door” that is independent of the cluster lifecycle. Nginx provides a proven reverse proxy pattern for TLS termination and routing.

**Reference:** https://docs.nginx.com/
{% endhint %}

> **Note:** Nginx configuration files and certificate management (ACME/PKI) deverá be documented in a dedicated “TLS & Certificates” page.

{% hint style="info" %}
**TLS management reference (optional):**\
If you manage certificates inside Kubernetes, `cert-manager` is the common standard.\
Reference: https://cert-manager.io/docs/
{% endhint %}

***

### 5. Validação de conectividade using httpbin

We deploy `httpbin` to confirm that ingress routing works **before** validating MOSIP modules.

{% hint style="info" %}
**Why httpbin?**\
It’s a lightweight echo service used to validate DNS → LB → ingress routing, TLS behavior, headers, and path rules without waiting for full application readiness.
{% endhint %}

#### 5.1 Install httpbin (utility)

```bash
cd $K8_ROOT/utils/httpbin
./install.sh
```

#### 5.2 Test routing (replace with your domains)

```bash
curl https://api.<env>.<domain>/httpbin/get?show_env=true
curl https://api-internal.<env>.<domain>/httpbin/get?show_env=true
```

***

### 6. Definição de Conclusão (DoD)

Before proceeding to “MOSIP platform installation”:

* [ ] `ingress-nginx` is installed and healthy on the Observation cluster
* [ ] Istio is installed and healthy on the MOSIP cluster
* [ ] DNS records resolve correctly to the right LB IPs
* [ ] `httpbin` tests succeed for both público (if applicable) and internal routes
* [ ] Admin endpoints are reachable only via WireGuard / allowlist