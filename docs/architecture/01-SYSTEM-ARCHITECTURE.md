# 01 - System Architecture

## 1. Architectural Style

**Primary:** Domain-Driven Design (DDD) with Event-Driven Architecture (EDA)
**Deployment:** Hybrid Cloud + Edge
**Communication:** Synchronous (REST/gRPC) for queries & commands, Asynchronous (Events) for state propagation
**Multi-tenancy:** Logical isolation via tenant context propagation; physical isolation optional for premium tenants

---

## 2. Platform Planes

The system is organized into five architectural planes, each with distinct responsibilities, latency profiles, and scaling characteristics.

### 2.1 Experience Plane

Responsible for all user-facing interfaces. Each interface is a thin client that communicates with the platform via the API Gateway/BFF layer.

```
Experience Plane
├── Customer-Facing
│   ├── Kiosk Touchscreen Application (Android/Linux kiosk mode)
│   │   └── Embedded Chromium / Native Android app
│   │   └── Runs on edge hardware (tablet/industrial panel PC)
│   │   └── Communicates with local Edge Controller first, cloud fallback
│   │
│   ├── Locker Interface
│   │   └── Compact screen UI for PIN/QR entry
│   │   └── Or mobile-first: customer's phone via QR → web app
│   │
│   ├── Store Entry App
│   │   └── Mobile app (QR code generation for gate entry)
│   │   └── NFC/contactless card tap alternative
│   │
│   ├── Customer Mobile App (iOS/Android)
│   │   └── Order tracking, receipts, disputes, account management
│   │
│   └── Customer Web Portal
│       └── Account management, order history, support
│
└── Operator-Facing
    ├── Admin Portal (Web)
    │   └── Tenant management, fleet monitoring, configuration
    │
    ├── Merchant Portal (Web)
    │   └── Inventory, pricing, reports, settlement
    │
    ├── Field Technician App (Mobile)
    │   └── Device diagnostics, maintenance workflows, parts inventory
    │
    └── Monitoring Dashboard (Web)
        └── Real-time fleet health, alerts, SLA tracking
```

**BFF (Backend-for-Frontend) Strategy:**

Each frontend category gets a dedicated BFF that aggregates multiple backend services into optimized payloads:

| BFF | Serves | Key Aggregations |
|---|---|---|
| `bff-kiosk` | Kiosk touch UI | Workflow steps + device commands + KYC status |
| `bff-locker` | Locker screen + mobile pickup | Compartment status + access tokens + order details |
| `bff-store` | Store entry app | Session management + basket state + payment |
| `bff-customer` | Mobile app + web portal | Orders + receipts + account + disputes |
| `bff-operator` | Admin/merchant/monitoring portals | Fleet state + analytics + configuration |
| `bff-technician` | Field tech mobile app | Device diagnostics + work orders + parts |

### 2.2 API Gateway Layer

```
┌─────────────────────────────────────────────────────┐
│                    API GATEWAY                       │
│                                                     │
│  ┌─────────────┐ ┌─────────────┐ ┌──────────────┐  │
│  │   AuthN /   │ │    Rate     │ │   Request    │  │
│  │   AuthZ     │ │  Limiting   │ │   Routing    │  │
│  └─────────────┘ └─────────────┘ └──────────────┘  │
│  ┌─────────────┐ ┌─────────────┐ ┌──────────────┐  │
│  │   Tenant    │ │   Request   │ │    API       │  │
│  │  Resolution │ │  Transform  │ │  Versioning  │  │
│  └─────────────┘ └─────────────┘ └──────────────┘  │
│  ┌─────────────┐ ┌─────────────┐                   │
│  │   Circuit   │ │   Audit     │                   │
│  │   Breaker   │ │   Logging   │                   │
│  └─────────────┘ └─────────────┘                   │
└─────────────────────────────────────────────────────┘
```

**Key behaviors:**
- JWT validation with tenant context extraction
- Per-tenant and per-device rate limiting
- API versioning via URL path (`/v1/`, `/v2/`)
- Request correlation ID injection for distributed tracing
- Circuit breaker for downstream service protection
- mTLS termination for device-to-cloud connections

