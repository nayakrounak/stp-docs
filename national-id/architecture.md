---
description: >-
  Target architecture and key flows for STP's National ID using MOSIP, including
  SIGA and Immigration integrations.
---

# Arquitetura

Esta secção describes the target architecture for the Identificação Nacional implementation in São Tomé & Príncipe (STP) using **MOSIP** as the foundational platform. It covers the logical building blocks, ambiente topology, network zoning, and the primary end-to-end flows for enrollment, processing, issuance, and verification—along with integrations with **SIGA** and **Immigration**.

***

### 1. Arquitetura Overview

The MOSIP-based Identificação Nacional platform is deployed as a set of secure, scalable services hosted on a Kubernetes-based infrastructure. The architecture is organized into:

* **Enrollment Edge**: registration workstations and MOSIP-compliant biometric devices used at enrollment sites.
* **Core MOSIP Cluster**: MOSIP services responsible for packet intake, processing workflows, identity repository management, and issuance.
* **Integration Layer**: secure interfaces connecting MOSIP to **SIGA** and **Immigration** systems for identity validation, data synchronization, and operational workflows.
* **Access & Identity**: Keycloak-based IAM (and eSignet if in scope) for authentication, authorization, and partner access.
* **Observability & Operations**: monitoring, logging, auditing, backups, and DR mechanisms supporting day-2 operations.

This architecture supports phased rollout (pilot → scale) and provides controlled exposure for público-facing portals and APIs.

***

### 2. Logical Building Blocks

#### 2.1 Enrollment & Registration

* **Registration Client** deployed on enrollment workstations for demographic capture, document capture, and biometric capture
* **MOSIP-compliant biometric devices** (fingerprint/face/iris as per STP scope) connected to the registration workstation
* **Packet creation**: enrollment data and artifacts are packaged securely for transmission to the backend

#### 2.2 Packet Intake & Processing

* **Packet receiver/intake services** to accept packets from enrollment sites
* **Registration Processor pipeline** to validate, extract, and process packet data through defined workflows
* **Operator workflows** (as applicable) for exception handling, quality checks, and adjudication

#### 2.3 Identity Repository

* Secure storage of:
  * Demographic identity data
  * Biometric references and metadata (as per STP policy and MOSIP configuration)
  * Document artifacts and enrollment evidence
  * Audit trails and system events
* APIs for internal services and authorized relying systems

#### 2.4 Issuance

* Identity finalization and assignment of identifiers (as per program policy)
* Credential issuance workflows, including:
  * Print/card integration (if applicable)
  * Digital credential issuance (if applicable)
* Status tracking and reporting

#### 2.5 Authentication & Verification (as applicable)

Depending on the scope:

* **Public verification portal** for document/credential authenticity checks
* **eSignet** for standards-based authentication and consent (OIDC) for relying parties
* **Partner onboarding and access controls** for consuming identity services

#### 2.6 Observability & Operations

* Central logging, metrics, dashboards, and alerts
* Audit logging and tamper-evidence controls
* Backup/restore and DR readiness

***

### 3. Environment Topology

MOSIP is deployed across multiple ambientes to support safe release management:

<table><thead><tr><th width="140.15234375">Environment</th><th>Purpose</th><th>Data Type</th><th>Notes</th></tr></thead><tbody><tr><td>DEV</td><td>Development</td><td>Synthetic</td><td>SI-only</td></tr><tr><td>SIT</td><td>Integration Testing</td><td>Synthetic/masked</td><td>Includes SIGA/Immigration test endpoints</td></tr><tr><td>UAT</td><td>Business Validation</td><td>Masked/controlled</td><td>UAT users and acceptance tests</td></tr><tr><td>PRE-PROD</td><td>Dress Rehearsal</td><td>Masked subset</td><td>Production-like topology</td></tr><tr><td>PROD</td><td>Live Operations</td><td>Real</td><td>Audited &#x26; change-controlled</td></tr><tr><td>DR</td><td>Failover</td><td>Real replica</td><td>Restricted access</td></tr></tbody></table>

**Promotion policy:** DEV → SIT → UAT → PRE-PROD → PROD\
**Key rule:** Same implementação method across ambientes (IaC/Helm), with scaling differences only.

***

### 4. Network Zones & Trust Boundaries

The implementação is segmented into zones to reduce blast radius and enforce least privilege.

#### 4.1 Recommended Zones

* **Public Zone (DMZ):**
  * Only what deve be internet-facing (e.g., público portals/APIs)
  * Protected by WAF, TLS, and rate limits
* **Application Zone:**
  * MOSIP application services, integration services, and internal APIs
* **Data Zone:**
  * Databases, object storage, backup repositories, KMS/HSM
* **Admin Zone:**
  * Bastion, ops consoles, cluster management ferramentas
  * MFA enforced and access logged
* **Enrollment Edge:**
  * Enrollment sites and secure connectivity to backend intake endpoints

#### 4.2 Exposure Principles

* Private admin services are accessible only via **WireGuard** and/or **admin allowlists**
* No direct access from the público zone to the data zone
* Service-to-service communication uses TLS where necessários
* All administrative actions deve be auditable

