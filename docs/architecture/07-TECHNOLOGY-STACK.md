# 07 - Technology Stack & Infrastructure

## 1. Technology Selection Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│                      IVM TECHNOLOGY STACK                             │
│                                                                      │
│  FRONTEND / EXPERIENCE                                               │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Kiosk UI:      React + TypeScript (Electron or Android WebView) │
│  │  Locker UI:     React + TypeScript (compact, responsive)     │    │
│  │  Store Entry:   React Native (mobile) + React (gate screen)  │    │
│  │  Customer App:  React Native (iOS + Android)                 │    │
│  │  Web Portal:    Next.js (SSR for SEO, React for SPA)         │    │
│  │  Admin Portals: Next.js + React                              │    │
│  │  Field Tech App: React Native                                │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  BACKEND / SERVICES                                                  │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Language:      TypeScript (Node.js) — primary               │    │
│  │                 Python — AI/ML services only                  │    │
│  │                 Go — edge controller (performance-critical)   │    │
│  │  Framework:     NestJS (Node.js services)                    │    │
│  │                 FastAPI (Python AI services)                  │    │
│  │  API:           REST (external) + gRPC (internal)            │    │
│  │  ORM:           Prisma (TypeScript) + SQLAlchemy (Python)    │    │
│  │  Validation:    Zod (TypeScript) + Pydantic (Python)         │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  EDGE / DEVICE                                                       │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Edge Controller:  Go (lightweight, cross-compilable)        │    │
│  │  Device Drivers:   Go + C/C++ (hardware interfaces)          │    │
│  │  AI Inference:     Python + C++ (TensorRT / ONNX Runtime)    │    │
│  │  Video Pipeline:   GStreamer + DeepStream (NVIDIA)            │    │
│  │  Container:        Docker + docker-compose                    │    │
│  │  Edge OS:          Ubuntu Core 24 / Balena OS                │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  DATA                                                                │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Primary DB:     PostgreSQL 16 (transactional)               │    │
│  │  Time-Series:    TimescaleDB (telemetry, device metrics)     │    │
│  │  Cache:          Redis 7 (session, catalog cache)            │    │
│  │  Search:         OpenSearch (logs, full-text search)         │    │
│  │  Event Stream:   Apache Kafka (event bus, audit stream)      │    │
│  │  Object Store:   S3-compatible (MinIO self-hosted or AWS S3) │    │
│  │  Data Warehouse: ClickHouse (OLAP, analytics)                │    │
│  │  Edge DB:        SQLite (offline state, queue)               │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  MESSAGING                                                           │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Event Bus:      Apache Kafka (CloudEvents envelope)         │    │
│  │  Device MQTT:    EMQX (MQTT 5.0 broker, clustered)           │    │
│  │  Task Queue:     BullMQ (Redis-backed, for async jobs)       │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  INFRASTRUCTURE                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Container Orch: Kubernetes (managed: EKS/GKE/AKS)          │    │
│  │  Service Mesh:   Istio or Linkerd (mTLS, observability)      │    │
│  │  API Gateway:    Kong or AWS API Gateway                     │    │
│  │  VPN:            WireGuard (edge-to-cloud tunnels)           │    │
│  │  CI/CD:          GitHub Actions + ArgoCD (GitOps)            │    │
│  │  IaC:            Terraform + Helm charts                     │    │
│  │  Secrets:        HashiCorp Vault                             │    │
│  │  Registry:       GitHub Container Registry / ECR             │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  OBSERVABILITY                                                       │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Metrics:        Prometheus + Grafana                        │    │
│  │  Logs:           Loki (or OpenSearch)                        │    │
│  │  Traces:         OpenTelemetry → Tempo (or Jaeger)           │    │
│  │  Alerts:         Alertmanager → PagerDuty/OpsGenie           │    │
│  │  Device Monitor: Custom dashboards (Grafana + TimescaleDB)   │    │
│  │  Uptime:         Synthetic monitoring (Grafana Cloud)        │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  AI/ML                                                               │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Training:       PyTorch + CUDA                              │    │
│  │  Experiment:     MLflow (experiment tracking)                │    │
│  │  Inference:      TensorRT (NVIDIA edge) / ONNX Runtime       │    │
│  │  Model Registry: MLflow Model Registry                       │    │
│  │  Data Pipeline:  Apache Airflow (ETL, training pipelines)    │    │
│  │  Annotation:     CVAT (Computer Vision Annotation Tool)      │    │
│  └──────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. Technology Selection Rationale