### 2.3 Core Platform Plane

This is the heart of the system — shared services that every channel module depends on.

```
Core Platform Services
│
├── Identity & Access Management (IAM)
│   ├── User authentication (password, OTP, biometric token)
│   ├── Device authentication (mTLS certificates, API keys)
│   ├── Role-based access control (RBAC) with tenant scoping
│   ├── OAuth2/OIDC provider for all clients
│   └── Session management (customer sessions, operator sessions, device sessions)
│
├── KYC / Verification Service
│   ├── Identity proofing workflow engine
│   ├── BVN verification adapter (→ NIBSS)
│   ├── NIN verification adapter (→ NIMC)
│   ├── Document verification (ID card OCR, liveness check)
│   ├── Biometric capture & matching adapter
│   ├── KYC tier classification (Tier 1/2/3 per CBN guidelines)
│   └── Verification result caching with TTL & audit trail
│
├── Catalog & Pricing Service
│   ├── Unified product/service catalog (physical goods, digital services, SIMs, policies)
│   ├── Hierarchical category taxonomy
│   ├── Tenant-scoped catalog views
│   ├── Dynamic pricing engine (rules-based + time-based + location-based)
│   ├── Promotions & discount engine
│   └── Bundle/package configuration
│
├── Order & Transaction Ledger
│   ├── Order lifecycle management (create → fulfill → complete/cancel)
│   ├── Immutable transaction ledger (append-only event log)
│   ├── Order composition from multiple fulfillment tasks
│   ├── Idempotent order creation (device retry safety)
│   └── Cross-channel order visibility
│
├── Payment Orchestration Service
│   ├── Payment method management (card tokenization, mobile money, bank transfer, wallet)
│   ├── Payment gateway adapter framework (pluggable: Paystack, Flutterwave, bank APIs)
│   ├── Pre-authorization & capture flow (for autonomous retail)
│   ├── Refund & adjustment processing
│   ├── Settlement engine (merchant payouts, commission splits)
│   ├── PCI DSS scope minimization (tokenization, no PAN storage)
│   └── Payment reconciliation & dispute management
│
├── Workflow Engine
│   ├── BPMN-inspired workflow definition (JSON/YAML DSL)
│   ├── Step types: user-input, system-action, approval, timer, branch, parallel
│   ├── Workflow versioning & migration
│   ├── Compensation/rollback handlers per step
│   ├── Workflow state persistence & resumability
│   └── Workflow analytics (completion rates, drop-off points, duration)
│
├── Notification Service
│   ├── Multi-channel delivery: SMS, email, push notification, in-app, webhook
│   ├── Template engine with localization (English, Yoruba, Hausa, Igbo, Pidgin)
│   ├── Delivery tracking & retry logic
│   ├── Notification preferences per customer
│   └── Provider adapter framework (pluggable: Twilio, Termii, Africa's Talking, FCM, APNS)
│
├── Audit Log Service
│   ├── Immutable, append-only audit log
│   ├── Structured event format (who, what, when, where, why, outcome)
│   ├── Compliance-grade retention (7+ years for financial transactions)
│   ├── Tamper-evident log chain (hash chaining)
│   └── Query API for compliance reporting & forensic investigation
│
├── Tenant / Merchant Management
│   ├── Tenant onboarding & configuration
│   ├── Tenant-scoped resource isolation
│   ├── White-label branding configuration (logos, colors, workflows)
│   ├── Merchant settlement accounts & commission structures
│   ├── Tenant feature flags & entitlements
│   └── Tenant hierarchy (parent org → sub-merchants → sites)
│
├── Device Registry & Management
│   ├── Device enrollment & provisioning
│   ├── Device identity (certificates, hardware fingerprint)
│   ├── Device configuration management (remote config push)
│   ├── Fleet grouping (by site, tenant, device type, firmware version)
│   ├── OTA update orchestration (TUF-signed packages)
│   ├── Device lifecycle (provisioned → active → maintenance → decommissioned)
│   └── Remote command execution (reboot, diagnostic dump, lock)
│
├── Analytics & Reporting Service
│   ├── Real-time operational dashboards (device health, transaction volume)
│   ├── Business intelligence (revenue, utilization, conversion funnels)
│   ├── Scheduled report generation (daily/weekly/monthly)
│   ├── Tenant-scoped data access
│   ├── Data warehouse ETL pipeline
│   └── KPI tracking (per-channel: uptime, completion rate, SLA adherence)
│
└── Support & Incident Management
    ├── Customer support ticket creation (from kiosk, app, or operator)
    ├── Incident auto-creation from device alerts
    ├── Escalation rules & SLA tracking
    ├── Field dispatch workflow integration
    └── Knowledge base for common resolutions
```

