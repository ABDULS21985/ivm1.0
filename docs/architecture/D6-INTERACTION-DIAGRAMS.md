# D6 - Interaction Diagrams

## Overview

This document provides formal sequence diagrams for the three core commerce workflows of the IVM Platform:
1. **Kiosk Service Purchase** (SIM Registration)
2. **Smart Locker Parcel Pickup**
3. **Autonomous Store Checkout**

Each diagram shows the full end-to-end flow including edge, cloud, device, and external system interactions.

---

## 1. Kiosk Service Purchase — SIM Registration

This sequence shows a customer purchasing and registering a SIM card at a kiosk, which is the most complex service vending workflow as it involves KYC, external telco integration, hardware dispensing, and payment.

```mermaid
sequenceDiagram
    autonumber
    participant C as Customer
    participant KUI as Kiosk Touchscreen
    participant EC as Edge Controller
    participant BFF as BFF-Kiosk
    participant WF as Workflow Engine
    participant CAT as Catalog Service
    participant KYC as KYC Service
    participant NIBSS as NIBSS (External)
    participant ORD as Order Service
    participant PAY as Payment Service
    participant PSP as Paystack (External)
    participant TELCO as MTN API (External)
    participant DEV as Device Service
    participant NOTIF as Notification Service
    participant AUDIT as Audit Service

    Note over C, AUDIT: Phase 1 — Service Selection
    C->>KUI: Tap "Buy SIM Card"
    KUI->>EC: GET /local/sessions (create session)
    EC->>BFF: GET /catalog/items?channelType=KIOSK&category=sim-cards
    BFF->>CAT: GET /api/v1/catalog/items?channelType=KIOSK&category=sim-cards
    CAT-->>BFF: CatalogItem[] (MTN, Airtel, Glo, 9mobile)
    BFF-->>EC: Available SIM products
    EC-->>KUI: Display SIM options
    C->>KUI: Select "MTN Prepaid SIM"

    Note over C, AUDIT: Phase 2 — Identity Verification (KYC)
    KUI->>C: Prompt: "Enter BVN for NCC compliance"
    C->>KUI: Enter BVN + Date of Birth
    KUI->>EC: POST /local/sessions/:id/advance (kyc step)
    EC->>BFF: POST /customers/:id/kyc/verify-bvn
    BFF->>KYC: POST /api/v1/customers/:id/kyc/verify-bvn
    KYC->>NIBSS: POST /bvn/verify (BVN lookup)
    NIBSS-->>KYC: BVN match result (firstName, lastName, DOB, phone)
    KYC-->>BFF: { status: "VERIFIED", kycTier: "TIER_1" }
    KYC--)AUDIT: Event: ivm.customer.identity_verified
    BFF-->>EC: KYC verified
    EC-->>KUI: "Identity verified ✓"

    Note over C, AUDIT: Phase 3 — Biometric Capture (NCC Compliance)
    KUI->>C: Prompt: "Place finger on scanner"
    C->>KUI: Fingerprint scan
    KUI->>EC: Biometric data captured
    EC->>BFF: POST /customers/:id/kyc/biometric
    BFF->>KYC: POST /api/v1/customers/:id/kyc/biometric
    KYC-->>BFF: Biometric stored
    KYC--)AUDIT: Event: ivm.customer.identity_evidence_added

    Note over C, AUDIT: Phase 4 — Order Creation & Payment
    EC->>BFF: POST /orders (Idempotency-Key)
    BFF->>ORD: POST /api/v1/orders
    ORD-->>BFF: { orderId: "ord_01HX...", status: "DRAFT", total: { amount: 50000, currency: "NGN" } }
    ORD--)WF: Event: ivm.order.created → Start workflow
    BFF-->>EC: Order created
    EC-->>KUI: "Total: ₦500.00 — Insert card or tap to pay"

    C->>KUI: Insert debit card
    KUI->>EC: Card data (EMV chip read via card reader)
    EC->>BFF: POST /orders/:id/confirm
    BFF->>ORD: POST /api/v1/orders/:id/confirm
    ORD->>PAY: POST /api/v1/payments (initiate)
    PAY->>PSP: POST /transaction/charge (tokenized)
    PSP-->>PAY: { status: "success", reference: "trx_abc123" }
    PAY-->>ORD: Payment authorized
    PAY--)ORD: Event: ivm.payment.authorized
    ORD--)WF: Event: ivm.order.payment_authorized
    PAY--)AUDIT: Event: ivm.payment.authorized
    BFF-->>EC: Payment successful
    EC-->>KUI: "Payment successful ✓"

    Note over C, AUDIT: Phase 5 — SIM Activation & Dispensing
    WF->>BFF: Advance to fulfillment step
    BFF->>TELCO: POST /sim/activate (ICCID, customer details)
    TELCO-->>BFF: { msisdn: "+2348012345678", status: "ACTIVATED" }
    BFF->>DEV: POST /api/v1/devices/:id/commands (DISPENSE_ITEM)
    DEV--)EC: MQTT command: dispense SIM from slot
    EC->>KUI: Dispense SIM card (vending motor)
    KUI-->>EC: SIM dispensed (sensor confirmed)
    EC--)DEV: MQTT ack: command executed

    Note over C, AUDIT: Phase 6 — Receipt & Completion
    BFF->>DEV: POST /api/v1/devices/:id/commands (PRINT_RECEIPT)
    DEV--)EC: MQTT command: print receipt
    EC->>KUI: Print receipt (thermal printer)
    BFF->>NOTIF: POST /api/v1/notifications/send (SMS receipt)
    NOTIF-->>C: SMS: "SIM activated. MSISDN: +2348012345678"
    ORD--)WF: Event: ivm.order.fulfillment_completed
    ORD--)AUDIT: Event: ivm.order.completed
    EC-->>KUI: "Transaction complete. Take your SIM card."
    C->>KUI: Take SIM card and receipt
```

