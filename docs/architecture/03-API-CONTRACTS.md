# 03 - API Contracts & Service Definitions

## 1. API Design Standards

| Aspect | Standard |
|---|---|
| Protocol | REST (external), gRPC (internal service-to-service) |
| Versioning | URL path: `/api/v1/...` |
| Authentication | Bearer JWT (user/operator), mTLS (device), API Key (partner) |
| Pagination | Cursor-based (`?cursor=xxx&limit=25`) |
| Filtering | Query parameters (`?status=ACTIVE&type=KIOSK`) |
| Sorting | `?sort=created_at:desc` |
| Error Format | RFC 7807 Problem Details |
| Date/Time | ISO 8601 UTC (`2026-03-12T14:30:00.000Z`) |
| Money | `{ "amount": 150000, "currency": "NGN" }` (minor units) |
| IDs | Prefixed ULIDs (`ord_01HX...`) |
| Rate Limiting | Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` |

### Error Response Format

```json
{
  "type": "https://api.ivm.io/errors/payment-failed",
  "title": "Payment Authorization Failed",
  "status": 402,
  "detail": "Card issuer declined the transaction. Please try a different payment method.",
  "instance": "/api/v1/payments/pay_01HX.../authorize",
  "traceId": "trace-abc-123",
  "errors": [
    {
      "code": "CARD_DECLINED",
      "field": "paymentMethodId",
      "message": "Card ending in 4242 was declined"
    }
  ]
}
```

---

## 2. Service API Definitions

### 2.1 Identity & Authentication Service

**Base URL:** `/api/v1/auth`

```
POST   /api/v1/auth/register              Register new user
POST   /api/v1/auth/login                 Authenticate (email/phone + password/OTP)
POST   /api/v1/auth/login/otp/request     Request OTP for phone-based login
POST   /api/v1/auth/login/otp/verify      Verify OTP
POST   /api/v1/auth/login/biometric       Authenticate via biometric token
POST   /api/v1/auth/token/refresh         Refresh access token
POST   /api/v1/auth/logout                Invalidate session
POST   /api/v1/auth/password/reset        Initiate password reset
POST   /api/v1/auth/password/change       Change password (authenticated)
POST   /api/v1/auth/mfa/enable            Enable MFA
POST   /api/v1/auth/mfa/verify            Verify MFA challenge
POST   /api/v1/auth/device/enroll         Enroll device (mTLS cert exchange)
POST   /api/v1/auth/device/authenticate   Device authentication
```

**Register User:**
```
POST /api/v1/auth/register
Content-Type: application/json

Request:
{
  "phone": "+2348012345678",
  "email": "user@example.com",
  "firstName": "Ade",
  "lastName": "Johnson",
  "password": "securePassword123!",
  "tenantId": "tnt_01HX..."
}

Response: 201 Created
{
  "userId": "usr_01HX...",
  "phone": "+2348012345678",
  "kycTier": "TIER_0",
  "accessToken": "eyJhbG...",
  "refreshToken": "eyJhbG...",
  "expiresIn": 3600
}
```

### 2.2 Customer & KYC Service

**Base URL:** `/api/v1/customers`

```
GET    /api/v1/customers/:id                  Get customer profile
PATCH  /api/v1/customers/:id                  Update customer profile
GET    /api/v1/customers/:id/kyc              Get KYC status & evidence
POST   /api/v1/customers/:id/kyc/verify-bvn   Initiate BVN verification
POST   /api/v1/customers/:id/kyc/verify-nin   Initiate NIN verification
POST   /api/v1/customers/:id/kyc/verify-doc   Upload document for verification
POST   /api/v1/customers/:id/kyc/biometric    Submit biometric data
GET    /api/v1/customers/:id/kyc/tier          Get current KYC tier
GET    /api/v1/customers/:id/orders            Get customer's order history
GET    /api/v1/customers/:id/payments          Get customer's payment history
```

**BVN Verification:**
```
POST /api/v1/customers/cust_01HX.../kyc/verify-bvn
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "bvn": "22012345678",
  "dateOfBirth": "1990-05-15",
  "consentGranted": true,
  "consentTimestamp": "2026-03-12T14:30:00.000Z",
  "deviceId": "dev_01HX...",
  "siteId": "site_01HX..."
}