### 2.4 Channel Module Plane

Channel modules compose core services into specialized workflows. They own no data stores of their own — they orchestrate core services.

#### A. Service Vending Module

```
Service Vending Module
│
├── SIM Registration Workflow
│   ├── Step: Select operator (MTN, Airtel, Glo, 9mobile)
│   ├── Step: Capture customer identity (NIN scan / BVN entry)
│   ├── Step: Verify identity via KYC Service → NIMC/NIBSS
│   ├── Step: Capture biometric (fingerprint/photo for NCC compliance)
│   ├── Step: Process payment (SIM cost + activation fee)
│   ├── Step: Activate SIM via Telco API adapter
│   ├── Step: Dispense SIM card (vending motor command)
│   ├── Step: Print receipt
│   └── Step: Emit completion event → Audit + Notification
│
├── Insurance Purchase Workflow
│   ├── Step: Select product from catalog (motor, health, travel, micro)
│   ├── Step: Capture customer details / verify identity
│   ├── Step: Risk questionnaire (dynamic form from rules engine)
│   ├── Step: Premium calculation (pricing engine or underwriter API)
│   ├── Step: Process payment
│   ├── Step: Issue policy via Insurer API adapter
│   ├── Step: Print/email policy certificate
│   └── Step: Emit completion event
│
├── Banking Services Workflow
│   ├── Account opening (Tier 1: BVN/NIN required per CBN circular)
│   ├── Cash deposit (if kiosk has cash acceptor)
│   ├── Bill payment
│   ├── Account balance / mini-statement
│   └── Card issuance request
│
├── Ticketing Workflow
│   ├── Event/transport ticket selection
│   ├── Payment processing
│   ├── Ticket generation (QR code)
│   └── Print or send to mobile
│
├── Document Services Workflow
│   ├── Document upload (USB, email, cloud storage)
│   ├── Print job management
│   ├── Scan & email/save
│   └── Photocopy
│
└── Adapter Registry
    ├── Telco API adapters (MTN, Airtel, Glo, 9mobile)
    ├── Insurance API adapters (per underwriter)
    ├── Banking API adapters (per bank/fintech)
    └── Government API adapters (FRSC, NIPOST, etc.)
```

#### B. Smart Locker Module

```
Smart Locker Module
│
├── Parcel Deposit Flow (Courier → Locker)
│   ├── Courier authentication (app QR / PIN)
│   ├── Parcel scan (barcode/QR)
│   ├── Compartment assignment (size matching algorithm)
│   ├── Door open command → Device Driver
│   ├── Deposit confirmation (door close sensor)
│   ├── Notify recipient → Notification Service
│   └── Start pickup SLA timer
│
├── Parcel Pickup Flow (Customer → Locker)
│   ├── Customer authentication (QR code / OTP / PIN)
│   ├── Locate assigned compartment
│   ├── Door open command → Device Driver
│   ├── Pickup confirmation (weight sensor / door close)
│   ├── Payment processing (if pay-on-pickup)
│   └── Emit completion event → update order status
│
├── Returns / Reverse Logistics Flow
│   ├── Customer initiates return (app or at locker)
│   ├── Compartment assignment for return deposit
│   ├── Customer deposits item
│   ├── Notify merchant/courier for collection
│   └── Refund processing upon collection confirmation
│
├── Failed Pickup Escalation
│   ├── SLA expiry detection (timer event)
│   ├── Reminder notification cascade (SMS → push → call)
│   ├── Reallocation to alternative pickup or return-to-sender
│   └── Incident creation if repeated failures
│
├── Compartment Management
│   ├── Compartment inventory (sizes: S, M, L, XL, refrigerated)
│   ├── Real-time availability tracking
│   ├── Maintenance/out-of-service flagging
│   ├── Cleaning schedule integration
│   └── Utilization analytics
│
└── Locker-Specific Integrations
    ├── Logistics partner APIs (courier assignment, tracking sync)
    ├── E-commerce platform webhooks (order status sync)
    └── Pharmacy/healthcare integrations (prescription dispensing compliance)
```

