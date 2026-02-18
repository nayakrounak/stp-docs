---
description: >-
  Target architecture and key flows for STP's National ID using MOSIP, including
  SIGA and Immigration integrations.
---

# Arquitetura

Esta secção descreve a arquitetura-alvo para a implementação de Identificação Nacional em São Tomé e Príncipe (STP), utilizando o **MOSIP** como plataforma fundacional. Abrange os blocos lógicos, a topologia por ambientes, o zonamento de rede e os principais fluxos ponta‑a‑ponta de enrolamento, processamento, emissão e verificação — bem como as integrações com **SIGA** e **Imigração**.

***

### 1. Visão geral da arquitetura

A plataforma de Identificação Nacional baseada no MOSIP é disponibilizada como um conjunto de serviços seguros e escaláveis, executados numa infraestrutura baseada em Kubernetes. A arquitetura organiza-se em:

* **Perímetro de Enrolamento (Enrollment Edge)**: postos de registo/enrolamento e dispositivos biométricos compatíveis com MOSIP utilizados nos locais de enrolamento.
* **Cluster Central MOSIP**: serviços MOSIP responsáveis pela receção de pacotes, workflows de processamento, gestão do repositório de identidade e emissão.
* **Camada de Integração**: interfaces seguras que ligam o MOSIP aos sistemas **SIGA** e **Imigração** para validação de identidade, sincronização de dados e workflows operacionais.
* **Acesso e Identidade**: IAM baseado em Keycloak (e eSignet, se estiver no âmbito) para autenticação, autorização e acesso de parceiros.
* **Observabilidade e Operações**: monitorização, logging, auditoria, backups e mecanismos de DR que suportam as operações _day‑2_.

Esta arquitetura suporta um rollout faseado (piloto → escala) e permite a exposição controlada de portais e APIs voltados ao público.

***

### 2. Blocos lógicos

#### 2.1 Enrolamento e Registo

* **Registration Client** instalado nos postos de enrolamento para captura de dados demográficos, documentos e biometria
* **Dispositivos biométricos compatíveis com MOSIP** (impressão digital/face/íris, conforme o âmbito em STP) ligados ao posto de enrolamento
* **Criação de pacotes**: os dados e artefactos do enrolamento são empacotados de forma segura para envio ao backend

#### 2.2 Receção e Processamento de Pacotes

* **Serviços de receção/ingestão de pacotes** para aceitar pacotes provenientes dos locais de enrolamento
* **Pipeline do Registration Processor** para validar, extrair e processar dados do pacote através de workflows definidos
* **Workflows operacionais** (quando aplicável) para tratamento de exceções, verificações de qualidade e adjudicação

#### 2.3 Repositório de Identidade

Armazenamento seguro de:

* Dados demográficos de identidade
* Referências biométricas e metadados (conforme a política de STP e a configuração MOSIP)
* Artefactos documentais e evidências de enrolamento
* Trilhos de auditoria e eventos do sistema

Inclui APIs para serviços internos e sistemas autorizados (_relying systems_).

#### 2.4 Emissão

* Finalização da identidade e atribuição de identificadores (conforme a política do programa)
* Workflows de emissão de credenciais, incluindo:
  * Integração com impressão/cartão (quando aplicável)
  * Emissão de credenciais digitais (quando aplicável)
* Acompanhamento de estado e reporting

#### 2.5 Autenticação e Verificação (quando aplicável)

Consoante o âmbito:

* **Portal público de verificação** para validação de autenticidade de documentos/credenciais
* **eSignet** para autenticação e consentimento baseados em standards (OIDC) para entidades relying parties
* **Onboarding de parceiros e controlos de acesso** para consumo de serviços de identidade

#### 2.6 Observabilidade e Operações

* Logging centralizado, métricas, dashboards e alertas
* Logging de auditoria e controlos de evidência de integridade (_tamper-evidence_)
* Backup/restore e prontidão de DR

***

### 3. Topologia por ambientes

O MOSIP é disponibilizado em múltiplos ambientes para suportar uma gestão de releases segura:

| Ambiente | Finalidade           | Tipo de Dados          | Notas                                       |
| -------- | -------------------- | ---------------------- | ------------------------------------------- |
| DEV      | Desenvolvimento      | Sintéticos             | Apenas SI                                   |
| SIT      | Testes de Integração | Sintéticos/mascarados  | Inclui endpoints de teste de SIGA/Imigração |
| UAT      | Validação de negócio | Mascarados/controlados | Utilizadores UAT e testes de aceitação      |
| PRE‑PROD | Ensaio geral         | Subconjunto mascarado  | Topologia semelhante a Produção             |
| PROD     | Operação em produção | Reais                  | Auditada e sujeita a controlo de mudanças   |
| DR       | Failover             | Réplica real           | Acesso restrito                             |

**Política de promoção:** DEV → SIT → UAT → PRE‑PROD → PROD\
**Regra-chave:** mesmo método de deployment em todos os ambientes (IaC/Helm), com diferenças apenas de dimensionamento.

***

### 4. Zonas de rede e fronteiras de confiança

A implementação é segmentada em zonas para reduzir o impacto de falhas e aplicar privilégio mínimo.

#### 4.1 Zonas recomendadas

* **Zona Pública (DMZ):**
  * Apenas o que tem de ser exposto à Internet (por exemplo, portais/APIs públicos)
  * Protegida por WAF, TLS e limites de taxa
* **Zona de Aplicação:**
  * Serviços aplicacionais MOSIP, serviços de integração e APIs internas
* **Zona de Dados:**
  * Bases de dados, object storage, repositórios de backup, KMS/HSM
* **Zona de Administração:**
  * Bastion, consolas de operações, ferramentas de gestão do cluster
  * MFA obrigatório e acesso registado
* **Perímetro de Enrolamento:**
  * Locais de enrolamento e conectividade segura para endpoints de ingestão no backend

#### 4.2 Princípios de exposição

* Serviços administrativos privados são acessíveis apenas via **WireGuard** e/ou **allowlists de administração**
* Sem acesso direto da zona pública para a zona de dados
* Comunicação serviço‑a‑serviço com TLS quando necessário
* Todas as ações administrativas devem ser auditáveis

***

### 5. Arquitetura de Integração (SIGA + Imigração)

A plataforma de Identificação Nacional integra-se com sistemas governamentais externos através de uma **Camada de Integração** controlada. Esta camada padroniza:

* **Conectividade segura** (VPN/ligação privada, mTLS, allowlists de IP)
* **Autenticação e autorização** (contas de serviço, acesso baseado em tokens)
* **Validação de esquema e versionamento**
* **Retry e reconciliação** (idempotência, DLQ quando aplicável)
* **Auditabilidade** (todos os pedidos/alterações registados e rastreáveis)

#### 5.1 Padrões de integração

Consoante a capacidade do sistema externo, podem ser usados:

* **Chamadas síncronas por API (REST)** para validação/consulta em tempo real
* **Troca assíncrona (fila/eventos)** quando é necessária maior desacoplagem e resiliência
* **Batch ETL/SFTP** para sincronizações em lote agendadas (quando APIs não existem ou não são estáveis)

#### 5.2 Integração com SIGA (âmbito funcional)

Âmbito típico de integração com o SIGA:

* **Validação/cruzamentos de identidade** durante o enrolamento ou adjudicação (por exemplo, verificação de dados existentes no registo civil)
* **Sincronização de dados de referência** (por exemplo, localizações, unidades administrativas, registos — quando o SIGA é a fonte de verdade)
* **Atualizações de estado** de Identificação Nacional para o SIGA (por exemplo, “enrolado”, “emitido”, “atualizado”, “desativado”) conforme governança acordada
* **Workflows de exceção** quando são detetadas discrepâncias (revisão/adjudicação manual)

**Controlos recomendados**

* Definir campos autoritativos (MOSIP vs SIGA) e regras de resolução de conflitos
* Usar operações de atualização idempotentes (evitar atualizações duplicadas)
* Manter relatórios de reconciliação (diários/semanais) para acompanhar divergências e retries

#### 5.3 Integração com Imigração (âmbito funcional)

Âmbito típico de integração com Imigração:

* **Verificação de estrangeiros/autorização de residência** durante o enrolamento de não cidadãos
* **Sincronização de estado** (por exemplo, expiração de autorização, alterações de visto) para atualizações de ciclo de vida
* **Suporte a workflows de fronteira/entrada** (quando aplicável), onde a verificação de Identificação Nacional é necessária para operações de imigração