### Key Observations
- **Offline resilience:** Steps 1-3 (catalog, KYC) can use edge-cached data during cloud partition.
- **Idempotency:** Order creation uses `Idempotency-Key` header to prevent duplicate orders on device retry.
- **PCI scope:** Card data is read by the EMV reader on-device and tokenized before reaching the BFF — raw PAN never enters the platform.
- **NCC compliance:** Biometric + BVN verification required before SIM activation per Nigerian Communications Commission regulations.

---

## 2. Smart Locker Parcel Pickup

This sequence shows a customer picking up an e-commerce parcel from a smart locker after receiving a notification.

```mermaid
sequenceDiagram
    autonumber
    participant C as Customer
    participant PHONE as Customer Phone
    participant LUI as Locker Screen
    participant EC as Edge Controller
    participant BFF as BFF-Locker
    participant LK as Locker Module
    participant ORD as Order Service
    participant DEV as Device Service
    participant NOTIF as Notification Service
    participant AUDIT as Audit Service

    Note over C, AUDIT: Pre-condition — Parcel Deposited by Courier
    Note right of LK: Courier has deposited parcel.<br/>Reservation status: READY_FOR_PICKUP.<br/>Customer received SMS with access code.

    Note over C, AUDIT: Phase 1 — Customer Arrives & Authenticates
    C->>PHONE: Open SMS with pickup code "847291"
    C->>LUI: Tap "Pick Up Parcel"
    LUI->>EC: Display access code entry screen
    C->>LUI: Enter access code "847291"

    LUI->>EC: POST /local/locker/verify-access { code: "847291" }
    EC->>BFF: POST /lockers/reservations/:id/verify-access { code: "847291" }
    BFF->>LK: POST /api/v1/lockers/reservations/:id/verify-access

    alt Access code valid
        LK-->>BFF: { valid: true, reservationId: "rsv_01HX...", compartmentNumber: "B3" }
        BFF-->>EC: Access verified
    else Access code invalid
        LK-->>BFF: { valid: false, attemptsRemaining: 2 }
        BFF-->>EC: Invalid code
        EC-->>LUI: "Invalid code. 2 attempts remaining."
    end

    Note over C, AUDIT: Phase 2 — Door Opens
    BFF->>LK: POST /api/v1/lockers/reservations/:id/pickup
    LK->>DEV: POST /api/v1/devices/:id/commands { type: "UNLOCK_DOOR", compartmentId: "cmp_01HX..." }
    DEV--)EC: MQTT command → unlock compartment B3
    EC->>EC: Device Driver: unlock door B3
    EC-->>LUI: "Compartment B3 is now open. Please take your parcel."
    LK--)AUDIT: Event: ivm.locker.door_opened

    Note over C, AUDIT: Phase 3 — Customer Takes Parcel
    C->>LUI: Opens door B3, takes parcel
    EC->>EC: Door sensor: door opened → weight change detected → door closed
    EC--)DEV: MQTT event: door_closed, weight_delta detected
    DEV--)LK: Event: ivm.locker.door_closed

    Note over C, AUDIT: Phase 4 — Pickup Confirmation & Completion
    LK->>LK: Verify: weight decreased (parcel removed)
    LK->>LK: Update reservation: READY_FOR_PICKUP → PICKED_UP
    LK->>LK: Release compartment: OCCUPIED → AVAILABLE

    LK--)ORD: Event: ivm.locker.parcel_picked_up
    ORD->>ORD: Update order: FULFILLING → COMPLETED
    ORD--)AUDIT: Event: ivm.order.completed

    LK--)NOTIF: Event: ivm.locker.parcel_picked_up
    NOTIF-->>C: SMS: "Parcel picked up from Locker B3 at Victoria Island Hub."
    NOTIF-->>PHONE: Push notification: "Pickup confirmed"

    LK--)AUDIT: Event: ivm.locker.compartment_released
    EC-->>LUI: "Pickup complete. Thank you!"

    Note over C, AUDIT: Phase 5 — Partner Notification (Async)
    LK->>LK: Send webhook to logistics partner
    Note right of LK: POST {partner_url}<br/>{ event: "locker.parcel.picked_up",<br/>  trackingNumber: "NG123456789" }
```

