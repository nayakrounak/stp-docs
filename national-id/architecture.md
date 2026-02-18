---
description: >-
  Target architecture and key flows for STP's National ID using MOSIP, including
  SIGA and Immigration integrations.
---

# Arquitetura

Esta secção descreve a arquitetura alvo para a implementação de Identificação Nacional em São Tomé e Príncipe (STP), utilizando o **MOSIP** como plataforma base. Abrange os blocos lógicos, a topologia do ambiente, a segmentação de rede e os principais fluxos ponta‑a‑ponta para registo, processamento, emissão e verificação — bem como as integrações com o **SIGA** e a **Imigração**.

***

### 1. Arquitetura Visão geral

A plataforma de Identificação Nacional baseada em MOSIP é implementada como um conjunto de serviços seguros e escaláveis, alojados numa infraestrutura baseada em Kubernetes. A arquitetura está organizada em:

* **Enrollment Edge**: registration workstations e MOSIP-compliant biometric devices used at registo locais.
* **Core MOSIP Cluster**: MOSIP services responsible para packet intake, processing workflows, identity repository management, e issuance.
* **Integration Layer**: secure interfaces connecting MOSIP para **SIGA** e **Immigration** systems para identity validação, data synchronization, e operacional workflows.
* **Acesso & Identity**: Keycloak-based IAM (e eSignet if in scope) para authentication, authorization, e partner acesso.
* **Observability & Operations**: monitorização, logging, auditing, backups, e DR mechanisms supporting day-2 operações.

This architecture supports phased implementação gradual (pilot → scale) e disponibiliza controlled exposure para público-facing portals e APIs.

***

### 2. Logical Building Blocks

#### 2.1 Enrollment & Registration

* **Registration Client** deployed on registo workstations para demographic capture, document capture, e biometric capture
* **MOSIP-compliant biometric devices** (fingerprint/face/iris as per STP scope) connected para the registration workstation
* **Packet creation**: registo data e artifacts are packaged securely para transmission para the backend

#### 2.2 Packet Intake & Processing

* **Packet receiver/intake services** para accept packets from registo locais
* **Registration Processor pipeline** para validate, extract, e process packet data through defined workflows
* **Operator workflows** (as applicable) para exception handling, quality checks, e adjudication

#### 2.3 Identity Repository

* Secure storage of:
  * Demographic identity data
  * Biometric references e metadata (as per STP policy e MOSIP configuration)
  * Document artifacts e registo evidence
  * Audit trails e sistema events
* APIs para internal services e authorized relying systems

#### 2.4 Issuance

* Identity finalization e assignment of identifiers (as per programa policy)
* Credential issuance workflows, including:
  * Print/card integration (if applicable)
  * Digital credential issuance (if applicable)
* Status tracking e reporting

#### 2.5 Authentication & Verification (as applicable)

Depending on the scope:

* **Público verification portal** para document/credential authenticity checks
* **eSignet** para standards-based authentication e consent (OIDC) para relying parties
* **Partner onboarding e acesso controls** para consuming identity services

#### 2.6 Observability & Operations

* Central logging, metrics, dashboards, e alerts
* Audit logging e tamper-evidence controls
* Backup/restore e DR prontidão

***

### 3. Environment Topologia

MOSIP is deployed across multiple ambientes para support safe release management:

<table><thead><tr><th width="140.15234375">Environment</th><th>Purpose</th><th>Data Type</th><th>Notes</th></tr></thead><tbody><tr><td>DEV</td><td>Development</td><td>Synthetic</td><td>SI-only</td></tr><tr><td>SIT</td><td>Integration Testing</td><td>Synthetic/masked</td><td>Includes SIGA/Immigration test endpoints</td></tr><tr><td>UAT</td><td>Business Validation</td><td>Masked/controlled</td><td>UAT users e acceptance tests</td></tr><tr><td>PRE-PROD</td><td>Dress Rehearsal</td><td>Masked subset</td><td>Production-like topologia</td></tr><tr><td>PROD</td><td>Live Operations</td><td>Real</td><td>Audited &#x26; change-controlled</td></tr><tr><td>DR</td><td>Failover</td><td>Real replica</td><td>Restricted acesso</td></tr></tbody></table>

**Promotion policy:** DEV → SIT → UAT → PRE-PROD → PROD\
**Key rule:** Same implementação method across ambientes (IaC/Helm), com scaling differences only.

***

### 4. Rede Zones & Trust Boundaries

The implementação is segmented into zones para reduce blast radius e enforce least privilege.

#### 4.1 Recomendado Zones

* **Público Zone (DMZ):**
  * Only what deve be internet-facing (e.g., público portals/APIs)
  * Protected by WAF, TLS, e rate limits
* **Application Zone:**
  * MOSIP application services, integration services, e internal APIs
* **Data Zone:**
  * Databases, object storage, cópias de segurança repositories, KMS/HSM
* **Admin Zone:**
  * Bastion, ops consoles, cluster management ferramentas
  * MFA enforced e acesso logged
* **Enrollment Edge:**
  * Enrollment locais e secure connectivity para backend intake endpoints

#### 4.2 Exposure Principles

* Privado admin services are accessible only via **WireGuard** e/ou **admin allowlists**
* Não existe acesso direto da zona pública para a zona de dados
* Service-para-service communication uses TLS where required
* All administrative actions deve be auditable