**Controlos recomendados**

* Controlos de acesso rigorosos (privilégio mínimo) e limitação de finalidade
* Minimização de PII (trocar apenas atributos necessários)
* Trilhos de auditoria fortes para todas as consultas e atualizações
* Regras de cache com limite temporal (se caching for permitido)

#### 5.4 Conectividade, segurança e exposição

Para integrações com SIGA e Imigração:

* Conectividade via **rede privada** sempre que possível (VPN/MPLS/peering privado)
* **mTLS** para comunicações serviço‑a‑serviço quando suportado
* **Allowlisting de IP** em ambos os lados
* **Conta de serviço dedicada** por integração e por ambiente
* Limites de taxa e timeouts para proteger o MOSIP e os sistemas externos

#### 5.5 Tratamento de erros e reconciliação

* Definir códigos de erro e regras de retry por integração
* Implementar chaves de idempotência para atualizações (quando aplicável)
* Capturar falhas numa fila de retry controlada ou mecanismo DLQ
* Produzir relatórios de reconciliação:
  * registos pendentes de sincronização
  * registos com falha (com motivo)
  * atualizações bem-sucedidas e timestamps

***

### 6. Fluxos de dados (ponta‑a‑ponta)

#### 6.1 Enrolamento → Submissão de Pacote

1. O operador captura dados demográficos, documentos e biometria no Registration Client
2. O Registration Client executa validações e verificações de qualidade localmente (conforme configuração)
3. Os dados do enrolamento são empacotados num pacote encriptado/assinado
4. O pacote é carregado para a ingestão backend (online ou _store‑and‑forward_, conforme o modelo do posto)

#### 6.2 Processamento de Pacote → Cruzamentos SIGA/Imigração (conforme configuração)

1. O Registration Processor valida a integridade do pacote e a conformidade com o esquema
2. Dados demográficos e artefactos são extraídos e colocados em staging
3. A Camada de Integração desencadeia as consultas/validações necessárias no SIGA e/ou Imigração
4. O workflow avalia respostas:
   * match/válido → prosseguir
   * discrepância → fluxo de exceção/adjudicação
   * indisponível → retry em fila ou fallback manual conforme SOP

#### 6.3 Finalização de Identidade → Emissão

1. Pacotes bem-sucedidos resultam na criação/finalização da identidade
2. Atribuição de identificador e criação/atualização do registo no Repositório de Identidade
3. Disparo do workflow de emissão:
   * impressão/cartão (quando aplicável)
   * credencial digital (quando aplicável)
4. Atualização do estado final e dos logs de auditoria

#### 6.4 Verificação/Autenticação (se estiver no âmbito)

* A verificação pública valida a autenticidade de credenciais/documentos impressos
* Relying parties usam APIs autenticadas (e eSignet, quando aplicável) para acesso baseado em consentimento
* Todos os eventos de verificação são registados e monitorizados

***

### 7. Inventário de componentes (alinhado com MOSIP)

* Registration Client (enrolamento em campo)
* Pipeline de ingestão e processamento de pacotes (Registration Processor)
* Serviços do Repositório de Identidade
* Serviços de configuração/administração e dados mestres
* Componentes da camada de integração (conectores SIGA + Imigração)
* Keycloak (IAM) e controlos de acesso administrativo
* eSignet (se ativado) para autenticação/consentimento OIDC
* Stack de observabilidade (logs, métricas, alertas)
* Serviços de dados: PostgreSQL, object storage, mensageria, pesquisa/armazenamento de logs (conforme adotado)

***

### 8. Metas não funcionais (arquitetura)

* **Segurança:** privilégio mínimo, encriptação em trânsito e em repouso, auditoria forte e acesso administrativo controlado
* **Disponibilidade:** HA para componentes críticos; capacidade de DR com RTO/RPO definidos
* **Escalabilidade:** escalabilidade horizontal para serviços stateless; planeamento de capacidade para serviços de dados
* **Desempenho:** latência p95 monitorizada para APIs-chave e metas de throughput de processamento
* **Operabilidade:** dashboards, runbooks, alertas e processos de manutenção
