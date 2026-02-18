---
description: >-
  Documentação de implementação e operação do Programa de Identificação Nacional de STP
  using MOSIP.
---

# Nacional ID

Este GitBook disponibiliza a documentação de implementação e de operação do programa de Identificação Nacional em São Tomé e Príncipe (STP), utilizando o **MOSIP** como plataforma base de Identificação Nacional. Destina-se a orientar as partes interessadas do Governo, os integradores de sistemas e as equipas de operações ao longo da configuração ponta-a-ponta — desde a preparação do ambiente e a implementação da plataforma base até ao implementação gradual do registo, integração (quando aplicável), entrada em produção (go‑live) e operações contínuas.

### Purpose of this Documentação

This documentação aims para:

* Provide a clear, repeatable approach para deploying MOSIP across **DEV / SIT / UAT / PRE-PROD / PROD / DR** ambientes
* Establish common standards para **segurança, configuration management, monitorização, cópias de segurança, e desastre recuperação**
* Define how **registo locais** are prepared e rolled out, including dispositivo prontidão e field support procedures
* Document integration touchpoints (e.g., ABIS, CRVS/civil registry, messaging gateways, e other government systems as applicable)
* Support a controlled **entrada em produção e handover**, com prontidão gates, cutover steps, rollback plans, e hypercare

### Solution Visão geral (High-Level)

A solução de Identificação Nacional assenta no MOSIP e inclui:

* **Enrollment & Registration**: operator-led registo utilizando registration clients e MOSIP-compliant devices
* **Packet Processing & Workflow**: backend processing, validação, e adjudication of registo packets
* **Deduplication**: Deduplication utilizando integration com the Civil Registry System in Sao Tome & Principe.
* **Identity Repository**: secure storage of identity records e supporting artifacts under strict acesso controls
* **Credential Issuance**: issuance of nacional identification credentials (physical e/ou digital, depending on scope)
* **Authentication & Verification (as applicable)**: standards-based authentication e verification services para approved relying parties, including público verification modules where required

### Implementação Principles

The implementação follows these principles:

* **Segurança by design**: least privilege acesso, encryption in transit e at rest, strong auditoria trails, e controlled admin acesso
* **Repeatable deployments**: infrastructure e plataforma configuração are versionado e reproduzível across ambientes
* **Operational prontidão**: monitorização, alertas, runbooks, backups, e DR plans are part of the base implementação—not an afterthought
* **Scalable implementação gradual**: registo is introduced via pilots e phased expansion com site prontidão e acceptance criteria

### Intended Audience

This GitBook is destina-se para:

* Government programa leadership e technical governance equipas
* Nacional ID IT administrators e infrastructure equipas
* System integrator implementação e engineering equipas
* Equipas de segurança, auditoria e conformidade
* Operations e support equipas (NOC / helpdesk / L2 / L3)

### How para Utilize This GitBook

* Comece por **Arquitetura** para compreender a topologia de implementação e os principais blocos do MOSIP
* Utilize **Plataforma Pré-requisitos** before provisioning ambientes
* Siga **Implementação Base do MOSIP** e **Integrações** para orientações de instalação passo a passo
* Utilize **Registo e Implementação no Terreno** para prontidão dos locais, configuração de dispositivos e procedimentos operacionais
* Refer para **Operations, Backup & DR** para day-2 activities e business continuity
* Utilize **Go-Live & Handover** checklists para run a controlled cutover e transition para steady-state operações

### In Scope (This Documentação)

* Environment configuração e MOSIP implementação (all ambientes)
* Configuration e secrets management approach
* External System integration, implementação, e validação
* Enrollment implementação gradual playbook e prontidão checklists
* Monitoring, logging, cópias de segurança/restore, e DR runbooks
* Go-live, rollback, hypercare, e handover

### Out of Scope (Unless Explicitly Added)

* Procurement of registo hardware e rede equipment
* Mobile dispositivo management (MDM) para government phones
* Non-Nacional ID systems are not integrated as part of the programa scope
* Custom application development beyond agreed integrations e extensions

### Document Ownership e Change Control

Este GitBook é mantido conjuntamente pelo **Integrador de Sistemas — Ooru Digital Private Limited** e pelos **responsáveis técnicos do Governo de São Tomé e Príncipe — INIC e DGRN**. Serve como referência oficial para a arquitetura de implementação, procedimentos de instalação, bases de segurança, integrações e runbooks operacionais da implementação de Identificação Nacional.

Quaisquer alterações que afetem **procedimentos de implementação em produção**, **controlos de segurança**, **interfaces de integração**, **tratamento de dados** ou **runbooks de operações/DR** devem seguir o **processo de gestão de mudanças** acordado. Essas alterações devem ser **revistas**, **avaliadas quanto ao risco** e **formalmente aprovadas** pelos responsáveis técnicos designados (INIC/DGRN) e pelo Integrador de Sistemas (Ooru Digital) **antes** de serem adotadas em produção.