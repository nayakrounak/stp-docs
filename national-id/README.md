---
description: >-
  Implementation and deployment documentation for the STP National ID program
  using MOSIP.
---

# Identificação Nacional

Este GitBook provides the implementation and implementação documentation for the Identificação Nacional program in São Tomé & Príncipe (STP), using **MOSIP** as the foundational Identificação Nacional platform. It is intended to guide government stakeholders, system integrators, and operations teams through the end-to-end setup—from ambiente preparation and core platform implementação to enrollment rollout, ABIS integration, go-live, and ongoing operations.

### Purpose of this Documentation

This documentation aims to:

* Provide a clear, repeatable approach to deploying MOSIP across **DEV / SIT / UAT / PRE-PROD / PROD / DR** ambientes
* Establish common standards for **segurança, configuration management, monitoring, backup, and disaster recovery**
* Define how **enrollment sites** are prepared and rolled out, including device readiness and field support procedures
* Document integration touchpoints (e.g., ABIS, CRVS/civil registry, messaging gateways, and other government systems as applicable)
* Support a controlled **go-live and handover**, with readiness gates, cutover steps, rollback plans, and hypercare

### Solution Overview (High-Level)

The Identificação Nacional solution is centered on MOSIP and includes:

* **Enrollment & Registration**: operator-led enrollment using registration clients and MOSIP-compliant devices
* **Packet Processing & Workflow**: backend processing, validation, and adjudication of enrollment packets
* **Deduplication**: Deduplication using integration with the Civil Registry System in Sao Tome & Principe.
* **Identity Repository**: secure storage of identity records and supporting artifacts under strict access controls
* **Credential Issuance**: issuance of national identification credentials (physical and/or digital, depending on scope)
* **Authentication & Verification (as applicable)**: standards-based authentication and verification services for approved relying parties, including público verification modules where necessários

### Deployment Principles

The implementation follows these principles:

* **Security by design**: least privilege access, encryption in transit and at rest, strong audit trails, and controlled admin access
* **Repeatable implementaçãos**: infrastructure and platform setup are versioned and reproducible across ambientes
* **Operational readiness**: monitoring, alerting, runbooks, backups, and DR plans are part of the baseline implementação—not an afterthought
* **Scalable rollout**: enrollment is introduced via pilots and phased expansion with site readiness and acceptance criteria

### Intended Audience

Este GitBook is intended for:

* Government program leadership and technical governance teams
* Identificação Nacional IT administrators and infrastructure teams
* System integrator implementação and engineering teams
* Security, audit, and compliance teams
* Operations and support teams (NOC / helpdesk / L2 / L3)

### How to Use Este GitBook

* Start with **Arquitetura** to understand the implementação topology and key MOSIP building blocks
* Use **Platform Pré-requisitos** before provisioning ambientes
* Follow **Core MOSIP Deployment** and **Integrations** for step-by-step installation guidance
* Use **Enrollment & Field Rollout** for site readiness, device setup, and operational procedures
* Refer to **Operations, Backup & DR** for day-2 activities and business continuity
* Use **Go-Live & Handover** checklists to run a controlled cutover and transition to steady-state operations

### In Scope (This Documentation)

* Environment setup and MOSIP implementação (all ambientes)
* Configuration and secrets management approach
* External System integration, implementação, and validation
* Enrollment rollout playbook and readiness checklists
* Monitoring, logging, backup/restore, and DR runbooks
* Go-live, rollback, hypercare, and handover

### Out of Scope (Unless Explicitly Added)

* Procurement of enrollment hardware and network equipment
* Mobile device management (MDM) for government phones
* Non-Identificação Nacional systems are not integrated as part of the program scope
* Custom application development beyond agreed integrations and extensions

### Propriedade do Documento e Controlo de Alterações

Este GitBook é mantido conjuntamente pelo **Integrador de Sistemas — Ooru Digital Private Limited** e pelos **responsáveis técnicos do Governo de São Tomé e Príncipe — INIC e DGRN**. Serve como referência oficial para a arquitetura de implementação, procedimentos de instalação, bases de segurança, integrações e runbooks operacionais da implementação de Identificação Nacional.

Quaisquer alterações que afetem **procedimentos de implementação em produção**, **controlos de segurança**, **interfaces de integração**, **tratamento de dados** ou **runbooks de operações/DR** devem seguir o **processo de gestão de mudanças** acordado. Essas alterações devem ser **revistas**, **avaliadas quanto ao risco** e **formalmente aprovadas** pelos responsáveis técnicos designados (INIC/DGRN) e pelo Integrador de Sistemas (Ooru Digital) **antes** de serem adotadas em produção.

Controlos recomendados:
- As alterações devem ser submetidas através de atualizações com controlo de versões (PR/MR), com um resumo claro, justificação e plano de rollback.
- Alterações com impacto em produção devem ser revistas por pares e aprovadas antes do merge.
- Cada release de produção deve referenciar um pedido de mudança aprovado e uma versão etiquetada da documentação.



Este GitBook is jointly maintained by the System Integrator — Ooru Digital Private Limited and the Government of São Tomé & Príncipe technical owners — INIC and DGRN. It serves as the authoritative reference for implementação architecture, installation procedures, segurança baselines, integrations, and operational runbooks for the Identificação Nacional implementation.

Any changes that affect production implementação procedures, segurança controls, integration interfaces, data handling, or operations/DR runbooks deve follow the agreed change management process. Such changes deve be reviewed, risk-assessed, and formally approved by the designated technical owners (INIC/DGRN) and the System Integrator (Ooru Digital) before being adopted in production.