#### C. Autonomous Retail Module

```
Autonomous Retail Module
│
├── Store Entry Flow
│   ├── Customer authentication at gate (QR / NFC / card tap / face)
│   ├── Payment pre-authorization (hold on card/wallet)
│   ├── Shopping session creation
│   ├── Gate open command
│   └── Session tracking initiated (camera + sensor fusion)
│
├── In-Store Session Tracking
│   ├── Computer vision pipeline (person tracking, item detection)
│   ├── Shelf sensor correlation (weight changes → item events)
│   ├── RFID correlation (if equipped)
│   ├── Virtual cart maintenance (add/remove item events)
│   ├── Multi-person disambiguation
│   └── Edge inference pipeline (real-time, <100ms latency)
│
├── Store Exit & Checkout Flow
│   ├── Exit detection (gate sensor / camera)
│   ├── Final basket reconciliation
│   ├── Confidence scoring per line item
│   ├── Payment capture (authorized amount → actual basket total)
│   ├── Receipt generation & delivery (push notification / email)
│   └── Session close event
│
├── Exception Handling
│   ├── Low-confidence item flagging for manual review
│   ├── Customer dispute workflow (app-based, with video evidence)
│   ├── Shrinkage detection & loss prevention alerts
│   ├── Multi-person session merge/split correction
│   └── Forensic replay tooling (reconstruct session from events + video)
│
├── Inventory & Replenishment
│   ├── Real-time shelf inventory from sensor fusion
│   ├── Replenishment alerts (low stock thresholds)
│   ├── Planogram compliance monitoring
│   └── Expiry date tracking
│
└── AI/ML Operations
    ├── Model registry & versioning
    ├── A/B model deployment (shadow mode → canary → full)
    ├── Continuous evaluation pipeline (accuracy, latency, drift)
    ├── Training data pipeline (anonymized video → labeled datasets)
    └── Privacy controls (face blurring, data retention limits, consent)
```

### 2.5 Edge Plane

See [04-EDGE-DEVICE-ARCHITECTURE.md](04-EDGE-DEVICE-ARCHITECTURE.md) for full detail.

```
Edge Controller (per site)
│
├── Device Driver Manager
│   ├── Driver lifecycle (load, start, stop, restart)
│   ├── Capability registry (what can this site's devices do?)
│   └── Health check orchestration
│
├── Local State Cache
│   ├── Active sessions / orders
│   ├── Catalog subset (site-relevant products/services)
│   ├── Customer authentication tokens (short-lived cache)
│   └── Compartment state (lockers) / shelf state (stores)
│
├── Offline Queue
│   ├── Transaction queue (store-and-forward when cloud unreachable)
│   ├── Telemetry buffer
│   ├── Event replay on reconnection
│   └── Conflict resolution strategy (last-write-wins for state, append for events)
│
├── Local Workflow Engine
│   ├── Subset of cloud workflow engine
│   ├── Handles critical-path steps locally during partitions
│   └── Syncs workflow state on reconnection
│
├── AI Inference Runtime (Autonomous Retail sites only)
│   ├── Model serving (TensorRT / ONNX Runtime / TFLite)
│   ├── Video pipeline (capture → preprocess → infer → postprocess)
│   ├── Model hot-swap (zero-downtime model updates)
│   └── GPU/NPU resource management
│
├── Telemetry Collector
│   ├── Device metrics aggregation
│   ├── Transaction outcome logging
│   ├── Network quality monitoring
│   └── MQTT publisher to cloud telemetry ingest
│
└── Secure Update Agent
    ├── TUF client (metadata verification, rollback protection)
    ├── Staged rollout support (canary → percentage → full)
    ├── Update scheduling (maintenance windows)
    └── Rollback capability (verified last-known-good image)
```