Response: 200 OK
{
  "verificationId": "kyc_01HX...",
  "status": "VERIFIED",
  "kycTier": "TIER_1",
  "matchedFields": ["firstName", "lastName", "dateOfBirth", "phone"],
  "verifiedAt": "2026-03-12T14:30:05.000Z",
  "provider": "nibss_bvn"
}

Response: 200 OK (mismatch)
{
  "verificationId": "kyc_01HX...",
  "status": "FAILED",
  "reason": "FIELD_MISMATCH",
  "mismatchedFields": ["dateOfBirth"],
  "canRetry": true,
  "maxRetries": 3,
  "retriesRemaining": 2
}
```

### 2.3 Catalog & Pricing Service

**Base URL:** `/api/v1/catalog`

```
GET    /api/v1/catalog/items                   List catalog items (filterable)
GET    /api/v1/catalog/items/:id               Get catalog item detail
POST   /api/v1/catalog/items                   Create catalog item (operator)
PATCH  /api/v1/catalog/items/:id               Update catalog item (operator)
GET    /api/v1/catalog/categories              List categories
POST   /api/v1/catalog/categories              Create category (operator)
GET    /api/v1/catalog/items/:id/price         Get resolved price (with promotions)
POST   /api/v1/catalog/price/calculate         Calculate price for a basket
GET    /api/v1/catalog/promotions              List active promotions
POST   /api/v1/catalog/promotions              Create promotion (operator)
```

**List Catalog Items (with site/channel filtering):**
```
GET /api/v1/catalog/items?channelType=KIOSK&siteId=site_01HX...&category=sim-cards&limit=20
Authorization: Bearer {token}

Response: 200 OK
{
  "items": [
    {
      "id": "cat_01HX...",
      "name": "MTN SIM Card - Prepaid",
      "type": "SIM_CARD",
      "description": "MTN prepaid SIM with free 1GB data",
      "categoryId": "catg_01HX...",
      "imageUrls": ["https://cdn.ivm.io/products/mtn-sim.png"],
      "price": { "amount": 50000, "currency": "NGN" },
      "channelAvailability": ["KIOSK"],
      "requiresKyc": true,
      "requiredKycTier": "TIER_1",
      "attributes": {
        "operator": "MTN",
        "simType": "PREPAID",
        "bundledData": "1GB"
      }
    }
  ],
  "cursor": "eyJpZCI6...",
  "hasMore": true
}
```

### 2.4 Order Service

**Base URL:** `/api/v1/orders`

```
POST   /api/v1/orders                         Create order
GET    /api/v1/orders/:id                      Get order detail
PATCH  /api/v1/orders/:id                      Update order
POST   /api/v1/orders/:id/confirm              Confirm order (trigger payment)
POST   /api/v1/orders/:id/cancel               Cancel order
GET    /api/v1/orders/:id/fulfillment          Get fulfillment status
POST   /api/v1/orders/:id/dispute              Raise dispute
GET    /api/v1/orders                          List orders (filterable)
```

**Create Order:**
```
POST /api/v1/orders
Content-Type: application/json
Authorization: Bearer {token}
Idempotency-Key: idem_01HX...

Request:
{
  "customerId": "cust_01HX...",
  "siteId": "site_01HX...",
  "deviceId": "dev_01HX...",
  "channelType": "KIOSK",
  "lineItems": [
    {
      "catalogItemId": "cat_01HX...",
      "quantity": 1
    }
  ],
  "paymentMethod": {
    "type": "CARD",
    "tokenizedCardId": "tok_01HX..."
  },
  "metadata": {
    "workflowType": "SIM_REGISTRATION",
    "operatorCode": "MTN"
  }
}

Response: 201 Created
{
  "id": "ord_01HX...",
  "status": "DRAFT",
  "lineItems": [...],
  "subtotal": { "amount": 50000, "currency": "NGN" },
  "tax": { "amount": 3750, "currency": "NGN" },
  "total": { "amount": 53750, "currency": "NGN" },
  "createdAt": "2026-03-12T14:30:00.000Z"
}
```

### 2.5 Payment Orchestration Service

**Base URL:** `/api/v1/payments`

```
POST   /api/v1/payments                       Initiate payment
POST   /api/v1/payments/:id/authorize          Authorize payment
POST   /api/v1/payments/:id/capture            Capture authorized payment
POST   /api/v1/payments/:id/void               Void authorized payment
POST   /api/v1/payments/:id/refund             Refund captured payment
GET    /api/v1/payments/:id                     Get payment detail
GET    /api/v1/payments/:id/status              Get payment status
POST   /api/v1/payments/methods                 Add payment method
GET    /api/v1/payments/methods                  List saved payment methods
DELETE /api/v1/payments/methods/:id              Remove payment method
GET    /api/v1/settlements                       List settlement records
GET    /api/v1/settlements/:id                    Get settlement detail
```

**Initiate Payment:**
```
POST /api/v1/payments
Content-Type: application/json
Authorization: Bearer {token}
Idempotency-Key: idem_pay_01HX...

