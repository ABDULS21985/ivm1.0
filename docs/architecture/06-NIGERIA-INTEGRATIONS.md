# 06 - Nigeria Integration Architecture

## 1. Integration Landscape

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    IVM PLATFORM                                          │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                    INTEGRATION GATEWAY                             │  │
│  │   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐    │  │
│  │   │ Adapter  │ │ Circuit  │ │ Retry    │ │ Response         │    │  │
│  │   │ Registry │ │ Breaker  │ │ Engine   │ │ Cache/Transform  │    │  │
│  │   └──────────┘ └──────────┘ └──────────┘ └──────────────────┘    │  │
│  └──────────────────────────┬─────────────────────────────────────────┘  │
│                              │                                           │
│  ┌───────────────────────────┼────────────────────────────────────────┐  │
│  │          ADAPTER LAYER    │                                        │  │
│  │                           │                                        │  │
│  │  ┌───────────────────┐ ┌──┴────────────────┐ ┌─────────────────┐  │  │
│  │  │  Identity         │ │  Payment          │ │  Telecom        │  │  │
│  │  │  Adapters         │ │  Adapters         │ │  Adapters       │  │  │
│  │  │  ├── NIBSS (BVN)  │ │  ├── Paystack     │ │  ├── MTN API    │  │  │
│  │  │  ├── NIMC (NIN)   │ │  ├── Flutterwave  │ │  ├── Airtel API │  │  │
│  │  │  ├── FRSC (DL)    │ │  ├── Bank APIs    │ │  ├── Glo API    │  │  │
│  │  │  └── INEC (VIN)   │ │  ├── Mobile Money │ │  └── 9mobile API│  │  │
│  │  └───────────────────┘ │  └── NIBSS (NIP)  │ └─────────────────┘  │  │
│  │                        └───────────────────┘                       │  │
│  │  ┌───────────────────┐ ┌───────────────────┐ ┌─────────────────┐  │  │
│  │  │  Insurance        │ │  Banking          │ │  Logistics      │  │  │
│  │  │  Adapters         │ │  Adapters         │ │  Adapters       │  │  │
│  │  │  ├── AXA Mansard  │ │  ├── CBN (ICAD)   │ │  ├── GIG Logist.│  │  │
│  │  │  ├── Leadway      │ │  ├── Core Banking  │ │  ├── DHL        │  │  │
│  │  │  ├── AIICO        │ │  │    (various)    │ │  ├── FedEx      │  │  │
│  │  │  └── Custodian    │ │  └── Microfinance  │ │  └── Kwik       │  │  │
│  │  └───────────────────┘ │     APIs           │ └─────────────────┘  │  │
│  │                        └───────────────────┘                       │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘

External Systems:
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│      NIBSS       │  │      NIMC        │  │      NCC         │
│  (BVN Database)  │  │  (NIN Database)  │  │ (Telecom Regul.) │
│  ├── BVN Verify  │  │  ├── NIN Verify  │  │  ├── SIM Reg DB  │
│  ├── BVN Lookup  │  │  ├── NIN Lookup  │  │  └── Compliance  │
│  ├── NIP (Payments│  │  ├── Virtual NIN │  │     Reporting    │
│  └── ICAD        │  │  └── Biometric   │  └──────────────────┘
└──────────────────┘  └──────────────────┘
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│      CBN         │  │      NDPC        │  │    FIRS / LIRS   │
│  (Central Bank)  │  │ (Data Protect.)  │  │  (Tax Authority) │
│  ├── Regulatory  │  │  ├── Registration │  │  ├── TIN Verify  │
│  │   Reporting   │  │  ├── Breach      │  │  └── Tax Receipt │
│  ├── KYC Tiers   │  │  │   Notification │  └──────────────────┘
│  └── AML Reporting│  │  └── Compliance  │
└──────────────────┘  │     Audits       │
                      └──────────────────┘
```

---

## 2. Identity Verification Integrations

### 2.1 BVN Verification (NIBSS)

**Purpose:** Verify customer identity using Bank Verification Number for CBN-mandated KYC.

```
Flow: BVN Verification at Kiosk

Customer  →  Kiosk UI  →  Edge Controller  →  Cloud KYC Service  →  NIBSS BVN API
                                                                          │
    ┌─────────────────────────────────────────────────────────────────────┘
    │
    ▼
1. Customer enters BVN on kiosk touchscreen
2. Kiosk captures biometric (fingerprint / photo)
3. Edge forwards to Cloud KYC Service
4. KYC Service calls NIBSS BVN Validation API:
   - Input: BVN, Date of Birth
   - NIBSS returns: First Name, Last Name, Middle Name, DOB, Phone,
                    Photo (base64), Gender, Registration Date
5. KYC Service performs matching:
   - Compare returned data with customer-provided data
   - Biometric match: compare NIBSS photo with kiosk-captured photo
   - Score match quality
6. If match passes threshold:
   - Store IdentityEvidence (encrypted) with verification result
   - Update customer KYC tier to TIER_1
   - Emit event: ivm.customer.identity_verified
7. If match fails:
   - Log failure reason
   - Allow retry (max 3 attempts)
   - Emit event: ivm.customer.identity_verification_failed
8. All interactions logged in audit trail
```

**Adapter Interface:**

```typescript
interface NIBSSBVNAdapter {
  // Verify BVN and retrieve associated data
  verifyBVN(request: BVNVerificationRequest): Promise<BVNVerificationResponse>;

  // Validate BVN format (pre-flight check)
  validateBVNFormat(bvn: string): boolean;
}

interface BVNVerificationRequest {
  bvn: string;           // 11-digit BVN
  dateOfBirth: string;   // YYYY-MM-DD
  firstName?: string;    // For additional matching
  lastName?: string;
  requestId: string;     // Idempotency
  consentRef: string;    // Link to consent record
}

interface BVNVerificationResponse {
  success: boolean;
  bvnValid: boolean;
  data?: {
    firstName: string;
    lastName: string;
    middleName?: string;
    dateOfBirth: string;
    phoneNumber: string;
    gender: string;
    photo?: string;        // Base64 encoded
    registrationDate: string;
    enrollmentBank: string;
  };
  error?: {
    code: string;
    message: string;
  };
}
```

**Resilience:**
- Circuit breaker: open after 5 consecutive failures, half-open after 30s
- Timeout: 15s per request
- Retry: 3 attempts with exponential backoff (1s, 2s, 4s)
- Fallback: queue verification for retry, allow limited-function TIER_0 access

### 2.2 NIN Verification (NIMC)

**Purpose:** Verify customer identity using National Identification Number.

```
Flow: NIN Verification

Similar to BVN flow, with differences:
1. Customer provides NIN (11 digits) or Virtual NIN (16 chars)
2. Platform calls NIMC NIN Verification API
3. NIMC returns: demographics + biometric data
4. Matching and scoring performed
5. Can be used as alternative to BVN or in combination

Virtual NIN (vNIN):
- NIMC issues vNIN to protect actual NIN
- vNIN is one-time-use, time-limited
- Customer generates vNIN via NIMC app or USSD (*346#)
- Platform accepts vNIN and resolves through NIMC API
```

**Adapter Interface:**

```typescript
interface NIMCNINAdapter {
  verifyNIN(request: NINVerificationRequest): Promise<NINVerificationResponse>;
  verifyVirtualNIN(request: VNINVerificationRequest): Promise<NINVerificationResponse>;
}

interface NINVerificationRequest {
  nin: string;             // 11-digit NIN
  dateOfBirth?: string;
  requestId: string;
  consentRef: string;
}

interface VNINVerificationRequest {
  virtualNIN: string;      // 16-character vNIN
  requestId: string;
  consentRef: string;
}

interface NINVerificationResponse {
  success: boolean;
  ninValid: boolean;
  data?: {
    firstName: string;
    lastName: string;
    middleName?: string;
    dateOfBirth: string;
    gender: string;
    phone: string;
    photo?: string;
    address?: string;
    birthState?: string;
    birthLGA?: string;
    residenceState?: string;
    residenceLGA?: string;
  };
  error?: {
    code: string;
    message: string;
  };
}
```

### 2.3 ICAD Profiling (NIBSS)

**Purpose:** Report customer identity to the Individual Customer Account Database per CBN directive.

```
Flow: ICAD Profiling (Post-Onboarding)

After successful KYC verification:
1. KYC Service emits: ivm.customer.identity_verified
2. ICAD Reporter (async consumer) receives event
3. Builds ICAD profile payload:
   - BVN
   - Account/wallet details
   - Verification timestamp
   - Tier classification
4. Submits to NIBSS ICAD API
5. Records ICAD submission status on customer profile
6. Handles errors: queue for retry if ICAD API unavailable

This is a compliance obligation, not a customer-facing workflow.
Runs asynchronously - does not block customer onboarding.
```

---

## 3. Payment Integrations

### 3.1 Payment Gateway Adapters

**Multi-Gateway Strategy:** Support multiple gateways for redundancy and optimal routing.

```
┌──────────────────────────────────────────────────────────┐
│              PAYMENT ORCHESTRATOR                         │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │             ROUTING ENGINE                        │    │
│  │                                                   │    │
│  │  Route by:                                        │    │
│  │  - Payment method (card → Paystack, bank → NIBSS) │    │
│  │  - Amount (micro-payments → specific gateway)     │    │
│  │  - Tenant preference (configurable per merchant)  │    │
│  │  - Gateway health (failover on outage)            │    │
│  │  - Cost optimization (lowest fee routing)         │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────┐ ┌──────────┐ ┌─────────┐ ┌─────────────┐  │
│  │Paystack │ │Flutterwave│ │  NIBSS  │ │Mobile Money │  │
│  │Adapter  │ │ Adapter   │ │NIP Adptr│ │  Adapters   │  │
│  │         │ │           │ │         │ │ ├── OPay    │  │
│  │- Card   │ │- Card     │ │- Bank   │ │ ├── Moniepoint│ │
│  │- Transfer││- Transfer │ │  Transfer│ │ └── PalmPay │  │
│  │- USSD   │ │- Mobile   │ │- Direct │ │             │  │
│  │- QR     │ │  Money    │ │  Debit  │ │             │  │
│  └─────────┘ └──────────┘ └─────────┘ └─────────────┘  │
└──────────────────────────────────────────────────────────┘
```

**Paystack Adapter (Primary):**

```typescript
interface PaystackAdapter implements IPaymentGateway {
  // Card payment (tokenized)
  chargeCard(request: CardChargeRequest): Promise<ChargeResponse>;

  // Bank transfer
  initializeTransfer(request: TransferRequest): Promise<TransferResponse>;

  // Verify transaction
  verifyTransaction(reference: string): Promise<VerificationResponse>;

  // Create customer
  createCustomer(request: CreateCustomerRequest): Promise<CustomerResponse>;

  // Refund
  createRefund(request: RefundRequest): Promise<RefundResponse>;

  // Webhook verification
  verifyWebhookSignature(payload: string, signature: string): boolean;

  // Subaccount (for merchant settlements)
  createSubaccount(request: SubaccountRequest): Promise<SubaccountResponse>;
}

interface CardChargeRequest {
  amount: number;          // Amount in kobo (NGN minor units)
  email: string;
  authorizationCode: string;  // Tokenized card reference
  reference: string;       // Unique transaction reference
  metadata: {
    orderId: string;
    tenantId: string;
    siteId: string;
    deviceId?: string;
  };
  subaccount?: string;     // For split payments
  bearer?: 'account' | 'subaccount';
}
```

### 3.2 Settlement & Reconciliation

```
Daily Settlement Flow:

┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│  End of Day    │────▶│  Reconcile     │────▶│  Calculate     │
│  Trigger       │     │  Transactions  │     │  Settlements   │
│  (00:00 UTC+1) │     │  vs Gateway    │     │  per Merchant  │
└────────────────┘     └────────────────┘     └───────┬────────┘
                                                       │
┌────────────────┐     ┌────────────────┐     ┌───────┴────────┐
│  Record in     │◀────│  Execute       │◀────│  Deduct        │
│  Ledger        │     │  Payouts       │     │  Commissions   │
└────────────────┘     └────────────────┘     └────────────────┘

Settlement Calculation:
  Gross Revenue = Sum(captured payments for period)
  Platform Commission = Gross × commission_rate (per tenant config)
  Gateway Fees = Sum(gateway fees for period)
  Net Payout = Gross - Platform Commission - Gateway Fees

  Output: SettlementRecord per merchant per period
```

---

## 4. Telecom Integrations (SIM Registration)

### 4.1 SIM Registration Workflow

```
Complete SIM Registration Flow at Kiosk:

┌──────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│Select│→│ Verify │→│Capture │→│ Link  │→│Process│→│Dispense│
│Telco │  │Identity│  │Biometr.│  │NIN to │  │Payment│  │SIM Card│
│& Plan│  │(BVN/NIN│  │(finger │  │SIM    │  │       │  │& Print │
│      │  │  )     │  │ /photo)│  │       │  │       │  │Receipt │
└──────┘  └────────┘  └────────┘  └────────┘  └────────┘  └────────┘
     │         │           │          │          │           │
     ▼         ▼           ▼          ▼          ▼           ▼
  Catalog   KYC Svc    Biometric  Telco API  Payment    Vending
  Service   → NIBSS    Scanner    Adapter    Orchestr.  Motor +
            → NIMC     Driver                           Printer
```

**Telco Adapter Interface:**

```typescript
interface TelcoAdapter {
  // Check SIM availability
  checkSIMAvailability(operator: string, plan: string): Promise<AvailabilityResponse>;

  // Register SIM with NCC-compliant data
  registerSIM(request: SIMRegistrationRequest): Promise<SIMRegistrationResponse>;

  // Activate SIM
  activateSIM(request: SIMActivationRequest): Promise<SIMActivationResponse>;

  // Get registration status
  getRegistrationStatus(registrationId: string): Promise<RegistrationStatusResponse>;
}

interface SIMRegistrationRequest {
  operator: 'MTN' | 'AIRTEL' | 'GLO' | '9MOBILE';
  simICCID: string;          // SIM card ICCID
  msisdn?: string;           // Phone number (if pre-assigned)
  plan: string;              // Tariff plan code

  // Identity (NCC-mandated)
  nin: string;               // Customer's NIN (verified)
  bvn?: string;              // Customer's BVN (verified)
  firstName: string;
  lastName: string;
  dateOfBirth: string;
  gender: string;
  address: string;
  photo: string;             // Base64 (captured at kiosk)
  fingerprint?: string;      // Base64 template

  // Agent info (kiosk as registered agent)
  agentId: string;
  agentSiteId: string;
  registrationTimestamp: string;

  // Audit
  requestId: string;
  consentRef: string;
}

interface SIMRegistrationResponse {
  success: boolean;
  registrationId: string;
  msisdn?: string;           // Assigned phone number
  activationStatus: 'PENDING' | 'ACTIVATED' | 'FAILED';
  nccRegistrationRef?: string;
  error?: {
    code: string;
    message: string;
    retryable: boolean;
  };
}
```

### 4.2 Telco-Specific Configurations

```yaml
# /config/integrations/telco.yml
telco_adapters:
  mtn:
    name: "MTN Nigeria"
    base_url: "${MTN_API_BASE_URL}"
    auth_type: "oauth2_client_credentials"
    credentials_ref: "vault://integrations/mtn"
    timeout_ms: 10000
    retry_count: 3
    circuit_breaker:
      failure_threshold: 5
      reset_timeout_ms: 30000
    sim_types: ["PREPAID", "POSTPAID"]
    plans:
      - code: "MTN_PULSE"
        name: "MTN Pulse"
        price: { amount: 20000, currency: "NGN" }
      - code: "MTN_XTRA_TALK"
        name: "MTN XtraTalk"
        price: { amount: 20000, currency: "NGN" }

  airtel:
    name: "Airtel Nigeria"
    base_url: "${AIRTEL_API_BASE_URL}"
    auth_type: "api_key"
    credentials_ref: "vault://integrations/airtel"
    timeout_ms: 10000
    retry_count: 3
    circuit_breaker:
      failure_threshold: 5
      reset_timeout_ms: 30000
    sim_types: ["PREPAID"]
    plans:
      - code: "AIRTEL_SMART_CONNECT"
        name: "Airtel SmartConnect"
        price: { amount: 20000, currency: "NGN" }

  glo:
    name: "Globacom"
    # ... similar structure

  9mobile:
    name: "9mobile"
    # ... similar structure
```

---

## 5. Insurance Integrations

### 5.1 Insurance Purchase Workflow

```
┌──────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│Select│→│Customer│→│ Risk   │→│ Quote │→│Process│→│ Issue │
│Product│  │Identity│  │Assess- │  │& Accept│  │Payment│  │Policy  │
│& Type│  │(if new)│  │ment    │  │        │  │       │  │& Print │
└──────┘  └────────┘  └────────┘  └────────┘  └────────┘  └────────┘
```

**Insurance Adapter Interface:**

```typescript
interface InsuranceAdapter {
  // Get available products
  getProducts(category: string): Promise<InsuranceProduct[]>;

  // Generate quote
  generateQuote(request: QuoteRequest): Promise<QuoteResponse>;

  // Purchase policy
  purchasePolicy(request: PolicyPurchaseRequest): Promise<PolicyResponse>;

  // Get policy details
  getPolicy(policyId: string): Promise<PolicyDetails>;

  // Cancel policy
  cancelPolicy(policyId: string, reason: string): Promise<CancellationResponse>;
}

interface QuoteRequest {
  productCode: string;
  customerDetails: {
    firstName: string;
    lastName: string;
    dateOfBirth: string;
    phone: string;
    email?: string;
  };
  riskDetails: JSON;       // Product-specific (e.g., vehicle info for motor)
  coverageStartDate: string;
  coverageDuration: string; // "1Y", "6M", "3M"
}

interface PolicyResponse {
  success: boolean;
  policyNumber: string;
  certificateUrl?: string;  // PDF download URL
  startDate: string;
  endDate: string;
  premium: { amount: number; currency: string };
  underwriter: string;
}
```

---

## 6. Logistics Integrations (Smart Lockers)

### 6.1 Courier Integration

```typescript
interface CourierAdapter {
  // Notify courier of locker assignment
  notifyDeposit(request: DepositNotification): Promise<void>;

  // Track parcel status
  trackParcel(trackingNumber: string): Promise<TrackingResponse>;

  // Request collection (for returns/expired parcels)
  requestCollection(request: CollectionRequest): Promise<CollectionResponse>;

  // Webhook receiver for parcel status updates
  handleWebhook(event: CourierWebhookEvent): Promise<void>;
}

interface DepositNotification {
  courierId: string;
  trackingNumber: string;
  lockerBankId: string;
  siteAddress: string;
  compartmentNumber: string;
  accessCode: string;       // Courier's access code (different from recipient's)
  depositDeadline: string;  // When the compartment reservation expires
}
```

### 6.2 E-Commerce Platform Integration

```typescript
interface ECommerceAdapter {
  // Receive order notification (order placed → locker selected as delivery)
  handleOrderPlaced(event: OrderPlacedEvent): Promise<void>;

  // Update order status (deposited, picked up, returned)
  updateOrderStatus(orderId: string, status: string): Promise<void>;

  // Sync available locker locations (for checkout display)
  syncLockerLocations(request: LocationSyncRequest): Promise<void>;
}
```

---

## 7. Integration Resilience Patterns

### 7.1 Adapter Resilience Configuration

```yaml
# Standard resilience config applied to all external adapters
adapter_resilience:
  circuit_breaker:
    failure_threshold: 5       # Open after 5 consecutive failures
    success_threshold: 3       # Close after 3 successes in half-open
    reset_timeout_ms: 30000    # Try half-open after 30s
    monitor_window_ms: 60000   # Sliding window for failure count

  retry:
    max_attempts: 3
    backoff_type: "exponential_with_jitter"
    initial_delay_ms: 1000
    max_delay_ms: 10000
    retryable_errors: [500, 502, 503, 504, "TIMEOUT", "CONNECTION_REFUSED"]

  timeout:
    connect_ms: 5000
    read_ms: 15000
    total_ms: 20000

  bulkhead:
    max_concurrent: 50         # Max concurrent calls to this adapter
    queue_size: 100            # Queue when at max concurrent
    queue_timeout_ms: 5000     # Reject if queued too long

  fallback:
    strategy: "QUEUE_FOR_RETRY"  # or "CACHED_RESPONSE" or "DEGRADED_MODE"
    queue_ttl_hours: 24
```

### 7.2 Integration Monitoring Dashboard

```
┌─────────────────────────────────────────────────────────────────┐
│              INTEGRATION HEALTH DASHBOARD                        │
│                                                                  │
│  NIBSS BVN API:  ████████░░ 82% success │ P99: 2.3s │ ⚠ SLOW  │
│  NIMC NIN API:   ██████████ 99% success │ P99: 1.1s │ ✓ OK    │
│  Paystack:       ██████████ 99% success │ P99: 0.8s │ ✓ OK    │
│  Flutterwave:    █████████░ 95% success │ P99: 1.5s │ ✓ OK    │
│  MTN SIM API:    ████████░░ 85% success │ P99: 3.1s │ ⚠ SLOW  │
│  Airtel SIM API: ██████████ 98% success │ P99: 1.2s │ ✓ OK    │
│  GIG Logistics:  █████████░ 93% success │ P99: 2.0s │ ✓ OK    │
│                                                                  │
│  Circuit Breakers: 0 OPEN │ 1 HALF-OPEN (NIBSS BVN) │ 6 CLOSED │
│  Retry Queue:     23 pending │ 5 failed (after max retries)     │
│  Last 24h Errors: 142 total │ 89 transient │ 53 permanent      │
└─────────────────────────────────────────────────────────────────┘
```
