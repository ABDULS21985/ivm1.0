# 05 - Security & Compliance Architecture

## 1. Threat Model

### 1.1 System Actors & Trust Boundaries

```
┌─────────────────────────────────────────────────────────────────┐
│                    TRUST BOUNDARY: INTERNET                      │
│                                                                  │
│  [Customer]  [Partner API]  [Attacker]                          │
│       │            │             │                               │
│       ▼            ▼             ▼                               │
│  ┌─────────────────────────────────────────────┐                │
│  │  TRUST BOUNDARY: CLOUD PERIMETER             │                │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │                │
│  │  │ WAF/DDoS │  │ API GW   │  │ MQTT     │   │                │
│  │  │ Shield   │  │ (AuthN)  │  │ Broker   │   │                │
│  │  └──────────┘  └──────────┘  └──────────┘   │                │
│  │       │             │             │          │                │
│  │  ┌──────────────────────────────────────┐    │                │
│  │  │ TRUST BOUNDARY: SERVICE MESH         │    │                │
│  │  │  ┌────────┐ ┌────────┐ ┌────────┐   │    │                │
│  │  │  │Services│ │Services│ │Services│   │    │                │
│  │  │  └────────┘ └────────┘ └────────┘   │    │                │
│  │  │                │                     │    │                │
│  │  │  ┌─────────────────────────────┐     │    │                │
│  │  │  │ TRUST BOUNDARY: DATA LAYER  │     │    │                │
│  │  │  │  ┌─────┐ ┌──────┐ ┌──────┐ │     │    │                │
│  │  │  │  │ DB  │ │Cache │ │Object│ │     │    │                │
│  │  │  │  │     │ │      │ │Store │ │     │    │                │
│  │  │  │  └─────┘ └──────┘ └──────┘ │     │    │                │
│  │  │  └─────────────────────────────┘     │    │                │
│  │  └──────────────────────────────────────┘    │                │
│  └──────────────────────────────────────────────┘                │
│                          │                                       │
│  ┌───────────────────────┼──────────────────────┐                │
│  │  TRUST BOUNDARY: VPN TUNNEL                  │                │
│  └───────────────────────┼──────────────────────┘                │
│                          │                                       │
│  ┌───────────────────────┼──────────────────────────────────┐    │
│  │  TRUST BOUNDARY: EDGE SITE                                │    │
│  │  ┌──────────────┐                                        │    │
│  │  │Edge Controller│                                       │    │
│  │  └──────┬───────┘                                        │    │
│  │         │                                                │    │
│  │  ┌──────────────────────────────────────────────────┐    │    │
│  │  │  TRUST BOUNDARY: DEVICE NETWORK                  │    │    │
│  │  │  [Scanner] [Printer] [Lock] [Camera] [Card Rdr]  │    │    │
│  │  └──────────────────────────────────────────────────┘    │    │
│  │                                                          │    │
│  │  [Physical Attacker] [Shoulder Surfer] [Tamper Attempt] │    │
│  └──────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Threat Categories

| Threat | Impact | Targets | Mitigations |
|---|---|---|---|
| **Credential theft** | Account takeover, fraud | Auth service, device certs | MFA, rate limiting, anomaly detection |
| **Payment fraud** | Financial loss | Payment service, card readers | PCI DSS, tokenization, fraud engine |
| **Device tampering** | Data exfiltration, skimming | Kiosks, card readers | Tamper detection, PCI PTS, secure enclosure |
| **Man-in-the-middle** | Data interception | Edge-to-cloud, device-to-edge | TLS 1.3, mTLS, certificate pinning |
| **Firmware compromise** | Full device control | Edge controllers, devices | Secure boot, TUF updates, integrity verification |
| **Supply chain attack** | Malicious software injection | Update pipeline | TUF, code signing, reproducible builds |
| **Data breach** | PII/PCI exposure, regulatory fines | Databases, object storage | Encryption at rest/transit, RLS, access control |
| **Insider threat** | Data exfiltration, sabotage | Admin portals, databases | RBAC, audit logging, principle of least privilege |
| **DDoS** | Service disruption | API gateway, MQTT broker | WAF, rate limiting, CDN, auto-scaling |
| **API abuse** | Resource exhaustion, data scraping | Public APIs | Rate limiting, API keys, anomaly detection |
| **Physical intrusion** | Theft, vandalism | Autonomous stores | Cameras, alarms, access control, insurance |
| **Biometric spoofing** | Identity fraud | Biometric scanners | Liveness detection, multi-factor verification |

---

## 2. Security Architecture

### 2.1 Authentication & Authorization

```
┌─────────────────────────────────────────────────────┐
│              AUTHENTICATION FLOWS                    │
│                                                     │
│  Customer (Kiosk/App/Web):                          │
│  ┌────────────────────────────────────────────┐     │
│  │ Phone + OTP (primary for Nigeria market)   │     │
│  │ Email + Password + MFA (optional)           │     │
│  │ Biometric token (returning kiosk customer)  │     │
│  │ QR code scan (store entry, locker pickup)   │     │
│  │ Card tap (NFC, for store entry)             │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  Operator (Admin/Merchant/Technician):              │
│  ┌────────────────────────────────────────────┐     │
│  │ Email + Password + MFA (mandatory)          │     │
│  │ SSO via OIDC (enterprise tenants)           │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  Device (Edge Controller / IoT):                    │
│  ┌────────────────────────────────────────────┐     │
│  │ mTLS certificate (X.509, unique per device) │     │
│  │ Hardware fingerprint validation              │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  Partner (API Integration):                         │
│  ┌────────────────────────────────────────────┐     │
│  │ API Key + HMAC signature                    │     │
│  │ OAuth2 client credentials flow              │     │
│  └────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────┘
```

**Authorization Model: RBAC with Tenant Scoping**

```
Permission Structure:
  {resource}:{action}:{scope}

