---
description: >-
  Provision Observation and MOSIP Kubernetes clusters using RKE with a hardened
  baseline.
---

# Provisionamento do Cluster e Configuração de Base

Esta página cobre os **passos ponta-a-ponta de provisionamento** para colocar em funcionamento o **cluster Kubernetes de Observação** e o **cluster Kubernetes do MOSIP** para São Tomé e Príncipe (STP), utilizando **RKE**, com uma baseline endurecida (swap desativado, Docker instalado, higiene de SSH) e **gates de validação** claros antes de avançar para ingress / Istio / instalações via Helm do MOSIP.

***

### Antes de começar

{% hint style="info" %}
**Porque esta página é importante**

A maioria das falhas de deployment ocorre porque os nós Kubernetes não estão configurados de forma consistente (swap ativo, definições incorretas de Docker/kernel, problemas de acesso SSH). Esta página garante que ambos os clusters são construídos da mesma forma, sempre — o que é especialmente importante para **reconstruções em DR**.
{% endhint %}

Defina estes valores (uma vez por ambiente):

* **Código do ambiente:** `<env>` (dev/sit/uat/prod/dr)
* **Bastion host:** `<bastion_ip>`
* **Nós de Observação:** `<obs_node_ips>`
* **Nós do MOSIP:** `<mosip_node_ips>`
* **Utilizador SSH:** `<ssh_user>` (exemplo: `ubuntu`)
* **Caminho da chave SSH:** `<ssh_private_key_path>`

***

### 1. Preparar a Bastion Host (Máquina do Operador)

Deve executar o provisionamento do cluster **a partir da bastion host** (recomendado).

#### 1.1 Definir variáveis de ambiente

```bash
export MOSIP_ROOT=/home/ubuntu
export K8_ROOT=$MOSIP_ROOT/k8s-infra
export INFRA_ROOT=$MOSIP_ROOT/mosip-infra
```

{% hint style="info" %}
**Porquê variáveis de ambiente?**\
Padronizam caminhos entre páginas/runbooks para que os operadores não “entrem na pasta errada” e para que os scripts funcionem de forma consistente.
{% endhint %}

#### 1.2 Confirmar que as ferramentas obrigatórias estão instaladas

No mínimo:

* Git
* Ansible
* RKE
* kubectl
* Helm

{% hint style="info" %}
**Referências de confiança**