***

### 5. Integration Arquitetura (SIGA + Immigration)

The Identificação Nacional platform integrates with external government systems through a controlled **Integration Layer**. This layer standardizes:

* **Secure connectivity** (VPN/privado link, mTLS, IP allowlists)
* **Authentication and authorization** (service accounts, token-based access)
* **Schema validation and versioning**
* **Retry and reconciliation** (idempotency, DLQ where applicable)
* **Auditability** (all requests/changes logged and traceable)

#### 5.1 Integration Patterns

The following patterns may be used depending on the external system capability:

* **Synchronous API calls (REST)** for real-time validation/lookup
* **Asynchronous exchange (queue/event)** when decoupling and resilience are necessários
* **Batch ETL/SFTP** for scheduled bulk sync (where APIs are not available or not stable)

#### 5.2 SIGA Integration (Functional Scope)

Typical scope of integration with SIGA includes:

* **Identity validation / cross-checks** during enrollment or adjudication (e.g., verifying existing civil registry data)
* **Reference data synchronization** (e.g., locations, administrative units, registries—where SIGA is the source of truth)
* **Status updates** from Identificação Nacional to SIGA (e.g., “enrolled”, “issued”, “updated”, “deactivated”) based on agreed governance
* **Exception handling workflows** when discrepancies are found (manual review/adjudication)

**Recommended Controls**

* Define authoritative fields (MOSIP vs SIGA) and conflict resolution rules
* Use idempotent update operations (avoid duplicate updates)
* Maintain reconciliation reports (daily/weekly) to track mismatches and retries

#### 5.3 Immigration Integration (Functional Scope)

Typical scope of integration with Immigration includes:

* **Foreigner/resident permit verification** during enrollment of non-citizens
* **Status synchronization** (e.g., permit expiry, visa status changes) for lifecycle updates
* **Border/entry workflows support** (as applicable), where Identificação Nacional verification is necessários for immigration operations

**Recommended Controls**

* Strict access controls (least privilege) and purpose limitation
* PII minimization (only exchange necessários attributes)
* Strong audit trails for all queries and updates
* Time-bound caching rules (if caching is allowed)

#### 5.4 Connectivity, Security, and Exposure

For both SIGA and Immigration integrations:

* Connectivity deverá be via **privado network** whenever feasible (VPN/MPLS/privado peering)
* Use **mTLS** for service-to-service communications where supported
* Enforce **IP allowlisting** at both ends
* Use a dedicated **service account** per integration per ambiente
* Apply rate limits and timeouts to protect MOSIP and external systems

#### 5.5 Error Handling and Reconciliation

* Define standard error codes and retry rules per integration
* Implement idempotency keys for updates (where applicable)
* Capture failures into a controlled retry queue or DLQ mechanism
* Produce reconciliation reports:
  * records pending sync
  * records failed sync (with reason)
  * successful updates and timestamps

***

### 6. Core Data Flows (End-to-End)

#### 6.1 Enrollment → Packet Submission

1. Operator captures demographics, documents, and biometrics on the Registration Client
2. Registration Client performs local validations and quality checks (as configured)
3. Enrollment data is packaged as an encrypted/signed packet
4. Packet is uploaded to backend intake (online or store-and-forward based on site model)

#### 6.2 Packet Processing → SIGA/Immigration Cross-Checks (as configured)

1. Registration Processor validates packet integrity and schema compliance
2. Demographics and artifacts are extracted and staged
3. Integration Layer triggers necessários lookups/validations against SIGA and/or Immigration
4. Workflow evaluates responses:
   * match/valid → continue
   * discrepancy → exception/adjudication flow
   * unavailable → queued retry or manual fallback as per SOP

#### 6.3 Identity Finalization → Issuance

1. Successful packets result in identity creation/finalization
2. Identifier assignment and record creation in the Identity Repository
3. Issuance workflow triggers:
   * Print/card generation (if applicable)
   * Digital credential issuance (if applicable)
4. Final status and audit logs are updated

#### 6.4 Verification / Authentication (if in scope)

* Public verification checks the authenticity of printed credentials/documents
* Relying parties use authenticated APIs (and eSignet, where applicable) for consent-based access
* All verification events are logged and monitored

***

### 7. High-Level Component Inventory (MOSIP-Aligned)

* Registration Client (field enrollment)
* Packet intake and processing pipeline (Registration Processor)
* Identity Repository services
* Master data and admin configuration services
* Integration layer components (SIGA + Immigration connectors)
* Keycloak (IAM) and admin access controls
* eSignet (if enabled) for OIDC authentication/consent
* Observability stack (logs, metrics, alerts)
* Data services: PostgreSQL, object storage, messaging, search/log store (as adopted)

***

### 8. Non-Functional Arquitetura Targets

* **Security:** least privilege, encryption in transit and at rest, strong auditing, controlled admin access
* **Availability:** HA for critical components; DR capability with defined RTO/RPO
* **Scalability:** horizontal scaling for stateless services; capacity planning for data services
* **Performance:** monitored p95 latency for key APIs and processing throughput targets
* **Operability:** dashboards, runbooks, alerts, and maintenance processes