Examples:
  orders:read:own          (customer: read own orders)
  orders:read:tenant       (merchant: read all tenant orders)
  orders:write:tenant      (operator: create orders for tenant)
  devices:manage:site      (technician: manage devices at assigned sites)
  devices:manage:fleet     (admin: manage all devices)
  payments:refund:tenant   (merchant admin: issue refunds)
  audit:read:platform      (compliance officer: read all audit logs)
  tenants:manage:platform  (platform admin: manage all tenants)

Role Hierarchy:
  Platform Admin → Tenant Admin → Merchant Admin → Operator → Customer
  Platform Admin → Compliance Officer
  Tenant Admin → Site Manager → Field Technician
```

### 2.2 Data Protection

**Encryption Standards:**

| Layer | Protocol/Algorithm | Minimum |
|---|---|---|
| Data in transit (external) | TLS 1.3 | Required everywhere |
| Data in transit (internal) | mTLS with TLS 1.3 | Service mesh enforced |
| Data in transit (MQTT) | TLS 1.3 + client cert | Required for all devices |
| Data at rest (database) | AES-256 TDE | All databases |
| Data at rest (field-level) | AES-256-GCM | PII, identity, financial |
| Data at rest (object storage) | AES-256 server-side | All buckets |
| Data at rest (edge) | LUKS full-disk encryption | All edge controllers |
| Key management | HashiCorp Vault / AWS KMS | HSM-backed |
| Certificate authority | Internal CA + Let's Encrypt | Automated rotation |

**Data Classification:**

| Level | Description | Examples | Controls |
|---|---|---|---|
| **CRITICAL** | Payment card data, biometrics | PAN (never stored), biometric templates | PCI DSS zone, HSM, minimal retention |
| **SENSITIVE** | PII, identity numbers | BVN, NIN, names, addresses, phone | Field encryption, access logging, consent |
| **CONFIDENTIAL** | Business data | Orders, pricing, analytics | Database encryption, RBAC |
| **INTERNAL** | Operational data | Logs, telemetry, configs | Standard encryption |
| **PUBLIC** | Marketing, product info | Catalog images, store locations | CDN, no special controls |

### 2.3 PCI DSS Scope Minimization

```
┌─────────────────────────────────────────────────────────┐
│               PCI DSS SCOPE STRATEGY                    │
│                                                          │
│  Principle: MINIMIZE cardholder data environment (CDE)   │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │  IN SCOPE (CDE)                                   │   │
│  │                                                   │   │
│  │  ┌─────────────────┐    ┌─────────────────┐       │   │
│  │  │ PCI PTS Approved│    │ Payment Gateway │       │   │
│  │  │ Card Terminal   │───▶│ (Paystack /     │       │   │
│  │  │ (PAX / Verifone)│    │  Flutterwave)   │       │   │
│  │  └─────────────────┘    └─────────────────┘       │   │
│  │                                                   │   │
│  │  PAN never touches IVM servers.                   │   │
│  │  Terminal connects directly to gateway.            │   │
│  │  IVM receives only: token + last4 + status.       │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │  OUT OF SCOPE (SAQ-A eligible)                    │   │
│  │                                                   │   │
│  │  All IVM services, databases, edge controllers.   │   │
│  │  Never process, store, or transmit PAN.           │   │
│  │  Only interact with tokenized references.         │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
│  For mobile/web payments:                                │
│  Use gateway-hosted payment page / SDK.                  │
│  IVM backend only receives payment tokens.               │
└─────────────────────────────────────────────────────────┘
```

### 2.4 Device Security

**ETSI EN 303 645 Alignment:**

| Provision | Implementation |
|---|---|
| No universal default passwords | Unique per-device password generated at provisioning |
| Implement vulnerability disclosure | security@ivm.io + responsible disclosure policy |
| Software updates | TUF-based OTA with signed packages |
| Securely store sensitive data | TPM-backed key storage, LUKS disk encryption |
| Communicate securely | TLS 1.3 + mTLS for all device communication |
| Minimize attack surfaces | Unused ports disabled, minimal OS, read-only rootfs |
| Software integrity | Secure boot chain, container image signing |
| Personal data protection | NDPA-aligned data handling, consent management |
| Resilient to outages | Offline queue, local state cache, watchdog |
| Telemetry/system data | Structured logging, no PII in telemetry |
| Easy device deletion | Factory reset command, certificate revocation |
| Installation/maintenance | Field technician app with secure authentication |
| Input validation | All device inputs validated at edge controller |

### 2.5 Network Security

```
┌─────────────────────────────────────────────────────────┐
│                 NETWORK SECURITY LAYERS                   │
│                                                          │
│  Layer 1: Edge Protection                                │
│  ┌────────────────────────────────────────────────┐      │
│  │ CDN + WAF + DDoS Protection (Cloudflare/AWS)   │      │
│  │ - Rate limiting per IP, per API key, per tenant │      │
│  │ - SQL injection / XSS / bot detection           │      │
│  │ - Geographic restrictions (optional)            │      │
│  └────────────────────────────────────────────────┘      │
│                                                          │
│  Layer 2: API Gateway                                    │
│  ┌────────────────────────────────────────────────┐      │
│  │ - JWT validation (signature + expiry + claims)  │      │
│  │ - mTLS termination for device connections       │      │
│  │ - Request size limits (1MB default)             │      │
│  │ - API versioning enforcement                    │      │
│  │ - CORS policy (strict origin allowlist)         │      │
│  └────────────────────────────────────────────────┘      │
│                                                          │
│  Layer 3: Service Mesh                                   │
│  ┌────────────────────────────────────────────────┐      │
│  │ - mTLS between all services (auto-managed)      │      │
│  │ - Network policies (Kubernetes NetworkPolicy)    │      │
│  │ - Service identity via SPIFFE/SPIRE              │      │
│  │ - Deny-by-default, explicit allow rules          │      │
│  └────────────────────────────────────────────────┘      │
│                                                          │
│  Layer 4: Data Layer                                     │
│  ┌────────────────────────────────────────────────┐      │
│  │ - Database accessible only from service mesh    │      │
│  │ - No public IP on any data store                │      │
│  │ - Encrypted connections required (TLS)          │      │
│  │ - Per-service database credentials (short-lived)│      │
│  └────────────────────────────────────────────────┘      │
│                                                          │
│  Layer 5: Edge-to-Cloud VPN                              │
│  ┌────────────────────────────────────────────────┐      │
│  │ - WireGuard tunnel per edge site                │      │
│  │ - Certificate-based authentication              │      │
│  │ - All edge traffic routed through VPN           │      │
│  │ - NAT traversal for sites behind firewalls      │      │
│  └────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Regulatory Compliance