### 2.1 Backend: TypeScript + NestJS (Primary)

**Why TypeScript:**
- Unified language with frontend (React/React Native) — reduced context switching
- Strong type system catches integration errors at compile time
- Excellent ecosystem for API development (Prisma, Zod, etc.)
- Large talent pool in Nigeria and globally
- Good performance for I/O-bound workloads (which most platform services are)

**Why NestJS:**
- Opinionated, modular architecture aligns with DDD/microservices
- First-class support for REST, gRPC, WebSockets, MQTT, Kafka
- Dependency injection for clean testability
- Built-in guards, interceptors, pipes for cross-cutting concerns
- Module system maps naturally to bounded contexts

**Why Go for Edge Controller:**
- Single binary deployment — no runtime dependencies on edge hardware
- Excellent cross-compilation (ARM64 for Raspberry Pi/Jetson)
- Low memory footprint (critical for constrained edge devices)
- Superior concurrency model (goroutines) for handling multiple device drivers
- Fast startup time (critical for edge reliability)

**Why Python for AI/ML:**
- De facto standard for ML/CV (PyTorch, TensorRT, OpenCV)
- FastAPI for serving ML models with async support
- Not used for platform services — only for AI-specific workloads

### 2.2 Frontend: React + React Native

**Why React everywhere:**
- One component model across web, kiosk, and embedded UIs
- React Native for mobile (iOS + Android from single codebase)
- Electron/WebView for kiosk (full control of hardware chrome)
- Next.js for web portals (SSR, routing, API routes)
- Shared component library across all UIs

### 2.3 Database: PostgreSQL

**Why PostgreSQL:**
- Battle-tested ACID compliance for financial transactions
- Row-Level Security (RLS) for multi-tenant isolation
- JSON/JSONB for flexible schema where needed (workflow context, device config)
- TimescaleDB extension for time-series (same engine, no new DB to manage)
- Excellent tooling and Nigeria hosting availability
- Supports full-text search (reduce need for separate search engine initially)

### 2.4 Event Streaming: Apache Kafka

**Why Kafka:**
- Durable, ordered event log — perfect for event sourcing
- High throughput for device telemetry ingestion
- Consumer groups for scaling event processing
- Long retention for audit compliance (configure 7+ year retention for audit topics)
- Kafka Connect for data pipeline integration
- CloudEvents serialization support

### 2.5 MQTT: EMQX

**Why EMQX:**
- MQTT 5.0 support (shared subscriptions, request/response, properties)
- Clusterable for high availability
- Built-in authentication (JWT, certificate)
- Rule engine for message routing (MQTT → Kafka bridge)
- Excellent performance at scale (millions of connections)
- Open source with commercial support

---

## 3. Service Architecture Map

