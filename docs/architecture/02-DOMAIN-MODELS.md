# 02 - Domain Models & Data Architecture

## 1. Bounded Contexts Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          IVM PLATFORM BOUNDED CONTEXTS                       │
│                                                                              │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │   IDENTITY   │────▶│   CUSTOMER   │────▶│     KYC      │                │
│  │   CONTEXT    │     │   CONTEXT    │     │   CONTEXT    │                │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘                │
│         │                    │                     │                         │
│         ▼                    ▼                     ▼                         │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │   TENANT     │     │   CATALOG    │     │   PRICING    │                │
│  │   CONTEXT    │     │   CONTEXT    │     │   CONTEXT    │                │
│  └──────┬───────┘     └──────┬───────┘     └──────────────┘                │
│         │                    │                                               │
│         ▼                    ▼                                               │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │    ORDER     │◀───▶│  FULFILLMENT │────▶│  INVENTORY   │                │
│  │   CONTEXT    │     │   CONTEXT    │     │   CONTEXT    │                │
│  └──────┬───────┘     └──────────────┘     └──────────────┘                │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │   PAYMENT    │     │   DEVICE     │────▶│    SITE      │                │
│  │   CONTEXT    │     │   CONTEXT    │     │   CONTEXT    │                │
│  └──────────────┘     └──────┬───────┘     └──────────────┘                │
│                              │                                               │
│                              ▼                                               │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │  WORKFLOW    │     │  TELEMETRY   │     │    AUDIT     │                │
│  │   CONTEXT    │     │   CONTEXT    │     │   CONTEXT    │                │
│  └──────────────┘     └──────────────┘     └──────────────┘                │
│                                                                              │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │  NOTIFICATION│     │   SESSION    │     │  AI/VISION   │                │
│  │   CONTEXT    │     │   CONTEXT    │     │   CONTEXT    │                │
│  └──────────────┘     └──────────────┘     └──────────────┘                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Core Entity Definitions

### 2.1 Identity Context

```
┌─────────────────────────────────────┐
│            User (Aggregate Root)     │
├─────────────────────────────────────┤
│ id: ULID                            │
│ type: CUSTOMER | OPERATOR | SYSTEM  │
│ status: ACTIVE | SUSPENDED | CLOSED │
│ email?: string                      │
│ phone: string (required for NG)     │
│ passwordHash?: string               │
│ mfaEnabled: boolean                 │
│ mfaMethods: MFAMethod[]             │
│ tenantId: ULID                      │
│ roles: Role[]                       │
│ lastLoginAt?: timestamp             │
│ createdAt: timestamp                │
│ updatedAt: timestamp                │
├─────────────────────────────────────┤
│ Methods:                            │
│  authenticate(credentials)          │
│  assignRole(role)                   │
│  suspend(reason)                    │
│  reactivate()                       │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│           Role                       │
├─────────────────────────────────────┤
│ id: ULID                            │
│ name: string                        │
│ tenantId: ULID                      │
│ permissions: Permission[]           │
│ scope: PLATFORM | TENANT | SITE     │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         DeviceIdentity               │
├─────────────────────────────────────┤
│ deviceId: ULID                      │
│ certificateFingerprint: string      │
│ hardwareFingerprint: string         │
│ enrolledAt: timestamp               │
│ lastAuthenticatedAt: timestamp      │
│ status: ENROLLED | ACTIVE | REVOKED │
└─────────────────────────────────────┘
```

### 2.2 Customer Context