### 3.1 Nigeria Data Protection Act (NDPA) 2023

| Obligation | Platform Implementation |
|---|---|
| **Lawful basis for processing** | Consent management service; contractual necessity for transactions |
| **Data minimization** | Collect only what's needed per workflow; configurable per tenant |
| **Purpose limitation** | Data tagged with processing purpose; access controls enforce purpose |
| **Data subject rights** | Self-service portal: access, rectification, erasure, portability |
| **Data breach notification** | Automated breach detection → incident workflow → 72-hour notification |
| **Data protection impact assessment** | Conducted per channel module; documented and reviewed annually |
| **Cross-border transfers** | Data residency enforced (Nigeria-first); transfer impact assessment for any replication |
| **Registration as controller/processor** | NDPC registration maintained; annual renewal tracked |
| **Record of processing activities** | Audit log service maintains comprehensive processing records |
| **Data retention limits** | Configurable per data type; automated deletion/anonymization pipelines |

**Data Retention Schedule:**

| Data Type | Retention | Post-Retention Action |
|---|---|---|
| Transaction records | 7 years | Archive to cold storage |
| KYC verification records | 7 years (CBN requirement) | Archive |
| Audit logs | 7 years | Archive |
| Customer PII | Duration of relationship + 2 years | Anonymize |
| Device telemetry | 90 days (hot) / 1 year (cold) | Delete |
| Video footage (stores) | 30 days | Delete |
| Session replay data | 90 days | Delete |
| Payment records | 7 years (tax/compliance) | Archive |
| Support tickets | 3 years | Archive |

