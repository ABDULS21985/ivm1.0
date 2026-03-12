# D4 - Event Contract Definitions

## Overview

This document defines the formal event contracts for the IVM Platform (Unified Automated Commerce & Service Platform). All domain events conform to the [CloudEvents v1.0 specification](https://cloudevents.io/) and are transported via Apache Kafka. Device-originated events are bridged from MQTT to Kafka via the MQTT-Kafka Bridge.

This is the authoritative reference for every event produced and consumed across bounded contexts.

---

## Table of Contents

1. [CloudEvents Envelope Standard](#1-cloudevents-envelope-standard)
2. [Event Versioning Strategy](#2-event-versioning-strategy)
3. [Event Ownership Matrix](#3-event-ownership-matrix)
4. [Event Contracts by Domain](#4-event-contracts-by-domain)
   - 4.1 [Identity Events](#41-identity-events)
   - 4.2 [Customer Events](#42-customer-events)
   - 4.3 [Order Events](#43-order-events)
   - 4.4 [Payment Events](#44-payment-events)
   - 4.5 [Device Events](#45-device-events)
   - 4.6 [Locker Events](#46-locker-events)
   - 4.7 [Store Events](#47-store-events)
   - 4.8 [Workflow Events](#48-workflow-events)
   - 4.9 [Notification Events](#49-notification-events)
5. [Event Routing](#5-event-routing)
6. [Dead Letter Queue Contracts](#6-dead-letter-queue-contracts)
7. [MQTT Event Bridge](#7-mqtt-event-bridge)

---

## 1. CloudEvents Envelope Standard

Every event on the IVM Platform uses the CloudEvents v1.0 structured-content mode with JSON encoding. The envelope consists of required CloudEvents attributes plus IVM-specific extension attributes.

### 1.1 Base Envelope Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/cloudevents/envelope.v1.json",
  "title": "IVM CloudEvents Envelope",
  "description": "Base envelope for all IVM Platform domain events, conforming to CloudEvents v1.0 with IVM extensions.",
  "type": "object",
  "required": [
    "specversion",
    "id",
    "source",
    "type",
    "time",
    "datacontenttype",
    "subject",
    "tenantid",
    "traceid",
    "correlationid",
    "data"
  ],
  "properties": {
    "specversion": {
      "type": "string",
      "const": "1.0",
      "description": "CloudEvents specification version."
    },
    "id": {
      "type": "string",
      "pattern": "^evt_[0-9A-HJKMNP-TV-Z]{26}$",
      "description": "Globally unique event identifier. Prefixed ULID (evt_...)."
    },
    "source": {
      "type": "string",
      "format": "uri",
      "description": "URI identifying the service instance that produced the event. Format: ivm://{environment}/services/{service-name}",
      "examples": [
        "ivm://production/services/ivm-identity-service",
        "ivm://production/services/ivm-payment-service",
        "ivm://production/devices/dev_01HX7V8K9M2N3P4Q5R6S7T8U9V"
      ]
    },
    "type": {
      "type": "string",
      "pattern": "^ivm\\.[a-z]+\\.[a-z_]+\\.v[0-9]+$",
      "description": "Event type with version suffix. Format: ivm.{domain}.{event_name}.v{version}",
      "examples": [
        "ivm.payment.authorized.v1",
        "ivm.order.created.v1",
        "ivm.device.online.v1"
      ]
    },
    "time": {
      "type": "string",
      "format": "date-time",
      "description": "Timestamp when the event was produced. ISO 8601 UTC."
    },
    "datacontenttype": {
      "type": "string",
      "const": "application/json",
      "description": "Content type of the data attribute."
    },
    "subject": {
      "type": "string",
      "description": "Subject of the event, identifying the entity. Format: {entity_type}/{entity_id}",
      "examples": [
        "user/usr_01HX7V8K9M2N3P4Q5R6S7T8U9V",
        "order/ord_01HX7V8K9M2N3P4Q5R6S7T8U9V",
        "device/dev_01HX7V8K9M2N3P4Q5R6S7T8U9V"
      ]
    },
    "tenantid": {
      "type": "string",
      "pattern": "^tnt_[0-9A-HJKMNP-TV-Z]{26}$",
      "description": "IVM Extension. Tenant that owns the resource. Prefixed ULID."
    },
    "traceid": {
      "type": "string",
      "description": "IVM Extension. Distributed trace identifier (OpenTelemetry W3C Trace Context)."
    },
    "causationid": {
      "type": "string",
      "pattern": "^evt_[0-9A-HJKMNP-TV-Z]{26}$",
      "description": "IVM Extension. ID of the event that directly caused this event. Null for user-initiated root events."
    },
    "correlationid": {
      "type": "string",
      "description": "IVM Extension. Business correlation identifier linking all events in a single business transaction (e.g., an order lifecycle). Typically the root command or order ID."
    },
    "data": {
      "type": "object",
      "description": "Event-specific payload. Schema varies by event type."
    }
  },
  "additionalProperties": false
}
```

### 1.2 Example Envelope

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-payment-service",
  "type": "ivm.payment.authorized.v1",
  "time": "2026-03-12T14:30:00.000Z",
  "datacontenttype": "application/json",
  "subject": "payment/pay_01J5M2WJ7P6N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "causationid": "evt_01J5M2VH5M4N3P4Q5R6S7T8U9V",
  "correlationid": "ord_01J5M2TF3K2N3P4Q5R6S7T8U9V",
  "data": {
    "paymentId": "pay_01J5M2WJ7P6N3P4Q5R6S7T8U9V",
    "orderId": "ord_01J5M2TF3K2N3P4Q5R6S7T8U9V",
    "customerId": "cst_01J5M2SG2H1N3P4Q5R6S7T8U9V",
    "amount": {
      "amount": 1500000,
      "currency": "NGN"
    },
    "method": "CARD",
    "gateway": "paystack",
    "gatewayRef": "PSK_abc123xyz"
  }
}
```

### 1.3 Attribute Summary Table

| Attribute | CloudEvents | Required | Description |
|---|---|---|---|
| `specversion` | Required | Yes | Always `"1.0"` |
| `id` | Required | Yes | Prefixed ULID `evt_...` |
| `source` | Required | Yes | URI of producing service or device |
| `type` | Required | Yes | Versioned event type `ivm.{domain}.{name}.v{N}` |
| `time` | Optional* | Yes | ISO 8601 UTC timestamp (*required for IVM) |
| `datacontenttype` | Optional* | Yes | Always `"application/json"` (*required for IVM) |
| `subject` | Optional* | Yes | Entity reference `{type}/{id}` (*required for IVM) |
| `tenantid` | Extension | Yes | Tenant ULID for multi-tenancy isolation |
| `traceid` | Extension | Yes | W3C Trace Context trace-id |
| `causationid` | Extension | No | Event ID of the direct cause |
| `correlationid` | Extension | Yes | Business transaction correlation ID |
| `data` | N/A | Yes | Event-specific JSON payload |

---

## 2. Event Versioning Strategy

### 2.1 Version Encoding

Event versions are encoded in the `type` attribute as a suffix:

```
ivm.{domain}.{event_name}.v{major_version}
```

Examples:
- `ivm.payment.authorized.v1`
- `ivm.order.created.v2`
- `ivm.device.health_reported.v1`

### 2.2 Versioning Rules

| Rule | Description |
|---|---|
| **Additive changes are non-breaking** | Adding new optional fields to `data` does NOT require a version bump. Consumers MUST ignore unknown fields. |
| **Removing or renaming fields is breaking** | Requires a new major version (`v1` -> `v2`). |
| **Changing field types is breaking** | Requires a new major version. |
| **Changing semantics is breaking** | Even if the shape is identical, a meaning change requires a new version. |
| **Parallel versions** | When `v2` is introduced, `v1` continues to be published for a deprecation period (minimum 90 days). Both versions are published simultaneously to separate Kafka topics or with type-based filtering. |

### 2.3 Schema Registry

All event schemas are registered in a Confluent-compatible Schema Registry:

- **Subject naming:** `{kafka_topic}-value` (e.g., `ivm.payment.events-value`)
- **Compatibility mode:** `FORWARD_TRANSITIVE` -- new schemas can add optional fields; consumers with older schemas can still deserialize.
- **Schema format:** JSON Schema (matching this document).

### 2.4 Deprecation Lifecycle

```
v1 ACTIVE ──────────────────────────────────────────────────────────────►
                    v2 introduced          v1 deprecated         v1 removed
                         │                      │                     │
                         ▼                      ▼                     ▼
v2          ─────────────┼──────────────────────┼─────────────────────┼──►
                         │◄──── dual publish ──►│◄── v1 consumers ──►│
                         │      (min 90 days)   │    must migrate     │
```

| Phase | Duration | Behavior |
|---|---|---|
| Dual Publish | >= 90 days | Both `v1` and `v2` events published. Consumers can read either. |
| v1 Deprecated | >= 30 days | `v1` still published but marked deprecated in schema registry. Alerts sent to consuming teams. |
| v1 Removed | After migration | `v1` no longer published. Dead-letter any unmigrated consumers. |

---

## 3. Event Ownership Matrix

### 3.1 Identity Events

| Event Type | Producer Service | Consumer Services | Kafka Topic | Retention |
|---|---|---|---|---|
| `ivm.user.created.v1` | `ivm-identity-service` | `ivm-customer-service`, `ivm-audit-service` | `ivm.identity.events` | 30 days |
| `ivm.user.authenticated.v1` | `ivm-identity-service` | `ivm-audit-service`, `ivm-analytics-service` | `ivm.identity.events` | 7 days |
| `ivm.user.authentication_failed.v1` | `ivm-identity-service` | `ivm-audit-service`, `ivm-notification-service` | `ivm.identity.events` | 30 days |
| `ivm.user.suspended.v1` | `ivm-identity-service` | `ivm-customer-service`, `ivm-order-service`, `ivm-audit-service` | `ivm.identity.events` | 30 days |
| `ivm.user.reactivated.v1` | `ivm-identity-service` | `ivm-customer-service`, `ivm-audit-service` | `ivm.identity.events` | 30 days |
| `ivm.device.enrolled.v1` | `ivm-identity-service` | `ivm-device-service`, `ivm-audit-service` | `ivm.identity.events` | 30 days |
| `ivm.device.authenticated.v1` | `ivm-identity-service` | `ivm-device-service`, `ivm-audit-service` | `ivm.identity.events` | 7 days |
| `ivm.device.certificate_rotated.v1` | `ivm-identity-service` | `ivm-device-service`, `ivm-audit-service` | `ivm.identity.events` | 90 days |

### 3.2 Customer Events

| Event Type | Producer Service | Consumer Services | Kafka Topic | Retention |
|---|---|---|---|---|
| `ivm.customer.created.v1` | `ivm-customer-service` | `ivm-kyc-service`, `ivm-audit-service`, `ivm-analytics-service` | `ivm.customer.events` | 30 days |
| `ivm.customer.updated.v1` | `ivm-customer-service` | `ivm-audit-service`, `ivm-analytics-service` | `ivm.customer.events` | 30 days |
| `ivm.customer.kyc_tier_changed.v1` | `ivm-customer-service` | `ivm-order-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.customer.events` | 90 days |
| `ivm.customer.identity_evidence_added.v1` | `ivm-customer-service` | `ivm-kyc-service`, `ivm-audit-service` | `ivm.customer.events` | 90 days |
| `ivm.customer.identity_verified.v1` | `ivm-kyc-service` | `ivm-customer-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.customer.events` | 90 days |
| `ivm.customer.identity_verification_failed.v1` | `ivm-kyc-service` | `ivm-customer-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.customer.events` | 90 days |

### 3.3 Order Events

| Event Type | Producer Service | Consumer Services | Kafka Topic | Retention |
|---|---|---|---|---|
| `ivm.order.created.v1` | `ivm-order-service` | `ivm-workflow-service`, `ivm-locker-module`, `ivm-audit-service`, `ivm-analytics-service` | `ivm.order.events` | 90 days |
| `ivm.order.confirmed.v1` | `ivm-order-service` | `ivm-payment-service`, `ivm-workflow-service`, `ivm-audit-service` | `ivm.order.events` | 90 days |
| `ivm.order.payment_authorized.v1` | `ivm-order-service` | `ivm-workflow-service`, `ivm-audit-service` | `ivm.order.events` | 90 days |
| `ivm.order.fulfillment_started.v1` | `ivm-order-service` | `ivm-workflow-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.order.events` | 90 days |
| `ivm.order.fulfillment_completed.v1` | `ivm-order-service` | `ivm-workflow-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.order.events` | 90 days |
| `ivm.order.completed.v1` | `ivm-order-service` | `ivm-notification-service`, `ivm-analytics-service`, `ivm-audit-service` | `ivm.order.events` | 90 days |
| `ivm.order.cancelled.v1` | `ivm-order-service` | `ivm-payment-service`, `ivm-locker-module`, `ivm-notification-service`, `ivm-audit-service` | `ivm.order.events` | 90 days |
| `ivm.order.disputed.v1` | `ivm-order-service` | `ivm-payment-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.order.events` | 365 days |
| `ivm.order.dispute_resolved.v1` | `ivm-order-service` | `ivm-payment-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.order.events` | 365 days |

### 3.4 Payment Events

| Event Type | Producer Service | Consumer Services | Kafka Topic | Retention |
|---|---|---|---|---|
| `ivm.payment.initiated.v1` | `ivm-payment-service` | `ivm-order-service`, `ivm-audit-service` | `ivm.payment.events` | 90 days |
| `ivm.payment.authorized.v1` | `ivm-payment-service` | `ivm-order-service`, `ivm-workflow-service`, `ivm-notification-service`, `ivm-store-module`, `ivm-audit-service` | `ivm.payment.events` | 90 days |
| `ivm.payment.captured.v1` | `ivm-payment-service` | `ivm-order-service`, `ivm-audit-service`, `ivm-analytics-service` | `ivm.payment.events` | 365 days |
| `ivm.payment.failed.v1` | `ivm-payment-service` | `ivm-order-service`, `ivm-workflow-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.payment.events` | 90 days |
| `ivm.payment.refunded.v1` | `ivm-payment-service` | `ivm-order-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.payment.events` | 365 days |
| `ivm.payment.settled.v1` | `ivm-payment-service` | `ivm-analytics-service`, `ivm-audit-service` | `ivm.payment.events` | 365 days |

### 3.5 Device Events

| Event Type | Producer Service | Consumer Services | Kafka Topic | Retention |
|---|---|---|---|---|
| `ivm.device.online.v1` | `ivm-device-service` | `ivm-site-service`, `ivm-telemetry-service`, `ivm-audit-service` | `ivm.device.events` | 7 days |
| `ivm.device.offline.v1` | `ivm-device-service` | `ivm-site-service`, `ivm-telemetry-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.device.events` | 7 days |
| `ivm.device.health_reported.v1` | `ivm-device-service` | `ivm-telemetry-service`, `ivm-analytics-service` | `ivm.device.events` | 3 days |
| `ivm.device.fault_detected.v1` | `ivm-device-service` | `ivm-site-service`, `ivm-notification-service`, `ivm-telemetry-service`, `ivm-audit-service` | `ivm.device.events` | 30 days |
| `ivm.device.command_executed.v1` | `ivm-device-service` | `ivm-workflow-service`, `ivm-locker-module`, `ivm-store-module`, `ivm-audit-service` | `ivm.device.events` | 30 days |
| `ivm.device.update_started.v1` | `ivm-device-service` | `ivm-telemetry-service`, `ivm-audit-service` | `ivm.device.events` | 30 days |
| `ivm.device.update_completed.v1` | `ivm-device-service` | `ivm-telemetry-service`, `ivm-audit-service` | `ivm.device.events` | 30 days |
| `ivm.device.update_failed.v1` | `ivm-device-service` | `ivm-notification-service`, `ivm-telemetry-service`, `ivm-audit-service` | `ivm.device.events` | 30 days |

### 3.6 Locker Events

| Event Type | Producer Service | Consumer Services | Kafka Topic | Retention |
|---|---|---|---|---|
| `ivm.locker.compartment_reserved.v1` | `ivm-locker-module` | `ivm-order-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.locker.events` | 30 days |
| `ivm.locker.parcel_deposited.v1` | `ivm-locker-module` | `ivm-order-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.locker.events` | 30 days |
| `ivm.locker.door_opened.v1` | `ivm-locker-module` | `ivm-telemetry-service`, `ivm-audit-service` | `ivm.locker.events` | 7 days |
| `ivm.locker.door_closed.v1` | `ivm-locker-module` | `ivm-telemetry-service`, `ivm-audit-service` | `ivm.locker.events` | 7 days |
| `ivm.locker.parcel_picked_up.v1` | `ivm-locker-module` | `ivm-order-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.locker.events` | 30 days |
| `ivm.locker.reservation_expired.v1` | `ivm-locker-module` | `ivm-order-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.locker.events` | 30 days |
| `ivm.locker.compartment_released.v1` | `ivm-locker-module` | `ivm-audit-service`, `ivm-analytics-service` | `ivm.locker.events` | 7 days |

### 3.7 Store Events

| Event Type | Producer Service | Consumer Services | Kafka Topic | Retention |
|---|---|---|---|---|
| `ivm.store.session_started.v1` | `ivm-store-module` | `ivm-audit-service`, `ivm-analytics-service` | `ivm.store.events` | 30 days |
| `ivm.store.customer_entered.v1` | `ivm-store-module` | `ivm-audit-service`, `ivm-analytics-service` | `ivm.store.events` | 30 days |
| `ivm.store.item_picked.v1` | `ivm-store-module` | `ivm-analytics-service` | `ivm.store.events` | 7 days |
| `ivm.store.item_returned.v1` | `ivm-store-module` | `ivm-analytics-service` | `ivm.store.events` | 7 days |
| `ivm.store.customer_exited.v1` | `ivm-store-module` | `ivm-audit-service`, `ivm-analytics-service` | `ivm.store.events` | 30 days |
| `ivm.store.basket_reconciled.v1` | `ivm-store-module` | `ivm-order-service`, `ivm-payment-service`, `ivm-audit-service` | `ivm.store.events` | 90 days |
| `ivm.store.session_charged.v1` | `ivm-store-module` | `ivm-notification-service`, `ivm-audit-service`, `ivm-analytics-service` | `ivm.store.events` | 90 days |
| `ivm.store.session_disputed.v1` | `ivm-store-module` | `ivm-notification-service`, `ivm-audit-service` | `ivm.store.events` | 365 days |
| `ivm.store.shrinkage_detected.v1` | `ivm-store-module` | `ivm-notification-service`, `ivm-audit-service`, `ivm-analytics-service` | `ivm.store.events` | 365 days |

### 3.8 Workflow Events

| Event Type | Producer Service | Consumer Services | Kafka Topic | Retention |
|---|---|---|---|---|
| `ivm.workflow.started.v1` | `ivm-workflow-service` | `ivm-audit-service`, `ivm-analytics-service` | `ivm.workflow.events` | 30 days |
| `ivm.workflow.step_completed.v1` | `ivm-workflow-service` | `ivm-audit-service`, `ivm-analytics-service` | `ivm.workflow.events` | 30 days |
| `ivm.workflow.step_failed.v1` | `ivm-workflow-service` | `ivm-notification-service`, `ivm-audit-service` | `ivm.workflow.events` | 30 days |
| `ivm.workflow.completed.v1` | `ivm-workflow-service` | `ivm-order-service`, `ivm-notification-service`, `ivm-audit-service`, `ivm-analytics-service` | `ivm.workflow.events` | 30 days |
| `ivm.workflow.failed.v1` | `ivm-workflow-service` | `ivm-order-service`, `ivm-notification-service`, `ivm-audit-service` | `ivm.workflow.events` | 30 days |
| `ivm.workflow.compensating.v1` | `ivm-workflow-service` | `ivm-order-service`, `ivm-payment-service`, `ivm-audit-service` | `ivm.workflow.events` | 90 days |

### 3.9 Notification Events

| Event Type | Producer Service | Consumer Services | Kafka Topic | Retention |
|---|---|---|---|---|
| `ivm.notification.sent.v1` | `ivm-notification-service` | `ivm-audit-service`, `ivm-analytics-service` | `ivm.notification.events` | 7 days |
| `ivm.notification.delivered.v1` | `ivm-notification-service` | `ivm-audit-service`, `ivm-analytics-service` | `ivm.notification.events` | 7 days |
| `ivm.notification.failed.v1` | `ivm-notification-service` | `ivm-audit-service` | `ivm.notification.events` | 30 days |

---

## 4. Event Contracts by Domain

Each event contract below includes:
- **Description:** When the event is emitted
- **CloudEvents Example:** Complete envelope with example `data`
- **Data JSON Schema:** Formal schema for the `data` payload
- **Consumers:** Services that react to this event

### 4.1 Identity Events

#### 4.1.1 ivm.user.created.v1

**Description:** Emitted when a new user account is registered on the platform. This is the root event for customer onboarding -- the Customer Service listens to create an associated customer record.

**Consumers:** `ivm-customer-service` (creates customer record), `ivm-audit-service` (logs account creation)

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-identity-service",
  "type": "ivm.user.created.v1",
  "time": "2026-03-12T10:00:00.000Z",
  "datacontenttype": "application/json",
  "subject": "user/usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef1234567890-1234567890abcdef-01",
  "correlationid": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "data": {
    "userId": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
    "userType": "CUSTOMER",
    "phone": "+2348012345678",
    "email": "ade.johnson@example.com",
    "firstName": "Ade",
    "lastName": "Johnson",
    "tenantId": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
    "registrationChannel": "MOBILE_APP",
    "createdAt": "2026-03-12T10:00:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/identity/user-created.v1.json",
  "title": "ivm.user.created.v1 Data",
  "type": "object",
  "required": ["userId", "userType", "phone", "tenantId", "createdAt"],
  "properties": {
    "userId": {
      "type": "string",
      "pattern": "^usr_[0-9A-HJKMNP-TV-Z]{26}$",
      "description": "Unique identifier for the newly created user."
    },
    "userType": {
      "type": "string",
      "enum": ["CUSTOMER", "OPERATOR", "SYSTEM"],
      "description": "Type of user account."
    },
    "phone": {
      "type": "string",
      "pattern": "^\\+234[0-9]{10}$",
      "description": "User phone number in E.164 format (Nigerian +234 prefix)."
    },
    "email": {
      "type": ["string", "null"],
      "format": "email",
      "description": "User email address. Optional for CUSTOMER accounts."
    },
    "firstName": {
      "type": "string",
      "minLength": 1,
      "description": "User first name."
    },
    "lastName": {
      "type": "string",
      "minLength": 1,
      "description": "User last name."
    },
    "tenantId": {
      "type": "string",
      "pattern": "^tnt_[0-9A-HJKMNP-TV-Z]{26}$",
      "description": "Tenant the user belongs to."
    },
    "registrationChannel": {
      "type": "string",
      "enum": ["MOBILE_APP", "WEB_PORTAL", "KIOSK", "OPERATOR_PORTAL", "API"],
      "description": "Channel through which the user registered."
    },
    "createdAt": {
      "type": "string",
      "format": "date-time",
      "description": "Timestamp of account creation."
    }
  },
  "additionalProperties": false
}
```

---

#### 4.1.2 ivm.user.authenticated.v1

**Description:** Emitted on every successful user authentication. Used for audit logging and analytics (login frequency, device patterns).

**Consumers:** `ivm-audit-service`, `ivm-analytics-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M3ABCD8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-identity-service",
  "type": "ivm.user.authenticated.v1",
  "time": "2026-03-12T10:05:00.000Z",
  "datacontenttype": "application/json",
  "subject": "user/usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef1234567891-1234567890abcde0-01",
  "correlationid": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "data": {
    "userId": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
    "method": "OTP",
    "channel": "MOBILE_APP",
    "ipAddress": "102.89.23.45",
    "userAgent": "IVMCustomerApp/2.1.0 (Android 14)",
    "deviceFingerprint": "fp_a1b2c3d4e5f6",
    "mfaUsed": false,
    "authenticatedAt": "2026-03-12T10:05:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/identity/user-authenticated.v1.json",
  "title": "ivm.user.authenticated.v1 Data",
  "type": "object",
  "required": ["userId", "method", "channel", "authenticatedAt"],
  "properties": {
    "userId": {
      "type": "string",
      "pattern": "^usr_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "method": {
      "type": "string",
      "enum": ["PASSWORD", "OTP", "BIOMETRIC", "SSO"],
      "description": "Authentication method used."
    },
    "channel": {
      "type": "string",
      "enum": ["MOBILE_APP", "WEB_PORTAL", "KIOSK", "OPERATOR_PORTAL", "API"],
      "description": "Channel from which the user authenticated."
    },
    "ipAddress": {
      "type": ["string", "null"],
      "description": "Client IP address."
    },
    "userAgent": {
      "type": ["string", "null"],
      "description": "Client user agent string."
    },
    "deviceFingerprint": {
      "type": ["string", "null"],
      "description": "Client device fingerprint for fraud detection."
    },
    "mfaUsed": {
      "type": "boolean",
      "description": "Whether multi-factor authentication was completed."
    },
    "authenticatedAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

#### 4.1.3 ivm.user.authentication_failed.v1

**Description:** Emitted when a user authentication attempt fails. Used for security monitoring, brute-force detection, and compliance audit.

**Consumers:** `ivm-audit-service`, `ivm-notification-service` (alerts on repeated failures)

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M3EFGH8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-identity-service",
  "type": "ivm.user.authentication_failed.v1",
  "time": "2026-03-12T10:06:00.000Z",
  "datacontenttype": "application/json",
  "subject": "user/usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef1234567892-1234567890abcde1-01",
  "correlationid": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "data": {
    "userId": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
    "method": "PASSWORD",
    "reason": "INVALID_CREDENTIALS",
    "channel": "WEB_PORTAL",
    "ipAddress": "102.89.23.99",
    "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
    "failedAttemptCount": 3,
    "accountLocked": false,
    "failedAt": "2026-03-12T10:06:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/identity/user-authentication-failed.v1.json",
  "title": "ivm.user.authentication_failed.v1 Data",
  "type": "object",
  "required": ["userId", "method", "reason", "channel", "failedAttemptCount", "accountLocked", "failedAt"],
  "properties": {
    "userId": {
      "type": "string",
      "pattern": "^usr_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "method": {
      "type": "string",
      "enum": ["PASSWORD", "OTP", "BIOMETRIC", "SSO"]
    },
    "reason": {
      "type": "string",
      "enum": ["INVALID_CREDENTIALS", "EXPIRED_OTP", "ACCOUNT_LOCKED", "ACCOUNT_SUSPENDED", "MFA_FAILED", "DEVICE_NOT_RECOGNIZED"],
      "description": "Reason for authentication failure."
    },
    "channel": {
      "type": "string",
      "enum": ["MOBILE_APP", "WEB_PORTAL", "KIOSK", "OPERATOR_PORTAL", "API"]
    },
    "ipAddress": {
      "type": ["string", "null"]
    },
    "userAgent": {
      "type": ["string", "null"]
    },
    "failedAttemptCount": {
      "type": "integer",
      "minimum": 1,
      "description": "Cumulative failed attempts in the current lockout window."
    },
    "accountLocked": {
      "type": "boolean",
      "description": "Whether this failure triggered an account lockout."
    },
    "failedAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

#### 4.1.4 ivm.user.suspended.v1

**Description:** Emitted when a user account is suspended by an operator or by automated policy (e.g., fraud detection). Active orders may need cancellation or hold.

**Consumers:** `ivm-customer-service` (marks customer inactive), `ivm-order-service` (holds active orders), `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M3JKLM8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-identity-service",
  "type": "ivm.user.suspended.v1",
  "time": "2026-03-12T11:00:00.000Z",
  "datacontenttype": "application/json",
  "subject": "user/usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef1234567893-1234567890abcde2-01",
  "correlationid": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "data": {
    "userId": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
    "reason": "FRAUD_SUSPECTED",
    "suspendedBy": "usr_01J5OPERATOR1234567890ABC",
    "note": "Multiple chargebacks reported within 7 days.",
    "suspendedAt": "2026-03-12T11:00:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/identity/user-suspended.v1.json",
  "title": "ivm.user.suspended.v1 Data",
  "type": "object",
  "required": ["userId", "reason", "suspendedBy", "suspendedAt"],
  "properties": {
    "userId": {
      "type": "string",
      "pattern": "^usr_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "reason": {
      "type": "string",
      "enum": ["FRAUD_SUSPECTED", "COMPLIANCE_VIOLATION", "TERMS_VIOLATION", "OPERATOR_REQUEST", "AUTOMATED_POLICY"],
      "description": "Reason for suspension."
    },
    "suspendedBy": {
      "type": "string",
      "description": "ID of the operator or system that initiated suspension."
    },
    "note": {
      "type": ["string", "null"],
      "description": "Free-text note explaining the suspension."
    },
    "suspendedAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

#### 4.1.5 ivm.user.reactivated.v1

**Description:** Emitted when a previously suspended user account is reactivated.

**Consumers:** `ivm-customer-service` (marks customer active), `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M3NOPQ8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-identity-service",
  "type": "ivm.user.reactivated.v1",
  "time": "2026-03-13T09:00:00.000Z",
  "datacontenttype": "application/json",
  "subject": "user/usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef1234567894-1234567890abcde3-01",
  "correlationid": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "data": {
    "userId": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
    "reactivatedBy": "usr_01J5OPERATOR1234567890ABC",
    "note": "Fraud investigation cleared. Customer is legitimate.",
    "reactivatedAt": "2026-03-13T09:00:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/identity/user-reactivated.v1.json",
  "title": "ivm.user.reactivated.v1 Data",
  "type": "object",
  "required": ["userId", "reactivatedBy", "reactivatedAt"],
  "properties": {
    "userId": {
      "type": "string",
      "pattern": "^usr_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "reactivatedBy": {
      "type": "string",
      "description": "ID of the operator or system that reactivated the account."
    },
    "note": {
      "type": ["string", "null"],
      "description": "Free-text note explaining the reactivation."
    },
    "reactivatedAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

#### 4.1.6 ivm.device.enrolled.v1

**Description:** Emitted when a new device identity is created and its mTLS certificate is provisioned. The Device Service listens to register the device in the fleet registry.

**Consumers:** `ivm-device-service` (registers device in fleet), `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M3RSTU8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-identity-service",
  "type": "ivm.device.enrolled.v1",
  "time": "2026-03-12T08:00:00.000Z",
  "datacontenttype": "application/json",
  "subject": "device/dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef1234567895-1234567890abcde4-01",
  "correlationid": "dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
  "data": {
    "deviceId": "dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
    "tenantId": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
    "siteId": "site_01J5M3QRST8N3P4Q5R6S7T8U9V",
    "deviceType": "LOCKER_CONTROLLER",
    "manufacturer": "IVM Hardware",
    "model": "LC-200",
    "serialNumber": "LC200-2026-00142",
    "certificateFingerprint": "SHA256:a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0",
    "hardwareFingerprint": "hwfp_x9y8z7w6v5u4t3s2",
    "enrolledBy": "usr_01J5TECHNICIAN12345678ABC",
    "enrolledAt": "2026-03-12T08:00:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/identity/device-enrolled.v1.json",
  "title": "ivm.device.enrolled.v1 Data",
  "type": "object",
  "required": ["deviceId", "tenantId", "siteId", "deviceType", "serialNumber", "certificateFingerprint", "enrolledAt"],
  "properties": {
    "deviceId": {
      "type": "string",
      "pattern": "^dev_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "tenantId": {
      "type": "string",
      "pattern": "^tnt_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "siteId": {
      "type": "string",
      "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "deviceType": {
      "type": "string",
      "enum": ["TOUCHSCREEN", "QR_SCANNER", "BARCODE_SCANNER", "RECEIPT_PRINTER", "THERMAL_PRINTER", "CARD_READER", "PIN_PAD", "BIOMETRIC_SCANNER", "CAMERA", "LOCKER_CONTROLLER", "SHELF_SENSOR", "WEIGHT_SENSOR", "ELECTRONIC_LOCK", "VENDING_MOTOR", "GATE_CONTROLLER", "CASH_ACCEPTOR", "CASH_DISPENSER", "EDGE_CONTROLLER", "CELLULAR_MODEM", "NFC_READER"]
    },
    "manufacturer": {
      "type": "string"
    },
    "model": {
      "type": "string"
    },
    "serialNumber": {
      "type": "string"
    },
    "certificateFingerprint": {
      "type": "string",
      "description": "SHA-256 fingerprint of the device mTLS certificate."
    },
    "hardwareFingerprint": {
      "type": ["string", "null"],
      "description": "Hardware-derived fingerprint (TPM-based)."
    },
    "enrolledBy": {
      "type": "string",
      "description": "User ID of the technician who enrolled the device."
    },
    "enrolledAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

#### 4.1.7 ivm.device.authenticated.v1

**Description:** Emitted on each successful device authentication via mTLS. High-volume event used primarily for telemetry and anomaly detection.

**Consumers:** `ivm-device-service`, `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M3VWXY8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-identity-service",
  "type": "ivm.device.authenticated.v1",
  "time": "2026-03-12T08:05:00.000Z",
  "datacontenttype": "application/json",
  "subject": "device/dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef1234567896-1234567890abcde5-01",
  "correlationid": "dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
  "data": {
    "deviceId": "dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
    "siteId": "site_01J5M3QRST8N3P4Q5R6S7T8U9V",
    "certificateFingerprint": "SHA256:a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0",
    "ipAddress": "10.0.1.42",
    "authenticatedAt": "2026-03-12T08:05:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/identity/device-authenticated.v1.json",
  "title": "ivm.device.authenticated.v1 Data",
  "type": "object",
  "required": ["deviceId", "siteId", "certificateFingerprint", "authenticatedAt"],
  "properties": {
    "deviceId": {
      "type": "string",
      "pattern": "^dev_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "siteId": {
      "type": "string",
      "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "certificateFingerprint": {
      "type": "string"
    },
    "ipAddress": {
      "type": ["string", "null"]
    },
    "authenticatedAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

#### 4.1.8 ivm.device.certificate_rotated.v1

**Description:** Emitted when a device mTLS certificate is rotated. The Device Service updates its certificate store. Long retention for compliance (certificate lifecycle audit).

**Consumers:** `ivm-device-service`, `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M3ZABC8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-identity-service",
  "type": "ivm.device.certificate_rotated.v1",
  "time": "2026-03-12T02:00:00.000Z",
  "datacontenttype": "application/json",
  "subject": "device/dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef1234567897-1234567890abcde6-01",
  "correlationid": "dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
  "data": {
    "deviceId": "dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
    "previousFingerprint": "SHA256:a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0",
    "newFingerprint": "SHA256:z9y8x7w6v5u4t3s2r1q0p9o8n7m6l5k4j3i2h1g0",
    "newCertificateExpiresAt": "2027-03-12T02:00:00.000Z",
    "rotationReason": "SCHEDULED",
    "rotatedAt": "2026-03-12T02:00:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/identity/device-certificate-rotated.v1.json",
  "title": "ivm.device.certificate_rotated.v1 Data",
  "type": "object",
  "required": ["deviceId", "previousFingerprint", "newFingerprint", "newCertificateExpiresAt", "rotationReason", "rotatedAt"],
  "properties": {
    "deviceId": {
      "type": "string",
      "pattern": "^dev_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "previousFingerprint": {
      "type": "string",
      "description": "SHA-256 fingerprint of the old certificate."
    },
    "newFingerprint": {
      "type": "string",
      "description": "SHA-256 fingerprint of the new certificate."
    },
    "newCertificateExpiresAt": {
      "type": "string",
      "format": "date-time",
      "description": "Expiry date of the new certificate."
    },
    "rotationReason": {
      "type": "string",
      "enum": ["SCHEDULED", "COMPROMISED", "MANUAL", "POLICY_CHANGE"],
      "description": "Reason for certificate rotation."
    },
    "rotatedAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

### 4.2 Customer Events

#### 4.2.1 ivm.customer.created.v1

**Description:** Emitted when a new customer profile is created, typically triggered by consuming `ivm.user.created.v1`. The KYC Service listens to auto-initiate basic verification if identity data is available.

**Consumers:** `ivm-kyc-service` (auto-initiate verification), `ivm-audit-service`, `ivm-analytics-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-customer-service",
  "type": "ivm.customer.created.v1",
  "time": "2026-03-12T10:00:01.000Z",
  "datacontenttype": "application/json",
  "subject": "customer/cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef1234567890-1234567890abcdef-01",
  "causationid": "evt_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "correlationid": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
  "data": {
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "userId": "usr_01J5M2XKQR8N3P4Q5R6S7T8U9V",
    "tenantId": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
    "firstName": "Ade",
    "lastName": "Johnson",
    "phone": "+2348012345678",
    "email": "ade.johnson@example.com",
    "kycTier": "TIER_0",
    "kycStatus": "PENDING",
    "createdAt": "2026-03-12T10:00:01.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/customer/customer-created.v1.json",
  "title": "ivm.customer.created.v1 Data",
  "type": "object",
  "required": ["customerId", "userId", "tenantId", "firstName", "lastName", "phone", "kycTier", "kycStatus", "createdAt"],
  "properties": {
    "customerId": {
      "type": "string",
      "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "userId": {
      "type": "string",
      "pattern": "^usr_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "tenantId": {
      "type": "string",
      "pattern": "^tnt_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "firstName": {
      "type": "string",
      "minLength": 1
    },
    "lastName": {
      "type": "string",
      "minLength": 1
    },
    "phone": {
      "type": "string",
      "pattern": "^\\+234[0-9]{10}$"
    },
    "email": {
      "type": ["string", "null"],
      "format": "email"
    },
    "kycTier": {
      "type": "string",
      "enum": ["TIER_0", "TIER_1", "TIER_2", "TIER_3"],
      "description": "Initial KYC tier (always TIER_0 on creation)."
    },
    "kycStatus": {
      "type": "string",
      "enum": ["PENDING", "VERIFIED", "FAILED", "EXPIRED"]
    },
    "createdAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

#### 4.2.2 ivm.customer.updated.v1

**Description:** Emitted when a customer profile is updated (name, address, email, preferences, etc.). Does not fire for KYC tier changes (separate event).

**Consumers:** `ivm-audit-service`, `ivm-analytics-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M4EFGH8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-customer-service",
  "type": "ivm.customer.updated.v1",
  "time": "2026-03-12T12:30:00.000Z",
  "datacontenttype": "application/json",
  "subject": "customer/cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef1234567898-1234567890abcde7-01",
  "correlationid": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "changedFields": ["address", "email"],
    "address": {
      "line1": "15 Admiralty Way",
      "line2": "Suite 200",
      "city": "Lekki",
      "state": "Lagos",
      "country": "NG",
      "postalCode": "106104"
    },
    "email": "ade.new@example.com",
    "updatedBy": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "updatedAt": "2026-03-12T12:30:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/customer/customer-updated.v1.json",
  "title": "ivm.customer.updated.v1 Data",
  "type": "object",
  "required": ["customerId", "changedFields", "updatedBy", "updatedAt"],
  "properties": {
    "customerId": {
      "type": "string",
      "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "changedFields": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1,
      "description": "List of field names that were modified."
    },
    "firstName": { "type": "string" },
    "lastName": { "type": "string" },
    "middleName": { "type": ["string", "null"] },
    "phone": { "type": "string" },
    "email": { "type": ["string", "null"] },
    "dateOfBirth": { "type": ["string", "null"], "format": "date" },
    "gender": { "type": ["string", "null"], "enum": ["MALE", "FEMALE", null] },
    "address": {
      "type": ["object", "null"],
      "properties": {
        "line1": { "type": "string" },
        "line2": { "type": ["string", "null"] },
        "city": { "type": "string" },
        "state": { "type": "string" },
        "country": { "type": "string" },
        "postalCode": { "type": ["string", "null"] }
      }
    },
    "updatedBy": {
      "type": "string",
      "description": "ID of the customer or operator who made the update."
    },
    "updatedAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

#### 4.2.3 ivm.customer.kyc_tier_changed.v1

**Description:** Emitted when a customer's KYC tier changes (e.g., TIER_0 to TIER_1 after BVN verification). CBN tiered KYC compliance requires tracking these transitions. The Order Service may gate certain transactions on minimum KYC tier.

**Consumers:** `ivm-order-service` (gate high-value transactions), `ivm-notification-service` (notify customer), `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M4JKLM8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-customer-service",
  "type": "ivm.customer.kyc_tier_changed.v1",
  "time": "2026-03-12T10:10:00.000Z",
  "datacontenttype": "application/json",
  "subject": "customer/cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef1234567899-1234567890abcde8-01",
  "causationid": "evt_01J5M4HIJK8N3P4Q5R6S7T8U9V",
  "correlationid": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "previousTier": "TIER_0",
    "newTier": "TIER_1",
    "reason": "BVN_VERIFIED",
    "verificationId": "vrf_01J5M4GHIJ8N3P4Q5R6S7T8U9V",
    "changedAt": "2026-03-12T10:10:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/customer/kyc-tier-changed.v1.json",
  "title": "ivm.customer.kyc_tier_changed.v1 Data",
  "type": "object",
  "required": ["customerId", "previousTier", "newTier", "reason", "changedAt"],
  "properties": {
    "customerId": {
      "type": "string",
      "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "previousTier": {
      "type": "string",
      "enum": ["TIER_0", "TIER_1", "TIER_2", "TIER_3"]
    },
    "newTier": {
      "type": "string",
      "enum": ["TIER_0", "TIER_1", "TIER_2", "TIER_3"]
    },
    "reason": {
      "type": "string",
      "enum": ["BVN_VERIFIED", "NIN_VERIFIED", "DOCUMENT_VERIFIED", "BIOMETRIC_VERIFIED", "MANUAL_UPGRADE", "TIER_EXPIRED", "COMPLIANCE_DOWNGRADE"],
      "description": "What caused the tier change."
    },
    "verificationId": {
      "type": ["string", "null"],
      "description": "Reference to the verification event that triggered this change."
    },
    "changedAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

#### 4.2.4 ivm.customer.identity_evidence_added.v1

**Description:** Emitted when a new piece of identity evidence (BVN, NIN, passport, etc.) is submitted for a customer. The KYC Service consumes this to begin verification.

**Consumers:** `ivm-kyc-service` (initiates verification), `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M4NOPQ8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-customer-service",
  "type": "ivm.customer.identity_evidence_added.v1",
  "time": "2026-03-12T10:08:00.000Z",
  "datacontenttype": "application/json",
  "subject": "customer/cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789a-1234567890abcde9-01",
  "correlationid": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "evidenceId": "evi_01J5M4MNOP8N3P4Q5R6S7T8U9V",
    "evidenceType": "BVN",
    "submissionChannel": "KIOSK",
    "siteId": "site_01J5M3QRST8N3P4Q5R6S7T8U9V",
    "addedAt": "2026-03-12T10:08:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/customer/identity-evidence-added.v1.json",
  "title": "ivm.customer.identity_evidence_added.v1 Data",
  "type": "object",
  "required": ["customerId", "evidenceId", "evidenceType", "submissionChannel", "addedAt"],
  "properties": {
    "customerId": {
      "type": "string",
      "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "evidenceId": {
      "type": "string",
      "pattern": "^evi_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "evidenceType": {
      "type": "string",
      "enum": ["BVN", "NIN", "PASSPORT", "DRIVERS_LICENSE", "VOTER_CARD", "INTL_PASSPORT"],
      "description": "Type of identity evidence submitted."
    },
    "submissionChannel": {
      "type": "string",
      "enum": ["KIOSK", "MOBILE_APP", "WEB_PORTAL", "OPERATOR_PORTAL"],
      "description": "Channel through which the evidence was submitted."
    },
    "siteId": {
      "type": ["string", "null"],
      "description": "Site ID if submitted at a physical kiosk."
    },
    "addedAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

#### 4.2.5 ivm.customer.identity_verified.v1

**Description:** Emitted when an identity evidence verification succeeds. Produced by the KYC Service after receiving a positive response from NIBSS (BVN), NIMC (NIN), or document verification providers.

**Consumers:** `ivm-customer-service` (updates evidence status, may trigger tier change), `ivm-notification-service` (notify customer), `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M4RSTU8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-kyc-service",
  "type": "ivm.customer.identity_verified.v1",
  "time": "2026-03-12T10:09:30.000Z",
  "datacontenttype": "application/json",
  "subject": "customer/cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789a-1234567890abcde9-01",
  "causationid": "evt_01J5M4NOPQ8N3P4Q5R6S7T8U9V",
  "correlationid": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "evidenceId": "evi_01J5M4MNOP8N3P4Q5R6S7T8U9V",
    "evidenceType": "BVN",
    "verificationProvider": "NIBSS",
    "providerRef": "NIBSS-VRF-2026031200142",
    "matchScore": 0.98,
    "verifiedFields": ["firstName", "lastName", "dateOfBirth", "phone"],
    "verifiedAt": "2026-03-12T10:09:30.000Z",
    "expiresAt": "2027-03-12T10:09:30.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/customer/identity-verified.v1.json",
  "title": "ivm.customer.identity_verified.v1 Data",
  "type": "object",
  "required": ["customerId", "evidenceId", "evidenceType", "verificationProvider", "verifiedAt"],
  "properties": {
    "customerId": {
      "type": "string",
      "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "evidenceId": {
      "type": "string",
      "pattern": "^evi_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "evidenceType": {
      "type": "string",
      "enum": ["BVN", "NIN", "PASSPORT", "DRIVERS_LICENSE", "VOTER_CARD", "INTL_PASSPORT"]
    },
    "verificationProvider": {
      "type": "string",
      "enum": ["NIBSS", "NIMC", "ICAD", "SMILE_ID", "MANUAL"],
      "description": "External provider that performed the verification."
    },
    "providerRef": {
      "type": ["string", "null"],
      "description": "Reference ID from the external verification provider."
    },
    "matchScore": {
      "type": ["number", "null"],
      "minimum": 0.0,
      "maximum": 1.0,
      "description": "Confidence score from the verification provider (0.0-1.0)."
    },
    "verifiedFields": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Fields that were verified against the evidence."
    },
    "verifiedAt": {
      "type": "string",
      "format": "date-time"
    },
    "expiresAt": {
      "type": ["string", "null"],
      "format": "date-time",
      "description": "When this verification result expires and re-verification is needed."
    }
  },
  "additionalProperties": false
}
```

---

#### 4.2.6 ivm.customer.identity_verification_failed.v1

**Description:** Emitted when an identity verification attempt fails. This may be due to a mismatch, provider timeout, or invalid evidence. The Customer Service updates the evidence status, and the Notification Service informs the customer.

**Consumers:** `ivm-customer-service` (updates evidence status), `ivm-notification-service` (notify customer of failure), `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M4VWXY8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-kyc-service",
  "type": "ivm.customer.identity_verification_failed.v1",
  "time": "2026-03-12T10:09:45.000Z",
  "datacontenttype": "application/json",
  "subject": "customer/cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789b-1234567890abcdea-01",
  "causationid": "evt_01J5M4NOPQ8N3P4Q5R6S7T8U9V",
  "correlationid": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "evidenceId": "evi_01J5M4MNOP8N3P4Q5R6S7T8U9V",
    "evidenceType": "NIN",
    "verificationProvider": "NIMC",
    "providerRef": "NIMC-VRF-2026031200098",
    "failureReason": "NAME_MISMATCH",
    "failureDetail": "First name on NIN record does not match submitted name.",
    "retryable": true,
    "failedAt": "2026-03-12T10:09:45.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/customer/identity-verification-failed.v1.json",
  "title": "ivm.customer.identity_verification_failed.v1 Data",
  "type": "object",
  "required": ["customerId", "evidenceId", "evidenceType", "verificationProvider", "failureReason", "retryable", "failedAt"],
  "properties": {
    "customerId": {
      "type": "string",
      "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "evidenceId": {
      "type": "string",
      "pattern": "^evi_[0-9A-HJKMNP-TV-Z]{26}$"
    },
    "evidenceType": {
      "type": "string",
      "enum": ["BVN", "NIN", "PASSPORT", "DRIVERS_LICENSE", "VOTER_CARD", "INTL_PASSPORT"]
    },
    "verificationProvider": {
      "type": "string",
      "enum": ["NIBSS", "NIMC", "ICAD", "SMILE_ID", "MANUAL"]
    },
    "providerRef": {
      "type": ["string", "null"]
    },
    "failureReason": {
      "type": "string",
      "enum": ["NAME_MISMATCH", "DOB_MISMATCH", "PHOTO_MISMATCH", "DOCUMENT_EXPIRED", "DOCUMENT_INVALID", "PROVIDER_TIMEOUT", "PROVIDER_ERROR", "FRAUD_SUSPECTED", "DATA_INSUFFICIENT"],
      "description": "Categorized reason for the verification failure."
    },
    "failureDetail": {
      "type": ["string", "null"],
      "description": "Human-readable detail about the failure."
    },
    "retryable": {
      "type": "boolean",
      "description": "Whether the customer can retry the verification."
    },
    "failedAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

---

### 4.3 Order Events

#### 4.3.1 ivm.order.created.v1

**Description:** Emitted when a new order is created. This is the starting point for the order lifecycle. The Workflow Service auto-starts the appropriate workflow, and the Locker Module may auto-reserve a compartment for locker-channel orders.

**Consumers:** `ivm-workflow-service` (auto-start workflow), `ivm-locker-module` (auto-reserve compartment), `ivm-audit-service`, `ivm-analytics-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-order-service",
  "type": "ivm.order.created.v1",
  "time": "2026-03-12T14:00:00.000Z",
  "datacontenttype": "application/json",
  "subject": "order/ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789c-1234567890abcdeb-01",
  "correlationid": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "orderId": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
    "tenantId": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "siteId": "site_01J5M3QRST8N3P4Q5R6S7T8U9V",
    "deviceId": "dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
    "channelType": "KIOSK",
    "orderType": "SERVICE",
    "status": "DRAFT",
    "lineItems": [
      {
        "lineItemId": "li_01J5M5BCDE8N3P4Q5R6S7T8U9V",
        "catalogItemId": "cat_01J5M5CDEF8N3P4Q5R6S7T8U9V",
        "name": "MTN SIM Registration",
        "quantity": 1,
        "unitPrice": { "amount": 150000, "currency": "NGN" },
        "discount": { "amount": 0, "currency": "NGN" },
        "tax": { "amount": 11250, "currency": "NGN" },
        "total": { "amount": 161250, "currency": "NGN" }
      }
    ],
    "subtotal": { "amount": 150000, "currency": "NGN" },
    "discount": { "amount": 0, "currency": "NGN" },
    "tax": { "amount": 11250, "currency": "NGN" },
    "total": { "amount": 161250, "currency": "NGN" },
    "idempotencyKey": "idem_kiosk_vi_001_20260312_001",
    "expiresAt": "2026-03-12T14:30:00.000Z",
    "createdAt": "2026-03-12T14:00:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/order/order-created.v1.json",
  "title": "ivm.order.created.v1 Data",
  "type": "object",
  "required": ["orderId", "tenantId", "customerId", "siteId", "channelType", "orderType", "status", "lineItems", "subtotal", "total", "createdAt"],
  "properties": {
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "tenantId": { "type": "string", "pattern": "^tnt_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "siteId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "deviceId": { "type": ["string", "null"] },
    "channelType": { "type": "string", "enum": ["KIOSK", "LOCKER", "STORE", "MOBILE", "WEB"] },
    "orderType": { "type": "string", "enum": ["SERVICE", "PRODUCT", "MIXED"] },
    "status": { "type": "string", "const": "DRAFT" },
    "lineItems": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["lineItemId", "catalogItemId", "name", "quantity", "unitPrice", "total"],
        "properties": {
          "lineItemId": { "type": "string" },
          "catalogItemId": { "type": "string" },
          "name": { "type": "string" },
          "quantity": { "type": "integer", "minimum": 1 },
          "unitPrice": { "$ref": "#/$defs/money" },
          "discount": { "$ref": "#/$defs/money" },
          "tax": { "$ref": "#/$defs/money" },
          "total": { "$ref": "#/$defs/money" }
        }
      }
    },
    "subtotal": { "$ref": "#/$defs/money" },
    "discount": { "$ref": "#/$defs/money" },
    "tax": { "$ref": "#/$defs/money" },
    "total": { "$ref": "#/$defs/money" },
    "idempotencyKey": { "type": "string", "description": "Client-provided idempotency key for safe retries." },
    "expiresAt": { "type": ["string", "null"], "format": "date-time" },
    "createdAt": { "type": "string", "format": "date-time" }
  },
  "$defs": {
    "money": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": {
        "amount": { "type": "integer", "minimum": 0, "description": "Amount in minor units (kobo for NGN)." },
        "currency": { "type": "string", "pattern": "^[A-Z]{3}$", "description": "ISO 4217 currency code." }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

---

#### 4.3.2 ivm.order.confirmed.v1

**Description:** Emitted when the customer confirms the order and it transitions from DRAFT to PENDING_PAYMENT. This triggers payment initiation.

**Consumers:** `ivm-payment-service` (triggers payment initiation), `ivm-workflow-service`, `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M5EFGH8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-order-service",
  "type": "ivm.order.confirmed.v1",
  "time": "2026-03-12T14:01:00.000Z",
  "datacontenttype": "application/json",
  "subject": "order/ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789c-1234567890abcdeb-01",
  "causationid": "evt_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "correlationid": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "orderId": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "previousStatus": "DRAFT",
    "newStatus": "PENDING_PAYMENT",
    "total": { "amount": 161250, "currency": "NGN" },
    "paymentMethod": "CARD",
    "confirmedAt": "2026-03-12T14:01:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/order/order-confirmed.v1.json",
  "title": "ivm.order.confirmed.v1 Data",
  "type": "object",
  "required": ["orderId", "customerId", "previousStatus", "newStatus", "total", "confirmedAt"],
  "properties": {
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "previousStatus": { "type": "string", "const": "DRAFT" },
    "newStatus": { "type": "string", "const": "PENDING_PAYMENT" },
    "total": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": {
        "amount": { "type": "integer", "minimum": 0 },
        "currency": { "type": "string", "pattern": "^[A-Z]{3}$" }
      }
    },
    "paymentMethod": { "type": ["string", "null"], "enum": ["CARD", "MOBILE_MONEY", "BANK_TRANSFER", "WALLET", "USSD", "QR", null] },
    "confirmedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.3.3 ivm.order.payment_authorized.v1

**Description:** Emitted when payment has been authorized for the order. The order transitions to PAYMENT_AUTHORIZED and fulfillment can begin.

**Consumers:** `ivm-workflow-service` (advance workflow step), `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M5JKLM8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-order-service",
  "type": "ivm.order.payment_authorized.v1",
  "time": "2026-03-12T14:01:15.000Z",
  "datacontenttype": "application/json",
  "subject": "order/ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789c-1234567890abcdeb-01",
  "causationid": "evt_01J5M5HIJK8N3P4Q5R6S7T8U9V",
  "correlationid": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "orderId": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
    "paymentId": "pay_01J5M5GHIJ8N3P4Q5R6S7T8U9V",
    "previousStatus": "PENDING_PAYMENT",
    "newStatus": "PAYMENT_AUTHORIZED",
    "authorizedAmount": { "amount": 161250, "currency": "NGN" },
    "authorizedAt": "2026-03-12T14:01:15.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/order/order-payment-authorized.v1.json",
  "title": "ivm.order.payment_authorized.v1 Data",
  "type": "object",
  "required": ["orderId", "paymentId", "previousStatus", "newStatus", "authorizedAmount", "authorizedAt"],
  "properties": {
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "paymentId": { "type": "string", "pattern": "^pay_[0-9A-HJKMNP-TV-Z]{26}$" },
    "previousStatus": { "type": "string" },
    "newStatus": { "type": "string", "const": "PAYMENT_AUTHORIZED" },
    "authorizedAmount": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": {
        "amount": { "type": "integer", "minimum": 0 },
        "currency": { "type": "string", "pattern": "^[A-Z]{3}$" }
      }
    },
    "authorizedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.3.4 ivm.order.fulfillment_started.v1

**Description:** Emitted when fulfillment begins for the order (e.g., SIM activation initiated, locker compartment being loaded, item dispensing started).

**Consumers:** `ivm-workflow-service`, `ivm-notification-service`, `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M5NOPQ8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-order-service",
  "type": "ivm.order.fulfillment_started.v1",
  "time": "2026-03-12T14:01:30.000Z",
  "datacontenttype": "application/json",
  "subject": "order/ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789c-1234567890abcdeb-01",
  "correlationid": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "orderId": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
    "previousStatus": "PAYMENT_AUTHORIZED",
    "newStatus": "FULFILLING",
    "fulfillmentTasks": [
      {
        "taskId": "ftk_01J5M5OPQR8N3P4Q5R6S7T8U9V",
        "type": "ACTIVATE",
        "status": "IN_PROGRESS",
        "description": "Activate MTN SIM via telco API"
      },
      {
        "taskId": "ftk_01J5M5PQRS8N3P4Q5R6S7T8U9V",
        "type": "DISPENSE",
        "status": "PENDING",
        "description": "Dispense SIM card from vending slot B3"
      }
    ],
    "startedAt": "2026-03-12T14:01:30.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/order/order-fulfillment-started.v1.json",
  "title": "ivm.order.fulfillment_started.v1 Data",
  "type": "object",
  "required": ["orderId", "previousStatus", "newStatus", "fulfillmentTasks", "startedAt"],
  "properties": {
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "previousStatus": { "type": "string" },
    "newStatus": { "type": "string", "const": "FULFILLING" },
    "fulfillmentTasks": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["taskId", "type", "status"],
        "properties": {
          "taskId": { "type": "string" },
          "type": { "type": "string", "enum": ["DISPENSE", "PRINT", "DELIVER", "ACTIVATE", "ISSUE", "PICKUP", "LOCKER_DEPOSIT", "LOCKER_PICKUP"] },
          "status": { "type": "string", "enum": ["PENDING", "IN_PROGRESS", "COMPLETED", "FAILED"] },
          "description": { "type": ["string", "null"] }
        }
      }
    },
    "startedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.3.5 ivm.order.fulfillment_completed.v1

**Description:** Emitted when all fulfillment tasks for an order have completed successfully.

**Consumers:** `ivm-workflow-service`, `ivm-notification-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/order/order-fulfillment-completed.v1.json",
  "title": "ivm.order.fulfillment_completed.v1 Data",
  "type": "object",
  "required": ["orderId", "previousStatus", "newStatus", "completedTasks", "completedAt"],
  "properties": {
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "previousStatus": { "type": "string", "const": "FULFILLING" },
    "newStatus": { "type": "string", "const": "COMPLETED" },
    "completedTasks": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["taskId", "type", "status"],
        "properties": {
          "taskId": { "type": "string" },
          "type": { "type": "string" },
          "status": { "type": "string", "const": "COMPLETED" },
          "completedAt": { "type": "string", "format": "date-time" }
        }
      }
    },
    "completedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.3.6 ivm.order.completed.v1

**Description:** Emitted when the order reaches its terminal successful state. Payment has been captured and all fulfillment tasks are done. Triggers receipt notification to the customer.

**Consumers:** `ivm-notification-service` (send receipt), `ivm-analytics-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/order/order-completed.v1.json",
  "title": "ivm.order.completed.v1 Data",
  "type": "object",
  "required": ["orderId", "customerId", "total", "paymentId", "channelType", "completedAt"],
  "properties": {
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "total": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": {
        "amount": { "type": "integer", "minimum": 0 },
        "currency": { "type": "string", "pattern": "^[A-Z]{3}$" }
      }
    },
    "paymentId": { "type": "string", "pattern": "^pay_[0-9A-HJKMNP-TV-Z]{26}$" },
    "channelType": { "type": "string", "enum": ["KIOSK", "LOCKER", "STORE", "MOBILE", "WEB"] },
    "siteId": { "type": "string" },
    "lineItemCount": { "type": "integer", "minimum": 1 },
    "completedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.3.7 ivm.order.cancelled.v1

**Description:** Emitted when an order is cancelled (by customer, operator, or system timeout). Triggers payment void/refund and locker compartment release if applicable.

**Consumers:** `ivm-payment-service` (void/refund), `ivm-locker-module` (release compartment), `ivm-notification-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/order/order-cancelled.v1.json",
  "title": "ivm.order.cancelled.v1 Data",
  "type": "object",
  "required": ["orderId", "customerId", "previousStatus", "cancellationReason", "cancelledBy", "cancelledAt"],
  "properties": {
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "previousStatus": { "type": "string", "enum": ["DRAFT", "PENDING_PAYMENT", "PAYMENT_AUTHORIZED", "FULFILLING"] },
    "cancellationReason": {
      "type": "string",
      "enum": ["CUSTOMER_REQUEST", "PAYMENT_FAILED", "FULFILLMENT_FAILED", "TIMEOUT", "OPERATOR_REQUEST", "SYSTEM_ERROR"],
      "description": "Reason for cancellation."
    },
    "cancelledBy": { "type": "string", "description": "ID of the customer, operator, or 'SYSTEM'." },
    "paymentId": { "type": ["string", "null"] },
    "refundRequired": { "type": "boolean", "description": "Whether a refund needs to be processed." },
    "cancelledAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.3.8 ivm.order.disputed.v1

**Description:** Emitted when a customer disputes a completed order. Long retention for compliance and dispute resolution.

**Consumers:** `ivm-payment-service` (hold settlement), `ivm-notification-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/order/order-disputed.v1.json",
  "title": "ivm.order.disputed.v1 Data",
  "type": "object",
  "required": ["orderId", "customerId", "disputeId", "disputeReason", "disputedAt"],
  "properties": {
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "disputeId": { "type": "string", "pattern": "^dsp_[0-9A-HJKMNP-TV-Z]{26}$" },
    "disputeReason": {
      "type": "string",
      "enum": ["ITEM_NOT_RECEIVED", "WRONG_ITEM", "OVERCHARGED", "QUALITY_ISSUE", "UNAUTHORIZED_CHARGE", "SERVICE_NOT_ACTIVATED", "OTHER"]
    },
    "disputeDescription": { "type": ["string", "null"], "description": "Customer-provided description of the dispute." },
    "orderTotal": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": {
        "amount": { "type": "integer" },
        "currency": { "type": "string" }
      }
    },
    "disputedAmount": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": {
        "amount": { "type": "integer" },
        "currency": { "type": "string" }
      }
    },
    "disputedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.3.9 ivm.order.dispute_resolved.v1

**Description:** Emitted when a dispute is resolved (in favor of customer or merchant). If resolved in favor of the customer, a refund is triggered.

**Consumers:** `ivm-payment-service` (process refund if applicable), `ivm-notification-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/order/order-dispute-resolved.v1.json",
  "title": "ivm.order.dispute_resolved.v1 Data",
  "type": "object",
  "required": ["orderId", "disputeId", "resolution", "resolvedBy", "resolvedAt"],
  "properties": {
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "disputeId": { "type": "string", "pattern": "^dsp_[0-9A-HJKMNP-TV-Z]{26}$" },
    "resolution": {
      "type": "string",
      "enum": ["REFUND_FULL", "REFUND_PARTIAL", "CREDIT_ISSUED", "RESOLVED_NO_ACTION", "MERCHANT_FAVOR"],
      "description": "Outcome of the dispute resolution."
    },
    "refundAmount": {
      "type": ["object", "null"],
      "properties": {
        "amount": { "type": "integer" },
        "currency": { "type": "string" }
      },
      "description": "Refund amount if applicable."
    },
    "resolvedBy": { "type": "string", "description": "Operator or system that resolved the dispute." },
    "resolutionNote": { "type": ["string", "null"] },
    "resolvedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

### 4.4 Payment Events

#### 4.4.1 ivm.payment.initiated.v1

**Description:** Emitted when a payment is created and sent to the payment gateway for processing. The order status remains PENDING_PAYMENT until authorization.

**Consumers:** `ivm-order-service` (track payment status), `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M6ABCD8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-payment-service",
  "type": "ivm.payment.initiated.v1",
  "time": "2026-03-12T14:01:05.000Z",
  "datacontenttype": "application/json",
  "subject": "payment/pay_01J5M5GHIJ8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789c-1234567890abcdeb-01",
  "correlationid": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "paymentId": "pay_01J5M5GHIJ8N3P4Q5R6S7T8U9V",
    "orderId": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "amount": { "amount": 161250, "currency": "NGN" },
    "method": "CARD",
    "gateway": "paystack",
    "idempotencyKey": "pay_idem_ord01J5M5ABCD_001",
    "initiatedAt": "2026-03-12T14:01:05.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/payment/payment-initiated.v1.json",
  "title": "ivm.payment.initiated.v1 Data",
  "type": "object",
  "required": ["paymentId", "orderId", "customerId", "amount", "method", "gateway", "initiatedAt"],
  "properties": {
    "paymentId": { "type": "string", "pattern": "^pay_[0-9A-HJKMNP-TV-Z]{26}$" },
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "amount": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": {
        "amount": { "type": "integer", "minimum": 0 },
        "currency": { "type": "string", "pattern": "^[A-Z]{3}$" }
      }
    },
    "method": { "type": "string", "enum": ["CARD", "MOBILE_MONEY", "BANK_TRANSFER", "WALLET", "USSD", "QR"] },
    "gateway": { "type": "string", "description": "Payment gateway identifier (e.g., paystack, flutterwave)." },
    "idempotencyKey": { "type": "string" },
    "initiatedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.4.2 ivm.payment.authorized.v1

**Description:** Emitted when the payment gateway confirms authorization (funds are held but not yet captured). This advances the order to PAYMENT_AUTHORIZED and the workflow to the next step.

**Consumers:** `ivm-order-service` (transition to PAYMENT_AUTHORIZED), `ivm-workflow-service` (advance step), `ivm-notification-service`, `ivm-store-module` (pre-auth confirmation for store entry), `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M6EFGH8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-payment-service",
  "type": "ivm.payment.authorized.v1",
  "time": "2026-03-12T14:01:12.000Z",
  "datacontenttype": "application/json",
  "subject": "payment/pay_01J5M5GHIJ8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789c-1234567890abcdeb-01",
  "causationid": "evt_01J5M6ABCD8N3P4Q5R6S7T8U9V",
  "correlationid": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "paymentId": "pay_01J5M5GHIJ8N3P4Q5R6S7T8U9V",
    "orderId": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "amount": { "amount": 161250, "currency": "NGN" },
    "method": "CARD",
    "gateway": "paystack",
    "gatewayRef": "PSK_txn_abc123xyz789",
    "cardLast4": "4242",
    "cardBrand": "VISA",
    "authorizedAt": "2026-03-12T14:01:12.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/payment/payment-authorized.v1.json",
  "title": "ivm.payment.authorized.v1 Data",
  "type": "object",
  "required": ["paymentId", "orderId", "customerId", "amount", "method", "gateway", "gatewayRef", "authorizedAt"],
  "properties": {
    "paymentId": { "type": "string", "pattern": "^pay_[0-9A-HJKMNP-TV-Z]{26}$" },
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "amount": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": {
        "amount": { "type": "integer", "minimum": 0 },
        "currency": { "type": "string", "pattern": "^[A-Z]{3}$" }
      }
    },
    "method": { "type": "string", "enum": ["CARD", "MOBILE_MONEY", "BANK_TRANSFER", "WALLET", "USSD", "QR"] },
    "gateway": { "type": "string" },
    "gatewayRef": { "type": "string", "description": "Gateway transaction reference." },
    "cardLast4": { "type": ["string", "null"], "pattern": "^[0-9]{4}$" },
    "cardBrand": { "type": ["string", "null"], "enum": ["VISA", "MASTERCARD", "VERVE", null] },
    "authorizedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.4.3 ivm.payment.captured.v1

**Description:** Emitted when funds are captured (settled from the gateway). For most kiosk flows this follows authorization immediately. For store pre-auth flows, capture happens after basket reconciliation.

**Consumers:** `ivm-order-service`, `ivm-audit-service`, `ivm-analytics-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/payment/payment-captured.v1.json",
  "title": "ivm.payment.captured.v1 Data",
  "type": "object",
  "required": ["paymentId", "orderId", "capturedAmount", "gateway", "gatewayRef", "capturedAt"],
  "properties": {
    "paymentId": { "type": "string", "pattern": "^pay_[0-9A-HJKMNP-TV-Z]{26}$" },
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "authorizedAmount": {
      "type": "object",
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "capturedAmount": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": { "amount": { "type": "integer", "minimum": 0 }, "currency": { "type": "string" } }
    },
    "gateway": { "type": "string" },
    "gatewayRef": { "type": "string" },
    "partialCapture": { "type": "boolean", "description": "True if captured amount differs from authorized amount." },
    "capturedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.4.4 ivm.payment.failed.v1

**Description:** Emitted when a payment attempt fails (card declined, insufficient funds, gateway error). The Order Service may transition the order to allow retry or cancel.

**Consumers:** `ivm-order-service` (handle payment failure), `ivm-workflow-service`, `ivm-notification-service` (notify customer), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/payment/payment-failed.v1.json",
  "title": "ivm.payment.failed.v1 Data",
  "type": "object",
  "required": ["paymentId", "orderId", "customerId", "amount", "failureReason", "failureCode", "failedAt"],
  "properties": {
    "paymentId": { "type": "string", "pattern": "^pay_[0-9A-HJKMNP-TV-Z]{26}$" },
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "amount": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "method": { "type": "string", "enum": ["CARD", "MOBILE_MONEY", "BANK_TRANSFER", "WALLET", "USSD", "QR"] },
    "gateway": { "type": "string" },
    "gatewayRef": { "type": ["string", "null"] },
    "failureReason": {
      "type": "string",
      "enum": ["CARD_DECLINED", "INSUFFICIENT_FUNDS", "EXPIRED_CARD", "INVALID_CARD", "GATEWAY_ERROR", "TIMEOUT", "FRAUD_SUSPECTED", "3DS_FAILED", "PROVIDER_UNAVAILABLE"],
      "description": "Categorized failure reason."
    },
    "failureCode": { "type": "string", "description": "Gateway-specific error code." },
    "failureMessage": { "type": ["string", "null"], "description": "Human-readable failure message." },
    "retryable": { "type": "boolean" },
    "attemptNumber": { "type": "integer", "minimum": 1 },
    "failedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.4.5 ivm.payment.refunded.v1

**Description:** Emitted when a payment is refunded (full or partial). Typically triggered by order cancellation or dispute resolution.

**Consumers:** `ivm-order-service`, `ivm-notification-service` (notify customer of refund), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/payment/payment-refunded.v1.json",
  "title": "ivm.payment.refunded.v1 Data",
  "type": "object",
  "required": ["paymentId", "orderId", "refundId", "refundedAmount", "refundType", "refundedAt"],
  "properties": {
    "paymentId": { "type": "string", "pattern": "^pay_[0-9A-HJKMNP-TV-Z]{26}$" },
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "refundId": { "type": "string", "pattern": "^rfd_[0-9A-HJKMNP-TV-Z]{26}$" },
    "refundedAmount": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": { "amount": { "type": "integer", "minimum": 1 }, "currency": { "type": "string" } }
    },
    "originalAmount": {
      "type": "object",
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "refundType": { "type": "string", "enum": ["FULL", "PARTIAL"] },
    "reason": { "type": "string", "enum": ["ORDER_CANCELLED", "DISPUTE_RESOLVED", "OVERCHARGE", "SERVICE_NOT_DELIVERED", "OPERATOR_REQUEST"] },
    "gateway": { "type": "string" },
    "gatewayRefundRef": { "type": "string" },
    "refundedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.4.6 ivm.payment.settled.v1

**Description:** Emitted when funds from a captured payment are settled to the merchant's bank account. This is a batch process that typically runs on the settlement schedule (daily, weekly, or instant).

**Consumers:** `ivm-analytics-service` (revenue tracking), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/payment/payment-settled.v1.json",
  "title": "ivm.payment.settled.v1 Data",
  "type": "object",
  "required": ["paymentId", "orderId", "settlementId", "grossAmount", "commissionAmount", "netAmount", "settledAt"],
  "properties": {
    "paymentId": { "type": "string", "pattern": "^pay_[0-9A-HJKMNP-TV-Z]{26}$" },
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "settlementId": { "type": "string", "pattern": "^stl_[0-9A-HJKMNP-TV-Z]{26}$" },
    "merchantId": { "type": "string", "description": "Merchant receiving the settlement." },
    "grossAmount": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "commissionAmount": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "netAmount": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "settlementBankCode": { "type": "string" },
    "settlementAccountNumber": { "type": "string", "description": "Masked account number (last 4 digits visible)." },
    "payoutRef": { "type": ["string", "null"], "description": "Bank payout reference." },
    "settledAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

### 4.5 Device Events

#### 4.5.1 ivm.device.online.v1

**Description:** Emitted when a device connects and reports as online (via MQTT heartbeat bridged to Kafka). Used for fleet health dashboards.

**Consumers:** `ivm-site-service`, `ivm-telemetry-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/device/device-online.v1.json",
  "title": "ivm.device.online.v1 Data",
  "type": "object",
  "required": ["deviceId", "siteId", "deviceType", "firmwareVersion", "onlineAt"],
  "properties": {
    "deviceId": { "type": "string", "pattern": "^dev_[0-9A-HJKMNP-TV-Z]{26}$" },
    "siteId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "deviceType": { "type": "string", "enum": ["TOUCHSCREEN", "QR_SCANNER", "BARCODE_SCANNER", "RECEIPT_PRINTER", "THERMAL_PRINTER", "CARD_READER", "PIN_PAD", "BIOMETRIC_SCANNER", "CAMERA", "LOCKER_CONTROLLER", "SHELF_SENSOR", "WEIGHT_SENSOR", "ELECTRONIC_LOCK", "VENDING_MOTOR", "GATE_CONTROLLER", "CASH_ACCEPTOR", "CASH_DISPENSER", "EDGE_CONTROLLER", "CELLULAR_MODEM", "NFC_READER"] },
    "firmwareVersion": { "type": "string" },
    "softwareVersion": { "type": ["string", "null"] },
    "ipAddress": { "type": ["string", "null"] },
    "connectionType": { "type": "string", "enum": ["ETHERNET", "WIFI", "CELLULAR", "USB", "SERIAL"], "description": "How the device connects to the edge controller or cloud." },
    "onlineAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.5.2 ivm.device.offline.v1

**Description:** Emitted when a device stops responding (heartbeat timeout). The Site Service assesses site health impact, and the Notification Service may alert operators.

**Consumers:** `ivm-site-service` (assess health impact), `ivm-telemetry-service`, `ivm-notification-service` (alert operators), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/device/device-offline.v1.json",
  "title": "ivm.device.offline.v1 Data",
  "type": "object",
  "required": ["deviceId", "siteId", "deviceType", "lastSeenAt", "detectedAt"],
  "properties": {
    "deviceId": { "type": "string", "pattern": "^dev_[0-9A-HJKMNP-TV-Z]{26}$" },
    "siteId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "deviceType": { "type": "string" },
    "lastSeenAt": { "type": "string", "format": "date-time", "description": "Last successful heartbeat timestamp." },
    "missedHeartbeats": { "type": "integer", "minimum": 1, "description": "Number of consecutive missed heartbeats." },
    "detectedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.5.3 ivm.device.health_reported.v1

**Description:** Emitted periodically when a device reports its health metrics (CPU, memory, disk, temperature, error counts). High-volume event with short retention.

**Consumers:** `ivm-telemetry-service` (store time-series), `ivm-analytics-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/device/device-health-reported.v1.json",
  "title": "ivm.device.health_reported.v1 Data",
  "type": "object",
  "required": ["deviceId", "siteId", "metrics", "reportedAt"],
  "properties": {
    "deviceId": { "type": "string", "pattern": "^dev_[0-9A-HJKMNP-TV-Z]{26}$" },
    "siteId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "metrics": {
      "type": "object",
      "properties": {
        "cpuUsagePercent": { "type": "number", "minimum": 0, "maximum": 100 },
        "memoryUsagePercent": { "type": "number", "minimum": 0, "maximum": 100 },
        "diskUsagePercent": { "type": "number", "minimum": 0, "maximum": 100 },
        "temperatureCelsius": { "type": ["number", "null"] },
        "uptimeSeconds": { "type": "integer", "minimum": 0 },
        "errorCount": { "type": "integer", "minimum": 0 },
        "networkLatencyMs": { "type": ["number", "null"] }
      },
      "description": "Device health metrics snapshot."
    },
    "overallStatus": { "type": "string", "enum": ["HEALTHY", "DEGRADED", "CRITICAL"], "description": "Computed overall health status." },
    "reportedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.5.4 ivm.device.fault_detected.v1

**Description:** Emitted when a device fault is detected (hardware failure, sensor malfunction, communication error). Triggers operator notification and may put device into maintenance mode.

**Consumers:** `ivm-site-service`, `ivm-notification-service` (alert operators/technicians), `ivm-telemetry-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/device/device-fault-detected.v1.json",
  "title": "ivm.device.fault_detected.v1 Data",
  "type": "object",
  "required": ["deviceId", "siteId", "faultCode", "faultCategory", "severity", "detectedAt"],
  "properties": {
    "deviceId": { "type": "string", "pattern": "^dev_[0-9A-HJKMNP-TV-Z]{26}$" },
    "siteId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "faultCode": { "type": "string", "description": "Machine-readable fault code (e.g., PRINTER_JAM, MOTOR_STUCK, SENSOR_TIMEOUT)." },
    "faultCategory": { "type": "string", "enum": ["HARDWARE", "SOFTWARE", "NETWORK", "SENSOR", "MECHANICAL", "POWER"] },
    "severity": { "type": "string", "enum": ["LOW", "MEDIUM", "HIGH", "CRITICAL"] },
    "description": { "type": "string", "description": "Human-readable fault description." },
    "affectedCapabilities": {
      "type": "array",
      "items": { "type": "string" },
      "description": "List of device capabilities affected by this fault."
    },
    "autoRecoveryAttempted": { "type": "boolean" },
    "autoRecoverySucceeded": { "type": ["boolean", "null"] },
    "detectedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.5.5 ivm.device.command_executed.v1

**Description:** Emitted when a device command (dispense, unlock door, open gate, print, etc.) completes execution. The Workflow Service uses this to advance workflow steps. The Locker Module uses this for door confirmation.

**Consumers:** `ivm-workflow-service` (advance step), `ivm-locker-module` (door confirmation), `ivm-store-module` (gate confirmation), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/device/device-command-executed.v1.json",
  "title": "ivm.device.command_executed.v1 Data",
  "type": "object",
  "required": ["deviceId", "siteId", "commandId", "commandType", "status", "executedAt"],
  "properties": {
    "deviceId": { "type": "string", "pattern": "^dev_[0-9A-HJKMNP-TV-Z]{26}$" },
    "siteId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "commandId": { "type": "string", "pattern": "^cmd_[0-9A-HJKMNP-TV-Z]{26}$" },
    "commandType": {
      "type": "string",
      "enum": ["DISPENSE", "UNLOCK_DOOR", "LOCK_DOOR", "OPEN_GATE", "CLOSE_GATE", "PRINT_RECEIPT", "PRINT_DOCUMENT", "SCAN_QR", "SCAN_BARCODE", "CAPTURE_PHOTO", "CAPTURE_FINGERPRINT", "REBOOT", "DIAGNOSTIC"],
      "description": "Type of command that was executed."
    },
    "status": { "type": "string", "enum": ["SUCCESS", "FAILED", "TIMEOUT"] },
    "durationMs": { "type": "integer", "minimum": 0, "description": "Command execution duration in milliseconds." },
    "failureReason": { "type": ["string", "null"], "description": "Reason for failure if status is FAILED." },
    "metadata": { "type": ["object", "null"], "description": "Command-specific result metadata." },
    "executedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.5.6 ivm.device.update_started.v1

**Description:** Emitted when an OTA firmware/software update begins on a device.

**Consumers:** `ivm-telemetry-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/device/device-update-started.v1.json",
  "title": "ivm.device.update_started.v1 Data",
  "type": "object",
  "required": ["deviceId", "siteId", "updateId", "currentVersion", "targetVersion", "updateType", "startedAt"],
  "properties": {
    "deviceId": { "type": "string", "pattern": "^dev_[0-9A-HJKMNP-TV-Z]{26}$" },
    "siteId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "updateId": { "type": "string", "pattern": "^upd_[0-9A-HJKMNP-TV-Z]{26}$" },
    "currentVersion": { "type": "string" },
    "targetVersion": { "type": "string" },
    "updateType": { "type": "string", "enum": ["FIRMWARE", "SOFTWARE", "CONFIG", "MODEL"], "description": "Type of update being applied." },
    "updateSizeBytes": { "type": "integer", "minimum": 0 },
    "rolloutStrategy": { "type": "string", "enum": ["CANARY", "PERCENTAGE", "FULL"], "description": "How this update is being rolled out." },
    "startedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.5.7 ivm.device.update_completed.v1

**Description:** Emitted when a device update completes successfully.

**Consumers:** `ivm-telemetry-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/device/device-update-completed.v1.json",
  "title": "ivm.device.update_completed.v1 Data",
  "type": "object",
  "required": ["deviceId", "siteId", "updateId", "previousVersion", "newVersion", "completedAt"],
  "properties": {
    "deviceId": { "type": "string", "pattern": "^dev_[0-9A-HJKMNP-TV-Z]{26}$" },
    "siteId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "updateId": { "type": "string", "pattern": "^upd_[0-9A-HJKMNP-TV-Z]{26}$" },
    "previousVersion": { "type": "string" },
    "newVersion": { "type": "string" },
    "updateType": { "type": "string", "enum": ["FIRMWARE", "SOFTWARE", "CONFIG", "MODEL"] },
    "durationSeconds": { "type": "integer", "minimum": 0 },
    "completedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.5.8 ivm.device.update_failed.v1

**Description:** Emitted when a device update fails. The device should automatically roll back to the previous version. Alerts operators for investigation.

**Consumers:** `ivm-notification-service` (alert operators), `ivm-telemetry-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/device/device-update-failed.v1.json",
  "title": "ivm.device.update_failed.v1 Data",
  "type": "object",
  "required": ["deviceId", "siteId", "updateId", "targetVersion", "failureReason", "rolledBack", "failedAt"],
  "properties": {
    "deviceId": { "type": "string", "pattern": "^dev_[0-9A-HJKMNP-TV-Z]{26}$" },
    "siteId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "updateId": { "type": "string", "pattern": "^upd_[0-9A-HJKMNP-TV-Z]{26}$" },
    "targetVersion": { "type": "string" },
    "failureReason": { "type": "string", "enum": ["DOWNLOAD_FAILED", "CHECKSUM_MISMATCH", "INSTALL_FAILED", "BOOT_FAILED", "TIMEOUT", "DISK_FULL", "SIGNATURE_INVALID"] },
    "failureDetail": { "type": ["string", "null"] },
    "rolledBack": { "type": "boolean", "description": "Whether the device successfully rolled back to the previous version." },
    "currentVersion": { "type": "string", "description": "Version the device is currently running after the failure." },
    "failedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

### 4.6 Locker Events

#### 4.6.1 ivm.locker.compartment_reserved.v1

**Description:** Emitted when a locker compartment is reserved for an order. Typically triggered by `ivm.order.created.v1` for locker-channel orders, or by a courier initiating a deposit.

**Consumers:** `ivm-order-service` (link reservation to order), `ivm-notification-service` (notify recipient), `ivm-audit-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M7ABCD8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-locker-module",
  "type": "ivm.locker.compartment_reserved.v1",
  "time": "2026-03-12T14:00:05.000Z",
  "datacontenttype": "application/json",
  "subject": "reservation/rsv_01J5M7ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789d-1234567890abcdec-01",
  "correlationid": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "reservationId": "rsv_01J5M7ABCD8N3P4Q5R6S7T8U9V",
    "lockerBankId": "lbk_01J5M7BCDE8N3P4Q5R6S7T8U9V",
    "compartmentId": "cmp_01J5M7CDEF8N3P4Q5R6S7T8U9V",
    "compartmentNumber": "B3",
    "compartmentSize": "MEDIUM",
    "orderId": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
    "recipientId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "reservationType": "PICKUP",
    "accessMethod": "QR",
    "siteId": "site_01J5M3QRST8N3P4Q5R6S7T8U9V",
    "expiresAt": "2026-03-14T14:00:05.000Z",
    "pickupSlaMinutes": 2880,
    "reservedAt": "2026-03-12T14:00:05.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/locker/compartment-reserved.v1.json",
  "title": "ivm.locker.compartment_reserved.v1 Data",
  "type": "object",
  "required": ["reservationId", "lockerBankId", "compartmentId", "compartmentNumber", "compartmentSize", "orderId", "recipientId", "reservationType", "accessMethod", "siteId", "expiresAt", "reservedAt"],
  "properties": {
    "reservationId": { "type": "string", "pattern": "^rsv_[0-9A-HJKMNP-TV-Z]{26}$" },
    "lockerBankId": { "type": "string", "pattern": "^lbk_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentId": { "type": "string", "pattern": "^cmp_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentNumber": { "type": "string", "description": "Human-readable compartment identifier (e.g., A1, B3)." },
    "compartmentSize": { "type": "string", "enum": ["SMALL", "MEDIUM", "LARGE", "XL", "REFRIGERATED"] },
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "recipientId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "depositorId": { "type": ["string", "null"], "description": "Courier or depositor ID if known at reservation time." },
    "reservationType": { "type": "string", "enum": ["PICKUP", "DROPOFF", "RETURN"] },
    "accessMethod": { "type": "string", "enum": ["PIN", "QR", "OTP", "APP"] },
    "siteId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "expiresAt": { "type": "string", "format": "date-time" },
    "pickupSlaMinutes": { "type": "integer", "minimum": 1 },
    "reservedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.6.2 ivm.locker.parcel_deposited.v1

**Description:** Emitted when a parcel is deposited into a reserved compartment (door closed and confirmed by sensor). Triggers pickup notification to the recipient.

**Consumers:** `ivm-order-service` (update fulfillment status), `ivm-notification-service` (notify recipient of pickup readiness), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/locker/parcel-deposited.v1.json",
  "title": "ivm.locker.parcel_deposited.v1 Data",
  "type": "object",
  "required": ["reservationId", "lockerBankId", "compartmentId", "orderId", "depositorId", "depositedAt"],
  "properties": {
    "reservationId": { "type": "string", "pattern": "^rsv_[0-9A-HJKMNP-TV-Z]{26}$" },
    "lockerBankId": { "type": "string", "pattern": "^lbk_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentId": { "type": "string", "pattern": "^cmp_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentNumber": { "type": "string" },
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "depositorId": { "type": "string", "description": "Courier or operator who deposited the parcel." },
    "recipientId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "siteId": { "type": "string" },
    "depositedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.6.3 ivm.locker.door_opened.v1

**Description:** Emitted when a locker compartment door is physically opened (detected by door sensor). High-frequency operational event.

**Consumers:** `ivm-telemetry-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/locker/door-opened.v1.json",
  "title": "ivm.locker.door_opened.v1 Data",
  "type": "object",
  "required": ["lockerBankId", "compartmentId", "compartmentNumber", "trigger", "openedAt"],
  "properties": {
    "lockerBankId": { "type": "string", "pattern": "^lbk_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentId": { "type": "string", "pattern": "^cmp_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentNumber": { "type": "string" },
    "reservationId": { "type": ["string", "null"] },
    "trigger": { "type": "string", "enum": ["CUSTOMER_ACCESS", "COURIER_DEPOSIT", "OPERATOR_OVERRIDE", "MAINTENANCE", "SYSTEM_COMMAND"] },
    "actorId": { "type": ["string", "null"], "description": "ID of the person or system that triggered the door open." },
    "siteId": { "type": "string" },
    "openedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.6.4 ivm.locker.door_closed.v1

**Description:** Emitted when a locker compartment door is closed (detected by door sensor).

**Consumers:** `ivm-telemetry-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/locker/door-closed.v1.json",
  "title": "ivm.locker.door_closed.v1 Data",
  "type": "object",
  "required": ["lockerBankId", "compartmentId", "compartmentNumber", "doorOpenDurationSeconds", "closedAt"],
  "properties": {
    "lockerBankId": { "type": "string", "pattern": "^lbk_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentId": { "type": "string", "pattern": "^cmp_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentNumber": { "type": "string" },
    "reservationId": { "type": ["string", "null"] },
    "doorOpenDurationSeconds": { "type": "integer", "minimum": 0, "description": "How long the door was open." },
    "siteId": { "type": "string" },
    "closedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.6.5 ivm.locker.parcel_picked_up.v1

**Description:** Emitted when a customer picks up their parcel (verified by access code and door close sensor). Completes the locker fulfillment task.

**Consumers:** `ivm-order-service` (complete fulfillment task), `ivm-notification-service` (send receipt), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/locker/parcel-picked-up.v1.json",
  "title": "ivm.locker.parcel_picked_up.v1 Data",
  "type": "object",
  "required": ["reservationId", "lockerBankId", "compartmentId", "orderId", "customerId", "accessMethod", "pickedUpAt"],
  "properties": {
    "reservationId": { "type": "string", "pattern": "^rsv_[0-9A-HJKMNP-TV-Z]{26}$" },
    "lockerBankId": { "type": "string", "pattern": "^lbk_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentId": { "type": "string", "pattern": "^cmp_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentNumber": { "type": "string" },
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "accessMethod": { "type": "string", "enum": ["PIN", "QR", "OTP", "APP"] },
    "dwellTimeMinutes": { "type": "integer", "minimum": 0, "description": "Time between deposit and pickup in minutes." },
    "siteId": { "type": "string" },
    "pickedUpAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.6.6 ivm.locker.reservation_expired.v1

**Description:** Emitted when a reservation exceeds its pickup SLA. Triggers escalation: reminder notifications, potential return-to-sender flow.

**Consumers:** `ivm-order-service` (update order status), `ivm-notification-service` (send escalation reminders), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/locker/reservation-expired.v1.json",
  "title": "ivm.locker.reservation_expired.v1 Data",
  "type": "object",
  "required": ["reservationId", "lockerBankId", "compartmentId", "orderId", "recipientId", "remindersSent", "expiredAt"],
  "properties": {
    "reservationId": { "type": "string", "pattern": "^rsv_[0-9A-HJKMNP-TV-Z]{26}$" },
    "lockerBankId": { "type": "string", "pattern": "^lbk_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentId": { "type": "string", "pattern": "^cmp_[0-9A-HJKMNP-TV-Z]{26}$" },
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "recipientId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "remindersSent": { "type": "integer", "minimum": 0, "description": "Number of reminder notifications sent before expiry." },
    "dwellTimeMinutes": { "type": "integer", "description": "Total time the parcel occupied the compartment." },
    "siteId": { "type": "string" },
    "expiredAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.6.7 ivm.locker.compartment_released.v1

**Description:** Emitted when a compartment is released back to the available pool (after pickup, expiry handling, or cancellation).

**Consumers:** `ivm-audit-service`, `ivm-analytics-service` (utilization tracking)

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/locker/compartment-released.v1.json",
  "title": "ivm.locker.compartment_released.v1 Data",
  "type": "object",
  "required": ["lockerBankId", "compartmentId", "compartmentNumber", "releaseReason", "releasedAt"],
  "properties": {
    "lockerBankId": { "type": "string", "pattern": "^lbk_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentId": { "type": "string", "pattern": "^cmp_[0-9A-HJKMNP-TV-Z]{26}$" },
    "compartmentNumber": { "type": "string" },
    "previousReservationId": { "type": ["string", "null"] },
    "releaseReason": { "type": "string", "enum": ["PARCEL_PICKED_UP", "RESERVATION_EXPIRED", "RESERVATION_CANCELLED", "MAINTENANCE_COMPLETE", "OPERATOR_OVERRIDE"] },
    "occupiedDurationMinutes": { "type": ["integer", "null"], "description": "How long the compartment was occupied." },
    "siteId": { "type": "string" },
    "releasedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

### 4.7 Store Events

#### 4.7.1 ivm.store.session_started.v1

**Description:** Emitted when an autonomous store shopping session is created after customer authentication and payment pre-authorization at the gate.

**Consumers:** `ivm-audit-service`, `ivm-analytics-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M8ABCD8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-store-module",
  "type": "ivm.store.session_started.v1",
  "time": "2026-03-12T15:00:00.000Z",
  "datacontenttype": "application/json",
  "subject": "session/ssn_01J5M8ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789e-1234567890abcded-01",
  "correlationid": "ssn_01J5M8ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "sessionId": "ssn_01J5M8ABCD8N3P4Q5R6S7T8U9V",
    "storeId": "site_01J5M3STORE8N3P4Q5R6S7T8U9V",
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "entryMethod": "QR",
    "preAuthPaymentId": "pay_01J5M8BCDE8N3P4Q5R6S7T8U9V",
    "preAuthAmount": { "amount": 5000000, "currency": "NGN" },
    "startedAt": "2026-03-12T15:00:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/store/session-started.v1.json",
  "title": "ivm.store.session_started.v1 Data",
  "type": "object",
  "required": ["sessionId", "storeId", "customerId", "entryMethod", "startedAt"],
  "properties": {
    "sessionId": { "type": "string", "pattern": "^ssn_[0-9A-HJKMNP-TV-Z]{26}$" },
    "storeId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "entryMethod": { "type": "string", "enum": ["QR", "NFC", "CARD", "FACE"] },
    "preAuthPaymentId": { "type": ["string", "null"] },
    "preAuthAmount": {
      "type": ["object", "null"],
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "startedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.7.2 ivm.store.customer_entered.v1

**Description:** Emitted when the gate opens and the customer physically enters the store (confirmed by gate sensor).

**Consumers:** `ivm-audit-service`, `ivm-analytics-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/store/customer-entered.v1.json",
  "title": "ivm.store.customer_entered.v1 Data",
  "type": "object",
  "required": ["sessionId", "storeId", "customerId", "gateId", "enteredAt"],
  "properties": {
    "sessionId": { "type": "string", "pattern": "^ssn_[0-9A-HJKMNP-TV-Z]{26}$" },
    "storeId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "gateId": { "type": "string", "description": "Device ID of the entry gate." },
    "enteredAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.7.3 ivm.store.item_picked.v1

**Description:** Emitted when the AI/vision system detects a customer picking an item from a shelf. High-volume event during shopping.

**Consumers:** `ivm-analytics-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/store/item-picked.v1.json",
  "title": "ivm.store.item_picked.v1 Data",
  "type": "object",
  "required": ["sessionId", "storeId", "customerId", "catalogItemId", "quantity", "confidence", "detectionSource", "pickedAt"],
  "properties": {
    "sessionId": { "type": "string", "pattern": "^ssn_[0-9A-HJKMNP-TV-Z]{26}$" },
    "storeId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "catalogItemId": { "type": "string", "description": "Product catalog item ID." },
    "itemName": { "type": "string" },
    "quantity": { "type": "integer", "minimum": 1 },
    "unitPrice": {
      "type": "object",
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "confidence": { "type": "number", "minimum": 0.0, "maximum": 1.0, "description": "AI detection confidence score." },
    "detectionSource": { "type": "string", "enum": ["VISION", "SHELF_SENSOR", "RFID", "WEIGHT"] },
    "shelfLocation": { "type": ["string", "null"], "description": "Shelf zone identifier." },
    "evidenceRefs": { "type": "array", "items": { "type": "string" }, "description": "Frame IDs or sensor event refs." },
    "pickedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.7.4 ivm.store.item_returned.v1

**Description:** Emitted when the AI/vision system detects a customer returning an item to a shelf (removing it from their virtual cart).

**Consumers:** `ivm-analytics-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/store/item-returned.v1.json",
  "title": "ivm.store.item_returned.v1 Data",
  "type": "object",
  "required": ["sessionId", "storeId", "customerId", "catalogItemId", "quantity", "confidence", "detectionSource", "returnedAt"],
  "properties": {
    "sessionId": { "type": "string", "pattern": "^ssn_[0-9A-HJKMNP-TV-Z]{26}$" },
    "storeId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "catalogItemId": { "type": "string" },
    "itemName": { "type": "string" },
    "quantity": { "type": "integer", "minimum": 1 },
    "confidence": { "type": "number", "minimum": 0.0, "maximum": 1.0 },
    "detectionSource": { "type": "string", "enum": ["VISION", "SHELF_SENSOR", "RFID", "WEIGHT"] },
    "shelfLocation": { "type": ["string", "null"] },
    "returnedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.7.5 ivm.store.customer_exited.v1

**Description:** Emitted when the customer exits the store through the gate. Triggers basket reconciliation.

**Consumers:** `ivm-audit-service`, `ivm-analytics-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/store/customer-exited.v1.json",
  "title": "ivm.store.customer_exited.v1 Data",
  "type": "object",
  "required": ["sessionId", "storeId", "customerId", "dwellTimeSeconds", "itemCount", "exitedAt"],
  "properties": {
    "sessionId": { "type": "string", "pattern": "^ssn_[0-9A-HJKMNP-TV-Z]{26}$" },
    "storeId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "gateId": { "type": "string" },
    "dwellTimeSeconds": { "type": "integer", "minimum": 0, "description": "Time spent in store in seconds." },
    "itemCount": { "type": "integer", "minimum": 0, "description": "Number of unique items in virtual basket at exit." },
    "exitedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.7.6 ivm.store.basket_reconciled.v1

**Description:** Emitted after the final basket is reconciled post-exit. This is the definitive basket used for charging. If confidence is below threshold, the session is flagged for manual review.

**Consumers:** `ivm-order-service` (create order from basket), `ivm-payment-service` (capture payment), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/store/basket-reconciled.v1.json",
  "title": "ivm.store.basket_reconciled.v1 Data",
  "type": "object",
  "required": ["sessionId", "storeId", "customerId", "basketItems", "basketTotal", "overallConfidence", "requiresReview", "reconciledAt"],
  "properties": {
    "sessionId": { "type": "string", "pattern": "^ssn_[0-9A-HJKMNP-TV-Z]{26}$" },
    "storeId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "basketItems": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["catalogItemId", "itemName", "quantity", "unitPrice", "lineTotal", "confidence"],
        "properties": {
          "catalogItemId": { "type": "string" },
          "itemName": { "type": "string" },
          "quantity": { "type": "integer", "minimum": 1 },
          "unitPrice": { "type": "object", "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } } },
          "lineTotal": { "type": "object", "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } } },
          "confidence": { "type": "number", "minimum": 0.0, "maximum": 1.0 }
        }
      }
    },
    "basketTotal": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "overallConfidence": { "type": "number", "minimum": 0.0, "maximum": 1.0 },
    "requiresReview": { "type": "boolean", "description": "True if overall confidence is below threshold." },
    "reconciledAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.7.7 ivm.store.session_charged.v1

**Description:** Emitted when the pre-authorized payment is captured for the reconciled basket amount. The receipt is sent to the customer.

**Consumers:** `ivm-notification-service` (send receipt), `ivm-audit-service`, `ivm-analytics-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/store/session-charged.v1.json",
  "title": "ivm.store.session_charged.v1 Data",
  "type": "object",
  "required": ["sessionId", "storeId", "customerId", "orderId", "paymentId", "chargedAmount", "chargedAt"],
  "properties": {
    "sessionId": { "type": "string", "pattern": "^ssn_[0-9A-HJKMNP-TV-Z]{26}$" },
    "storeId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "orderId": { "type": "string", "pattern": "^ord_[0-9A-HJKMNP-TV-Z]{26}$" },
    "paymentId": { "type": "string", "pattern": "^pay_[0-9A-HJKMNP-TV-Z]{26}$" },
    "preAuthAmount": {
      "type": "object",
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "chargedAmount": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "itemCount": { "type": "integer", "minimum": 0 },
    "dwellTimeSeconds": { "type": "integer" },
    "chargedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.7.8 ivm.store.session_disputed.v1

**Description:** Emitted when a customer disputes a store session charge (e.g., wrong items, overcharged). Video evidence is preserved for review.

**Consumers:** `ivm-notification-service`, `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/store/session-disputed.v1.json",
  "title": "ivm.store.session_disputed.v1 Data",
  "type": "object",
  "required": ["sessionId", "storeId", "customerId", "disputeId", "disputeReason", "disputedAt"],
  "properties": {
    "sessionId": { "type": "string", "pattern": "^ssn_[0-9A-HJKMNP-TV-Z]{26}$" },
    "storeId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "customerId": { "type": "string", "pattern": "^cst_[0-9A-HJKMNP-TV-Z]{26}$" },
    "orderId": { "type": "string" },
    "disputeId": { "type": "string", "pattern": "^dsp_[0-9A-HJKMNP-TV-Z]{26}$" },
    "disputeReason": { "type": "string", "enum": ["WRONG_ITEMS", "OVERCHARGED", "ITEM_NOT_TAKEN", "UNAUTHORIZED_CHARGE", "OTHER"] },
    "disputeDescription": { "type": ["string", "null"] },
    "chargedAmount": {
      "type": "object",
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "videoSegmentRefs": { "type": "array", "items": { "type": "string" }, "description": "References to video evidence for dispute review." },
    "disputedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.7.9 ivm.store.shrinkage_detected.v1

**Description:** Emitted when the system detects potential inventory shrinkage (items leaving store without being associated with a session, or inventory discrepancies after reconciliation). Used for loss prevention.

**Consumers:** `ivm-notification-service` (alert store operators), `ivm-audit-service`, `ivm-analytics-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/store/shrinkage-detected.v1.json",
  "title": "ivm.store.shrinkage_detected.v1 Data",
  "type": "object",
  "required": ["storeId", "shrinkageId", "detectionMethod", "estimatedLoss", "detectedAt"],
  "properties": {
    "storeId": { "type": "string", "pattern": "^site_[0-9A-HJKMNP-TV-Z]{26}$" },
    "shrinkageId": { "type": "string", "pattern": "^shr_[0-9A-HJKMNP-TV-Z]{26}$" },
    "sessionId": { "type": ["string", "null"], "description": "Associated session if shrinkage is tied to a specific customer." },
    "detectionMethod": { "type": "string", "enum": ["INVENTORY_RECONCILIATION", "SENSOR_ANOMALY", "VIDEO_ANALYSIS", "WEIGHT_DISCREPANCY"] },
    "affectedItems": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "catalogItemId": { "type": "string" },
          "itemName": { "type": "string" },
          "quantity": { "type": "integer" },
          "estimatedValue": { "type": "object", "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } } }
        }
      }
    },
    "estimatedLoss": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": { "amount": { "type": "integer" }, "currency": { "type": "string" } }
    },
    "videoSegmentRefs": { "type": "array", "items": { "type": "string" } },
    "severity": { "type": "string", "enum": ["LOW", "MEDIUM", "HIGH"] },
    "detectedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

### 4.8 Workflow Events

#### 4.8.1 ivm.workflow.started.v1

**Description:** Emitted when a new workflow instance is started (e.g., SIM registration workflow, locker pickup workflow). Typically triggered by an order creation or manual operator action.

**Consumers:** `ivm-audit-service`, `ivm-analytics-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5M9ABCD8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-workflow-service",
  "type": "ivm.workflow.started.v1",
  "time": "2026-03-12T14:00:01.000Z",
  "datacontenttype": "application/json",
  "subject": "workflow/wf_01J5M9ABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef123456789f-1234567890abcdee-01",
  "correlationid": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "workflowInstanceId": "wf_01J5M9ABCD8N3P4Q5R6S7T8U9V",
    "workflowDefinitionId": "wfd_01J5M9BCDE8N3P4Q5R6S7T8U9V",
    "workflowName": "sim-registration-mtn",
    "workflowVersion": 3,
    "channelType": "KIOSK",
    "orderId": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
    "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "siteId": "site_01J5M3QRST8N3P4Q5R6S7T8U9V",
    "deviceId": "dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
    "firstStepId": "step_select_operator",
    "totalSteps": 8,
    "startedAt": "2026-03-12T14:00:01.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/workflow/workflow-started.v1.json",
  "title": "ivm.workflow.started.v1 Data",
  "type": "object",
  "required": ["workflowInstanceId", "workflowDefinitionId", "workflowName", "workflowVersion", "channelType", "firstStepId", "totalSteps", "startedAt"],
  "properties": {
    "workflowInstanceId": { "type": "string", "pattern": "^wf_[0-9A-HJKMNP-TV-Z]{26}$" },
    "workflowDefinitionId": { "type": "string", "pattern": "^wfd_[0-9A-HJKMNP-TV-Z]{26}$" },
    "workflowName": { "type": "string" },
    "workflowVersion": { "type": "integer", "minimum": 1 },
    "channelType": { "type": "string", "enum": ["KIOSK", "LOCKER", "STORE"] },
    "triggerType": { "type": "string", "enum": ["EVENT", "MANUAL", "SCHEDULE"] },
    "orderId": { "type": ["string", "null"] },
    "customerId": { "type": ["string", "null"] },
    "siteId": { "type": ["string", "null"] },
    "deviceId": { "type": ["string", "null"] },
    "firstStepId": { "type": "string" },
    "totalSteps": { "type": "integer", "minimum": 1 },
    "startedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.8.2 ivm.workflow.step_completed.v1

**Description:** Emitted when a workflow step completes successfully. Carries the step output and the next step to execute.

**Consumers:** `ivm-audit-service`, `ivm-analytics-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/workflow/step-completed.v1.json",
  "title": "ivm.workflow.step_completed.v1 Data",
  "type": "object",
  "required": ["workflowInstanceId", "stepId", "stepName", "stepType", "stepIndex", "nextStepId", "completedAt"],
  "properties": {
    "workflowInstanceId": { "type": "string", "pattern": "^wf_[0-9A-HJKMNP-TV-Z]{26}$" },
    "stepId": { "type": "string" },
    "stepName": { "type": "string" },
    "stepType": { "type": "string", "enum": ["USER_INPUT", "SYSTEM_ACTION", "APPROVAL", "TIMER", "BRANCH", "PARALLEL", "COMPENSATION"] },
    "stepIndex": { "type": "integer", "minimum": 0, "description": "Zero-based index of the step in the workflow." },
    "durationMs": { "type": "integer", "minimum": 0, "description": "Time to complete this step in milliseconds." },
    "output": { "type": ["object", "null"], "description": "Step output data (varies by step type)." },
    "nextStepId": { "type": ["string", "null"], "description": "Next step to execute. Null if this was the last step." },
    "completedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.8.3 ivm.workflow.step_failed.v1

**Description:** Emitted when a workflow step fails. May trigger retry, compensation, or workflow failure depending on the step's retry policy.

**Consumers:** `ivm-notification-service` (alert operators if critical), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/workflow/step-failed.v1.json",
  "title": "ivm.workflow.step_failed.v1 Data",
  "type": "object",
  "required": ["workflowInstanceId", "stepId", "stepName", "stepType", "failureReason", "retryable", "failedAt"],
  "properties": {
    "workflowInstanceId": { "type": "string", "pattern": "^wf_[0-9A-HJKMNP-TV-Z]{26}$" },
    "stepId": { "type": "string" },
    "stepName": { "type": "string" },
    "stepType": { "type": "string", "enum": ["USER_INPUT", "SYSTEM_ACTION", "APPROVAL", "TIMER", "BRANCH", "PARALLEL", "COMPENSATION"] },
    "failureReason": { "type": "string", "description": "Categorized failure reason." },
    "failureDetail": { "type": ["string", "null"] },
    "attemptNumber": { "type": "integer", "minimum": 1 },
    "maxRetries": { "type": "integer", "minimum": 0 },
    "retryable": { "type": "boolean" },
    "willRetry": { "type": "boolean", "description": "Whether the system will automatically retry." },
    "compensationStepId": { "type": ["string", "null"], "description": "Compensation step to execute if retries exhausted." },
    "failedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.8.4 ivm.workflow.completed.v1

**Description:** Emitted when a workflow instance completes all steps successfully. The Order Service may use this to finalize the order.

**Consumers:** `ivm-order-service` (finalize order), `ivm-notification-service`, `ivm-audit-service`, `ivm-analytics-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/workflow/workflow-completed.v1.json",
  "title": "ivm.workflow.completed.v1 Data",
  "type": "object",
  "required": ["workflowInstanceId", "workflowName", "orderId", "stepsCompleted", "totalDurationMs", "completedAt"],
  "properties": {
    "workflowInstanceId": { "type": "string", "pattern": "^wf_[0-9A-HJKMNP-TV-Z]{26}$" },
    "workflowDefinitionId": { "type": "string" },
    "workflowName": { "type": "string" },
    "orderId": { "type": ["string", "null"] },
    "customerId": { "type": ["string", "null"] },
    "stepsCompleted": { "type": "integer", "minimum": 1 },
    "totalDurationMs": { "type": "integer", "minimum": 0 },
    "output": { "type": ["object", "null"], "description": "Final workflow output/result data." },
    "completedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.8.5 ivm.workflow.failed.v1

**Description:** Emitted when a workflow instance fails terminally (all retries exhausted, no compensation possible, or critical error). The Order Service may cancel the associated order.

**Consumers:** `ivm-order-service` (cancel order), `ivm-notification-service` (notify customer/operator), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/workflow/workflow-failed.v1.json",
  "title": "ivm.workflow.failed.v1 Data",
  "type": "object",
  "required": ["workflowInstanceId", "workflowName", "failedStepId", "failureReason", "failedAt"],
  "properties": {
    "workflowInstanceId": { "type": "string", "pattern": "^wf_[0-9A-HJKMNP-TV-Z]{26}$" },
    "workflowDefinitionId": { "type": "string" },
    "workflowName": { "type": "string" },
    "orderId": { "type": ["string", "null"] },
    "customerId": { "type": ["string", "null"] },
    "failedStepId": { "type": "string" },
    "failedStepName": { "type": "string" },
    "failureReason": { "type": "string" },
    "failureDetail": { "type": ["string", "null"] },
    "stepsCompletedBeforeFailure": { "type": "integer", "minimum": 0 },
    "compensationTriggered": { "type": "boolean" },
    "failedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.8.6 ivm.workflow.compensating.v1

**Description:** Emitted when a workflow enters compensation mode (saga rollback). Each previously completed step's compensation action is executed in reverse order.

**Consumers:** `ivm-order-service` (handle compensation), `ivm-payment-service` (refund if needed), `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/workflow/workflow-compensating.v1.json",
  "title": "ivm.workflow.compensating.v1 Data",
  "type": "object",
  "required": ["workflowInstanceId", "workflowName", "triggerStepId", "compensationSteps", "compensatingAt"],
  "properties": {
    "workflowInstanceId": { "type": "string", "pattern": "^wf_[0-9A-HJKMNP-TV-Z]{26}$" },
    "workflowName": { "type": "string" },
    "orderId": { "type": ["string", "null"] },
    "triggerStepId": { "type": "string", "description": "The step whose failure triggered compensation." },
    "triggerReason": { "type": "string" },
    "compensationSteps": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["stepId", "compensationAction"],
        "properties": {
          "stepId": { "type": "string" },
          "stepName": { "type": "string" },
          "compensationAction": { "type": "string", "description": "Action to reverse the step (e.g., REFUND_PAYMENT, RELEASE_COMPARTMENT)." },
          "status": { "type": "string", "enum": ["PENDING", "IN_PROGRESS", "COMPLETED", "FAILED"] }
        }
      },
      "description": "Compensation steps to execute in reverse order."
    },
    "compensatingAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

### 4.9 Notification Events

#### 4.9.1 ivm.notification.sent.v1

**Description:** Emitted when a notification is sent to a delivery provider (SMS gateway, email service, push notification service). Note: "sent" means handed off to the provider, not necessarily delivered to the end user.

**Consumers:** `ivm-audit-service`, `ivm-analytics-service`

**CloudEvents Example:**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5MAABCD8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-notification-service",
  "type": "ivm.notification.sent.v1",
  "time": "2026-03-12T14:02:00.000Z",
  "datacontenttype": "application/json",
  "subject": "notification/ntf_01J5MAABCD8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-abcdef1234567890abcdef12345678a0-1234567890abcdef-01",
  "correlationid": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
  "data": {
    "notificationId": "ntf_01J5MAABCD8N3P4Q5R6S7T8U9V",
    "recipientId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
    "channel": "SMS",
    "templateId": "tpl_order_completed",
    "provider": "termii",
    "providerRef": "TERMII-MSG-20260312-abc123",
    "recipient": "+2348012345678",
    "triggerEvent": "ivm.order.completed.v1",
    "sentAt": "2026-03-12T14:02:00.000Z"
  }
}
```

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/notification/notification-sent.v1.json",
  "title": "ivm.notification.sent.v1 Data",
  "type": "object",
  "required": ["notificationId", "recipientId", "channel", "templateId", "provider", "sentAt"],
  "properties": {
    "notificationId": { "type": "string", "pattern": "^ntf_[0-9A-HJKMNP-TV-Z]{26}$" },
    "recipientId": { "type": "string", "description": "Customer or operator ID." },
    "channel": { "type": "string", "enum": ["SMS", "EMAIL", "PUSH", "IN_APP", "WEBHOOK"] },
    "templateId": { "type": "string", "description": "Notification template identifier." },
    "provider": { "type": "string", "description": "Delivery provider (termii, africas_talking, fcm, apns, ses)." },
    "providerRef": { "type": ["string", "null"], "description": "Provider message reference for tracking." },
    "recipient": { "type": "string", "description": "Masked recipient address (phone, email, device token)." },
    "locale": { "type": "string", "description": "Locale used for template rendering (e.g., en, yo, ha, ig, pcm)." },
    "triggerEvent": { "type": ["string", "null"], "description": "Event type that triggered this notification." },
    "sentAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.9.2 ivm.notification.delivered.v1

**Description:** Emitted when the delivery provider confirms the notification reached the end user (delivery receipt from SMS gateway, email delivery confirmation, push delivered).

**Consumers:** `ivm-audit-service`, `ivm-analytics-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/notification/notification-delivered.v1.json",
  "title": "ivm.notification.delivered.v1 Data",
  "type": "object",
  "required": ["notificationId", "recipientId", "channel", "provider", "deliveredAt"],
  "properties": {
    "notificationId": { "type": "string", "pattern": "^ntf_[0-9A-HJKMNP-TV-Z]{26}$" },
    "recipientId": { "type": "string" },
    "channel": { "type": "string", "enum": ["SMS", "EMAIL", "PUSH", "IN_APP", "WEBHOOK"] },
    "provider": { "type": "string" },
    "providerRef": { "type": "string" },
    "deliveryLatencyMs": { "type": "integer", "minimum": 0, "description": "Time between send and delivery confirmation in milliseconds." },
    "deliveredAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

#### 4.9.3 ivm.notification.failed.v1

**Description:** Emitted when notification delivery fails after all retry attempts. Used for monitoring delivery reliability and escalation.

**Consumers:** `ivm-audit-service`

**Data JSON Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/events/notification/notification-failed.v1.json",
  "title": "ivm.notification.failed.v1 Data",
  "type": "object",
  "required": ["notificationId", "recipientId", "channel", "provider", "failureReason", "attemptCount", "failedAt"],
  "properties": {
    "notificationId": { "type": "string", "pattern": "^ntf_[0-9A-HJKMNP-TV-Z]{26}$" },
    "recipientId": { "type": "string" },
    "channel": { "type": "string", "enum": ["SMS", "EMAIL", "PUSH", "IN_APP", "WEBHOOK"] },
    "provider": { "type": "string" },
    "providerRef": { "type": ["string", "null"] },
    "failureReason": {
      "type": "string",
      "enum": ["INVALID_RECIPIENT", "PROVIDER_ERROR", "PROVIDER_TIMEOUT", "RATE_LIMITED", "BLOCKED", "UNSUBSCRIBED", "DEVICE_UNREACHABLE"],
      "description": "Categorized reason for delivery failure."
    },
    "failureDetail": { "type": ["string", "null"] },
    "attemptCount": { "type": "integer", "minimum": 1, "description": "Total delivery attempts made." },
    "firstAttemptAt": { "type": "string", "format": "date-time" },
    "failedAt": { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

---

## 5. Event Routing

### 5.1 Kafka Topic Mapping

All domain events are routed to Kafka topics organized by bounded context. Each topic uses the CloudEvents JSON envelope as the message value.

| Kafka Topic | Domain | Partitioning Key | Partitions | Replication Factor |
|---|---|---|---|---|
| `ivm.identity.events` | Identity & Access | `tenantid` | 12 | 3 |
| `ivm.customer.events` | Customer & KYC | `tenantid` | 12 | 3 |
| `ivm.order.events` | Order & Transaction | `subject` (order ID) | 24 | 3 |
| `ivm.payment.events` | Payment | `subject` (payment ID) | 24 | 3 |
| `ivm.device.events` | Device Fleet | `subject` (device ID) | 24 | 3 |
| `ivm.locker.events` | Smart Locker | `subject` (reservation/compartment ID) | 12 | 3 |
| `ivm.store.events` | Autonomous Store | `subject` (session ID) | 12 | 3 |
| `ivm.workflow.events` | Workflow Engine | `subject` (workflow instance ID) | 12 | 3 |
| `ivm.notification.events` | Notification | `tenantid` | 6 | 3 |

### 5.2 Partitioning Strategy

Events are partitioned to guarantee ordering within a single entity lifecycle:

- **Order events**: Partitioned by order ID -- all events for a single order land on the same partition, preserving creation-to-completion ordering.
- **Payment events**: Partitioned by payment ID for the same reason.
- **Device events**: Partitioned by device ID -- all events from one device are ordered.
- **Store events**: Partitioned by session ID -- all events within a shopping session are ordered.
- **Identity/Customer events**: Partitioned by tenant ID for even distribution.

### 5.3 Consumer Group Naming Convention

```
{consuming-service-name}.{topic-name}
```

Examples:
- `ivm-order-service.ivm.payment.events`
- `ivm-workflow-service.ivm.order.events`
- `ivm-audit-service.ivm.order.events`
- `ivm-notification-service.ivm.locker.events`

### 5.4 Topic Configuration

```yaml
# Standard domain event topic configuration
retention.ms: 604800000          # 7 days default (overridden per topic)
retention.bytes: -1              # No size-based retention
cleanup.policy: delete           # Delete old segments
min.insync.replicas: 2           # At least 2 replicas must acknowledge
compression.type: lz4            # LZ4 compression for throughput
max.message.bytes: 1048576       # 1 MB max message size
message.timestamp.type: CreateTime

# High-retention topics (financial events)
# ivm.order.events, ivm.payment.events:
#   retention.ms: 7776000000     # 90 days
#
# Dispute topics:
#   retention.ms: 31536000000    # 365 days
#
# High-volume topics (device telemetry, store item events):
# ivm.device.events:
#   retention.ms: 259200000      # 3 days for health_reported
#
# Short-retention topics:
# ivm.notification.events:
#   retention.ms: 604800000      # 7 days
```

### 5.5 Event Flow Diagram

```
                             ┌──────────────────────────┐
                             │      KAFKA CLUSTER        │
                             │                          │
  ivm-identity-service ────►│ ivm.identity.events      │───► ivm-customer-service
                             │                          │───► ivm-audit-service
                             │                          │
  ivm-customer-service ────►│ ivm.customer.events      │───► ivm-kyc-service
  ivm-kyc-service ─────────►│                          │───► ivm-order-service
                             │                          │───► ivm-notification-service
                             │                          │
  ivm-order-service ───────►│ ivm.order.events         │───► ivm-payment-service
                             │                          │───► ivm-workflow-service
                             │                          │───► ivm-locker-module
                             │                          │───► ivm-notification-service
                             │                          │
  ivm-payment-service ─────►│ ivm.payment.events       │───► ivm-order-service
                             │                          │───► ivm-workflow-service
                             │                          │───► ivm-notification-service
                             │                          │───► ivm-store-module
                             │                          │
  ivm-device-service ──────►│ ivm.device.events        │───► ivm-site-service
  (via MQTT Bridge) ────────►│                          │───► ivm-telemetry-service
                             │                          │───► ivm-workflow-service
                             │                          │───► ivm-locker-module
                             │                          │
  ivm-locker-module ───────►│ ivm.locker.events        │───► ivm-order-service
                             │                          │───► ivm-notification-service
                             │                          │
  ivm-store-module ────────►│ ivm.store.events         │───► ivm-order-service
                             │                          │───► ivm-payment-service
                             │                          │───► ivm-notification-service
                             │                          │
  ivm-workflow-service ────►│ ivm.workflow.events      │───► ivm-order-service
                             │                          │───► ivm-notification-service
                             │                          │
  ivm-notification-service ►│ ivm.notification.events  │───► (terminal)
                             │                          │
                             │          ALL TOPICS       │───► ivm-audit-service
                             │                          │───► ivm-analytics-service
                             └──────────────────────────┘
```

---

## 6. Dead Letter Queue Contracts

### 6.1 Overview

When a consumer fails to process an event after exhausting its retry policy, the event is published to a Dead Letter Queue (DLQ) topic. DLQ events wrap the original event with failure metadata for investigation and replay.

### 6.2 DLQ Topic Naming

```
{original_topic}.dlq
```

Examples:
- `ivm.order.events.dlq`
- `ivm.payment.events.dlq`
- `ivm.device.events.dlq`

### 6.3 DLQ Topic Configuration

```yaml
retention.ms: 2592000000     # 30 days
cleanup.policy: delete
min.insync.replicas: 2
compression.type: lz4
```

### 6.4 DLQ Event Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/cloudevents/dlq-envelope.v1.json",
  "title": "IVM Dead Letter Queue Event",
  "description": "Envelope for events that failed processing and were sent to the Dead Letter Queue.",
  "type": "object",
  "required": [
    "specversion",
    "id",
    "source",
    "type",
    "time",
    "datacontenttype",
    "data"
  ],
  "properties": {
    "specversion": { "type": "string", "const": "1.0" },
    "id": {
      "type": "string",
      "pattern": "^dlq_[0-9A-HJKMNP-TV-Z]{26}$",
      "description": "Unique DLQ event identifier."
    },
    "source": {
      "type": "string",
      "description": "The consumer service that failed to process the event."
    },
    "type": {
      "type": "string",
      "const": "ivm.dlq.event_failed.v1",
      "description": "DLQ event type."
    },
    "time": { "type": "string", "format": "date-time" },
    "datacontenttype": { "type": "string", "const": "application/json" },
    "data": {
      "type": "object",
      "required": [
        "originalEvent",
        "failingConsumer",
        "failureReason",
        "failureDetail",
        "attemptCount",
        "firstAttemptAt",
        "lastAttemptAt",
        "originalTopic",
        "originalPartition",
        "originalOffset"
      ],
      "properties": {
        "originalEvent": {
          "type": "object",
          "description": "The complete original CloudEvents event that failed processing."
        },
        "failingConsumer": {
          "type": "string",
          "description": "Consumer group ID that failed to process the event.",
          "examples": ["ivm-order-service.ivm.payment.events"]
        },
        "failureReason": {
          "type": "string",
          "enum": ["DESERIALIZATION_ERROR", "VALIDATION_ERROR", "PROCESSING_ERROR", "DEPENDENCY_UNAVAILABLE", "TIMEOUT", "UNKNOWN"],
          "description": "Categorized failure reason."
        },
        "failureDetail": {
          "type": "string",
          "description": "Detailed error message or stack trace summary."
        },
        "attemptCount": {
          "type": "integer",
          "minimum": 1,
          "description": "Total number of processing attempts before DLQ."
        },
        "firstAttemptAt": {
          "type": "string",
          "format": "date-time",
          "description": "Timestamp of the first processing attempt."
        },
        "lastAttemptAt": {
          "type": "string",
          "format": "date-time",
          "description": "Timestamp of the last processing attempt."
        },
        "originalTopic": {
          "type": "string",
          "description": "Kafka topic the original event came from."
        },
        "originalPartition": {
          "type": "integer",
          "description": "Kafka partition number."
        },
        "originalOffset": {
          "type": "integer",
          "description": "Kafka offset of the original message."
        },
        "replayable": {
          "type": "boolean",
          "description": "Whether this event can be safely replayed after fixing the issue."
        }
      },
      "additionalProperties": false
    }
  }
}
```

### 6.5 DLQ Example

```json
{
  "specversion": "1.0",
  "id": "dlq_01J5MBABCD8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/services/ivm-order-service",
  "type": "ivm.dlq.event_failed.v1",
  "time": "2026-03-12T14:05:00.000Z",
  "datacontenttype": "application/json",
  "data": {
    "originalEvent": {
      "specversion": "1.0",
      "id": "evt_01J5M6EFGH8N3P4Q5R6S7T8U9V",
      "source": "ivm://production/services/ivm-payment-service",
      "type": "ivm.payment.authorized.v1",
      "time": "2026-03-12T14:01:12.000Z",
      "datacontenttype": "application/json",
      "subject": "payment/pay_01J5M5GHIJ8N3P4Q5R6S7T8U9V",
      "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
      "traceid": "00-abcdef1234567890abcdef123456789c-1234567890abcdeb-01",
      "correlationid": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
      "data": {
        "paymentId": "pay_01J5M5GHIJ8N3P4Q5R6S7T8U9V",
        "orderId": "ord_01J5M5ABCD8N3P4Q5R6S7T8U9V",
        "customerId": "cst_01J5M4ABCD8N3P4Q5R6S7T8U9V",
        "amount": { "amount": 161250, "currency": "NGN" },
        "method": "CARD",
        "gateway": "paystack",
        "gatewayRef": "PSK_txn_abc123xyz789",
        "cardLast4": "4242",
        "cardBrand": "VISA",
        "authorizedAt": "2026-03-12T14:01:12.000Z"
      }
    },
    "failingConsumer": "ivm-order-service.ivm.payment.events",
    "failureReason": "PROCESSING_ERROR",
    "failureDetail": "Order ord_01J5M5ABCD not found in database. Possible replication lag.",
    "attemptCount": 5,
    "firstAttemptAt": "2026-03-12T14:01:13.000Z",
    "lastAttemptAt": "2026-03-12T14:04:55.000Z",
    "originalTopic": "ivm.payment.events",
    "originalPartition": 7,
    "originalOffset": 142857,
    "replayable": true
  }
}
```

### 6.6 DLQ Processing Workflow

```
1. Event fails processing after N retries (exponential backoff)
2. Consumer publishes to {topic}.dlq with failure metadata
3. Consumer commits the original offset and moves on
4. DLQ monitor alerts on-call team (PagerDuty/OpsGenie)
5. Operator investigates via DLQ query API
6. After fix, operator replays event from DLQ back to original topic
7. DLQ event is marked as resolved
```

### 6.7 Retry Policy (Default)

| Parameter | Value |
|---|---|
| Max Retries | 5 |
| Initial Backoff | 1 second |
| Backoff Multiplier | 2x (exponential) |
| Max Backoff | 30 seconds |
| Jitter | +/- 20% |
| DLQ After | 5 failed attempts |

---

## 7. MQTT Event Bridge

### 7.1 Overview

Devices communicate with the cloud via MQTT 5.0 (TLS + mTLS). The MQTT-Kafka Bridge service subscribes to device MQTT topics and translates device messages into CloudEvents published to the appropriate Kafka topics.

### 7.2 MQTT Topic Structure

```
ivm/{tenant_id}/{site_id}/{device_id}/telemetry/health
ivm/{tenant_id}/{site_id}/{device_id}/telemetry/metrics
ivm/{tenant_id}/{site_id}/{device_id}/events/{event_type}
ivm/{tenant_id}/{site_id}/{device_id}/commands/{command_id}
ivm/{tenant_id}/{site_id}/{device_id}/commands/{command_id}/ack
ivm/fleet/{tenant_id}/alerts
ivm/fleet/{tenant_id}/config-updates
```

### 7.3 MQTT-to-Kafka Translation

The bridge performs the following translation for each MQTT message:

| MQTT Topic Pattern | Kafka Topic | CloudEvents `type` | CloudEvents `source` |
|---|---|---|---|
| `ivm/{t}/{s}/{d}/telemetry/health` | `ivm.device.events` | `ivm.device.health_reported.v1` | `ivm://production/devices/{device_id}` |
| `ivm/{t}/{s}/{d}/events/online` | `ivm.device.events` | `ivm.device.online.v1` | `ivm://production/devices/{device_id}` |
| `ivm/{t}/{s}/{d}/events/offline` | `ivm.device.events` | `ivm.device.offline.v1` | `ivm://production/devices/{device_id}` |
| `ivm/{t}/{s}/{d}/events/fault` | `ivm.device.events` | `ivm.device.fault_detected.v1` | `ivm://production/devices/{device_id}` |
| `ivm/{t}/{s}/{d}/commands/{c}/ack` | `ivm.device.events` | `ivm.device.command_executed.v1` | `ivm://production/devices/{device_id}` |
| `ivm/{t}/{s}/{d}/events/update_started` | `ivm.device.events` | `ivm.device.update_started.v1` | `ivm://production/devices/{device_id}` |
| `ivm/{t}/{s}/{d}/events/update_completed` | `ivm.device.events` | `ivm.device.update_completed.v1` | `ivm://production/devices/{device_id}` |
| `ivm/{t}/{s}/{d}/events/update_failed` | `ivm.device.events` | `ivm.device.update_failed.v1` | `ivm://production/devices/{device_id}` |
| `ivm/{t}/{s}/{d}/events/door_opened` | `ivm.locker.events` | `ivm.locker.door_opened.v1` | `ivm://production/devices/{device_id}` |
| `ivm/{t}/{s}/{d}/events/door_closed` | `ivm.locker.events` | `ivm.locker.door_closed.v1` | `ivm://production/devices/{device_id}` |

### 7.4 MQTT Payload Format (Device Side)

Devices publish lightweight JSON payloads. The bridge enriches these into full CloudEvents:

**MQTT Health Report (from device):**

```json
{
  "ts": 1710252000000,
  "cpu": 45.2,
  "mem": 62.1,
  "disk": 33.0,
  "temp": 38.5,
  "uptime": 86400,
  "errors": 0,
  "net_latency": 12
}
```

**Translated CloudEvent (by bridge):**

```json
{
  "specversion": "1.0",
  "id": "evt_01J5MCABCD8N3P4Q5R6S7T8U9V",
  "source": "ivm://production/devices/dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
  "type": "ivm.device.health_reported.v1",
  "time": "2026-03-12T14:00:00.000Z",
  "datacontenttype": "application/json",
  "subject": "device/dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
  "tenantid": "tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V",
  "traceid": "00-bridge-generated-trace-id-00000000-0000000000000001-01",
  "correlationid": "dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
  "data": {
    "deviceId": "dev_01J5M3RSTU8N3P4Q5R6S7T8U9V",
    "siteId": "site_01J5M3QRST8N3P4Q5R6S7T8U9V",
    "metrics": {
      "cpuUsagePercent": 45.2,
      "memoryUsagePercent": 62.1,
      "diskUsagePercent": 33.0,
      "temperatureCelsius": 38.5,
      "uptimeSeconds": 86400,
      "errorCount": 0,
      "networkLatencyMs": 12
    },
    "overallStatus": "HEALTHY",
    "reportedAt": "2026-03-12T14:00:00.000Z"
  }
}
```

### 7.5 Bridge Enrichment Logic

The bridge performs the following enrichments when translating MQTT messages to CloudEvents:

1. **Event ID Generation**: Generate a new `evt_` prefixed ULID for each message.
2. **Source Construction**: Build the `source` URI from the MQTT topic segments: `ivm://production/devices/{device_id}`.
3. **Type Mapping**: Map MQTT topic suffix to CloudEvents `type` (see table above).
4. **Timestamp Normalization**: Convert device timestamp (Unix epoch ms) to ISO 8601 UTC.
5. **Tenant/Site Resolution**: Extract `tenant_id` and `site_id` from the MQTT topic path.
6. **Device ID Extraction**: Extract `device_id` from the MQTT topic path.
7. **Trace ID Generation**: Generate a new trace ID for device-originated events (no incoming trace context from devices).
8. **Correlation ID**: Use `device_id` as the correlation ID for device lifecycle events.
9. **Health Status Computation**: Compute `overallStatus` from device metrics thresholds:
   - HEALTHY: All metrics within normal range
   - DEGRADED: Any metric exceeds warning threshold
   - CRITICAL: Any metric exceeds critical threshold

### 7.6 Bridge Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    MQTT-KAFKA BRIDGE                      │
│                                                          │
│  ┌──────────────┐   ┌───────────────┐   ┌────────────┐  │
│  │  MQTT Client │   │  Transformer  │   │   Kafka    │  │
│  │  (Subscriber)│──►│  & Enricher   │──►│  Producer  │  │
│  │              │   │               │   │            │  │
│  │ QoS 1 (at   │   │ - ID gen      │   │ - Async    │  │
│  │  least once) │   │ - Type map    │   │ - Batched  │  │
│  │              │   │ - Timestamp   │   │ - Idempotent│  │
│  │              │   │ - Enrichment  │   │            │  │
│  └──────────────┘   └───────────────┘   └────────────┘  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │              Device Registry Cache                │    │
│  │  (tenant/site/device metadata for enrichment)    │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │           Health Threshold Config                 │    │
│  │  CPU Warning: 80%  Critical: 95%                 │    │
│  │  Memory Warning: 85%  Critical: 95%              │    │
│  │  Disk Warning: 80%  Critical: 95%                │    │
│  │  Temperature Warning: 45C  Critical: 60C         │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
         │                                    │
         │ MQTT (TLS+mTLS)                   │ Kafka
         ▼                                    ▼
┌─────────────────┐              ┌──────────────────┐
│   MQTT Broker   │              │  Kafka Cluster   │
│  (EMQX/HiveMQ)  │              │                  │
└─────────────────┘              └──────────────────┘
         ▲
         │ MQTT (TLS+mTLS)
         │
┌─────────────────┐
│  Edge Controller │
│  (per site)      │
└─────────────────┘
```

### 7.7 Reverse Path: Cloud-to-Device Commands

Commands flow from cloud services to devices via the reverse path:

1. Service publishes command to Kafka topic `ivm.device.commands` (internal).
2. MQTT-Kafka Bridge consumes the command.
3. Bridge translates to lightweight MQTT payload.
4. Bridge publishes to MQTT topic: `ivm/{tenant_id}/{site_id}/{device_id}/commands/{command_id}`.
5. Edge Controller receives and routes to device driver.
6. Device executes and publishes acknowledgment to: `ivm/{tenant_id}/{site_id}/{device_id}/commands/{command_id}/ack`.
7. Bridge translates ack to `ivm.device.command_executed.v1` CloudEvent on `ivm.device.events`.

### 7.8 Delivery Guarantees

| Path | MQTT QoS | Kafka Guarantee | End-to-End |
|---|---|---|---|
| Device -> Cloud | QoS 1 (at-least-once) | Idempotent producer (exactly-once) | At-least-once (deduplicated by event ID) |
| Cloud -> Device | QoS 1 (at-least-once) | N/A (Kafka -> MQTT) | At-least-once (device ack required) |

### 7.9 Offline Handling

When the edge controller loses cloud connectivity:

1. MQTT messages queue locally on the edge controller (SQLite-backed offline queue).
2. Upon reconnection, queued messages are published in order with original timestamps.
3. The bridge detects replayed messages via monotonic sequence numbers per device.
4. Duplicate detection at the Kafka producer level (idempotent producer) prevents duplicate events.

---

## Appendix A: Shared Schema Definitions

### A.1 Money

Used across all event payloads that include monetary amounts.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.ivm.io/common/money.v1.json",
  "title": "Money",
  "description": "Monetary amount in minor units. For NGN, 1 Naira = 100 Kobo, so 1500.00 NGN = 150000.",
  "type": "object",
  "required": ["amount", "currency"],
  "properties": {
    "amount": {
      "type": "integer",
      "minimum": 0,
      "description": "Amount in minor units (kobo for NGN, cents for USD)."
    },
    "currency": {
      "type": "string",
      "pattern": "^[A-Z]{3}$",
      "description": "ISO 4217 currency code. Primary: NGN.",
      "examples": ["NGN", "USD"]
    }
  },
  "additionalProperties": false
}
```

### A.2 Prefixed ULID Patterns

| Entity | Prefix | Pattern | Example |
|---|---|---|---|
| Event | `evt_` | `^evt_[0-9A-HJKMNP-TV-Z]{26}$` | `evt_01J5M2XKQR8N3P4Q5R6S7T8U9V` |
| User | `usr_` | `^usr_[0-9A-HJKMNP-TV-Z]{26}$` | `usr_01J5M2XKQR8N3P4Q5R6S7T8U9V` |
| Customer | `cst_` | `^cst_[0-9A-HJKMNP-TV-Z]{26}$` | `cst_01J5M4ABCD8N3P4Q5R6S7T8U9V` |
| Tenant | `tnt_` | `^tnt_[0-9A-HJKMNP-TV-Z]{26}$` | `tnt_01HX7V8K9M2N3P4Q5R6S7T8U9V` |
| Order | `ord_` | `^ord_[0-9A-HJKMNP-TV-Z]{26}$` | `ord_01J5M5ABCD8N3P4Q5R6S7T8U9V` |
| Payment | `pay_` | `^pay_[0-9A-HJKMNP-TV-Z]{26}$` | `pay_01J5M5GHIJ8N3P4Q5R6S7T8U9V` |
| Refund | `rfd_` | `^rfd_[0-9A-HJKMNP-TV-Z]{26}$` | `rfd_01J5M6ABCD8N3P4Q5R6S7T8U9V` |
| Settlement | `stl_` | `^stl_[0-9A-HJKMNP-TV-Z]{26}$` | `stl_01J5M6BCDE8N3P4Q5R6S7T8U9V` |
| Device | `dev_` | `^dev_[0-9A-HJKMNP-TV-Z]{26}$` | `dev_01J5M3RSTU8N3P4Q5R6S7T8U9V` |
| Site | `site_` | `^site_[0-9A-HJKMNP-TV-Z]{26}$` | `site_01J5M3QRST8N3P4Q5R6S7T8U9V` |
| Locker Bank | `lbk_` | `^lbk_[0-9A-HJKMNP-TV-Z]{26}$` | `lbk_01J5M7BCDE8N3P4Q5R6S7T8U9V` |
| Compartment | `cmp_` | `^cmp_[0-9A-HJKMNP-TV-Z]{26}$` | `cmp_01J5M7CDEF8N3P4Q5R6S7T8U9V` |
| Reservation | `rsv_` | `^rsv_[0-9A-HJKMNP-TV-Z]{26}$` | `rsv_01J5M7ABCD8N3P4Q5R6S7T8U9V` |
| Session | `ssn_` | `^ssn_[0-9A-HJKMNP-TV-Z]{26}$` | `ssn_01J5M8ABCD8N3P4Q5R6S7T8U9V` |
| Workflow Inst. | `wf_` | `^wf_[0-9A-HJKMNP-TV-Z]{26}$` | `wf_01J5M9ABCD8N3P4Q5R6S7T8U9V` |
| Workflow Def. | `wfd_` | `^wfd_[0-9A-HJKMNP-TV-Z]{26}$` | `wfd_01J5M9BCDE8N3P4Q5R6S7T8U9V` |
| Notification | `ntf_` | `^ntf_[0-9A-HJKMNP-TV-Z]{26}$` | `ntf_01J5MAABCD8N3P4Q5R6S7T8U9V` |
| Evidence | `evi_` | `^evi_[0-9A-HJKMNP-TV-Z]{26}$` | `evi_01J5M4MNOP8N3P4Q5R6S7T8U9V` |
| Dispute | `dsp_` | `^dsp_[0-9A-HJKMNP-TV-Z]{26}$` | `dsp_01J5M5WXYZ8N3P4Q5R6S7T8U9V` |
| Command | `cmd_` | `^cmd_[0-9A-HJKMNP-TV-Z]{26}$` | `cmd_01J5M6CDEF8N3P4Q5R6S7T8U9V` |
| Update | `upd_` | `^upd_[0-9A-HJKMNP-TV-Z]{26}$` | `upd_01J5M6DEFG8N3P4Q5R6S7T8U9V` |
| Shrinkage | `shr_` | `^shr_[0-9A-HJKMNP-TV-Z]{26}$` | `shr_01J5M8EFGH8N3P4Q5R6S7T8U9V` |
| DLQ | `dlq_` | `^dlq_[0-9A-HJKMNP-TV-Z]{26}$` | `dlq_01J5MBABCD8N3P4Q5R6S7T8U9V` |