```
┌─────────────────────────────────────┐
│       Customer (Aggregate Root)      │
├─────────────────────────────────────┤
│ id: ULID                            │
│ userId: ULID (→ User)               │
│ tenantId: ULID                      │
│ firstName: string                   │
│ lastName: string                    │
│ middleName?: string                 │
│ dateOfBirth?: date                  │
│ gender?: MALE | FEMALE              │
│ phone: string                       │
│ email?: string                      │
│ address?: Address                   │
│ kycTier: TIER_0 | TIER_1 | TIER_2 | TIER_3 │
│ kycStatus: PENDING | VERIFIED | FAILED | EXPIRED │
│ identityEvidence: IdentityEvidence[]│
│ preferences: CustomerPreferences    │
│ walletId?: ULID                     │
│ createdAt: timestamp                │
│ updatedAt: timestamp                │
├─────────────────────────────────────┤
│ Methods:                            │
│  updateKycTier(tier, evidence)      │
│  addIdentityEvidence(evidence)      │
│  updatePreferences(prefs)           │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│        IdentityEvidence              │
├─────────────────────────────────────┤
│ id: ULID                            │
│ customerId: ULID                    │
│ type: BVN | NIN | PASSPORT |        │
│       DRIVERS_LICENSE | VOTER_CARD | │
│       INTL_PASSPORT                  │
│ identifierValue: string (encrypted) │
│ verificationStatus: PENDING |       │
│   VERIFIED | FAILED | EXPIRED       │
│ verifiedAt?: timestamp              │
│ verifiedBy: string (provider ID)    │
│ verificationResponse: JSON (encrypted) │
│ expiresAt?: timestamp               │
│ createdAt: timestamp                │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│            Address                   │
├─────────────────────────────────────┤
│ line1: string                       │
│ line2?: string                      │
│ city: string                        │
│ state: string                       │
│ country: string (ISO 3166-1)        │
│ postalCode?: string                 │
│ coordinates?: {lat, lng}            │
└─────────────────────────────────────┘
```

### 2.3 Tenant Context

```
┌─────────────────────────────────────┐
│        Tenant (Aggregate Root)       │
├─────────────────────────────────────┤
│ id: ULID                            │
│ name: string                        │
│ slug: string (unique)               │
│ type: PLATFORM | MERCHANT | PARTNER │
│ status: ACTIVE | SUSPENDED | TRIAL  │
│ parentTenantId?: ULID               │
│ branding: BrandingConfig            │
│ features: FeatureFlags              │
│ billingPlan: string                 │
│ settlementConfig: SettlementConfig  │
│ contactEmail: string                │
│ contactPhone: string                │
│ address: Address                    │
│ ndpcRegistration?: string           │
│ createdAt: timestamp                │
│ updatedAt: timestamp                │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         BrandingConfig               │
├─────────────────────────────────────┤
│ logoUrl: string                     │
│ primaryColor: string                │
│ secondaryColor: string              │
│ fontFamily?: string                 │
│ kioskTheme?: ThemeConfig            │
│ receiptHeader?: string              │
│ receiptFooter?: string              │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│        SettlementConfig              │
├─────────────────────────────────────┤
│ bankCode: string                    │
│ accountNumber: string (encrypted)   │
│ accountName: string                 │
│ settlementSchedule: DAILY | WEEKLY | INSTANT │
│ commissionRate: decimal             │
│ minimumPayout: decimal              │
│ currency: string (NGN)              │
└─────────────────────────────────────┘
```

### 2.4 Site & Device Context

