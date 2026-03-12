# 08 - Implementation Roadmap

## 1. Phased Delivery Strategy

```
TIMELINE OVERVIEW (18-24 months to full platform)

Month:  1  2  3  4  5  6  7  8  9  10  11  12  13  14  15  16  17  18
        ├──────────────────┤
        Phase 1: Foundation
                           ├──────────────────┤
                           Phase 2: Service Vending
                                              ├──────────────────┤
                                              Phase 3: Smart Lockers
                                                                 ├──────────────┤
                                                                 Phase 4: Auto Store (R&D)

        ├─────────────────────────────────────────────────────────────────────────┤
        Continuous: Security, Compliance, DevOps, Edge Platform
```

---

## 2. Phase 1: Platform Foundation (Months 1-6)

**Goal:** Build the minimum complete platform that every channel module will depend on.

### 2.1 Sprint Breakdown

**Sprint 1-2 (Month 1): Project Bootstrap & Core Infrastructure**

```
Deliverables:
├── Monorepo setup (Turborepo/Nx)
├── CI/CD pipeline (GitHub Actions → container build → staging deploy)
├── Kubernetes cluster provisioned (staging)
├── PostgreSQL, Redis, Kafka deployed (staging)
├── Shared packages initialized:
│   ├── shared-types (ULID, Money, enums)
│   ├── shared-utils (ID generation, date handling, crypto)
│   └── event-contracts (base CloudEvents schema)
├── NestJS service template (logging, tracing, health checks, error handling)
├── API Gateway (Kong) deployed with basic routing
├── Development environment (docker-compose.dev.yml)
└── Architecture Decision Records (ADR) started
```

**Sprint 3-4 (Month 2): Identity, Tenant & Auth**

```
Deliverables:
├── Identity Service
│   ├── User registration (phone + OTP)
│   ├── User authentication (phone + OTP, email + password)
│   ├── JWT issuance & validation
│   ├── Session management
│   ├── Device authentication (mTLS skeleton)
│   └── RBAC engine (roles, permissions, tenant scoping)
├── Tenant Service
│   ├── Tenant CRUD
│   ├── Branding configuration
│   ├── Feature flags
│   └── Tenant hierarchy (parent → child)
├── API Gateway integration (JWT validation, tenant resolution)
├── Row-Level Security (RLS) implementation on PostgreSQL
└── Admin Portal skeleton (Next.js + auth flow)
```

**Sprint 5-6 (Month 3): KYC, Catalog & Payment**

```
Deliverables:
├── KYC Service
│   ├── BVN verification adapter (NIBSS integration)
│   ├── NIN verification adapter (NIMC integration)
│   ├── KYC tier management (Tier 0/1/2/3)
│   ├── Verification result storage (encrypted)
│   └── Consent management
├── Catalog Service
│   ├── Product/service catalog CRUD
│   ├── Category management
│   ├── Tenant-scoped catalog views
│   └── Channel availability filtering
├── Pricing Service
│   ├── Base price management
│   ├── Promotion engine (simple rules)
│   └── Price calculation API
├── Payment Orchestration Service
│   ├── Paystack adapter (card, transfer)
│   ├── Payment initiation & authorization
│   ├── Webhook handling
│   ├── Tokenized card storage
│   └── Refund processing
└── Integration tests (KYC → NIBSS stub, Payment → Paystack sandbox)
```

**Sprint 7-8 (Month 4): Orders, Workflow & Notifications**

```
Deliverables:
├── Order Service
│   ├── Order creation & lifecycle
│   ├── Order line items
│   ├── Fulfillment task management
│   ├── Idempotent order creation
│   └── Order status state machine
├── Workflow Engine
│   ├── Workflow definition DSL (JSON/YAML)
│   ├── Step execution runtime
│   ├── Workflow state persistence
│   ├── Timer/timeout handling
│   └── Compensation/rollback
├── Notification Service
│   ├── SMS adapter (Termii / Africa's Talking)
│   ├── Email adapter (SendGrid / SES)
│   ├── Push notification (FCM)
│   ├── Template engine
│   └── Delivery tracking
└── Event bus integration (all services emit CloudEvents to Kafka)
```

**Sprint 9-10 (Month 5): Audit, Analytics & Device Management**

```
Deliverables:
├── Audit Service
│   ├── Immutable audit log (append-only)
│   ├── Hash chain integrity
│   ├── Query API
│   └── All services wired to emit audit events
├── Analytics Service
│   ├── Data pipeline (Kafka → ClickHouse)
│   ├── Real-time dashboards (Grafana)
│   ├── Transaction volume metrics
│   └── Basic reporting API
├── Device Registry Service
│   ├── Device enrollment API
│   ├── Device configuration management
│   ├── Device health tracking
│   ├── Fleet listing & filtering
│   └── Remote command framework
├── MQTT Broker deployment (EMQX)
├── Device telemetry ingestion (MQTT → TimescaleDB)
└── Monitoring dashboards (device health, service health)
```