### Key Observations
- **Offline support:** Access code verification can occur locally at the edge if the code was pre-synced. Door unlock is entirely local.
- **Sensor fusion:** Weight sensor + door sensor combine to confirm parcel removal (prevents "opened but not taken" false positives).
- **SLA enforcement:** If pickup doesn't happen before `expiresAt`, the Locker Module triggers a `ivm.locker.reservation_expired` event, escalating to reminders and eventually return-to-sender.
- **Pay-on-pickup:** If `paymentRequired` flag is set, a payment step is inserted between access verification and door unlock.

---

## 3. Autonomous Store Checkout

This sequence shows the full flow from store entry to automatic checkout and payment capture.

```mermaid
sequenceDiagram
    autonumber
    participant C as Customer
    participant APP as Customer App
    participant GATE as Gate Controller
    participant CAM as Camera Array
    participant SHELF as Shelf Sensors
    participant EC as Edge Controller
    participant AI as AI Inference (Edge)
    participant BFF as BFF-Store
    participant STORE as Store Module
    participant PAY as Payment Service
    participant PSP as Paystack (External)
    participant CAT as Catalog Service
    participant NOTIF as Notification Service
    participant AUDIT as Audit Service

    Note over C, AUDIT: Phase 1 — Store Entry
    C->>APP: Open app → "Enter Store"
    APP->>BFF: POST /api/v1/store/sessions
    Note right of APP: { storeId, customerId,<br/>entryMethod: "QR",<br/>preAuthAmount: { amount: 5000000, currency: "NGN" },<br/>paymentMethodId: "tok_01HX..." }

    BFF->>STORE: POST /api/v1/store/sessions
    STORE->>PAY: POST /api/v1/payments (pre-authorize ₦50,000)
    PAY->>PSP: POST /transaction/charge (auth-only)
    PSP-->>PAY: { status: "authorized", reference: "trx_xyz" }
    PAY-->>STORE: Pre-auth successful (pay_01HX...)
    PAY--)AUDIT: Event: ivm.payment.authorized (pre-auth)

    STORE-->>BFF: { sessionId: "ssn_01HX...", status: "ENTRY_AUTHORIZED", gateCommand: { action: "OPEN" } }
    BFF->>EC: Open gate for customer
    EC->>GATE: UNLOCK_DOOR command
    GATE-->>EC: Gate opened
    STORE--)AUDIT: Event: ivm.store.session_started

    APP-->>C: "Welcome! Walk in and shop."
    C->>GATE: Walks through gate
    GATE-->>EC: Gate sensor: person entered
    EC--)STORE: Event: ivm.store.customer_entered
    STORE->>STORE: Session status: ENTRY_AUTHORIZED → SHOPPING

    Note over C, AUDIT: Phase 2 — In-Store Shopping (Continuous)
    loop Real-time tracking (every frame ~30fps)
        CAM->>EC: Video frames
        EC->>AI: Inference pipeline (person tracking + item detection)
        AI-->>EC: Detection results
    end

    Note over C, CAM: Customer picks up a bottle of water
    CAM->>EC: Frame: hand reaches shelf
    SHELF-->>EC: Weight change: -350g on shelf position A3
    EC->>AI: Correlate: vision + shelf sensor
    AI-->>EC: { action: "PICKED", itemSku: "WATER-500ML", confidence: 0.97 }
    EC--)STORE: Event: ivm.store.item_picked
    STORE->>STORE: Add to basket: WATER-500ML (₦200, confidence: 0.97)

    Note over C, CAM: Customer picks up a sandwich
    CAM->>EC: Frame: hand reaches refrigerated shelf
    SHELF-->>EC: Weight change: -180g on shelf position C7
    EC->>AI: Correlate: vision + shelf sensor
    AI-->>EC: { action: "PICKED", itemSku: "SANDWICH-CHICKEN", confidence: 0.94 }
    EC--)STORE: Event: ivm.store.item_picked
    STORE->>STORE: Add to basket: SANDWICH-CHICKEN (₦1,500, confidence: 0.94)

    Note over C, CAM: Customer puts back the water
    CAM->>EC: Frame: hand returns item to shelf
    SHELF-->>EC: Weight change: +350g on shelf position A3
    EC->>AI: Correlate: vision + shelf sensor
    AI-->>EC: { action: "RETURNED", itemSku: "WATER-500ML", confidence: 0.96 }
    EC--)STORE: Event: ivm.store.item_returned
    STORE->>STORE: Remove from basket: WATER-500ML

    Note over C, AUDIT: Phase 3 — Exit & Checkout
    C->>GATE: Walks toward exit
    GATE-->>EC: Exit sensor: person approaching
    CAM->>EC: Frame: customer at exit zone
    EC->>AI: Confirm: person exiting, final basket state
    AI-->>EC: Exit confirmed, person ID matched to session

    EC--)STORE: Event: ivm.store.customer_exited
    STORE->>STORE: Session status: SHOPPING → EXIT_DETECTED

    Note over C, AUDIT: Phase 4 — Basket Reconciliation
    STORE->>STORE: Final basket reconciliation
    STORE->>CAT: Verify item prices (latest catalog)
    CAT-->>STORE: Confirmed prices

    STORE->>STORE: Calculate totals
    Note right of STORE: Basket: 1× SANDWICH-CHICKEN = ₦1,500<br/>Confidence score: 0.94<br/>Minimum threshold: 0.85 ✓<br/>No manual review required

    STORE->>STORE: Session status: EXIT_DETECTED → RECONCILING → CHARGING
    STORE--)AUDIT: Event: ivm.store.basket_reconciled

    Note over C, AUDIT: Phase 5 — Payment Capture
    STORE->>PAY: POST /api/v1/payments/:id/capture { amount: { amount: 150000, currency: "NGN" } }
    Note right of PAY: Capture ₦1,500 against<br/>₦50,000 pre-auth.<br/>Release remaining hold.
    PAY->>PSP: POST /transaction/capture (₦1,500)
    PSP-->>PAY: { status: "captured" }
    PAY-->>STORE: Payment captured
    PAY--)AUDIT: Event: ivm.payment.captured

    STORE->>STORE: Session status: CHARGING → COMPLETED
    STORE--)AUDIT: Event: ivm.store.session_charged

    Note over C, AUDIT: Phase 6 — Receipt Delivery
    STORE--)NOTIF: Event: ivm.store.session_charged
    NOTIF-->>APP: Push notification: "₦1,500 charged for your visit"
    NOTIF-->>C: Email receipt with itemized basket

    Note right of APP: Digital Receipt:<br/>1× Chicken Sandwich — ₦1,500<br/>Total: ₦1,500<br/>Payment: Visa •••4242<br/>Store: IVM AutoMart Lekki
```