---

## 3. Communication Patterns

### 3.1 Synchronous (Request-Response)

| Pattern | Protocol | Use Case |
|---|---|---|
| Client → API Gateway → BFF → Services | HTTPS/REST | UI-initiated queries and commands |
| Service → Service | gRPC | Internal service calls requiring strong typing and low latency |
| Edge → Cloud | HTTPS/REST + mTLS | Edge controller syncing state with cloud services |

### 3.2 Asynchronous (Event-Driven)

| Pattern | Protocol | Use Case |
|---|---|---|
| Service → Event Bus → Services | CloudEvents over NATS/Kafka | Domain events (order.created, payment.authorized, kyc.verified) |
| Device → Edge → Cloud | MQTT 5.0 | Device telemetry, health heartbeats, sensor events |
| Cloud → Edge | MQTT 5.0 | Remote commands, config updates, OTA triggers |
| Event Bus → Audit Log | CloudEvents | Every domain event is persisted for audit trail |

### 3.3 Event Envelope (CloudEvents Standard)

```json
{
  "specversion": "1.0",
  "id": "evt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "source": "ivm://services/payment-orchestrator",
  "type": "ivm.payment.authorized",
  "datacontenttype": "application/json",
  "time": "2026-03-12T14:30:00.000Z",
  "subject": "order/ord_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_acme_bank",
  "traceid": "trace-abc-123",
  "data": {
    "orderId": "ord_01HX7V8K9M2N3P4Q5R6S7T8U9V",
    "amount": 15000,
    "currency": "NGN",
    "paymentMethod": "card_tokenized",
    "gatewayRef": "pay_xyz_789"
  }
}
```

### 3.4 MQTT Topic Structure

```
ivm/{tenant_id}/{site_id}/{device_id}/telemetry/health
ivm/{tenant_id}/{site_id}/{device_id}/telemetry/metrics
ivm/{tenant_id}/{site_id}/{device_id}/events/{event_type}
ivm/{tenant_id}/{site_id}/{device_id}/commands/{command_id}
ivm/{tenant_id}/{site_id}/{device_id}/commands/{command_id}/ack
ivm/fleet/{tenant_id}/alerts
ivm/fleet/{tenant_id}/config-updates
```

---

## 4. Data Architecture Overview

### 4.1 Storage Strategy

| Data Category | Storage | Rationale |
|---|---|---|
| Transactional data (orders, payments, customers) | PostgreSQL (per-service schemas) | ACID guarantees, relational integrity |
| Event log / audit trail | Apache Kafka (retention) + PostgreSQL (queryable) | Immutable append-only, high throughput |
| Device state & telemetry (hot) | TimescaleDB (time-series extension on PostgreSQL) | Efficient time-series queries for dashboards |
| Session state (edge) | SQLite (edge) + Redis (cloud) | Lightweight edge storage, fast cloud lookups |
| Catalog & configuration | PostgreSQL + Redis cache | Relational structure with fast read cache |
| AI model artifacts | Object storage (S3-compatible) | Large binary artifacts with versioning |
| Video / media | Object storage (S3-compatible) with lifecycle policies | Retention-managed, privacy-compliant |
| Search & analytics | OpenSearch / Elasticsearch | Full-text search, log analytics, dashboards |
| Data warehouse | ClickHouse or BigQuery | OLAP workloads, business intelligence |

### 4.2 Data Flow

```
Devices → MQTT → Edge Controller → Event Buffer → Cloud Event Bus
                                                        │
                    ┌───────────────┬───────────────────┤
                    ▼               ▼                   ▼
              Service DBs     Event Store        Telemetry DB
              (PostgreSQL)    (Kafka→PG)        (TimescaleDB)
                    │               │                   │
                    └───────────────┴───────────────────┘
                                    │
                                    ▼
                            Data Warehouse
                            (ClickHouse)
                                    │
                                    ▼
                            Analytics & BI
                            (Dashboards)
```

---

## 5. Deployment Topology

