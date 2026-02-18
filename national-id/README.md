---
description: >-
  Documentação de implementação e operação do Programa de Identificação Nacional
  de STP using MOSIP.
---

# Identificação Nacional

Este GitBook disponibiliza a documentação de implementação e de disponibilização (deployment) do programa de Identificação Nacional em São Tomé e Príncipe (STP), utilizando o **MOSIP** como plataforma base (fundacional) de Identificação Nacional. Destina-se a orientar stakeholders do Governo, integradores de sistemas e equipas de operações ao longo de toda a configuração ponta‑a‑ponta — desde a preparação do ambiente e a disponibilização do núcleo da plataforma até ao rollout do registo/enrolamento, à integração com ABIS, à entrada em produção (go‑live) e às operações contínuas.

#### Finalidade desta documentação

Esta documentação visa:

* Fornecer uma abordagem clara e repetível para disponibilizar o MOSIP nos ambientes **DEV / SIT / UAT / PRE‑PROD / PROD / DR**
* Estabelecer normas comuns para **segurança, gestão de configuração, monitorização, backup e recuperação de desastre**
* Definir como os **postos de enrolamento** são preparados e colocados em funcionamento, incluindo prontidão dos dispositivos e procedimentos de suporte em campo
* Documentar pontos de integração (por exemplo, ABIS, CRVS/registo civil, gateways de mensageria e outros sistemas governamentais, conforme aplicável)
* Suportar um **go‑live e handover** controlados, com gates de prontidão, passos de cutover, planos de rollback e período de hypercare

#### Visão geral da solução (alto nível)

A solução de Identificação Nacional é centrada no MOSIP e inclui:

* **Enrolamento e Registo**: enrolamento conduzido por operador através de clientes de registo e dispositivos compatíveis com MOSIP
* **Processamento de Pacotes e Workflow**: processamento backend, validação e adjudicação dos pacotes de enrolamento
* **Desduplicação**: desduplicação através de integração com o Sistema de Registo Civil em São Tomé e Príncipe
* **Repositório de Identidade**: armazenamento seguro de registos de identidade e artefactos de suporte sob controlos de acesso rigorosos
* **Emissão de Credenciais**: emissão de credenciais de identificação nacional (físicas e/ou digitais, dependendo do âmbito)
* **Autenticação e Verificação (quando aplicável)**: serviços de autenticação e verificação baseados em standards para entidades relying parties aprovadas, incluindo módulos públicos de verificação quando necessário

#### Princípios de disponibilização

A implementação segue estes princípios:

* **Segurança por conceção**: acesso com privilégio mínimo, encriptação em trânsito e em repouso, trilhos de auditoria robustos e acesso administrativo controlado
* **Disponibilizações repetíveis**: a infraestrutura e a configuração da plataforma são versionadas e reproduzíveis entre ambientes
* **Prontidão operacional**: monitorização, alertas, runbooks, backups e planos de DR fazem parte da base — não são um complemento posterior
* **Rollout escalável**: o enrolamento é introduzido através de pilotos e expansão faseada, com prontidão do posto e critérios de aceitação

#### Público-alvo

Este GitBook destina-se a:

* Liderança do programa e equipas de governação técnica do Governo
* Administradores de TI e equipas de infraestrutura da Identificação Nacional
* Equipas de deployment e engenharia do integrador de sistemas
* Equipas de segurança, auditoria e conformidade
* Equipas de operações e suporte (NOC / helpdesk / L2 / L3)

#### Como utilizar este GitBook

* Comece por **Arquitetura** para compreender a topologia de implementação e os principais building blocks do MOSIP
* Utilize **pré‑requisitos da Plataforma** antes de provisionar os ambientes
* Siga **Implementação MOSIP (Core)** e **Integrações** para orientação de instalação passo‑a‑passo
* Utilize **Enrolamento e Rollout em Campo** para prontidão do posto, configuração de dispositivos e procedimentos operacionais
* Consulte **Operações, Backup e DR** para atividades de day‑2 e continuidade de negócio
* Utilize checklists de **Go‑Live e Handover** para executar um cutover controlado e transitar para operações em regime permanente

#### No âmbito (desta documentação)

* Preparação do ambiente e disponibilização do MOSIP (todos os ambientes)
* Abordagem de gestão de configuração e secrets
* Integração com sistemas externos, disponibilização e validação
* Playbook de rollout de enrolamento e checklists de prontidão
* Runbooks de monitorização, logging, backup/restore e DR
* Go‑live, rollback, hypercare e handover

#### Fora do âmbito (salvo inclusão explícita)

* Aquisição de hardware de enrolamento e equipamentos de rede
* Mobile Device Management (MDM) para telemóveis do Governo
* Sistemas não pertencentes à Identificação Nacional não são integrados como parte do âmbito do programa
* Desenvolvimento de aplicações customizadas para além das integrações e extensões acordadas

#### Propriedade do documento e controlo de alterações

Este GitBook é mantido conjuntamente pelo **Integrador de Sistemas — Ooru Digital Private Limited** — e pelos **responsáveis técnicos do Governo de São Tomé e Príncipe — INIC e DGRN**. Serve como referência oficial para a arquitetura de implementação, os procedimentos de instalação, as bases de segurança, as integrações e os runbooks operacionais da implementação de Identificação Nacional.

Quaisquer alterações que afetem **procedimentos de implementação em produção**, **controlos de segurança**, **interfaces de integração**, **tratamento de dados** ou **runbooks de operações/DR** devem seguir o **processo de gestão de mudanças** acordado. Essas alterações devem ser **revistas**, **avaliadas quanto ao risco** e **formalmente aprovadas** pelos responsáveis técnicos designados (INIC/DGRN) e pelo Integrador de Sistemas (Ooru Digital) **antes** de serem adotadas em produção.
