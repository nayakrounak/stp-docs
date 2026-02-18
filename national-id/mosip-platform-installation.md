---
description: >-
  Instalar a plataforma MOSIP no cluster Kubernetes do MOSIP utilizando Helm e
  validate core endpoints.
---

# MOSIP Plataforma Instalação

Esta página documenta os passos para instalar a **plataforma MOSIP** no **cluster Kubernetes do MOSIP**, utilizando a abordagem padrão de **implementação baseada em Helm** usada neste runbook de STP.

It assumes you have already completed:

1. **Plataforma Pre-requisites**
2. **Implementação Topologia & Rede Plano**
3. **Cluster Provisioning & Baseline Configuração**
4. **Ingress & Edge Routing Configuração** (Istio installed on MOSIP cluster, edge routing validated)

***

### 0. References (trusted)

{% hint style="info" %}
**Primary references (official):**

* MOSIP Helm Charts guiar: https://docs.mosip.io/1.2.0/setup/deploymentnew/getting-started/helm-charts
* MOSIP k8s-infra repository: https://github.com/mosip/k8s-infra
* Helm documentação: https://helm.sh/docs/
* Kubernetes documentação: https://kubernetes.io/docs/home/
{% endhint %}

***

### 1. Instalação approach

We deploy MOSIP utilizando:

* **Helm charts** para repeatable installs/upgrades
* A **shared configuration layer** (ConfigMaps / Secrets) para ambiente-specific values
* **Namespaces** para separate plataforma components e simplify operações

{% hint style="info" %}
**Porquê Helm?**\
MOSIP components are packaged as Helm charts para standardize instalação e upgrades across ambientes. Helm also supports clean rollbacks e versionado releases.\
Reference: https://helm.sh/docs/
{% endhint %}

{% hint style="info" %}
**Porquê namespaces?**\
Namespaces allow logical separation of workloads e simplify RBAC, rede policies, e operacional workflows.\
Reference: https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
{% endhint %}

***

### 2. Pre-checks (MOSIP cluster)

#### 2.1 Confirm kubecontext points para MOSIP cluster

```bash
kubectl config current-context
kubectl get nodes
```

#### 2.2 Confirm Istio is healthy (prerequisite para MOSIP ingress routing)

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

{% hint style="info" %}
**Porquê this gate?**\
MOSIP endpoint exposure typically depends on Istio ingress gateways (internal/external). If Istio is unhealthy, MOSIP endpoints won’t route correctly.\
Reference: https://istio.io/latest/docs/tasks/traffic-management/ingress/
{% endhint %}

***

### 3. Prepare Helm repositories

Ensure the MOSIP Helm repo is added e up para date:

```bash
helm repo add mosip https://mosip.github.io/mosip-helm
helm repo update
helm search repo mosip | head
```

{% hint style="info" %}
**Porquê usar the MOSIP Helm repository?**\
It contains the official MOSIP charts e versionado dependencies needed para a consistent plataforma instalação.
{% endhint %}

***

### 4. Environment configuration (ConfigMaps & Secrets)

MOSIP deployments rely on ambiente-specific configuration such as domain names, database endpoints, object store endpoints, IAM endpoints, e keys.

{% hint style="info" %}
**Porquê ConfigMaps?**\
ConfigMaps store non-sensitive configuration (URLs, feature flags, ambiente identifiers) e keep Helm values para a minimum.\
Reference: https://kubernetes.io/docs/concepts/configuration/configmap/
{% endhint %}

{% hint style="info" %}
**Porquê Secrets?**\
Secrets store sensitive configuration (passwords, tokens, privado keys). They deverá be rotated e protected com RBAC e, if supported, encryption at rest.\
Reference: https://kubernetes.io/docs/concepts/configuration/secret/
{% endhint %}

> **Nota STP:** Mantenha uma separação rigorosa entre endpoints internos (por exemplo, `api-internal.<env>.<domain>`) e endpoints públicos (por exemplo, `api.<env>.<domain>`), de acordo com a política de DNS.

***

### 5. Instalar order (recomendado)

A typical safe instalar order is:

1. **Foundational dependencies** (if not already installed): storage class, ingress/gateways (already done), logging/monitorização (optional)
2. **MOSIP base services** (plataforma services e dependencies)
3. **Portals** (Admin, Resident, Pre-registration, Partner portals if in scope)
4. **Registration client distribution endpoint** (if used)
5. **Post-instalar jobs/migrations**
6. **Smoke tests & validação**

{% hint style="info" %}
**Porquê instalar in phases?**\
It isolates failures early e makes upgrades safer by limiting changes para a subset of services at a time.
{% endhint %}

***

### 6. Instalação steps (runbook-aligned)

> The runbook uses scripts e Helm values maintained under `$K8_ROOT`. Keep your ambiente’s repository layout consistent.

#### 6.1 Navigate para MOSIP infra repo (if applicable)

```bash
cd $K8_ROOT
```

#### 6.2 Apply ambiente config (ConfigMaps / Secrets)

Utilize the YAMLs in your repo para ConfigMaps/Secrets.

```bash
kubectl apply -f <path-to-configmaps>
kubectl apply -f <path-to-secrets>
```

#### 6.3 Instalar MOSIP Helm releases

Depending on your repository structure, you will either run an instalar script ou usar Helm para instalar/upgrade per chart.

Typical pattern:

```bash
helm upgrade --install <release-name> mosip/<chart-name> -n <namespace> -f <values.yaml>
```

#### 6.4 Verify implementação gradual

```bash
kubectl get pods -n <namespace>
kubectl rollout status deployment/<deployment-name> -n <namespace>
kubectl get svc -n <namespace>
```

***

### 7. Post-instalar validação

#### 7.1 Plataforma health

```bash
kubectl get pods -A | egrep -i "mosip|crash|error|pending" || true
```

#### 7.2 Logs para troubleshooting

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
**Porquê verify endpoints early?**\
Many instalação issues appear only when routing/TLS headers are applied at the edge. Early checks reduce rework later.
{% endhint %}

***

### 8. Definition of Done (DoD)

Antes moving para the **Integrations** secção:

* [ ] MOSIP Helm releases installed successfully
* [ ] All MOSIP namespaces are healthy (pods Running / Completed)
* [ ] Required portals/services respond via destina-se FQDNs
* [ ] Core authentication flows (admin login / IAM endpoints) behave as expected
* [ ] ConfigMaps/Secrets are stored in the repo (com secret material handled securely)