**Sprint 11-12 (Month 6): Edge Controller v1 & Integration Testing**

```
Deliverables:
├── Edge Controller (Go) v1
│   ├── Local REST API (gateway)
│   ├── Device driver manager (framework + 2 drivers: scanner, printer)
│   ├── State cache (SQLite)
│   ├── Offline queue (store-and-forward)
│   ├── Telemetry collector (MQTT)
│   ├── Cloud sync engine
│   ├── VPN client (WireGuard)
│   └── Update agent (TUF) skeleton
├── BFF-Kiosk (first BFF)
│   ├── Auth flow aggregation
│   ├── Catalog fetching
│   ├── Order + payment flow
│   └── Device command proxy
├── End-to-end integration testing
│   ├── Full customer journey: register → KYC → browse → order → pay
│   ├── Edge ↔ cloud communication
│   └── Offline mode basic testing
├── Production Kubernetes cluster provisioned
├── Security audit of Phase 1 deliverables
└── Load testing & performance baseline
```

### 2.2 Phase 1 Exit Criteria

- [ ] User can register, authenticate (OTP), and be KYC-verified (BVN/NIN)
- [ ] Catalog items can be created and queried per tenant
- [ ] Orders can be placed and paid for (Paystack sandbox)
- [ ] All events flow through Kafka, audit trail is complete
- [ ] Edge controller connects, syncs state, survives network partition
- [ ] Admin portal: tenant management, device fleet view
- [ ] CI/CD pipeline deploys to staging automatically
- [ ] >80% test coverage on core services
- [ ] Security scan clean (no critical/high vulnerabilities)

---

## 3. Phase 2: Service Vending (Months 5-9, overlapping Phase 1)

**Goal:** Deploy the first revenue-generating channel — self-service kiosks.

### 3.1 Deliverables

```
Month 5-6 (parallel with Phase 1 tail):
├── Vending Module Service
│   ├── SIM registration workflow
│   │   ├── Telco adapter: MTN (first integration)
│   │   ├── Identity capture + KYC verification flow
│   │   ├── Biometric capture step
│   │   ├── SIM activation step
│   │   ├── Dispensing command (motor driver)
│   │   └── Receipt printing step
│   ├── Insurance purchase workflow (1 underwriter)
│   └── Bill payment workflow (basic)
├── Device Drivers (Go)
│   ├── Card reader driver (PAX)
│   ├── Biometric scanner driver (Suprema)
│   ├── Vending motor driver (GPIO)
│   └── Driver hot-plug support
└── Kiosk UI (React) v1
    ├── Welcome / language selection
    ├── Service selection menu
    ├── Customer identification flow
    ├── Service-specific workflow screens
    ├── Payment screen (card reader integration)
    └── Receipt / completion screen

Month 7-8:
├── Additional telco adapters (Airtel, Glo, 9mobile)
├── Additional insurance underwriter adapters (2-3)
├── Banking services workflow (account opening, balance inquiry)
├── Kiosk UI polish & user testing
├── Field Technician App v1 (React Native)
│   ├── Device diagnostics
│   ├── Maintenance checklist
│   └── QR-based device identification
├── Merchant Portal v1 (Next.js)
│   ├── Transaction monitoring
│   ├── Revenue reports
│   └── Catalog management
└── OTA update system (TUF) operational

Month 9:
├── Pilot deployment: 5-10 kiosks in controlled locations
│   ├── Bank branches (banking + SIM services)
│   ├── Telco partner offices
│   └── Campus locations
├── Operational monitoring & alerting tuned
├── Customer support tooling (ticket creation from kiosk)
├── Settlement engine (daily merchant payouts)
└── NCC compliance verification (SIM registration audit)
```

### 3.2 Phase 2 Exit Criteria

- [ ] Kiosk can complete: SIM registration, insurance purchase, banking service
- [ ] KYC flow completes with live NIBSS/NIMC APIs
- [ ] Payment processing live with real card transactions
- [ ] OTA updates deployed to pilot devices successfully
- [ ] Pilot kiosks operational for 30 days with <2% failure rate
- [ ] Merchant can view transactions and reports
- [ ] Field technician can diagnose and maintain devices via app
- [ ] NCC compliance requirements met for SIM registration

### 3.3 Phase 2 KPIs