* kubectl: [https://kubernetes.io/docs/tasks/tools/](https://kubernetes.io/docs/tasks/tools/)
* RKE (RKE1): [https://rke.docs.rancher.com/](https://rke.docs.rancher.com/)
* Ansible: [https://docs.ansible.com/ansible/latest/installation\_guide/intro\_installation.html](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
* Helm: [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)
{% endhint %}

***

### 2. Higiene de SSH e Acesso sem Password (Opcional, mas recomendado)

#### 2.1 Gerar chave SSH (se necessário)

```bash
ssh-keygen -t rsa
```

#### 2.2 Copiar a chave pública para todos os nós

```bash
ssh-copy-id <remote-user>@<remote-ip>
```

#### 2.3 Verificar conectividade SSH

```bash
ssh -i ~/.ssh/<your private key> <remote-user>@<remote-ip>
```

#### 2.4 Garantir permissões da chave privada

```bash
chmod 400 ~/.ssh/<your private key>
```

{% hint style="info" %}
**Porquê SSH sem password?**\
O RKE e o Ansible exigem acesso consistente aos nós. SSH sem password reduz erros durante o provisionamento e facilita reconstruções automatizadas (por exemplo, via CI).
{% endhint %}

***

### 3. Preparar Nós com Ansible (Baseline)

Aplicamos primeiro a mesma baseline aos nós do cluster de Observação e, em seguida, repetimos o processo para os **nós do cluster MOSIP**.

#### 3.1 Ir para a diretoria do Ansible (Observação)

```bash
cd $K8_ROOT/rancher/on-prem
```

#### 3.2 Criar um ficheiro de inventário

```bash
cp hosts.ini.sample hosts.ini
```

Atualize o `hosts.ini` com os IPs/utilizadores dos seus nós.

#### 3.3 Verificar estado do swap (antes de desativar)

```bash
swapon --show
```

#### 3.4 Desativar swap em todos os nós do cluster

```bash
ansible-playbook -i hosts.ini swap.yaml
```

{% hint style="info" %}
**Porquê desativar swap?**\
O Kubernetes requer que o swap esteja desativado (ou explicitamente configurado), pois pode causar comportamento imprevisível de memória para pods e agendamento.

Referência: [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin)
{% endhint %}

#### 3.5 Instalar Docker + adicionar utilizador ao grupo Docker

```bash
ansible-playbook -i hosts.ini docker.yaml
```

{% hint style="info" %}
**Porquê uma baseline de Docker?**\
O RKE1 usa frequentemente Docker como runtime de containers. Uma configuração Docker consistente em todos os nós evita problemas de runtime durante o _bring-up_ do cluster.

Referência: [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)
{% endhint %}

***

### 4. Criar o Cluster Kubernetes de Observação (RKE)

#### 4.1 Gerar `cluster.yml` de forma interativa

```bash
rke config
```

Serão pedidos detalhes dos nós e o caminho da chave SSH.

#### 4.2 Editar `cluster.yml` (se necessário)

```bash
vi cluster.yml
```

Se quiser desativar a instalação de ingress por defeito (comum quando gere ingress separadamente):

```yaml
ingress:
  provider: none
```

#### 4.3 Criar o cluster

```bash
rke up
```

#### 4.4 Definir kubeconfig

Opção A (copiar para o padrão):

```bash
cp $HOME/.kube/<cluster_name>_config $HOME/.kube/config
```

Opção B (exportar explicitamente):

```bash
export KUBECONFIG="$HOME/.kube/<cluster_name>_config"
```

#### 4.5 Validar a saúde do cluster

```bash
kubectl get nodes
```

{% hint style="info" %}
**Porquê estes passos de validação?**\
Se os nós não estiverem `Ready` agora, tudo o que vem a seguir (ingress, Istio, instalações Helm do MOSIP) torna-se instável e difícil de diagnosticar.
{% endhint %}

#### 4.6 Guardar artefactos do RKE de forma segura

Mantenha estes ficheiros num local seguro (necessário para upgrades/substituição de nós/reconstruções):

* `cluster.yml`
* `cluster.rkestate`
* `kube_config_cluster.yml` (ou o kubeconfig gerado)

***

### 5. Criar o Cluster Kubernetes do MOSIP (RKE)

Repita exatamente o mesmo fluxo para o cluster MOSIP.

#### 5.1 Preparar inventário e baseline (nós MOSIP)

(Se mantiver inventários separados para o cluster MOSIP)

```bash
cd $K8_ROOT/rancher/on-prem
cp hosts.ini.sample hosts.ini
ansible-playbook -i hosts.ini swap.yaml
ansible-playbook -i hosts.ini docker.yaml
```

#### 5.2 Gerar e criar o cluster MOSIP

```bash
rke config
vi cluster.yml
rke up
```

#### 5.3 Definir kubeconfig e validar

```bash
cp $HOME/.kube/<cluster_name>_config $HOME/.kube/config
kubectl get nodes
```

***

### 6. Definição de Pronto (DoD) — Provisionamento do Cluster

Antes de avançar para **Ingress & Istio / Instalação da Plataforma**, confirme:

* [ ] Consegue fazer SSH da bastion para cada nó usando a chave pretendida
* [ ] Swap está desativado em todos os nós (`swapon --show` não devolve nada)
* [ ] Docker está instalado e o utilizador pretendido consegue executar Docker (sem problemas de sudo)
* [ ] `rke up` terminou com sucesso (sem hosts com falha)
* [ ] `kubectl get nodes` mostra todos os nós em estado `Ready`
* [ ] Artefactos do RKE guardados de forma segura (`cluster.yml`, `cluster.rkestate`, kubeconfig)