```
┌─────────────────────────────────────┐
│          Site (Aggregate Root)        │
├─────────────────────────────────────┤
│ id: ULID                            │
│ tenantId: ULID                      │
│ name: string                        │
│ type: KIOSK_STATION | LOCKER_BANK | │
│       AUTONOMOUS_STORE | HYBRID     │
│ status: ACTIVE | MAINTENANCE |      │
│         OFFLINE | DECOMMISSIONED    │
│ address: Address                    │
│ coordinates: {lat, lng}             │
│ timezone: string                    │
│ operatingHours: OperatingSchedule   │
│ devices: Device[]                   │
│ edgeControllerId?: ULID             │
│ networkConfig: NetworkConfig        │
│ createdAt: timestamp                │
│ updatedAt: timestamp                │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│        Device (Aggregate Root)       │
├─────────────────────────────────────┤
│ id: ULID                            │
│ siteId: ULID                        │
│ tenantId: ULID                      │
│ type: DeviceType                    │
│ manufacturer: string                │
│ model: string                       │
│ serialNumber: string                │
│ firmwareVersion: string             │
│ softwareVersion: string             │
│ status: PROVISIONED | ONLINE |      │
│         OFFLINE | MAINTENANCE |     │
│         FAULT | DECOMMISSIONED      │
│ capabilities: Capability[]          │
│ config: JSON                        │
│ lastHeartbeatAt?: timestamp         │
│ lastMaintenanceAt?: timestamp       │
│ enrolledAt: timestamp               │
│ createdAt: timestamp                │
│ updatedAt: timestamp                │
├─────────────────────────────────────┤
│ Methods:                            │
│  executeCommand(command)            │
│  updateConfig(config)               │
│  reportHealth(healthData)           │
│  scheduleUpdate(updatePackage)      │
└─────────────────────────────────────┘

DeviceType enum:
  TOUCHSCREEN, QR_SCANNER, BARCODE_SCANNER,
  RECEIPT_PRINTER, THERMAL_PRINTER, CARD_READER,
  PIN_PAD, BIOMETRIC_SCANNER, CAMERA,
  LOCKER_CONTROLLER, SHELF_SENSOR, WEIGHT_SENSOR,
  ELECTRONIC_LOCK, VENDING_MOTOR, GATE_CONTROLLER,
  CASH_ACCEPTOR, CASH_DISPENSER, EDGE_CONTROLLER,
  CELLULAR_MODEM, NFC_READER

Capability enum:
  SCAN_QR, SCAN_BARCODE, PRINT_RECEIPT,
  PRINT_DOCUMENT, READ_CARD_EMV, READ_CARD_NFC,
  CAPTURE_PIN, CAPTURE_FINGERPRINT, CAPTURE_PHOTO,
  CAPTURE_VIDEO, UNLOCK_DOOR, LOCK_DOOR,
  DETECT_WEIGHT, DISPENSE_ITEM, ACCEPT_CASH,
  DISPENSE_CASH, OPEN_GATE, CLOSE_GATE
```

### 2.5 Order & Transaction Context

```
┌─────────────────────────────────────┐
│         Order (Aggregate Root)       │
├─────────────────────────────────────┤
│ id: ULID                            │
│ tenantId: ULID                      │
│ customerId: ULID                    │
│ siteId: ULID                        │
│ deviceId?: ULID                     │
│ channelType: KIOSK | LOCKER | STORE │
│              | MOBILE | WEB         │
│ type: SERVICE | PRODUCT | MIXED     │
│ status: OrderStatus                 │
│ lineItems: OrderLineItem[]          │
│ subtotal: Money                     │
│ discount: Money                     │
│ tax: Money                          │
│ total: Money                        │
│ paymentId?: ULID                    │
│ fulfillmentTasks: FulfillmentTask[] │
│ workflowInstanceId?: ULID          │
│ metadata: JSON                      │
│ createdAt: timestamp                │
│ updatedAt: timestamp                │
│ completedAt?: timestamp             │
│ cancelledAt?: timestamp             │
│ expiresAt?: timestamp               │
├─────────────────────────────────────┤
│ Methods:                            │
│  addLineItem(item)                  │
│  removeLineItem(itemId)             │
│  applyDiscount(discount)            │
│  confirm()                          │
│  fulfill(task)                      │
│  complete()                         │
│  cancel(reason)                     │
│  dispute(reason)                    │
└─────────────────────────────────────┘

OrderStatus state machine:
  DRAFT → PENDING_PAYMENT → PAYMENT_AUTHORIZED →
  FULFILLING → COMPLETED
                          ↘ PARTIALLY_FULFILLED → COMPLETED
  Any → CANCELLED
  COMPLETED → DISPUTED → RESOLVED

┌─────────────────────────────────────┐
│          OrderLineItem               │
├─────────────────────────────────────┤
│ id: ULID                            │
│ orderId: ULID                       │
│ catalogItemId: ULID                 │
│ name: string                        │
│ description?: string                │
│ quantity: integer                   │
│ unitPrice: Money                    │
│ discount: Money                     │
│ tax: Money                          │
│ total: Money                        │
│ metadata: JSON                      │
│ confidence?: decimal (AI store)     │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│        FulfillmentTask               │
├─────────────────────────────────────┤
│ id: ULID                            │
│ orderId: ULID                       │
│ type: DISPENSE | PRINT | DELIVER |  │
│       ACTIVATE | ISSUE | PICKUP |   │
│       LOCKER_DEPOSIT | LOCKER_PICKUP│
│ status: PENDING | IN_PROGRESS |     │
│         COMPLETED | FAILED          │
│ deviceId?: ULID                     │
│ assignedTo?: string                 │
│ details: JSON                       │
│ startedAt?: timestamp               │
│ completedAt?: timestamp             │
│ failedAt?: timestamp                │
│ failureReason?: string              │
│ retryCount: integer                 │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│             Money                    │
├─────────────────────────────────────┤
│ amount: bigint (minor units, e.g. kobo) │
│ currency: string (ISO 4217, e.g. "NGN") │
├─────────────────────────────────────┤
│ Invariants:                         │
│  - amount >= 0                      │
│  - currency always 3 chars          │
│  - arithmetic only on same currency │
└─────────────────────────────────────┘
```