### Key Observations
- **Edge-first processing:** All AI inference (person tracking, item detection) runs on-device at the edge (NVIDIA Jetson), achieving <100ms latency.
- **Sensor fusion:** Computer vision + shelf weight sensors corroborate each other for higher confidence scores.
- **Pre-authorization pattern:** ₦50,000 hold is placed at entry; actual basket total (₦1,500) is captured at exit; remaining hold is released.
- **Confidence threshold:** Items below 0.85 confidence trigger `REVIEW_REQUIRED` status, routing to human review with video evidence.
- **No checkout friction:** Customer walks out. Payment is automatic. Receipt arrives on phone.

---

## 4. Exception Flow — Low Confidence Item (Autonomous Store)

```mermaid
sequenceDiagram
    autonumber
    participant STORE as Store Module
    participant AI as AI Inference
    participant REVIEW as Review Dashboard
    participant OP as Store Operator
    participant PAY as Payment Service
    participant C as Customer App

    AI-->>STORE: { action: "PICKED", itemSku: "UNKNOWN", confidence: 0.62 }
    STORE->>STORE: Basket item added with confidence 0.62 (below 0.85 threshold)
    Note right of STORE: Session flagged: requiresReview = true

    STORE->>STORE: Customer exits → Session status: RECONCILING → REVIEW_REQUIRED
    STORE--)REVIEW: Event: ivm.store.session_disputed (auto)
    REVIEW-->>OP: Alert: "Session ssn_01HX... requires review"

    OP->>REVIEW: Open session replay
    REVIEW->>STORE: GET /api/v1/store/sessions/:id/replay
    STORE-->>REVIEW: Video clips + tracking data + shelf sensor readings

    alt Operator confirms item
        OP->>REVIEW: Confirm: "Item is CHIPS-PLANTAIN (₦500)"
        REVIEW->>STORE: POST /api/v1/store/sessions/:id/reconcile { corrections: [...] }
        STORE->>STORE: Update basket, session → CHARGING
        STORE->>PAY: Capture corrected amount
        PAY-->>STORE: Captured
        STORE--)C: Push: "₦2,000 charged (₦1,500 sandwich + ₦500 plantain chips)"
    else Operator removes item (false positive)
        OP->>REVIEW: Remove item (false detection)
        REVIEW->>STORE: POST /api/v1/store/sessions/:id/reconcile { removals: [...] }
        STORE->>STORE: Update basket, session → CHARGING
        STORE->>PAY: Capture original amount (₦1,500)
    end
```

