---
description: >-
  Instalar a plataforma MOSIP no cluster Kubernetes do MOSIP utilizando Helm e
  validate core endpoints.
---

# Instalação da Plataforma MOSIP

Esta página documenta os passos para instalar a **plataforma MOSIP** no **cluster Kubernetes do MOSIP** usando a abordagem padrão de **deployment com Helm** utilizada neste _runbook_ de STP.

Assume que já concluiu:

1. **Pré‑requisitos da Plataforma**
2. **Topologia de Deployment e Plano de Rede**
3. **Provisionamento do Cluster e Configuração Base**
4. **Configuração de Ingress e Edge Routing** (Istio instalado no cluster MOSIP e edge routing validado)

***

### 0. Referências (de confiança)

{% hint style="info" %}
**Referências principais (oficiais):**

* Guia de Helm Charts do MOSIP: [https://docs.mosip.io/1.2.0/setup/deploymentnew/getting-started/helm-charts](https://docs.mosip.io/1.2.0/setup/deploymentnew/getting-started/helm-charts)
* Repositório MOSIP k8s-infra: [https://github.com/mosip/k8s-infra](https://github.com/mosip/k8s-infra)
* Documentação do Helm: [https://helm.sh/docs/](https://helm.sh/docs/)
* Documentação do Kubernetes: [https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home/)
{% endhint %}

***

### 1. Abordagem de instalação

Fazemos o deployment do MOSIP usando:

* **Helm charts** para instalações/atualizações repetíveis
* Uma **camada partilhada de configuração** (ConfigMaps / Secrets) para valores específicos por ambiente
* **Namespaces** para separar componentes da plataforma e simplificar operações

{% hint style="info" %}
**Porquê Helm?**\
Os componentes MOSIP são empacotados como Helm charts para padronizar instalação e upgrades entre ambientes. O Helm também suporta _rollbacks_ limpos e releases versionadas.\
Referência: [https://helm.sh/docs/](https://helm.sh/docs/)
{% endhint %}

{% hint style="info" %}
**Porquê namespaces?**\
Os namespaces permitem separação lógica de workloads e simplificam RBAC, políticas de rede e fluxos operacionais.\
Referência: [https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
{% endhint %}

***

### 2. Pré‑checks (cluster MOSIP)

#### 2.1 Confirmar que o kubecontext aponta para o cluster MOSIP

```bash
kubectl config current-context
kubectl get nodes
```

#### 2.2 Confirmar que o Istio está saudável (pré-requisito para routing de ingress do MOSIP)

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

{% hint style="info" %}
**Porquê este gate?**\
A exposição de endpoints do MOSIP depende tipicamente dos ingress gateways do Istio (interno/externo). Se o Istio estiver degradado, os endpoints do MOSIP não vão encaminhar corretamente.\
Referência: [https://istio.io/latest/docs/tasks/traffic-management/ingress/](https://istio.io/latest/docs/tasks/traffic-management/ingress/)
{% endhint %}

***

### 3. Preparar repositórios Helm

Garanta que o repositório Helm do MOSIP está adicionado e atualizado:

```bash
helm repo add mosip https://mosip.github.io/mosip-helm
helm repo update
helm search repo mosip | head
```

{% hint style="info" %}
**Porquê usar o repositório Helm do MOSIP?**\
Contém os charts oficiais do MOSIP e dependências versionadas necessárias para uma instalação consistente da plataforma.
{% endhint %}

***

### 4. Configuração por ambiente (ConfigMaps e Secrets)

Deployments MOSIP dependem de configuração específica por ambiente, como nomes de domínio, endpoints de base de dados, endpoints de object store, endpoints de IAM e chaves.

{% hint style="info" %}
**Porquê ConfigMaps?**\
ConfigMaps armazenam configuração não sensível (URLs, _feature flags_, identificadores de ambiente) e ajudam a manter os valores Helm no mínimo.\
Referência: [https://kubernetes.io/docs/concepts/configuration/configmap/](https://kubernetes.io/docs/concepts/configuration/configmap/)
{% endhint %}

{% hint style="info" %}
**Porquê Secrets?**\
Secrets armazenam configuração sensível (passwords, tokens, chaves privadas). Devem ser rotacionados e protegidos com RBAC e, quando suportado, encriptação em repouso.\
Referência: [https://kubernetes.io/docs/concepts/configuration/secret/](https://kubernetes.io/docs/concepts/configuration/secret/)
{% endhint %}

> **Nota STP:** Mantenha uma separação rigorosa entre endpoints internos (por exemplo, `api-internal.<env>.<domain>`) e endpoints públicos (por exemplo, `api.<env>.<domain>`) conforme a política de DNS.

***

### 5. Ordem de instalação (recomendada)

Uma ordem segura típica é:

1. **Dependências fundacionais** (se ainda não instaladas): _storage class_, ingress/gateways (já feito), logging/monitorização (opcional)
2. **Serviços core do MOSIP** (serviços da plataforma e dependências)
3. **Portais** (Admin, Resident, Pré‑registo, Portais de Parceiros — se estiverem no âmbito)
4. **Endpoint de distribuição do Registration Client** (se usado)
5. **Jobs/migrações pós‑instalação**
6. **Smoke tests e validação**

{% hint style="info" %}
**Porquê instalar por fases?**\
Isola falhas mais cedo e torna upgrades mais seguros, limitando alterações a um subconjunto de serviços de cada vez.
{% endhint %}

***

### 6. Passos de instalação (alinhados com o runbook)

> Este runbook usa scripts e valores Helm mantidos em `$K8_ROOT`. Mantenha o layout do repositório consistente por ambiente.

#### 6.1 Navegar para o repositório de infra do MOSIP (se aplicável)

```bash
cd $K8_ROOT
```

#### 6.2 Aplicar configuração do ambiente (ConfigMaps / Secrets)

Use os YAMLs no seu repositório para ConfigMaps/Secrets:

```bash
kubectl apply -f <path-to-configmaps>
kubectl apply -f <path-to-secrets>
```

#### 6.3 Instalar releases Helm do MOSIP

Dependendo da estrutura do seu repositório, irá executar um script de instalação ou usar o Helm para instalar/atualizar por chart.

Padrão típico:

```bash
helm upgrade --install <release-name> mosip/<chart-name> -n <namespace> -f <values.yaml>
```

#### 6.4 Verificar o rollout

```bash
kubectl get pods -n <namespace>
kubectl rollout status deployment/<deployment-name> -n <namespace>
kubectl get svc -n <namespace>
```

***

### 7. Validação pós‑instalação

#### 7.1 Saúde da plataforma

```bash
kubectl get pods -A | egrep -i "mosip|crash|error|pending" || true
```

#### 7.2 Logs para troubleshooting

```bash
kubectl logs -n <namespace> deploy/<deployment-name> --tail=200
kubectl describe pod -n <namespace> <pod-name>
```

#### 7.3 Verificação de endpoints (a partir da bastion)

```bash
curl -k https://api.<env>.<domain>/
curl -k https://api-internal.<env>.<domain>/
```

{% hint style="info" %}
**Porquê verificar endpoints cedo?**\
Muitos problemas de instalação só aparecem quando o routing/TLS e headers são aplicados no edge. Verificações precoces reduzem retrabalho mais tarde.
{% endhint %}

***

### 8. Definição de Pronto (DoD)

Antes de avançar para a secção de **Integrações**:

* [ ] Releases Helm do MOSIP instaladas com sucesso
* [ ] Todos os namespaces do MOSIP estão saudáveis (pods em `Running` / `Completed`)
* [ ] Portais/serviços necessários respondem através dos FQDNs pretendidos
* [ ] Fluxos core de autenticação (login admin / endpoints IAM) comportam-se como esperado
* [ ] ConfigMaps/Secrets estão versionados no repositório (com material sensível tratado de forma segura)