```
┌─────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                             │
│                                                                  │
│  Namespace: ivm-gateway                                          │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Kong API Gateway (2+ replicas, auto-scaled)             │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Namespace: ivm-bff                                              │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │bff-kiosk   │ │bff-locker  │ │bff-customer│ │bff-operator│   │
│  │ NestJS     │ │ NestJS     │ │ NestJS     │ │ NestJS     │   │
│  │ 2 replicas │ │ 2 replicas │ │ 3 replicas │ │ 2 replicas │   │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘   │
│                                                                  │
│  Namespace: ivm-core                                             │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │identity-svc│ │kyc-svc     │ │catalog-svc │ │order-svc   │   │
│  │ NestJS     │ │ NestJS     │ │ NestJS     │ │ NestJS     │   │
│  │ 3 replicas │ │ 2 replicas │ │ 2 replicas │ │ 3 replicas │   │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘   │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │payment-svc │ │workflow-svc│ │notify-svc  │ │audit-svc   │   │
│  │ NestJS     │ │ NestJS     │ │ NestJS     │ │ NestJS     │   │
│  │ 3 replicas │ │ 2 replicas │ │ 2 replicas │ │ 2 replicas │   │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘   │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                  │
│  │device-svc  │ │tenant-svc  │ │analytics-svc│                 │
│  │ NestJS     │ │ NestJS     │ │ NestJS      │                 │
│  │ 2 replicas │ │ 2 replicas │ │ 2 replicas  │                 │
│  └────────────┘ └────────────┘ └─────────────┘                  │
│                                                                  │
│  Namespace: ivm-channels                                         │
│  ┌────────────┐ ┌────────────┐ ┌──────────────┐                │
│  │vending-svc │ │locker-svc  │ │store-svc     │                │
│  │ NestJS     │ │ NestJS     │ │ NestJS       │                │
│  │ 2 replicas │ │ 2 replicas │ │ 2 replicas   │                │
│  └────────────┘ └────────────┘ └──────────────┘                │
│                                                                  │
│  Namespace: ivm-ai (Autonomous store sites only)                │
│  ┌──────────────────┐ ┌──────────────────┐                     │
│  │model-registry-svc│ │training-pipeline │                     │
│  │ FastAPI (Python)  │ │ Airflow + PyTorch│                     │
│  │ 1 replica         │ │ On-demand        │                     │
│  └──────────────────┘ └──────────────────┘                     │
│                                                                  │
│  Namespace: ivm-data                                             │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │PostgreSQL  │ │Redis       │ │Kafka       │ │EMQX        │   │
│  │ Primary +  │ │ Cluster    │ │ 3-broker   │ │ Cluster    │   │
│  │ 2 Replicas │ │ 3 nodes    │ │ cluster    │ │ 3 nodes    │   │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘   │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                  │
│  │TimescaleDB │ │OpenSearch  │ │ClickHouse  │                  │
│  │ (telemetry)│ │ (logs/srch)│ │ (analytics)│                  │
│  └────────────┘ └────────────┘ └────────────┘                  │
│                                                                  │
│  Namespace: ivm-monitoring                                       │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │Prometheus  │ │Grafana     │ │Loki        │ │Tempo       │   │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘   │
│                                                                  │
│  Namespace: ivm-infra                                            │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                  │
│  │Vault       │ │Cert-Manager│ │ArgoCD      │                  │
│  └────────────┘ └────────────┘ └────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Repository Structure

```
ivm-platform/
├── README.md
├── .github/
│   ├── workflows/          # CI/CD pipelines
│   │   ├── ci.yml
│   │   ├── cd-staging.yml
│   │   └── cd-production.yml
│   └── CODEOWNERS
│
├── packages/               # Shared libraries (monorepo)
│   ├── shared-types/       # TypeScript types shared across services
│   ├── shared-utils/       # Common utilities (ID generation, money, etc.)
│   ├── event-contracts/    # CloudEvents schema definitions
│   ├── api-contracts/      # OpenAPI specs + generated clients
│   └── ui-components/      # Shared React component library
│
├── services/               # Backend microservices
│   ├── identity-service/
│   ├── kyc-service/
│   ├── catalog-service/
│   ├── order-service/
│   ├── payment-service/
│   ├── workflow-service/
│   ├── notification-service/
│   ├── audit-service/
│   ├── device-service/
│   ├── tenant-service/
│   ├── analytics-service/
│   ├── vending-module/
│   ├── locker-module/
│   └── store-module/
│
├── bff/                    # Backend-for-Frontend services
│   ├── bff-kiosk/
│   ├── bff-locker/
│   ├── bff-customer/
│   └── bff-operator/
│
├── apps/                   # Frontend applications
│   ├── kiosk-ui/           # React (Electron/WebView)
│   ├── locker-ui/          # React (compact screen)
│   ├── customer-app/       # React Native (iOS/Android)
│   ├── customer-web/       # Next.js
│   ├── admin-portal/       # Next.js
│   ├── merchant-portal/    # Next.js
│   └── technician-app/     # React Native
│
├── edge/                   # Edge controller (Go)
│   ├── cmd/edge-controller/
│   ├── internal/
│   │   ├── gateway/        # Local REST API
│   │   ├── drivers/        # Device driver manager
│   │   ├── cache/          # State cache (SQLite)
│   │   ├── queue/          # Offline queue
│   │   ├── sync/           # Cloud synchronization
│   │   ├── telemetry/      # Telemetry collector
│   │   └── update/         # TUF update agent
│   └── drivers/            # Device driver implementations
│       ├── scanner/
│       ├── printer/
│       ├── lock-controller/
│       ├── card-reader/
│       ├── biometric/
│       ├── camera/
│       └── gpio/
│
├── ai/                     # AI/ML services (Python)
│   ├── inference-service/  # Edge inference runtime
│   ├── model-registry/     # Model management API
│   ├── training-pipeline/  # Training workflows
│   └── models/             # Model definitions
│
├── infra/                  # Infrastructure as Code
│   ├── terraform/
│   │   ├── environments/
│   │   │   ├── staging/
│   │   │   └── production/
│   │   └── modules/
│   │       ├── kubernetes/
│   │       ├── database/
│   │       ├── networking/
│   │       ├── monitoring/
│   │       └── security/
│   ├── helm/               # Helm charts
│   │   ├── ivm-platform/
│   │   ├── ivm-edge/
│   │   └── ivm-monitoring/
│   └── argocd/             # GitOps application definitions
│
├── docs/                   # Documentation
│   ├── architecture/       # Architecture docs (this folder)
│   ├── api/                # API documentation
│   ├── runbooks/           # Operational runbooks
│   └── adr/                # Architecture Decision Records
│
└── tools/                  # Developer tooling
    ├── scripts/
    ├── docker-compose.dev.yml
    └── Makefile