Request:
{
  "orderId": "ord_01HX...",
  "amount": { "amount": 53750, "currency": "NGN" },
  "method": "CARD",
  "tokenizedCardId": "tok_01HX...",
  "captureMode": "AUTOMATIC",
  "callbackUrl": "https://edge.site01.ivm.io/callbacks/payment"
}

Response: 201 Created
{
  "id": "pay_01HX...",
  "status": "INITIATED",
  "amount": { "amount": 53750, "currency": "NGN" },
  "gateway": "paystack",
  "gatewayRef": "trx_abc123",
  "nextAction": null,
  "createdAt": "2026-03-12T14:30:00.000Z"
}
```

### 2.6 Device Management Service

**Base URL:** `/api/v1/devices`

```
POST   /api/v1/devices                        Register device
GET    /api/v1/devices/:id                     Get device detail
PATCH  /api/v1/devices/:id                     Update device config
POST   /api/v1/devices/:id/commands            Send command to device
GET    /api/v1/devices/:id/commands/:cmdId     Get command status
GET    /api/v1/devices/:id/health              Get health status
GET    /api/v1/devices/:id/telemetry           Get telemetry data (time range)
POST   /api/v1/devices/:id/update              Trigger OTA update
GET    /api/v1/devices                         List devices (filterable by site/status/type)
GET    /api/v1/devices/fleet/health             Fleet health summary
POST   /api/v1/devices/fleet/commands           Send command to device group
```

**Send Command to Device:**
```
POST /api/v1/devices/dev_01HX.../commands
Content-Type: application/json
Authorization: Bearer {operator_token}

Request:
{
  "type": "UNLOCK_DOOR",
  "parameters": {
    "compartmentId": "cmp_01HX...",
    "timeoutSeconds": 30
  },
  "correlationId": "rsv_01HX...",
  "priority": "HIGH"
}

Response: 202 Accepted
{
  "commandId": "cmd_01HX...",
  "status": "PENDING",
  "deviceId": "dev_01HX...",
  "type": "UNLOCK_DOOR",
  "createdAt": "2026-03-12T14:30:00.000Z",
  "expectedCompletionMs": 2000
}
```

### 2.7 Locker Module Service

**Base URL:** `/api/v1/lockers`

```
GET    /api/v1/lockers/banks                   List locker banks
GET    /api/v1/lockers/banks/:id               Get locker bank detail
GET    /api/v1/lockers/banks/:id/availability  Get compartment availability
POST   /api/v1/lockers/reservations            Create reservation
GET    /api/v1/lockers/reservations/:id        Get reservation detail
POST   /api/v1/lockers/reservations/:id/deposit   Record deposit (courier)
POST   /api/v1/lockers/reservations/:id/pickup     Initiate pickup (customer)
POST   /api/v1/lockers/reservations/:id/cancel     Cancel reservation
GET    /api/v1/lockers/reservations/:id/access-code  Get access code (authenticated)
POST   /api/v1/lockers/reservations/:id/verify-access  Verify access code at locker
```

**Create Reservation:**
```
POST /api/v1/lockers/reservations
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "lockerBankId": "lbk_01HX...",
  "orderId": "ord_01HX...",
  "recipientId": "cust_01HX...",
  "size": "MEDIUM",
  "type": "PICKUP",
  "accessMethod": "QR",
  "pickupSlaMinutes": 2880,
  "metadata": {
    "trackingNumber": "NG123456789",
    "courierCompany": "GIG Logistics"
  }
}