### 2.6 Payment Context

```
┌─────────────────────────────────────┐
│       Payment (Aggregate Root)       │
├─────────────────────────────────────┤
│ id: ULID                            │
│ tenantId: ULID                      │
│ orderId: ULID                       │
│ customerId: ULID                    │
│ method: CARD | MOBILE_MONEY | BANK_TRANSFER │
│         | WALLET | USSD | QR        │
│ status: PaymentStatus               │
│ amount: Money                       │
│ authorizedAmount?: Money            │
│ capturedAmount?: Money              │
│ refundedAmount?: Money              │
│ gateway: string                     │
│ gatewayRef: string                  │
│ gatewayResponse: JSON (encrypted)   │
│ tokenizedCard?: TokenizedCard       │
│ retryCount: integer                 │
│ idempotencyKey: string              │
│ initiatedAt: timestamp              │
│ authorizedAt?: timestamp            │
│ capturedAt?: timestamp              │
│ failedAt?: timestamp                │
│ refundedAt?: timestamp              │
│ metadata: JSON                      │
├─────────────────────────────────────┤
│ Methods:                            │
│  authorize()                        │
│  capture(amount?)                   │
│  refund(amount, reason)             │
│  void()                             │
└─────────────────────────────────────┘

PaymentStatus state machine:
  INITIATED → AUTHORIZED → CAPTURED → SETTLED
  INITIATED → FAILED
  AUTHORIZED → VOIDED
  CAPTURED → PARTIALLY_REFUNDED → FULLY_REFUNDED
  CAPTURED → FULLY_REFUNDED

┌─────────────────────────────────────┐
│         TokenizedCard                │
├─────────────────────────────────────┤
│ token: string (gateway-issued)      │
│ last4: string                       │
│ brand: VISA | MASTERCARD | VERVE    │
│ expiryMonth: integer                │
│ expiryYear: integer                 │
│ bankName?: string                   │
│ NOTE: NO PAN storage. Ever.         │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│       SettlementRecord               │
├─────────────────────────────────────┤
│ id: ULID                            │
│ tenantId: ULID                      │
│ merchantId: ULID                    │
│ period: DateRange                   │
│ grossAmount: Money                  │
│ commissionAmount: Money             │
│ netAmount: Money                    │
│ transactionCount: integer           │
│ status: PENDING | PROCESSING |      │
│         SETTLED | FAILED            │
│ payoutRef?: string                  │
│ settledAt?: timestamp               │
└─────────────────────────────────────┘
```

### 2.7 Catalog Context