```

**Monorepo Management:** Use Turborepo or Nx for monorepo orchestration — shared builds, caching, and task dependencies.

---

## 5. CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                      CI/CD PIPELINE                              │
│                                                                  │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────────┐  │
│  │  Code   │───▶│  Build  │───▶│  Test   │───▶│  Security   │  │
│  │  Push   │    │  & Lint │    │         │    │  Scan       │  │
│  └─────────┘    └─────────┘    └─────────┘    └──────┬──────┘  │
│                                                       │         │
│  ┌─────────────┐    ┌──────────┐    ┌────────────────┘         │
│  │  Container  │◀───│  Quality │◀───┘                          │
│  │  Build &    │    │  Gate    │                                │
│  │  Push       │    │  (pass?) │                                │
│  └──────┬──────┘    └──────────┘                                │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               GITOPS (ArgoCD)                             │   │
│  │                                                          │   │
│  │  ┌────────┐     ┌──────────┐     ┌──────────────────┐   │   │
│  │  │Staging │────▶│ Approval │────▶│ Production       │   │   │
│  │  │ Auto   │     │ (manual) │     │ Progressive      │   │   │
│  │  │ Deploy │     │          │     │ Rollout (canary)  │   │   │
│  │  └────────┘     └──────────┘     └──────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Stages:                                                         │
│  1. Lint: ESLint, Prettier, Go vet, Python ruff                 │
│  2. Build: TypeScript compile, Go build, Docker build            │
│  3. Unit Tests: Jest (TS), Go test, pytest                       │
│  4. Integration Tests: Testcontainers (DB, Kafka, MQTT)          │
│  5. Security: Trivy (container scan), Snyk (dependency scan),    │
│     SAST (Semgrep), secrets detection (Gitleaks)                 │
│  6. Quality Gate: Coverage > 80%, no critical vulnerabilities    │
│  7. Container Push: Build & push to registry                     │
│  8. Staging Deploy: Auto-deploy via ArgoCD                       │
│  9. E2E Tests: Playwright (web), Detox (mobile)                 │
│  10. Production Deploy: Manual approval → canary → full          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Cloud Provider Strategy

**Primary Recommendation:** AWS (best presence in Africa, Lagos region pending, currently South Africa)

**Alternative:** Google Cloud (good Kubernetes tooling) or Azure (if enterprise partners mandate it)

| Service | AWS | GCP Alternative | Azure Alternative |
|---|---|---|---|
| Kubernetes | EKS | GKE | AKS |
| Database (PG) | RDS PostgreSQL | Cloud SQL | Azure Database for PG |
| Cache | ElastiCache Redis | Memorystore | Azure Cache for Redis |
| Object Storage | S3 | Cloud Storage | Blob Storage |
| Event Streaming | MSK (Managed Kafka) | Pub/Sub | Event Hubs |
| Secrets | Secrets Manager | Secret Manager | Key Vault |
| Container Registry | ECR | Artifact Registry | ACR |
| CDN | CloudFront | Cloud CDN | Azure CDN |
| DNS | Route 53 | Cloud DNS | Azure DNS |
| VPN | Site-to-Site VPN | Cloud VPN | VPN Gateway |

**Nigeria Hosting Consideration:**
- No major cloud provider has a Lagos data center yet
- Use Africa (Cape Town) region as primary with CDN edge in Nigeria
- Consider co-location in Lagos data center (Rack Centre / MDXi) for latency-sensitive components
- Edge controllers handle latency-critical operations locally