***

### 5. Integration Arquitetura (SIGA + Immigration)

A plataforma de Identificação Nacional integra-se com sistemas governamentais externos através de uma **Camada de Integração** controlada. Esta camada padroniza:

* **Secure connectivity** (VPN/privado link, mTLS, IP allowlists)
* **Authentication e authorization** (service accounts, token-based acesso)
* **Schema validação e versioning**
* **Retry e reconciliation** (idempotency, DLQ where applicable)
* **Auditability** (all requests/changes logged e traceable)

#### 5.1 Integration Patterns

Os seguintes padrões podem ser utilizados, dependendo das capacidades do sistema externo:

* **Synchronous API calls (REST)** para real-time validação/lookup
* **Asynchronous exchange (queue/event)** when decoupling e resilience are required
* **Batch ETL/SFTP** para scheduled bulk sync (where APIs are not available ou not stable)

#### 5.2 SIGA Integration (Functional Scope)

Typical scope of integration com SIGA inclui:

* **Identity validação / cross-checks** during registo ou adjudication (e.g., verifying existing civil registry data)
* **Reference data synchronization** (e.g., locations, administrative units, registries—where SIGA is the source of truth)
* **Status updates** from Nacional ID para SIGA (e.g., “enrolled”, “issued”, “updated”, “deactivated”) based on agreed governance
* **Exception handling workflows** when discrepancies are found (manual review/adjudication)

**Recomendado Controls**

* Define authoritative fields (MOSIP vs SIGA) e conflict resolution rules
* Utilize idempotent update operações (avoid duplicate updates)
* Maintain reconciliation reports (daily/weekly) para track mismatches e retries

#### 5.3 Immigration Integration (Functional Scope)

Typical scope of integration com Immigration inclui:

* **Foreigner/resident permit verification** during registo of non-citizens
* **Status synchronization** (e.g., permit expiry, visa status changes) para lifecycle updates
* **Border/entry workflows support** (as applicable), where Nacional ID verification is required para immigration operações

**Recomendado Controls**

* Strict acesso controls (least privilege) e purpose limitation
* PII minimization (only exchange required attributes)
* Strong auditoria trails para all queries e updates
* Time-bound caching rules (if caching is allowed)

#### 5.4 Connectivity, Segurança, e Exposure

For both SIGA e Immigration integrations:

* Connectivity deverá be via **privado rede** whenever feasible (VPN/MPLS/privado peering)
* Utilize **mTLS** para service-para-service communications where supported
* Enforce **IP allowlisting** at both ends
* Utilize a dedicated **service account** per integration per ambiente
* Apply rate limits e timeouts para protect MOSIP e external systems

#### 5.5 Error Handling e Reconciliation

* Define standard error codes e retry rules per integration
* Implement idempotency keys para updates (where applicable)
* Capture failures into a controlled retry queue ou DLQ mechanism
* Produce reconciliation reports:
  * records pending sync
  * records failed sync (com reason)
  * successful updates e timestamps

***

### 6. Core Data Flows (End-para-End)

#### 6.1 Enrollment → Packet Submission

1. Operator captures demographics, documents, e biometrics on the Registration Client
2. Registration Client performs local validations e quality checks (as configured)
3. Enrollment data is packaged as an encrypted/signed packet
4. Packet is uploaded para backend intake (online ou store-e-forward based on site model)

#### 6.2 Packet Processing → SIGA/Immigration Cross-Checks (as configured)

1. Registration Processor validates packet integrity e schema conformidade
2. Demographics e artifacts are extracted e staged
3. Integration Layer triggers required lookups/validations against SIGA e/ou Immigration
4. Workflow evaluates responses:
   * match/valid → continue
   * discrepancy → exception/adjudication fluxo
   * unavailable → queued retry ou manual fallback as per SOP

#### 6.3 Identity Finalization → Issuance

1. Successful packets result in identity creation/finalization
2. Identifier assignment e record creation in the Identity Repository
3. Issuance workflow triggers:
   * Print/card generation (if applicable)
   * Digital credential issuance (if applicable)
4. Final status e auditoria logs are updated

#### 6.4 Verification / Authentication (if in scope)

* Público verification checks the authenticity of printed credentials/documents
* Relying parties usar authenticated APIs (e eSignet, where applicable) para consent-based acesso
* All verification events are logged e monitored

***

### 7. High-Level Component Inventory (MOSIP-Aligned)

* Registration Client (field registo)
* Packet intake e processing pipeline (Registration Processor)
* Identity Repository services
* Master data e admin configuration services
* Integration layer components (SIGA + Immigration connectors)
* Keycloak (IAM) e admin acesso controls
* eSignet (if enabled) para OIDC authentication/consent
* Observability stack (logs, metrics, alerts)
* Data services: PostgreSQL, object storage, messaging, search/log store (as adopted)

***

### 8. Non-Functional Arquitetura Targets

* **Segurança:** least privilege, encryption in transit e at rest, strong auditing, controlled admin acesso
* **Availability:** HA para critical components; DR capability com defined RTO/RPO
* **Scalability:** horizontal scaling para stateless services; capacity planning para data services
* **Performance:** monitored p95 latency para key APIs e processing throughput targets
* **Operability:** dashboards, runbooks, alerts, e maintenance processes