### 3.2 CBN Regulatory Compliance

| Requirement | Implementation |
|---|---|
| BVN/NIN for Tier-1 accounts | Mandatory KYC step in all financial service workflows |
| Electronic retrieval from NIBSS/NIMC | Direct API integration (no manual entry) |
| ICAD profiling | Automated ICAD reporting post-onboarding |
| Account restriction deadlines | Automated compliance checks; restriction workflow if non-compliant |
| No manual profile creation | KYC service enforces: all identity data must come from verified sources |
| Transaction limits per tier | Payment service enforces per-tier limits automatically |

### 3.3 NCC Compliance (SIM Registration)

| Requirement | Implementation |
|---|---|
| Compulsory SIM registration | SIM workflow enforces registration before activation |
| NIN linkage | NIN verification required in SIM registration workflow |
| Biometric capture | Biometric step included in SIM workflow |
| Agent verification | Kiosk operates as registered agent; agent ID in all transactions |
| Audit trail | All SIM registration events logged with full data lineage |

### 3.4 PCI DSS v4.0.1 Compliance

| Requirement Area | Approach |
|---|---|
| **Scope** | Minimized: card terminals connect directly to payment gateways |
| **Build/maintain secure network** | Network segmentation, firewall rules, no default credentials |
| **Protect cardholder data** | No PAN storage; tokenization only |
| **Vulnerability management** | Automated scanning, dependency updates, penetration testing |
| **Access control** | RBAC, MFA for admin access, unique IDs per user |
| **Monitor/test networks** | Logging, monitoring, annual penetration tests |
| **Information security policy** | Documented, reviewed annually, training for all staff |