```
┌─────────────────────────────────────┐
│     CatalogItem (Aggregate Root)     │
├─────────────────────────────────────┤
│ id: ULID                            │
│ tenantId: ULID                      │
│ type: PHYSICAL_PRODUCT | DIGITAL_SERVICE │
│       | SIM_CARD | INSURANCE_POLICY │
│       | BANKING_SERVICE | TICKET    │
│       | DOCUMENT_SERVICE            │
│ name: string                        │
│ description: string                 │
│ sku?: string                        │
│ categoryId: ULID                    │
│ imageUrls: string[]                 │
│ basePrice: Money                    │
│ taxRate?: decimal                   │
│ status: ACTIVE | INACTIVE | DRAFT   │
│ channelAvailability: ChannelType[]  │
│ siteAvailability?: ULID[] (site IDs)│
│ attributes: JSON                    │
│ requiresKyc: boolean               │
│ requiredKycTier?: KycTier           │
│ requiresInventory: boolean          │
│ fulfillmentType: FulfillmentType    │
│ createdAt: timestamp                │
│ updatedAt: timestamp                │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│           Category                   │
├─────────────────────────────────────┤
│ id: ULID                            │
│ tenantId: ULID                      │
│ parentId?: ULID                     │
│ name: string                        │
│ slug: string                        │
│ sortOrder: integer                  │
│ imageUrl?: string                   │
│ status: ACTIVE | INACTIVE           │
└─────────────────────────────────────┘
```

### 2.8 Locker Context (Channel-Specific)

```
┌─────────────────────────────────────┐
│   LockerBank (Aggregate Root)        │
├─────────────────────────────────────┤
│ id: ULID                            │
│ siteId: ULID                        │
│ tenantId: ULID                      │
│ name: string                        │
│ compartments: Compartment[]         │
│ status: ACTIVE | MAINTENANCE        │
│ totalCapacity: integer              │
│ availableCount: integer             │
│ location: Address                   │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│          Compartment                 │
├─────────────────────────────────────┤
│ id: ULID                            │
│ lockerBankId: ULID                  │
│ number: string (e.g., "A1", "B3")  │
│ size: SMALL | MEDIUM | LARGE | XL  │
│       | REFRIGERATED                │
│ status: AVAILABLE | OCCUPIED |      │
│         RESERVED | MAINTENANCE |    │
│         OUT_OF_SERVICE              │
│ currentReservationId?: ULID         │
│ lockDeviceId: ULID (→ Device)       │
│ sensorDeviceId?: ULID              │
│ dimensions: {width, height, depth}  │
│ hasTemperatureControl: boolean      │
│ lastOpenedAt?: timestamp            │
│ lastCleanedAt?: timestamp           │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│       LockerReservation              │
├─────────────────────────────────────┤
│ id: ULID                            │
│ lockerBankId: ULID                  │
│ compartmentId: ULID                 │
│ orderId: ULID                       │
│ type: PICKUP | DROPOFF | RETURN     │
│ status: RESERVED | LOADED |         │
│         READY_FOR_PICKUP | PICKED_UP│
│         | EXPIRED | CANCELLED       │
│ depositorId?: ULID (courier)        │
│ recipientId: ULID (customer)        │
│ accessCode: string (hashed)         │
│ accessMethod: PIN | QR | OTP | APP  │
│ reservedAt: timestamp               │
│ loadedAt?: timestamp                │
│ notifiedAt?: timestamp              │
│ pickedUpAt?: timestamp              │
│ expiresAt: timestamp                │
│ pickupSlaMinutes: integer           │
│ remindersSent: integer              │
└─────────────────────────────────────┘

Reservation state machine:
  RESERVED → LOADED → READY_FOR_PICKUP → PICKED_UP
  RESERVED → CANCELLED
  READY_FOR_PICKUP → EXPIRED → (return-to-sender flow)
```

### 2.9 Shopping Session Context (Autonomous Retail)