### 5.1 Cloud Infrastructure

```
Cloud Region (Primary: Lagos / West Africa)
├── Kubernetes Cluster (managed: EKS, GKE, or AKS)
│   ├── Namespace: ivm-core (platform services)
│   ├── Namespace: ivm-channels (channel modules)
│   ├── Namespace: ivm-bff (backend-for-frontend)
│   ├── Namespace: ivm-gateway (API gateway, ingress)
│   ├── Namespace: ivm-data (databases, caches)
│   ├── Namespace: ivm-messaging (Kafka, MQTT broker)
│   ├── Namespace: ivm-monitoring (Prometheus, Grafana, Loki)
│   └── Namespace: ivm-jobs (batch processing, ETL, reports)
│
├── Managed Services
│   ├── PostgreSQL (RDS/Cloud SQL) — primary transactional DB
│   ├── Redis Cluster — caching, session store
│   ├── Kafka (managed) — event streaming
│   ├── MQTT Broker (EMQX/HiveMQ) — device messaging
│   ├── Object Storage (S3) — media, models, backups
│   ├── Container Registry — Docker images
│   └── Secrets Manager — credentials, certificates, API keys
│
└── DR Region (Secondary)
    ├── Read replicas of primary databases
    ├── Event stream replication
    └── Warm standby Kubernetes cluster
```

### 5.2 Edge Site Deployment

```
Per-Site Edge Hardware
├── Edge Controller (Industrial PC / Raspberry Pi CM4 / NVIDIA Jetson)
│   ├── OS: Ubuntu Core / Balena OS / custom Linux
│   ├── Container Runtime: Docker / containerd
│   ├── Services:
│   │   ├── Edge Gateway (local API)
│   │   ├── Device Driver Manager
│   │   ├── State Cache (SQLite + in-memory)
│   │   ├── Telemetry Collector
│   │   ├── Offline Queue
│   │   └── [AI Inference Runtime — stores only]
│   │
│   ├── Connectivity:
│   │   ├── Primary: Ethernet / Wi-Fi
│   │   ├── Failover: 4G/5G cellular modem
│   │   └── Cloud VPN tunnel (WireGuard)
│   │
│   └── Security:
│       ├── TPM 2.0 for device identity
│       ├── Encrypted local storage
│       ├── Secure boot chain
│       └── mTLS certificates for cloud communication
│
└── Connected Devices (site-specific)
    ├── Kiosk: touchscreen, scanner, printer, card reader, biometric, dispenser
    ├── Locker: door controllers, lock actuators, sensors, display
    └── Store: camera array, shelf sensors, gate controllers, weight sensors
```

### 5.3 Network Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLOUD                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   API    │  │   MQTT   │  │  Admin   │  │   CDN    │       │
│  │ Gateway  │  │  Broker  │  │  Portal  │  │ (Static) │       │
│  │ :443     │  │ :8883    │  │  :443    │  │  :443    │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │              │             │              │              │
│  ─────┼──────────────┼─────────────┼──────────────┼─────────────│
│       │        TLS + mTLS         │              │              │
└───────┼──────────────┼─────────────┼──────────────┼─────────────┘
        │              │             │              │
   ┌────┼──────────────┼─────────────┼──────────────┼────┐
   │    │    WireGuard VPN Tunnel    │              │    │
   │    ▼              ▼             ▼              ▼    │
   │  ┌──────────────────────────────────────────────┐  │
   │  │            EDGE CONTROLLER                   │  │
   │  │  ┌────────┐  ┌─────────┐  ┌──────────────┐  │  │
   │  │  │Local   │  │  MQTT   │  │    Device     │  │  │
   │  │  │API     │  │  Client │  │   Drivers     │  │  │
   │  │  └───┬────┘  └─────────┘  └──────┬───────┘  │  │
   │  └──────┼────────────────────────────┼──────────┘  │
   │         │       Local Network        │             │
   │         ▼       (Ethernet/USB)       ▼             │
   │  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
   │  │ Display  │  │ Scanner  │  │  Printer │  ...   │
   │  └──────────┘  └──────────┘  └──────────┘        │
   │                    SITE                            │
   └────────────────────────────────────────────────────┘