---

## 4. Security Operations

### 4.1 Security Monitoring

```
┌─────────────────────────────────────────────────────────┐
│              SECURITY MONITORING STACK                    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Layer 1: Log Collection & Aggregation             │    │
│  │  - All service logs → Loki / Elasticsearch        │    │
│  │  - All device logs → MQTT → TimescaleDB           │    │
│  │  - All audit events → Audit Service               │    │
│  │  - Network flow logs → Cloud provider logging     │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Layer 2: Detection & Alerting                     │    │
│  │  - Failed auth threshold → alert (brute force)    │    │
│  │  - Unusual API patterns → alert (abuse)           │    │
│  │  - Device offline unexpectedly → alert (tamper?)  │    │
│  │  - Certificate near expiry → alert                │    │
│  │  - Unusual payment patterns → fraud engine        │    │
│  │  - Data access anomalies → alert (insider threat) │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Layer 3: Incident Response                        │    │
│  │  - Automated: block IP, revoke token, lock device │    │
│  │  - Manual: incident ticket → response team        │    │
│  │  - Escalation: PagerDuty/OpsGenie integration     │    │
│  │  - Post-incident: RCA → playbook update           │    │
│  └──────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Fraud Detection Engine

```
Fraud Engine Signals:
  - Transaction velocity (too many transactions per minute)
  - Unusual transaction amounts (vs. historical pattern)
  - Geographic anomaly (transaction at site far from usual)
  - Device anomaly (new device for known customer)
  - Basket anomaly (autonomous store: unusual item combinations)
  - Failed KYC + retry patterns
  - Multiple SIM registrations from same kiosk in short window
  - Locker: multiple failed access attempts

Fraud Engine Actions:
  - Score transactions (0.0 = safe, 1.0 = fraudulent)
  - Auto-block if score > 0.9
  - Flag for manual review if score > 0.7
  - Require step-up authentication if score > 0.5
  - Log all decisions for model improvement
```

---

## 5. Privacy by Design

### 5.1 Video & Biometric Data (Autonomous Stores)

```
Privacy Controls for Vision Systems:
  1. PURPOSE LIMITATION
     - Video used only for: basket tracking, loss prevention, dispute resolution
     - NOT for: marketing profiling, customer tracking across stores, facial recognition DB

  2. DATA MINIMIZATION
     - Process video frames in real-time on edge; do NOT stream to cloud
     - Store only: event metadata + short video clips for dispute evidence
     - Face detection used for person tracking, NOT face recognition
     - Biometric templates: stored encrypted, deleted after session

  3. RETENTION
     - Raw video: 30 days max, auto-deleted
     - Dispute evidence clips: retained only if dispute open, deleted on resolution
     - Inference results (events): 90 days

  4. ACCESS CONTROL
     - Video access: restricted to security team + dispute resolution
     - Audit logged: every video access recorded
     - Customer access: can request their session data via data subject request

  5. TRANSPARENCY
     - Signage at store entrance: "This store uses cameras for checkout"
     - Privacy notice in app during session creation
     - Opt-out: customer can request manual checkout (staffed hours only)
```

### 5.2 Consent Management

```
Consent Service:
  - Granular consent per processing purpose
  - Consent recorded with: timestamp, version, channel, evidence
  - Consent withdrawal: immediate effect on future processing
  - Consent for third-party sharing: separate, explicit
  - Consent for marketing: opt-in only
  - Under-18: parental consent workflow (if applicable)

Consent Purposes:
  - TRANSACTION_PROCESSING (contractual, not optional)
  - IDENTITY_VERIFICATION (legal requirement for KYC)
  - MARKETING_COMMUNICATIONS (opt-in)
  - ANALYTICS (legitimate interest, opt-out available)
  - VIDEO_MONITORING (legitimate interest, signage required)
  - BIOMETRIC_PROCESSING (explicit consent required)
  - THIRD_PARTY_SHARING (explicit consent required)
```
