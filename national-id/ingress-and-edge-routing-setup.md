---
title: Ingress & Edge Routing Setup
description: >-
  Configure ingress and edge routing (ingress-nginx + Istio gateways) and
  validate routing using httpbin.
---

# Ingress & Edge Routing Setup

This page explains how we expose **private admin tools** (Observation cluster) and **MOSIP platform endpoints** (MOSIP cluster) using a layered approach:

***

### 0. Prerequisites (from previous pages)

* Observation cluster is **Ready** (`kubectl get nodes`)
* MOSIP cluster is **Ready** (`kubectl get nodes`)
* DNS plan exists (private vs public FQDNs)
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

This step installs the `ingress-nginx` controller on the **Observation** cluster to expose internal tooling (e.g., Rancher UI) via private DNS and private access policy.

{% hint style="info" %}
**Why ingress-nginx here?**\
It’s the most widely used Kubernetes ingress controller, with strong community support and clear operational patterns for private dashboards.

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

#### 2.2 Validation checks

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
Gateways provide explicit control over what is exposed and allow consistent traffic management patterns (host-based routing, canary, header-based routing), with strong tooling for troubleshooting.

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

#### 3.2 Validation checks

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
On-prem environments often need a predictable “front door” that is independent of the cluster lifecycle. Nginx provides a proven reverse proxy pattern for TLS termination and routing.

**Reference:** https://docs.nginx.com/
{% endhint %}

> **Note:** Nginx configuration files and certificate management (ACME/PKI) should be documented in a dedicated “TLS & Certificates” page.

{% hint style="info" %}
**TLS management reference (optional):**\
If you manage certificates inside Kubernetes, `cert-manager` is the common standard.\
Reference: https://cert-manager.io/docs/
{% endhint %}

***

### 5. Connectivity validation using httpbin

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

### 6. Definition of Done (DoD)

Before proceeding to “MOSIP platform installation”:

* [ ] `ingress-nginx` is installed and healthy on the Observation cluster
* [ ] Istio is installed and healthy on the MOSIP cluster
* [ ] DNS records resolve correctly to the right LB IPs
* [ ] `httpbin` tests succeed for both public (if applicable) and internal routes
* [ ] Admin endpoints are reachable only via WireGuard / allowlist