| KPI | Target | Measurement |
|---|---|---|
| KYC completion rate | >85% | Successful verifications / total attempts |
| Transaction success rate | >95% | Completed orders / initiated orders |
| Device uptime | >98% | Healthy heartbeats / expected heartbeats |
| Average transaction time | <5 min (SIM reg) | Workflow start → complete |
| Payment authorization rate | >90% | Authorized / attempted |
| OTA update success rate | 100% | Successful updates / attempted |

---

## 4. Phase 3: Smart Lockers (Months 9-14)

**Goal:** Add parcel locker capability using the existing platform core.

### 4.1 Deliverables

```
Month 9-10:
├── Locker Module Service
│   ├── Compartment management
│   ├── Reservation engine (size matching, assignment)
│   ├── Deposit flow (courier authentication, door control)
│   ├── Pickup flow (customer authentication, door control)
│   ├── Pickup SLA tracking & reminders
│   └── Failed pickup escalation
├── Lock Controller Driver (Go)
│   ├── Electronic lock control (serial/GPIO)
│   ├── Door sensor monitoring
│   ├── Weight sensor integration (optional)
│   └── Multi-compartment management
├── Locker UI (React)
│   ├── Customer pickup screen (PIN / QR entry)
│   ├── Courier deposit screen
│   ├── Compartment status display
│   └── Help / support screen
└── BFF-Locker

Month 11-12:
├── Returns / reverse logistics flow
├── Pay-on-pickup workflow (locker + payment integration)
├── Courier integration adapters (GIG Logistics, 1-2 others)
├── E-commerce webhook integration (order status sync)
├── Customer Mobile App v1 (React Native)
│   ├── Pickup notification → QR code display
│   ├── Order tracking
│   ├── Locker bank locator (map)
│   └── Dispute / support ticket
├── Compartment utilization analytics
└── Locker bank monitoring dashboard

Month 13-14:
├── Pilot deployment: 3-5 locker banks
│   ├── Shopping mall (retail partnership)
│   ├── Corporate campus (employee parcels)
│   └── Residential estate (last-mile delivery)
├── Courier onboarding & training
├── Notification optimization (reminder cadence tuning)
├── Exception handling refinement
└── Pharmacutical dispensing exploration (compliance research)
```

### 4.2 Phase 3 Exit Criteria

- [ ] Full deposit → notify → pickup flow operational
- [ ] Returns flow operational
- [ ] Courier app integration working with 1+ logistics partner
- [ ] Customer app shows pickup notifications with QR codes
- [ ] Compartment utilization tracked and displayed
- [ ] Failed pickup escalation working (SLA timers, reminders)
- [ ] Pilot locker banks operational for 30 days

### 4.3 Phase 3 KPIs

| KPI | Target | Measurement |
|---|---|---|
| Pickup success rate | >90% | Picked up before SLA / total deposits |
| Average pickup time | <24 hours | Deposit → pickup |
| Failed delivery rate | 0% | (vs. door-to-door benchmark) |
| Courier deposit time | <2 min | Scan → deposit → confirm |
| Customer pickup time | <1 min | Auth → door open → close |
| Compartment utilization | >50% | Occupied hours / available hours |

---

## 5. Phase 4: Autonomous Retail (Months 14-18+)

**Goal:** R&D-intensive program to add cashierless store capability.

### 5.1 Deliverables

```
Month 14-15 (R&D):
├── AI/ML Foundation
│   ├── Training data collection pipeline
│   ├── Product image dataset creation
│   ├── Person detection & tracking model (baseline)
│   ├── Item detection model (top 50 products)
│   └── Action recognition model (pick / put-back)
├── AI Inference Runtime (Python + C++)
│   ├── Video pipeline (GStreamer + camera ingestion)
│   ├── Model serving (TensorRT / ONNX)
│   ├── Multi-camera synchronization
│   └── Sensor fusion engine (vision + weight)
├── Edge hardware procurement & testing (Jetson Orin)
└── Store layout simulation environment

Month 16-17 (Prototype):
├── Store Module Service
│   ├── Shopping session management
│   ├── Entry authorization + gate control
│   ├── Basket reconciliation engine
│   ├── Exit detection + checkout trigger
│   ├── Pre-authorization + capture flow
│   └── Dispute resolution workflow
├── Store Entry App (React + React Native)
│   ├── QR code generation for entry
│   ├── Session tracking (live basket view)
│   ├── Receipt display
│   └── Dispute flow
├── Forensic replay tooling
│   ├── Session reconstruction from events + video
│   ├── Human review interface
│   └── Model accuracy reporting
├── Model monitoring & continuous evaluation
├── Privacy controls implementation
│   ├── Face blurring in stored video
│   ├── Data retention automation
│   └── NDPA compliance verification
└── Lab store (controlled environment for testing)

Month 18+ (Pilot):
├── Pilot deployment: 1 test store
│   ├── Small format (20-50 SKUs)
│   ├── Staffed during business hours (hybrid model)
│   ├── Unstaffed after hours (full autonomous)
│   └── 8-week evaluation period
├── Model iteration based on real-world data
├── Customer experience optimization
├── Shrinkage & accuracy analysis
├── Unit economics validation
└── Scale-up plan based on pilot results
```