```
┌─────────────────────────────────────┐
│   ShoppingSession (Aggregate Root)   │
├─────────────────────────────────────┤
│ id: ULID                            │
│ storeId: ULID (→ Site)              │
│ tenantId: ULID                      │
│ customerId: ULID                    │
│ status: SessionStatus               │
│ entryMethod: QR | NFC | CARD | FACE │
│ enteredAt: timestamp                │
│ exitedAt?: timestamp                │
│ basketItems: BasketItem[]           │
│ basketTotal: Money                  │
│ paymentPreAuthId?: ULID             │
│ paymentCaptureId?: ULID             │
│ confidenceScore: decimal (0.0-1.0)  │
│ requiresReview: boolean             │
│ receiptSentAt?: timestamp           │
│ videoSegmentRefs: string[]          │
│ sensorEvents: SensorEventRef[]      │
│ createdAt: timestamp                │
│ updatedAt: timestamp                │
└─────────────────────────────────────┘

SessionStatus state machine:
  ENTRY_AUTHORIZED → SHOPPING → EXIT_DETECTED →
  RECONCILING → CHARGING → COMPLETED
  RECONCILING → REVIEW_REQUIRED → RESOLVED → COMPLETED
  COMPLETED → DISPUTED → RESOLVED

┌─────────────────────────────────────┐
│           BasketItem                 │
├─────────────────────────────────────┤
│ id: ULID                            │
│ sessionId: ULID                     │
│ catalogItemId: ULID                 │
│ action: PICKED | RETURNED           │
│ quantity: integer                   │
│ unitPrice: Money                    │
│ confidence: decimal (0.0-1.0)       │
│ detectionSource: VISION | SHELF_SENSOR │
│                  | RFID | WEIGHT    │
│ timestamp: timestamp                │
│ evidenceRefs: string[] (frame IDs)  │
└─────────────────────────────────────┘
```

### 2.10 Workflow Context

```
┌─────────────────────────────────────┐
│  WorkflowDefinition (Aggregate Root) │
├─────────────────────────────────────┤
│ id: ULID                            │
│ tenantId: ULID                      │
│ name: string                        │
│ version: integer                    │
│ channelType: KIOSK | LOCKER | STORE │
│ triggerType: MANUAL | EVENT | SCHEDULE │
│ steps: WorkflowStep[]               │
│ status: DRAFT | ACTIVE | DEPRECATED │
│ createdAt: timestamp                │
│ updatedAt: timestamp                │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         WorkflowStep                 │
├─────────────────────────────────────┤
│ id: string                          │
│ type: USER_INPUT | SYSTEM_ACTION |  │
│       APPROVAL | TIMER | BRANCH |   │
│       PARALLEL | COMPENSATION       │
│ name: string                        │
│ config: JSON                        │
│ nextSteps: ConditionalNext[]        │
│ timeoutSeconds?: integer            │
│ retryPolicy?: RetryPolicy           │
│ compensationStepId?: string         │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│      WorkflowInstance                │
├─────────────────────────────────────┤
│ id: ULID                            │
│ definitionId: ULID                  │
│ definitionVersion: integer          │
│ tenantId: ULID                      │
│ orderId?: ULID                      │
│ customerId?: ULID                   │
│ deviceId?: ULID                     │
│ siteId?: ULID                       │
│ status: RUNNING | PAUSED | COMPLETED │
│         | FAILED | COMPENSATING     │
│ currentStepId: string               │
│ stepHistory: StepExecution[]        │
│ context: JSON (workflow variables)  │
│ startedAt: timestamp                │
│ completedAt?: timestamp             │
│ failedAt?: timestamp                │
└─────────────────────────────────────┘
```

### 2.11 Audit Context

```
┌─────────────────────────────────────┐
│        AuditEntry (Immutable)        │
├─────────────────────────────────────┤
│ id: ULID                            │
│ tenantId: ULID                      │
│ timestamp: timestamp                │
│ actor: AuditActor                   │
│ action: string (e.g., "kyc.verify") │
│ resource: AuditResource             │
│ outcome: SUCCESS | FAILURE | DENIED │
│ details: JSON                       │
│ sourceIp?: string                   │
│ deviceId?: ULID                     │
│ siteId?: ULID                       │
│ traceId: string                     │
│ previousHash: string                │
│ hash: string                        │
├─────────────────────────────────────┤
│ NOTE: Append-only. Never updated    │
│ or deleted. Hash chain for tamper   │
│ evidence.                           │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│           AuditActor                 │
├─────────────────────────────────────┤
│ type: USER | DEVICE | SYSTEM | API  │
│ id: string                          │
│ name?: string                       │
│ roles?: string[]                    │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         AuditResource                │
├─────────────────────────────────────┤
│ type: string (e.g., "order")        │
│ id: string                          │
│ attributes?: JSON                   │
└─────────────────────────────────────┘
```

