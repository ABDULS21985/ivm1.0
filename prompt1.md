Architecture & Design — Define Bounded Contexts and Service Boundaries
Objective

Design the core architecture and domain decomposition of a Unified Automated Commerce & Service Platform (UACSP) that supports the following commerce channels within a single unified platform:

Service Vending Machines

SIM card vending

Ticket vending

Insurance purchase kiosks

Banking service kiosks

Digital document printing

Smart Locker Systems

E-commerce parcel pickup

Pharmacy pickup lockers

Secure parcel storage

Logistics fulfillment lockers

AI-Powered Autonomous Retail Stores

Entry via QR code / NFC / identity

Computer vision product detection

Automatic checkout and payment

The system must support multi-tenant deployment, event-driven workflows, edge devices, and hybrid cloud + edge computing.

The platform will be implemented using:

Backend: ASP.NET Core (.NET 8)

Frontend: Razor Pages / MVC

Database: PostgreSQL

Messaging: MQTT (device communication) + Kafka/Event bus (platform events)

Deployment: Containerized microservices (Docker/Kubernetes)

System Goals

Design an architecture that:

Enables modular commerce services

Supports multiple vending channels in a single platform

Enables event-driven workflows

Supports offline edge device operations

Ensures strong security and compliance

Enables scalable device fleet management

The architecture must support deployments for:

Banks

Telecom providers

Retail chains

Logistics companies

Government service kiosks

Expected Architectural Principles

The architecture must adhere to the following principles.

1. Domain Driven Design (DDD)

Define bounded contexts with:

Clear domain boundaries

Independent data ownership

Explicit APIs

Event-driven interactions

No shared database across domains.

2. API-First Architecture

All services must expose:

REST APIs (ASP.NET Core)

Event contracts

OpenAPI documentation

Internal service communication must occur through:

APIs

Event streams

3. Event-Driven Architecture

Use events for asynchronous workflows.

Example events:

DeviceRegistered
LockerDoorOpened
ServicePurchaseRequested
PaymentAuthorized
OrderReadyForPickup
StoreEntryGranted
BasketClosed
ReceiptGenerated

Event formats must follow CloudEvents specification.

4. Multi-Tenant Platform

The system must support:

Multiple organizations

Device fleets per tenant

Independent configuration

Tenant isolation must include:

Data

devices

payment integrations

configuration

5. Edge + Cloud Hybrid Architecture

Some services must run at edge sites.

Edge node responsibilities:

Device communication

Local command queue

Offline transaction buffering

Sensor / CV inference

Cloud responsibilities:

orchestration

analytics

identity

payment settlement

monitoring

Domain Areas To Design

The AI agent must identify and define bounded contexts across the platform.

Minimum expected domains include:

1. Identity & Access Domain

Responsibilities:

User identity management

Tenant identity

Authentication

Role based access control

Device identity

Technologies:

ASP.NET Identity

OAuth2 / OpenID Connect

Core entities:

User
Role
Tenant
Permission
DeviceIdentity
2. Device Fleet Management Domain

Responsible for managing:

kiosks

lockers

autonomous store sensors

cameras

IoT devices

Capabilities:

device registration

firmware management

health monitoring

remote command execution

Events:

DeviceRegistered
DeviceHeartbeatReceived
DeviceOfflineDetected
FirmwareUpdateAvailable

Protocols:

MQTT

secure device provisioning

3. Commerce Orchestration Domain

Responsible for coordinating transactions.

Examples:

service purchase
locker pickup
retail checkout

Capabilities:

workflow orchestration

order lifecycle

transaction state

Entities:

Order
Transaction
Basket
FulfillmentTask
4. Payment Processing Domain

Responsible for:

payment authorization

PSP integration

tokenized transactions

wallet support

Integrations may include:

Remita

Flutterwave

Paystack

card terminals

PCI DSS design must minimize card exposure.

Events:

PaymentRequested
PaymentAuthorized
PaymentFailed
RefundIssued
5. Locker Management Domain

Responsible for:

locker allocation

door control

parcel tracking

pickup authentication

Entities:

Locker
LockerBank
Parcel
PickupCode

Events:

LockerReserved
LockerDoorOpened
ParcelCollected
ParcelExpired
6. Service Catalog Domain

Responsible for vending service definitions.

Examples:

SIM registration

insurance purchase

document printing

banking services

Entities:

Service
ServiceProvider
ServiceWorkflow
ServicePricing
7. Autonomous Store Domain

Responsible for:

store entry

sensor fusion

basket creation

automated checkout

Entities:

StoreSession
CustomerBasket
ProductInteraction
SensorEvent

Events:

CustomerEnteredStore
ProductPicked
ProductReturned
BasketClosed
ReceiptGenerated
8. Inventory & Supply Domain

Responsible for:

kiosk product inventory

store product inventory

locker capacity

Entities:

Product
InventoryLocation
StockLevel
ReplenishmentOrder
9. Notification Domain

Responsible for sending:

SMS

email

push notifications

locker pickup codes

Events:

NotificationRequested
SMSDelivered
EmailDelivered
10. Compliance & KYC Domain

Responsible for:

KYC verification

identity verification

SIM registration compliance

AML monitoring

Integrations:

NIMC

NIBSS

BVN

NDPC compliance

Data Architecture Requirements

Database technology:

PostgreSQL

Each bounded context must own its own schema.

Example:

identity_db
device_db
commerce_db
payment_db
locker_db
inventory_db

No cross-domain table joins.

Messaging Architecture

Device messaging:

MQTT broker

Example topics:

device/{deviceId}/telemetry
device/{deviceId}/commands
device/{deviceId}/events

Platform events:

Kafka / event streaming.

Frontend Architecture

Admin UI must be implemented using:

ASP.NET Core Razor Pages

MVC pattern

Capabilities:

device fleet dashboard
tenant management
locker monitoring
commerce analytics
store monitoring
Security Requirements

Architecture must enforce:

zero trust device authentication

encrypted device communication

secure OTA updates

RBAC

audit trails

IoT security must align with:

NISTIR 8259A

ETSI EN 303 645

Deliverables Required From The AI Agent

The AI agent must produce the following artifacts.

1. Domain Context Map

Visual diagram of all bounded contexts.

2. Service Responsibility Matrix

Table showing each service and responsibilities.

3. API Contract Definitions

OpenAPI specs for each service.

4. Event Contract Definitions

CloudEvents formatted event schema.

5. Database Ownership Map

Mapping of services → PostgreSQL schemas.

6. Interaction Diagrams

Sequence diagrams for workflows:

Examples:

kiosk purchase

locker pickup

autonomous store checkout

7. Deployment Architecture

Diagram showing:

Cloud services
Edge nodes
Devices
Messaging layers
Acceptance Criteria

Architecture is considered complete when:

All services have clearly defined bounded contexts

No shared database exists across contexts

All cross-service communication uses APIs or events

Device communication uses MQTT

Platform events use event streaming

Multi-tenant isolation is enforced

Edge and cloud responsibilities are clearly separated

Priority

P1 — Foundational

This architecture defines the core platform foundation and must be completed before implementation of other domains.