### 5.2 Phase 4 KPIs

| KPI | Target (Pilot) | Target (Scale) |
|---|---|---|
| Basket accuracy | >95% | >99% |
| Receipt latency | <30 seconds | <10 seconds |
| Dispute rate | <5% | <1% |
| Shrinkage rate | <3% | <1% |
| Entry → exit time | N/A (customer-driven) | N/A |
| Inference latency (P99) | <200ms | <100ms |
| Customer NPS | >40 | >60 |

---

## 6. Team Structure

### 6.1 Recommended Team (Phase 1-2)

```
Engineering Team (Phase 1): ~12-15 people
│
├── Platform Team (5-6)
│   ├── 1 Tech Lead / Architect
│   ├── 2 Backend Engineers (NestJS: identity, KYC, payments)
│   ├── 1 Backend Engineer (NestJS: orders, workflow, notifications)
│   └── 1 DevOps / Infrastructure Engineer
│
├── Edge & Device Team (3-4)
│   ├── 1 Edge Lead (Go)
│   ├── 1 Embedded/Driver Engineer (Go + hardware)
│   ├── 1 Edge Backend Engineer (Go: gateway, sync, queue)
│   └── 1 Hardware Integration Specialist (part-time)
│
├── Frontend Team (3)
│   ├── 1 Frontend Lead (React + React Native)
│   ├── 1 Kiosk UI Engineer (React)
│   └── 1 Portal/App Engineer (Next.js + React Native)
│
├── QA (1-2)
│   ├── 1 QA Engineer (E2E, integration)
│   └── 1 QA Engineer (device testing, field testing)
│
└── Cross-functional
    ├── Product Manager (1)
    ├── UX Designer (1)
    └── Security Engineer (0.5, shared or consultant)
```

### 6.2 Scaling Team (Phase 3-4)

```
Additional Hires for Phase 3 (Lockers):
├── 1 Backend Engineer (locker module)
├── 1 Integration Engineer (logistics partners)
└── 1 Mobile Engineer (customer app)

Additional Hires for Phase 4 (Autonomous Retail):
├── 2 ML Engineers (computer vision)
├── 1 ML Ops Engineer
├── 1 Edge AI Engineer (TensorRT, optimization)
└── 1 Data Engineer (training pipelines)

Total at Scale: ~22-25 people
```

---

## 7. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| NIBSS/NIMC API instability | High | High | Circuit breaker, retry queues, fallback to TIER_0 access |
| Payment gateway outages | Medium | High | Multi-gateway routing, offline card processing |
| Edge hardware failures in field | High | Medium | Watchdog, remote diagnostics, spare parts inventory |
| Regulatory changes (CBN/NCC/NDPC) | Medium | High | Compliance monitoring, policy engine (configurable rules) |
| Poor network connectivity at sites | High | Medium | Offline-first design, 4G failover, local state cache |
| Talent attrition (key engineers) | Medium | High | Documentation, code reviews, knowledge sharing sessions |
| Scope creep across phases | High | Medium | Strict phase gates, phased tenanting |
| Hardware vendor lock-in | Medium | Medium | Device abstraction layer, multi-vendor testing |
| AI model accuracy insufficient | Medium (Phase 4) | High | Phased approach, hybrid staffed model, continuous eval |
| Data breach / security incident | Low | Critical | Security-first design, penetration testing, incident response plan |

---

## 8. Success Metrics (Platform Level)

| Metric | Phase 1 Target | Phase 2 Target | Phase 3 Target |
|---|---|---|---|
| Platform uptime | 99.5% | 99.9% | 99.9% |
| API latency P99 | <500ms | <300ms | <300ms |
| Deployment frequency | Weekly | 2x/week | Daily |
| Mean time to recovery | <2 hours | <1 hour | <30 min |
| Test coverage | >80% | >80% | >80% |
| Security vulnerabilities (critical) | 0 | 0 | 0 |
| Active devices managed | 0 | 10-50 | 50-200 |
| Monthly transactions | 0 | 5,000-50,000 | 50,000-500,000 |
| Active tenants | 1 (internal) | 3-5 | 10-20 |