---

## 3. Entity Relationship Diagram

```
┌──────────┐     ┌──────────┐     ┌──────────────┐
│  Tenant  │────<│   Site   │────<│    Device    │
└────┬─────┘     └────┬─────┘     └──────────────┘
     │                │
     │                │            ┌──────────────┐
     │                ├───────────<│ LockerBank   │
     │                │            │  └─Compartment│
     │                │            └──────────────┘
     │                │
     ├───────────<┌───┴────┐      ┌──────────────┐
     │            │Customer│─────<│ IdentityEvid.│
     │            └───┬────┘      └──────────────┘
     │                │
     │                │
     ├──────<┌────────┴──────┐    ┌──────────────┐
     │       │    Order      │───<│  LineItem    │
     │       └────────┬──────┘    └──────────────┘
     │                │
     │                ├──────────<┌──────────────┐
     │                │           │Fulfillment   │
     │                │           │  Task        │
     │                │           └──────────────┘
     │                │
     │                ├──────────>┌──────────────┐
     │                │           │   Payment    │
     │                │           └──────────────┘
     │                │
     │                ├──────────>┌──────────────┐
     │                │           │  Workflow    │
     │                │           │  Instance    │
     │                │           └──────────────┘
     │                │
     │                └──────────>┌──────────────┐
     │                            │   Locker     │
     │                            │ Reservation  │
     │                            └──────────────┘
     │
     ├──────<┌────────────────┐
     │       │ CatalogItem    │
     │       │  └─Category    │
     │       └────────────────┘
     │
     ├──────<┌────────────────┐
     │       │ Shopping       │
     │       │ Session        │
     │       │  └─BasketItem  │
     │       └────────────────┘
     │
     └──────<┌────────────────┐
              │  AuditEntry    │
              └────────────────┘
```

---

## 4. Database Schema Strategy

### 4.1 Schema-Per-Service

Each core service owns its database schema. No direct cross-schema queries — services communicate via APIs and events.

```
Database: ivm_identity
  Tables: users, roles, permissions, user_roles, device_identities, sessions

Database: ivm_customer
  Tables: customers, identity_evidence, addresses, customer_preferences

Database: ivm_tenant
  Tables: tenants, branding_configs, settlement_configs, feature_flags

Database: ivm_site
  Tables: sites, operating_schedules, network_configs

Database: ivm_device
  Tables: devices, device_capabilities, device_configs, device_health_snapshots

Database: ivm_catalog
  Tables: catalog_items, categories, item_attributes, item_availability

Database: ivm_order
  Tables: orders, order_line_items, fulfillment_tasks

Database: ivm_payment
  Tables: payments, tokenized_cards, settlement_records, refunds

Database: ivm_workflow
  Tables: workflow_definitions, workflow_steps, workflow_instances, step_executions

Database: ivm_notification
  Tables: notification_templates, notification_deliveries, delivery_attempts

Database: ivm_audit
  Tables: audit_entries (partitioned by month, never deleted)

Database: ivm_locker
  Tables: locker_banks, compartments, locker_reservations

Database: ivm_store
  Tables: shopping_sessions, basket_items, sensor_event_refs

Database: ivm_telemetry (TimescaleDB)
  Hypertables: device_metrics, device_events, transaction_metrics
```

### 4.2 Multi-Tenancy Implementation

**Strategy:** Row-Level Security (RLS) with tenant_id column on every table.

```sql
-- Example: Enable RLS on orders table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.current_tenant_id')::ulid);

-- Set tenant context at connection level (from middleware)
SET app.current_tenant_id = 'tnt_01HX...';
```

**Benefits:**
- Single database instance for cost efficiency
- Automatic tenant data isolation at the database level
- No application-layer filtering mistakes
- Can be promoted to separate databases for premium tenants

### 4.3 ID Strategy