Response: 201 Created
{
  "id": "rsv_01HX...",
  "compartmentId": "cmp_01HX...",
  "compartmentNumber": "B3",
  "status": "RESERVED",
  "accessCode": "847291",
  "accessMethod": "QR",
  "qrCodeUrl": "https://api.ivm.io/qr/rsv_01HX...",
  "expiresAt": "2026-03-14T14:30:00.000Z",
  "createdAt": "2026-03-12T14:30:00.000Z"
}
```

### 2.8 Autonomous Store Module Service

**Base URL:** `/api/v1/store`

```
POST   /api/v1/store/sessions                  Create shopping session (entry)
GET    /api/v1/store/sessions/:id              Get session detail
POST   /api/v1/store/sessions/:id/exit         Record exit
GET    /api/v1/store/sessions/:id/basket       Get current basket
POST   /api/v1/store/sessions/:id/reconcile    Trigger manual reconciliation
POST   /api/v1/store/sessions/:id/charge       Charge for session
POST   /api/v1/store/sessions/:id/dispute      Customer disputes session
GET    /api/v1/store/sessions/:id/replay       Get session replay data
GET    /api/v1/store/:storeId/inventory        Get real-time store inventory
GET    /api/v1/store/:storeId/analytics        Get store analytics
```

**Create Shopping Session (Entry):**
```
POST /api/v1/store/sessions
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "storeId": "site_01HX...",
  "customerId": "cust_01HX...",
  "entryMethod": "QR",
  "preAuthAmount": { "amount": 5000000, "currency": "NGN" },
  "paymentMethodId": "tok_01HX..."
}

Response: 201 Created
{
  "id": "ssn_01HX...",
  "status": "ENTRY_AUTHORIZED",
  "gateCommand": {
    "action": "OPEN",
    "gateDeviceId": "dev_01HX...",
    "openDurationSeconds": 10
  },
  "preAuthPaymentId": "pay_01HX...",
  "enteredAt": "2026-03-12T14:30:00.000Z"
}
```

### 2.9 Workflow Service

**Base URL:** `/api/v1/workflows`

```
GET    /api/v1/workflows/definitions               List workflow definitions
POST   /api/v1/workflows/definitions               Create workflow definition
GET    /api/v1/workflows/definitions/:id            Get workflow definition
PATCH  /api/v1/workflows/definitions/:id            Update workflow definition
POST   /api/v1/workflows/instances                  Start workflow instance
GET    /api/v1/workflows/instances/:id              Get workflow instance
POST   /api/v1/workflows/instances/:id/advance      Advance to next step
POST   /api/v1/workflows/instances/:id/cancel       Cancel workflow
GET    /api/v1/workflows/instances/:id/history       Get step execution history
```

### 2.10 Notification Service

**Base URL:** `/api/v1/notifications`

```
POST   /api/v1/notifications/send               Send notification
GET    /api/v1/notifications/:id                 Get notification status
POST   /api/v1/notifications/templates           Create template
GET    /api/v1/notifications/templates            List templates
PATCH  /api/v1/notifications/templates/:id        Update template
GET    /api/v1/notifications/preferences/:userId   Get user preferences
PATCH  /api/v1/notifications/preferences/:userId   Update user preferences
```

### 2.11 Tenant Management Service

**Base URL:** `/api/v1/tenants`

```
POST   /api/v1/tenants                        Create tenant
GET    /api/v1/tenants/:id                     Get tenant detail
PATCH  /api/v1/tenants/:id                     Update tenant
GET    /api/v1/tenants/:id/sites               List tenant's sites
GET    /api/v1/tenants/:id/merchants           List sub-merchants
GET    /api/v1/tenants/:id/analytics           Tenant analytics
PATCH  /api/v1/tenants/:id/branding            Update branding
PATCH  /api/v1/tenants/:id/features            Update feature flags
GET    /api/v1/tenants/:id/billing             Get billing/usage info
```

### 2.12 Audit Service

**Base URL:** `/api/v1/audit`

```
GET    /api/v1/audit/entries                  Query audit entries (filterable)
GET    /api/v1/audit/entries/:id              Get audit entry detail
GET    /api/v1/audit/report                   Generate compliance report
GET    /api/v1/audit/chain/verify             Verify hash chain integrity
```

---

## 3. Internal gRPC Service Definitions

For high-throughput internal communication, services use gRPC:

```protobuf
// identity.proto
service IdentityService {
  rpc ValidateToken(ValidateTokenRequest) returns (ValidateTokenResponse);
  rpc GetUserPermissions(GetPermissionsRequest) returns (PermissionsResponse);
  rpc AuthenticateDevice(DeviceAuthRequest) returns (DeviceAuthResponse);
}