---

## 5. Exception Flow — Locker Pickup SLA Expiry

```mermaid
sequenceDiagram
    autonumber
    participant TIMER as SLA Timer
    participant LK as Locker Module
    participant NOTIF as Notification Service
    participant C as Customer
    participant COURIER as Courier System
    participant AUDIT as Audit Service

    TIMER->>LK: SLA expired for rsv_01HX... (48h elapsed)
    LK--)AUDIT: Event: ivm.locker.reservation_expired

    Note over LK, C: Escalation Level 1 — Reminder
    LK--)NOTIF: Send reminder SMS
    NOTIF-->>C: SMS: "Your parcel at Victoria Island Hub expires in 24h. Pick up code: 847291"

    Note over LK, C: +12 hours — Escalation Level 2
    LK--)NOTIF: Send urgent reminder (SMS + Push)
    NOTIF-->>C: SMS: "URGENT: Parcel expires in 12h"
    NOTIF-->>C: Push notification

    Note over LK, COURIER: +24 hours — Escalation Level 3 (Return to Sender)
    LK->>LK: Reservation status: EXPIRED
    LK->>LK: Compartment status: OCCUPIED → needs retrieval
    LK--)COURIER: Webhook: return-to-sender request
    LK--)AUDIT: Event: ivm.locker.return_initiated

    COURIER-->>LK: Courier dispatched for retrieval
    Note right of LK: Courier picks up parcel.<br/>Compartment released.<br/>Customer notified of return.
```

---

## 6. Offline Kiosk Transaction (Edge Resilience)

```mermaid
sequenceDiagram
    autonumber
    participant C as Customer
    participant KUI as Kiosk Touchscreen
    participant EC as Edge Controller
    participant CACHE as Edge Cache (SQLite)
    participant QUEUE as Offline Queue
    participant CLOUD as Cloud Services

    Note over EC, CLOUD: Network partition detected
    EC->>EC: Cloud connectivity: OFFLINE

    C->>KUI: Select service → "Bill Payment"
    KUI->>EC: Create session
    EC->>CACHE: Load cached catalog (bill payment items)
    CACHE-->>EC: Cached catalog items
    EC-->>KUI: Display bill payment options

    C->>KUI: Pay electricity bill (₦5,000)
    KUI->>EC: Process payment
    EC->>EC: Local card read (EMV)
    EC->>CACHE: Store transaction locally
    EC->>QUEUE: Queue: { type: "order", data: {...}, timestamp: "..." }
    EC-->>KUI: "Payment queued. Receipt #TXN-EDGE-001"
    EC->>KUI: Print offline receipt (marked "pending confirmation")

    Note over EC, CLOUD: Network restored
    EC->>EC: Cloud connectivity: ONLINE
    EC->>CLOUD: Replay queued transactions (FIFO)
    QUEUE->>CLOUD: POST /api/v1/orders (with idempotency key)
    CLOUD-->>EC: Order confirmed + payment processed
    EC->>QUEUE: Mark transaction synced
    EC->>EC: Reconcile local state with cloud state

    Note right of EC: If payment fails on sync:<br/>Compensation flow triggered.<br/>Customer notified of reversal.
```