**ULID (Universally Unique Lexicographically Sortable Identifier):**
- Globally unique across all services
- Time-sortable (useful for ordering without timestamps)
- URL-safe, 26-character string representation
- No coordination needed (can be generated at edge)

**Prefixed IDs for human readability:**
```
usr_01HX7V8K9M2N3P4Q5R6S7T8U9V  (user)
tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V  (tenant)
ord_01HX7V8K9M2N3P4Q5R6S7T8U9V  (order)
pay_01HX7V8K9M2N3P4Q5R6S7T8U9V  (payment)
dev_01HX7V8K9M2N3P4Q5R6S7T8U9V  (device)
site_01HX7V8K9M2N3P4Q5R6S7T8U9V (site)
cmp_01HX7V8K9M2N3P4Q5R6S7T8U9V  (compartment)
rsv_01HX7V8K9M2N3P4Q5R6S7T8U9V  (reservation)
ssn_01HX7V8K9M2N3P4Q5R6S7T8U9V  (shopping session)
wf_01HX7V8K9M2N3P4Q5R6S7T8U9V   (workflow)
evt_01HX7V8K9M2N3P4Q5R6S7T8U9V  (event)
```

### 4.4 Encryption at Rest

| Data Classification | Encryption | Key Management |
|---|---|---|
| PII (names, addresses, phone) | AES-256-GCM field-level | Application-managed keys in Vault |
| Identity evidence (BVN, NIN) | AES-256-GCM field-level | Dedicated key per tenant |
| Payment tokens | Gateway-managed tokenization | Gateway owns keys |
| Biometric templates | AES-256-GCM, separate storage | Hardware-backed keys |
| Audit logs | Database-level TDE | Infrastructure-managed |
| Video/media | Object storage encryption | Infrastructure-managed |
| Device credentials | AES-256-GCM | Hardware-backed (TPM) |

---

## 5. Event Catalog

### 5.1 Core Domain Events

```
Identity Events:
  ivm.user.created
  ivm.user.authenticated
  ivm.user.authentication_failed
  ivm.user.suspended
  ivm.user.reactivated
  ivm.device.enrolled
  ivm.device.authenticated
  ivm.device.certificate_rotated

Customer Events:
  ivm.customer.created
  ivm.customer.updated
  ivm.customer.kyc_tier_changed
  ivm.customer.identity_evidence_added
  ivm.customer.identity_verified
  ivm.customer.identity_verification_failed

Order Events:
  ivm.order.created
  ivm.order.confirmed
  ivm.order.payment_authorized
  ivm.order.fulfillment_started
  ivm.order.fulfillment_completed
  ivm.order.completed
  ivm.order.cancelled
  ivm.order.disputed
  ivm.order.dispute_resolved

Payment Events:
  ivm.payment.initiated
  ivm.payment.authorized
  ivm.payment.captured
  ivm.payment.failed
  ivm.payment.refunded
  ivm.payment.settled

Device Events:
  ivm.device.online
  ivm.device.offline
  ivm.device.health_reported
  ivm.device.fault_detected
  ivm.device.command_executed
  ivm.device.update_started
  ivm.device.update_completed
  ivm.device.update_failed

Locker Events:
  ivm.locker.compartment_reserved
  ivm.locker.parcel_deposited
  ivm.locker.door_opened
  ivm.locker.door_closed
  ivm.locker.parcel_picked_up
  ivm.locker.reservation_expired
  ivm.locker.compartment_released

Store Events:
  ivm.store.session_started
  ivm.store.customer_entered
  ivm.store.item_picked
  ivm.store.item_returned
  ivm.store.customer_exited
  ivm.store.basket_reconciled
  ivm.store.session_charged
  ivm.store.session_disputed
  ivm.store.shrinkage_detected

Workflow Events:
  ivm.workflow.started
  ivm.workflow.step_completed
  ivm.workflow.step_failed
  ivm.workflow.completed
  ivm.workflow.failed
  ivm.workflow.compensating

Notification Events:
  ivm.notification.sent
  ivm.notification.delivered
  ivm.notification.failed
```
