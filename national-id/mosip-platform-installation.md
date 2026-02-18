---
title: MOSIP Platform Installation
description: >-
  Install the MOSIP platform on the MOSIP Kubernetes cluster using Helm and
  validate core endpoints.
---

# MOSIP Platform Installation

This page documents the steps to install the **MOSIP platform** on the **MOSIP Kubernetes cluster** using the standard **Helm-based deployment** approach used in this STP runbook.

It assumes you have already completed:

1. **Platform Pre-requisites**
2. **Deployment Topology & Network Plan**
3. **Cluster Provisioning & Baseline Setup**
4. **Ingress & Edge Routing Setup** (Istio installed on MOSIP cluster, edge routing validated)

***

### 0. References (trusted)

{% hint style="info" %}
**Primary references (official):**

* MOSIP Helm Charts guide: https://docs.mosip.io/1.2.0/setup/deploymentnew/getting-started/helm-charts
* MOSIP k8s-infra repository: https://github.com/mosip/k8s-infra
* Helm documentation: https://helm.sh/docs/
* Kubernetes documentation: https://kubernetes.io/docs/home/
{% endhint %}

***

### 1. Installation approach

We deploy MOSIP using:

* **Helm charts** for repeatable installs/upgrades
* A **shared configuration layer** (ConfigMaps / Secrets) for environment-specific values
* **Namespaces** to separate platform components and simplify operations

{% hint style="info" %}
**Why Helm?**\
MOSIP components are packaged as Helm charts to standardize installation and upgrades across environments. Helm also supports clean rollbacks and versioned releases.\
Reference: https://helm.sh/docs/
{% endhint %}

{% hint style="info" %}
**Why namespaces?**\
Namespaces allow logical separation of workloads and simplify RBAC, network policies, and operational workflows.\
Reference: https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
{% endhint %}

***

### 2. Pre-checks (MOSIP cluster)

#### 2.1 Confirm kubecontext points to MOSIP cluster

```bash
kubectl config current-context
kubectl get nodes
```

#### 2.2 Confirm Istio is healthy (prerequisite for MOSIP ingress routing)

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

{% hint style="info" %}
**Why this gate?**\
MOSIP endpoint exposure typically depends on Istio ingress gateways (internal/external). If Istio is unhealthy, MOSIP endpoints won’t route correctly.\
Reference: https://istio.io/latest/docs/tasks/traffic-management/ingress/
{% endhint %}

***

### 3. Prepare Helm repositories

Ensure the MOSIP Helm repo is added and up to date:

```bash
helm repo add mosip https://mosip.github.io/mosip-helm
helm repo update
helm search repo mosip | head
```

{% hint style="info" %}
**Why use the MOSIP Helm repository?**\
It contains the official MOSIP charts and versioned dependencies needed for a consistent platform installation.
{% endhint %}

***

### 4. Environment configuration (ConfigMaps & Secrets)

MOSIP deployments rely on environment-specific configuration such as domain names, database endpoints, object store endpoints, IAM endpoints, and keys.

{% hint style="info" %}
**Why ConfigMaps?**\
ConfigMaps store non-sensitive configuration (URLs, feature flags, environment identifiers) and keep Helm values to a minimum.\
Reference: https://kubernetes.io/docs/concepts/configuration/configmap/
{% endhint %}

{% hint style="info" %}
**Why Secrets?**\
Secrets store sensitive configuration (passwords, tokens, private keys). They should be rotated and protected with RBAC and, if supported, encryption at rest.\
Reference: https://kubernetes.io/docs/concepts/configuration/secret/
{% endhint %}

> **STP note:** Keep a strict split between internal endpoints (e.g., `api-internal.<env>.<domain>`) and public endpoints (e.g., `api.<env>.<domain>`) as per the DNS policy.

***

### 5. Install order (recommended)

A typical safe install order is:

1. **Foundational dependencies** (if not already installed): storage class, ingress/gateways (already done), logging/monitoring (optional)
2. **MOSIP core services** (platform services and dependencies)
3. **Portals** (Admin, Resident, Pre-registration, Partner portals if in scope)
4. **Registration client distribution endpoint** (if used)
5. **Post-install jobs/migrations**
6. **Smoke tests & validation**

{% hint style="info" %}
**Why install in phases?**\
It isolates failures early and makes upgrades safer by limiting changes to a subset of services at a time.
{% endhint %}

***

### 6. Installation steps (runbook-aligned)

> The runbook uses scripts and Helm values maintained under `$K8_ROOT`. Keep your environment’s repository layout consistent.

#### 6.1 Navigate to MOSIP infra repo (if applicable)

```bash
cd $K8_ROOT
```

#### 6.2 Apply environment config (ConfigMaps / Secrets)

Use the YAMLs in your repo for ConfigMaps/Secrets.

```bash
kubectl apply -f <path-to-configmaps>
kubectl apply -f <path-to-secrets>
```

#### 6.3 Install MOSIP Helm releases

Depending on your repository structure, you will either run an install script or use Helm to install/upgrade per chart.

Typical pattern:

```bash
helm upgrade --install <release-name> mosip/<chart-name> -n <namespace> -f <values.yaml>
```

#### 6.4 Verify rollout

```bash
kubectl get pods -n <namespace>
kubectl rollout status deployment/<deployment-name> -n <namespace>
kubectl get svc -n <namespace>
```

***

### 7. Post-install validation

#### 7.1 Platform health

```bash
kubectl get pods -A | egrep -i "mosip|crash|error|pending" || true
```

#### 7.2 Logs for troubleshooting

```bash
kubectl logs -n <namespace> deploy/<deployment-name> --tail=200
kubectl describe pod -n <namespace> <pod-name>
```

#### 7.3 Endpoint verification (from bastion)

```bash
curl -k https://api.<env>.<domain>/
curl -k https://api-internal.<env>.<domain>/
```

{% hint style="info" %}
**Why verify endpoints early?**\
Many installation issues appear only when routing/TLS headers are applied at the edge. Early checks reduce rework later.
{% endhint %}

***

### 8. Definition of Done (DoD)

Before moving to the **Integrations** section:

* [ ] MOSIP Helm releases installed successfully
* [ ] All MOSIP namespaces are healthy (pods Running / Completed)
* [ ] Required portals/services respond via intended FQDNs
* [ ] Core authentication flows (admin login / IAM endpoints) behave as expected
* [ ] ConfigMaps/Secrets are stored in the repo (with secret material handled securely)