// kyc.proto
service KycService {
  rpc VerifyBVN(BVNVerificationRequest) returns (VerificationResponse);
  rpc VerifyNIN(NINVerificationRequest) returns (VerificationResponse);
  rpc GetKycStatus(KycStatusRequest) returns (KycStatusResponse);
  rpc CheckKycTier(KycTierCheckRequest) returns (KycTierCheckResponse);
}

// payment.proto
service PaymentService {
  rpc InitiatePayment(InitiatePaymentRequest) returns (PaymentResponse);
  rpc AuthorizePayment(AuthorizeRequest) returns (PaymentResponse);
  rpc CapturePayment(CaptureRequest) returns (PaymentResponse);
  rpc RefundPayment(RefundRequest) returns (PaymentResponse);
  rpc GetPaymentStatus(PaymentStatusRequest) returns (PaymentStatusResponse);
}

// device.proto
service DeviceService {
  rpc SendCommand(DeviceCommandRequest) returns (CommandResponse);
  rpc GetDeviceStatus(DeviceStatusRequest) returns (DeviceStatusResponse);
  rpc ReportHealth(stream HealthReport) returns (HealthAck);
  rpc StreamTelemetry(stream TelemetryEvent) returns (TelemetryAck);
}

// workflow.proto
service WorkflowService {
  rpc StartWorkflow(StartWorkflowRequest) returns (WorkflowInstanceResponse);
  rpc AdvanceStep(AdvanceStepRequest) returns (WorkflowInstanceResponse);
  rpc GetState(GetWorkflowStateRequest) returns (WorkflowStateResponse);
  rpc CancelWorkflow(CancelWorkflowRequest) returns (WorkflowInstanceResponse);
}

// notification.proto
service NotificationService {
  rpc Send(SendNotificationRequest) returns (NotificationResponse);
  rpc SendBatch(SendBatchRequest) returns (BatchNotificationResponse);
}
```

---

## 4. Webhook / Callback Contracts

### 4.1 Payment Gateway Callbacks

```
POST /callbacks/payment/{gateway}
Content-Type: application/json
X-Signature: {hmac_signature}

{
  "event": "charge.success",
  "data": {
    "reference": "pay_01HX...",
    "amount": 53750,
    "currency": "NGN",
    "status": "success",
    "gatewayRef": "trx_abc123",
    "cardLast4": "4242",
    "bank": "First Bank"
  }
}
```

### 4.2 Partner Integration Webhooks (Outbound)

IVM sends webhooks to integrated partners (e-commerce platforms, logistics companies):

```
POST {partner_webhook_url}
Content-Type: application/json
X-IVM-Signature: {hmac_sha256}
X-IVM-Event: locker.parcel.picked_up

{
  "eventId": "evt_01HX...",
  "eventType": "locker.parcel.picked_up",
  "timestamp": "2026-03-12T14:30:00.000Z",
  "data": {
    "reservationId": "rsv_01HX...",
    "orderId": "ord_01HX...",
    "trackingNumber": "NG123456789",
    "pickedUpAt": "2026-03-12T14:30:00.000Z",
    "lockerBankId": "lbk_01HX...",
    "siteId": "site_01HX..."
  }
}
```

---

## 5. Edge Controller Local API

The edge controller exposes a local REST API for on-site devices and UI applications. This API mirrors a subset of cloud APIs but operates independently during network partitions.

**Base URL:** `http://edge.local:8080/api/v1`

```
# Device commands (local, low-latency)
POST   /api/v1/local/devices/:id/commands     Send command (no cloud round-trip)
GET    /api/v1/local/devices/:id/status        Get device status

# Session management (for kiosk/store UI)
POST   /api/v1/local/sessions                  Create local session
GET    /api/v1/local/sessions/:id              Get session state
POST   /api/v1/local/sessions/:id/advance      Advance workflow step

# Locker operations (for locker UI)
POST   /api/v1/local/locker/verify-access      Verify access code locally
POST   /api/v1/local/locker/open-door          Open compartment door
GET    /api/v1/local/locker/availability        Get compartment availability

# Health & sync
GET    /api/v1/local/health                    Edge controller health
GET    /api/v1/local/sync/status               Cloud sync status
POST   /api/v1/local/sync/force                Force sync with cloud
```
