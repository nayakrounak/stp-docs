---
title: National ID
description: Implementation and deployment documentation for the STP National ID program using MOSIP.
---

# National ID

This GitBook provides the implementation and deployment documentation for the National ID program in São Tomé & Príncipe (STP), using **MOSIP** as the foundational National ID platform. It is intended to guide government stakeholders, system integrators, and operations teams through the end-to-end setup—from environment preparation and core platform deployment to enrollment rollout, ABIS integration, go-live, and ongoing operations.

## Related pages

- Start here: [Architecture](./architecture.md)

- Prepare environment: [Platform Pre-requisites](./platform-pre-requisites.md)

- Build clusters: [Cluster Provisioning & Baseline Setup](./cluster-provisioning-and-baseline-setup.md)

- Route traffic: [Ingress & Edge Routing Setup](./ingress-and-edge-routing-setup.md)

- Install MOSIP: [MOSIP Platform Installation](./mosip-platform-installation.md)

### Purpose of this Documentation

This documentation aims to:

* Provide a clear, repeatable approach to deploying MOSIP across **DEV / SIT / UAT / PRE-PROD / PROD / DR** environments
* Establish common standards for **security, configuration management, monitoring, backup, and disaster recovery**
* Define how **enrollment sites** are prepared and rolled out, including device readiness and field support procedures
* Document integration touchpoints (e.g., ABIS, CRVS/civil registry, messaging gateways, and other government systems as applicable)
* Support a controlled **go-live and handover**, with readiness gates, cutover steps, rollback plans, and hypercare

### Solution Overview (High-Level)

The National ID solution is centered on MOSIP and includes:

* **Enrollment & Registration**: operator-led enrollment using registration clients and MOSIP-compliant devices
* **Packet Processing & Workflow**: backend processing, validation, and adjudication of enrollment packets
* **Deduplication**: Deduplication using integration with the Civil Registry System in Sao Tome & Principe.
* **Identity Repository**: secure storage of identity records and supporting artifacts under strict access controls
* **Credential Issuance**: issuance of national identification credentials (physical and/or digital, depending on scope)
* **Authentication & Verification (as applicable)**: standards-based authentication and verification services for approved relying parties, including public verification modules where required

### Deployment Principles

The implementation follows these principles:

* **Security by design**: least privilege access, encryption in transit and at rest, strong audit trails, and controlled admin access
* **Repeatable deployments**: infrastructure and platform setup are versioned and reproducible across environments
* **Operational readiness**: monitoring, alerting, runbooks, backups, and DR plans are part of the baseline deployment—not an afterthought
* **Scalable rollout**: enrollment is introduced via pilots and phased expansion with site readiness and acceptance criteria

### Intended Audience

This GitBook is intended for:

* Government program leadership and technical governance teams
* National ID IT administrators and infrastructure teams
* System integrator deployment and engineering teams
* Security, audit, and compliance teams
* Operations and support teams (NOC / helpdesk / L2 / L3)

### How to Use This GitBook

* Start with **Architecture** to understand the deployment topology and key MOSIP building blocks
* Use **Platform Prerequisites** before provisioning environments
* Follow **Core MOSIP Deployment** and **Integrations** for step-by-step installation guidance
* Use **Enrollment & Field Rollout** for site readiness, device setup, and operational procedures
* Refer to **Operations, Backup & DR** for day-2 activities and business continuity
* Use **Go-Live & Handover** checklists to run a controlled cutover and transition to steady-state operations

### In Scope (This Documentation)

* Environment setup and MOSIP deployment (all environments)
* Configuration and secrets management approach
* External System integration, deployment, and validation
* Enrollment rollout playbook and readiness checklists
* Monitoring, logging, backup/restore, and DR runbooks
* Go-live, rollback, hypercare, and handover

### Out of Scope (Unless Explicitly Added)

* Procurement of enrollment hardware and network equipment
* Mobile device management (MDM) for government phones
* Non-National ID systems are not integrated as part of the program scope
* Custom application development beyond agreed integrations and extensions

### Document Ownership and Change Control

This GitBook is maintained by the implementation team in coordination with the designated government technical owners. Changes to production deployment procedures, security controls, and operational runbooks should follow the agreed change management process and must be reviewed and approved before adoption in production.

---
## Navigation
- **Next:** [Architecture](./architecture.md)