```

---

## 6. Scalability & Resilience

### 6.1 Scaling Strategy

| Component | Scaling Model | Trigger |
|---|---|---|
| API Gateway | Horizontal (pod autoscaling) | Request rate, latency P99 |
| Core Services | Horizontal (stateless pods) | CPU/memory, queue depth |
| Event Bus (Kafka) | Partition scaling | Topic throughput |
| MQTT Broker | Cluster scaling | Connected device count |
| Databases | Vertical + read replicas | Query load, storage |
| Edge Controllers | Per-site (fixed) | N/A — one per site |
| AI Inference | GPU scaling (stores only) | Camera count, FPS requirements |

### 6.2 Resilience Patterns

| Pattern | Implementation | Scope |
|---|---|---|
| **Circuit Breaker** | Per-service, with fallback responses | All inter-service calls |
| **Retry with Backoff** | Exponential backoff + jitter | External API calls, event publishing |
| **Bulkhead** | Thread pool isolation per dependency | Payment gateway, KYC providers |
| **Saga / Compensation** | Orchestrated sagas for multi-step workflows | Order creation, SIM registration |
| **Idempotency** | Idempotency keys on all mutating operations | All commands from devices/clients |
| **Event Sourcing** | Append-only event log for critical domains | Orders, payments, audit |
| **CQRS** | Separate read/write models where beneficial | Catalog reads, analytics queries |
| **Offline Queue** | Store-and-forward at edge | All edge-to-cloud communication |
| **Health Checks** | Liveness + readiness probes | All services and devices |
| **Graceful Degradation** | Feature flags + fallback modes | KYC provider down → queue for retry |

### 6.3 Availability Targets

| Component | Target | Justification |
|---|---|---|
| Cloud Platform Services | 99.9% | Standard SaaS availability |
| API Gateway | 99.95% | Customer-facing entry point |
| Payment Processing | 99.95% | Revenue-critical path |
| Edge Controller (per-site) | 99.5% online, 100% operational* | *Offline mode ensures operation continues |
| MQTT Broker | 99.9% | Device connectivity |
| Device Uptime | 98% | Physical hardware has maintenance needs |

---

## 7. Cross-Cutting Concerns

### 7.1 Observability Stack

```
┌─────────────────────────────────────────────┐
│             OBSERVABILITY                    │
│                                             │
│  Metrics    → Prometheus → Grafana          │
│  Logs       → Structured JSON → Loki        │
│  Traces     → OpenTelemetry → Tempo/Jaeger  │
│  Alerts     → Alertmanager → PagerDuty/OpsGenie │
│  Device     → MQTT Telemetry → TimescaleDB → Grafana │
│  Business   → Event Stream → ClickHouse → Custom Dashboards │
└─────────────────────────────────────────────┘
```

### 7.2 Tenant Context Propagation

Every request carries a `TenantContext` through the entire call chain:

```
TenantContext {
  tenantId: string        // "tnt_acme_bank"
  merchantId?: string     // "mrc_branch_lagos_001"
  siteId?: string         // "site_victoria_island_01"
  deviceId?: string       // "dev_kiosk_vi_001"
  userId?: string         // "usr_customer_abc"
  operatorId?: string     // "opr_admin_xyz"
  permissions: string[]   // ["orders:read", "payments:write", ...]
  traceId: string         // distributed trace correlation
}
```

This context is:
- Injected by the API Gateway from JWT claims
- Propagated via gRPC metadata / HTTP headers
- Used for database query scoping (row-level security)
- Included in every event envelope
- Used for rate limiting and billing

### 7.3 Configuration Management

| Level | Mechanism | Example |
|---|---|---|
| Platform defaults | Environment config + config maps | Service timeouts, retry counts |
| Tenant overrides | Tenant config service (DB-backed) | Branding, enabled features, workflow variants |
| Site overrides | Site config (pushed to edge) | Device layout, local catalog, operating hours |
| Device overrides | Device config (pushed via MQTT) | Firmware version, driver parameters |
| Feature flags | Feature flag service (LaunchDarkly-style) | Gradual rollout of new workflows |
