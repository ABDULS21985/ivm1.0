# 04 - Edge & Device Architecture

## 1. Edge Controller Design

The Edge Controller is the on-premises compute unit at every IVM site. It acts as the local brain — driving devices, caching state, handling offline scenarios, and running AI inference for autonomous stores.

### 1.1 Edge Controller Hardware Profiles

| Site Type | Recommended Hardware | CPU | RAM | Storage | GPU/NPU | Connectivity |
|---|---|---|---|---|---|---|
| **Service Kiosk** | Raspberry Pi CM4 / Intel NUC | ARM64 / x86-64 | 4-8 GB | 64 GB eMMC/SSD | None | Ethernet + 4G |
| **Smart Locker** | Raspberry Pi CM4 / Industrial SBC | ARM64 | 4 GB | 32 GB eMMC | None | Ethernet + 4G |
| **Autonomous Store** | NVIDIA Jetson Orin / x86+GPU | ARM64 / x86-64 | 16-32 GB | 256 GB NVMe | GPU (Jetson) / dGPU | Ethernet + 5G |
| **Hybrid Site** | NVIDIA Jetson Orin NX | ARM64 | 16 GB | 128 GB NVMe | GPU | Ethernet + 4G/5G |

### 1.2 Edge Software Stack

```
┌─────────────────────────────────────────────────────────┐
│                  EDGE CONTROLLER OS                      │
│            (Ubuntu Core 24 / Balena OS)                  │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              Container Runtime (Docker)              │ │
│  ├─────────────────────────────────────────────────────┤ │
│  │                                                     │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │ │
│  │  │ Edge Gateway │  │  Device     │  │  State     │  │ │
│  │  │ (REST API)  │  │  Driver Mgr │  │  Cache     │  │ │
│  │  │  :8080      │  │             │  │ (SQLite +  │  │ │
│  │  │             │  │             │  │  in-memory)│  │ │
│  │  └─────────────┘  └──────┬──────┘  └────────────┘  │ │
│  │                          │                          │ │
│  │  ┌─────────────┐  ┌─────┴───────┐  ┌────────────┐  │ │
│  │  │  Workflow   │  │  Telemetry  │  │  Offline   │  │ │
│  │  │  Runtime    │  │  Collector  │  │  Queue     │  │ │
│  │  │  (local)    │  │  → MQTT     │  │  (SQLite)  │  │ │
│  │  └─────────────┘  └─────────────┘  └────────────┘  │ │
│  │                                                     │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │ │
│  │  │  Update     │  │  VPN Client │  │  Log       │  │ │
│  │  │  Agent      │  │ (WireGuard) │  │  Forwarder │  │ │
│  │  │  (TUF)      │  │             │  │  (Fluent)  │  │ │
│  │  └─────────────┘  └─────────────┘  └────────────┘  │ │
│  │                                                     │ │
│  │  ┌──────────────────────────────────────────────┐   │ │
│  │  │  AI Inference Runtime (Stores only)          │   │ │
│  │  │  ┌──────────┐ ┌───────────┐ ┌────────────┐  │   │ │
│  │  │  │  Video   │ │  Model    │ │  Event     │  │   │ │
│  │  │  │ Pipeline │ │  Server   │ │  Emitter   │  │   │ │
│  │  │  │ (GStreamer│ │(TensorRT/ │ │            │  │   │ │
│  │  │  │ /DeepStr)│ │ ONNX)     │ │            │  │   │ │
│  │  │  └──────────┘ └───────────┘ └────────────┘  │   │ │
│  │  └──────────────────────────────────────────────┘   │ │
│  │                                                     │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              OS-Level Services                       │ │
│  │  ┌──────────┐ ┌──────────┐ ┌────────────────────┐  │ │
│  │  │  TPM 2.0 │ │ Network  │ │ Watchdog / Health  │  │ │
│  │  │  (secure  │ │ Manager  │ │ Monitor            │  │ │
│  │  │   boot)   │ │ (failovr)│ │ (auto-recovery)    │  │ │
│  │  └──────────┘ └──────────┘ └────────────────────┘  │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 1.3 Edge Container Composition

```yaml
# docker-compose.edge.yml (simplified)
services:
  edge-gateway:
    image: ivm/edge-gateway:${VERSION}
    ports: ["8080:8080"]
    environment:
      SITE_ID: ${SITE_ID}
      TENANT_ID: ${TENANT_ID}
      CLOUD_API_URL: ${CLOUD_API_URL}
    volumes:
      - edge-state:/data/state
    restart: always

  device-driver-manager:
    image: ivm/device-driver-manager:${VERSION}
    privileged: true  # hardware access
    devices:
      - /dev/ttyUSB0  # serial devices
      - /dev/video0   # cameras
    volumes:
      - /dev:/dev:ro
      - driver-config:/config
    restart: always

  telemetry-collector:
    image: ivm/telemetry-collector:${VERSION}
    environment:
      MQTT_BROKER: ${CLOUD_MQTT_BROKER}
      MQTT_PORT: 8883
      MQTT_CLIENT_ID: edge_${SITE_ID}
    restart: always

  offline-queue:
    image: ivm/offline-queue:${VERSION}
    volumes:
      - offline-data:/data/queue
    restart: always

  update-agent:
    image: ivm/update-agent:${VERSION}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - update-keys:/keys:ro
    restart: always

  vpn-client:
    image: ivm/vpn-client:${VERSION}
    cap_add: [NET_ADMIN]
    sysctls:
      net.ipv4.conf.all.src_valid_mark: 1
    restart: always

  # Only for autonomous store sites:
  ai-inference:
    image: ivm/ai-inference:${VERSION}
    runtime: nvidia  # GPU access
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: always

volumes:
  edge-state:
  driver-config:
  offline-data:
  update-keys:
```

---

## 2. Device Abstraction Layer (DAL)

### 2.1 Architecture

The DAL provides a uniform interface for all hardware interactions, regardless of manufacturer, protocol, or connection type.

```
┌────────────────────────────────────────────────────────────┐
│                  Application Layer                          │
│          (Edge Gateway, Workflow Runtime)                   │
│                                                            │
│    Calls:  dal.scan()  dal.print()  dal.unlockDoor()       │
└────────────────────────┬───────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────┐
│               Device Abstraction Layer                      │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Capability Interface Registry            │  │
│  │                                                      │  │
│  │  IScanner       → scan(): ScanResult                 │  │
│  │  IPrinter       → print(doc): PrintResult            │  │
│  │  ICardReader    → readCard(): CardData               │  │
│  │  IBiometric     → capture(): BiometricTemplate       │  │
│  │  ILockController→ unlock(id): LockResult             │  │
│  │  ILockController→ lock(id): LockResult               │  │
│  │  IDispenser     → dispense(slot): DispenseResult     │  │
│  │  IGateController→ open(): GateResult                 │  │
│  │  ICamera        → startStream(): VideoStream         │  │
│  │  IWeightSensor  → getWeight(): WeightReading         │  │
│  │  ICashHandler   → acceptCash(): CashResult           │  │
│  │  IDisplay       → show(content): DisplayResult       │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                  │
│  ┌──────────────────────┼──────────────────────────────┐  │
│  │         Driver Manager (Registry + Lifecycle)       │  │
│  │                      │                              │  │
│  │  Discovers → Loads → Initializes → Monitors → Restarts │
│  └──────────────────────┼──────────────────────────────┘  │
│                         │                                  │
│  ┌──────────┬──────────┬┴─────────┬──────────┬─────────┐  │
│  │ Driver:  │ Driver:  │ Driver:  │ Driver:  │ Driver: │  │
│  │ Honeywell│ Zebra    │ PAX      │ Suprema  │ Custom  │  │
│  │ Scanner  │ Printer  │ Terminal │ Biometric│ Lock    │  │
│  │ (USB/HID)│ (USB/ESC)│ (Serial) │ (USB/SDK)│ (GPIO) │  │
│  └──────────┴──────────┴──────────┴──────────┴─────────┘  │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Transport Layer                          │  │
│  │  USB HID │ Serial/RS232 │ TCP/IP │ GPIO │ Bluetooth  │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### 2.2 Driver Interface Definitions

```typescript
// Core driver interface - all drivers must implement
interface IDeviceDriver {
  // Identity
  readonly driverId: string;
  readonly deviceType: DeviceType;
  readonly manufacturer: string;
  readonly model: string;
  readonly version: string;

  // Lifecycle
  initialize(config: DriverConfig): Promise<void>;
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  reset(): Promise<void>;
  healthCheck(): Promise<HealthStatus>;

  // Events
  on(event: 'connected' | 'disconnected' | 'error' | 'data', handler: EventHandler): void;
}

// Scanner capability
interface IScanner extends IDeviceDriver {
  scan(options?: ScanOptions): Promise<ScanResult>;
  startContinuousScan(callback: ScanCallback): Promise<void>;
  stopContinuousScan(): Promise<void>;
}

interface ScanResult {
  success: boolean;
  format: 'QR' | 'BARCODE_1D' | 'BARCODE_2D' | 'NFC';
  data: string;
  rawData?: Buffer;
  timestamp: Date;
}

// Printer capability
interface IPrinter extends IDeviceDriver {
  print(document: PrintDocument): Promise<PrintResult>;
  getStatus(): Promise<PrinterStatus>;
  cut(): Promise<void>;
  openCashDrawer?(): Promise<void>;
}

interface PrintDocument {
  type: 'RECEIPT' | 'DOCUMENT' | 'LABEL' | 'TICKET';
  content: PrintContent[];  // Lines, images, barcodes, QR codes
  copies?: number;
}

// Lock controller capability
interface ILockController extends IDeviceDriver {
  unlock(compartmentId: string, timeoutMs?: number): Promise<LockResult>;
  lock(compartmentId: string): Promise<LockResult>;
  getStatus(compartmentId: string): Promise<LockStatus>;
  getAllStatuses(): Promise<Map<string, LockStatus>>;
  onDoorEvent(callback: DoorEventCallback): void;
}

interface LockResult {
  success: boolean;
  compartmentId: string;
  state: 'LOCKED' | 'UNLOCKED' | 'OPEN' | 'ERROR';
  timestamp: Date;
}

interface DoorEvent {
  compartmentId: string;
  event: 'OPENED' | 'CLOSED' | 'FORCED' | 'TIMEOUT';
  timestamp: Date;
}

// Card reader capability
interface ICardReader extends IDeviceDriver {
  readCard(options?: CardReadOptions): Promise<CardReadResult>;
  readNFC(): Promise<NFCReadResult>;
  cancelRead(): Promise<void>;
}

// Biometric scanner capability
interface IBiometricScanner extends IDeviceDriver {
  captureFingerprint(): Promise<BiometricResult>;
  capturePhoto(): Promise<BiometricResult>;
  verifyFingerprint(template: Buffer): Promise<VerificationResult>;
}

// Camera capability (for autonomous stores)
interface ICamera extends IDeviceDriver {
  startStream(config: StreamConfig): Promise<VideoStream>;
  stopStream(): Promise<void>;
  captureFrame(): Promise<Frame>;
  getStreamInfo(): Promise<StreamInfo>;
}

// Weight sensor capability
interface IWeightSensor extends IDeviceDriver {
  getWeight(): Promise<WeightReading>;
  tare(): Promise<void>;
  onWeightChange(threshold: number, callback: WeightChangeCallback): void;
}
```

### 2.3 Driver Discovery & Configuration

```yaml
# /config/drivers/site-config.yml
site:
  id: "site_01HX..."
  type: "KIOSK_STATION"

drivers:
  - id: "scanner-main"
    type: "SCANNER"
    driver: "honeywell-voyager-1400g"
    connection:
      type: "USB_HID"
      vendorId: "0x0c2e"
      productId: "0x0b81"
    config:
      scanMode: "TRIGGER"
      formats: ["QR", "BARCODE_1D", "BARCODE_2D"]

  - id: "printer-receipt"
    type: "PRINTER"
    driver: "epson-tm-t88vi"
    connection:
      type: "USB"
      vendorId: "0x04b8"
      productId: "0x0202"
    config:
      paperWidth: 80
      autoCut: true

  - id: "card-reader"
    type: "CARD_READER"
    driver: "pax-a920-reader"
    connection:
      type: "SERIAL"
      port: "/dev/ttyUSB0"
      baudRate: 115200
    config:
      supportedCards: ["VISA", "MASTERCARD", "VERVE"]
      contactless: true

  - id: "biometric"
    type: "BIOMETRIC_SCANNER"
    driver: "suprema-sfm-6000"
    connection:
      type: "USB"
    config:
      captureMode: "FINGERPRINT"
      quality: "HIGH"
```

---

## 3. Offline Strategy

### 3.1 Offline Capability Matrix

| Operation | Offline Support | Strategy |
|---|---|---|
| **Kiosk: Display catalog** | Full | Cached catalog at edge |
| **Kiosk: KYC verification** | Degraded | Queue for verification, limited to cached results |
| **Kiosk: Payment (card)** | Full (store-and-forward) | Terminal processes offline, syncs later |
| **Kiosk: SIM dispensing** | Partial | Can dispense, activation queued |
| **Locker: Pickup (cached code)** | Full | Access codes cached at edge |
| **Locker: Pickup (new OTP)** | Degraded | Cannot generate new OTPs, use cached PIN |
| **Locker: Deposit** | Full | Local compartment management |
| **Store: Entry** | Degraded | Allow cached/pre-authorized customers only |
| **Store: Shopping + exit** | Full | All inference is local |
| **Store: Charging** | Queued | Charge deferred until connectivity restored |
| **All: Telemetry** | Buffered | Queue locally, sync on reconnect |
| **All: Audit events** | Buffered | Queue locally, sync on reconnect (hash chain maintained) |

### 3.2 State Synchronization

```
┌──────────────────────────────────────────────────┐
│               SYNC ENGINE                         │
│                                                   │
│  Cloud State ←──── Bidirectional ────→ Edge State │
│                                                   │
│  Strategies:                                      │
│  ┌────────────────────────────────────────────┐   │
│  │ 1. CLOUD-TO-EDGE (config, catalog, access) │   │
│  │    Cloud is source of truth                 │   │
│  │    Edge pulls on schedule + on reconnect    │   │
│  │    Conflict: Cloud always wins              │   │
│  └────────────────────────────────────────────┘   │
│  ┌────────────────────────────────────────────┐   │
│  │ 2. EDGE-TO-CLOUD (events, telemetry)       │   │
│  │    Edge is source of truth for local events │   │
│  │    Append-only, no conflicts possible       │   │
│  │    Queue and forward on reconnect           │   │
│  └────────────────────────────────────────────┘   │
│  ┌────────────────────────────────────────────┐   │
│  │ 3. BIDIRECTIONAL (orders, sessions)        │   │
│  │    Edge creates locally, syncs to cloud     │   │
│  │    Cloud may update (payment status)        │   │
│  │    Conflict resolution: version vectors     │   │
│  │    + last-writer-wins for non-critical      │   │
│  │    + manual resolution queue for conflicts  │   │
│  └────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

### 3.3 Offline Queue Implementation

```
┌──────────────────────────────────────────────────────┐
│            OFFLINE QUEUE (SQLite-backed)              │
│                                                       │
│  Table: outbound_queue                                │
│  ┌────────┬──────────┬────────┬──────────┬─────────┐ │
│  │ id     │ type     │ payload│ priority │ status   │ │
│  ├────────┼──────────┼────────┼──────────┼─────────┤ │
│  │ 1      │ event    │ {...}  │ NORMAL   │ PENDING  │ │
│  │ 2      │ payment  │ {...}  │ HIGH     │ SENDING  │ │
│  │ 3      │ telemetry│ {...}  │ LOW      │ PENDING  │ │
│  │ 4      │ audit    │ {...}  │ HIGH     │ SENT     │ │
│  └────────┴──────────┴────────┴──────────┴─────────┘ │
│                                                       │
│  Behaviors:                                           │
│  - Persisted to SQLite (survives power loss)          │
│  - Priority ordering: HIGH → NORMAL → LOW             │
│  - Automatic retry with exponential backoff           │
│  - Deduplication via idempotency keys                 │
│  - Max queue size with LRU eviction (LOW only)        │
│  - Automatic drain on network reconnection            │
│  - Batch sending for efficiency                       │
└──────────────────────────────────────────────────────┘
```

---

## 4. AI Inference Pipeline (Autonomous Stores)

### 4.1 Pipeline Architecture

```
Camera Array (4-16 cameras per store)
      │
      ▼
┌──────────────────────────────────────────────┐
│           VIDEO INGESTION LAYER              │
│                                              │
│  Camera 1 ──→ ┐                              │
│  Camera 2 ──→ ├──→ Frame Synchronizer ──→    │
│  Camera 3 ──→ ┤    (timestamp alignment)     │
│  Camera N ──→ ┘                              │
└──────────────────────┬───────────────────────┘
                       │ Synchronized frames
                       ▼
┌──────────────────────────────────────────────┐
│          PREPROCESSING LAYER                  │
│                                              │
│  Decode → Resize → Normalize → Batch         │
│  (Hardware-accelerated: NVDEC / VA-API)      │
└──────────────────────┬───────────────────────┘
                       │ Preprocessed tensors
                       ▼
┌──────────────────────────────────────────────┐
│           INFERENCE LAYER                     │
│                                              │
│  ┌────────────────┐  ┌───────────────────┐   │
│  │ Person Detector │  │ Item Detector     │   │
│  │ + Tracker       │  │ (product recog.)  │   │
│  │ (YOLOv8/RT-DETR)│  │ (custom model)    │   │
│  └────────┬───────┘  └────────┬──────────┘   │
│           │                    │              │
│           ▼                    ▼              │
│  ┌────────────────────────────────────────┐   │
│  │       Action Recognition               │   │
│  │  (pick / put-back / shelf interaction) │   │
│  └────────────────────┬───────────────────┘   │
│                       │                       │
│  ┌────────────────────┼───────────────────┐   │
│  │    Shelf Sensor Fusion                 │   │
│  │  (weight Δ correlation with vision)    │   │
│  └────────────────────┬───────────────────┘   │
└──────────────────────┬───────────────────────┘
                       │ Item events with confidence
                       ▼
┌──────────────────────────────────────────────┐
│        EVENT PROCESSING LAYER                 │
│                                              │
│  Item Event → Session Associator → Basket    │
│               (person-item binding)  Update   │
│                                              │
│  Confidence < threshold? → Flag for review   │
│  Shelf sensor disagrees? → Flag for review   │
│                                              │
│  Output: BasketItem events → Edge Gateway    │
└──────────────────────────────────────────────┘
```

### 4.2 Model Management

```
┌────────────────────────────────────────────────────────┐
│               MODEL LIFECYCLE                           │
│                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │  Train   │───▶│  Package │───▶│ Validate │          │
│  │  (Cloud) │    │ (ONNX/   │    │ (test    │          │
│  │          │    │  TensorRT│    │  dataset) │          │
│  └──────────┘    └──────────┘    └────┬─────┘          │
│                                       │                 │
│                    ┌──────────────────┘                 │
│                    ▼                                    │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐         │
│  │  Shadow  │───▶│  Canary  │───▶│  Full    │         │
│  │  Deploy  │    │  Deploy  │    │  Rollout │         │
│  │ (no real │    │ (1 site) │    │(all sites│         │
│  │  effect) │    │          │    │          │         │
│  └──────────┘    └──────────┘    └──────────┘         │
│                                                         │
│  Shadow mode: new model runs in parallel,               │
│  results compared but not used for billing              │
│                                                         │
│  Canary: new model used at 1 test site,                │
│  monitored for accuracy/latency regression              │
│                                                         │
│  Rollout: progressive deployment to all sites           │
│  with automatic rollback on metric degradation          │
└────────────────────────────────────────────────────────┘
```

---

## 5. Device Enrollment & Provisioning

### 5.1 Zero-Touch Provisioning Flow

```
1. Factory: Device manufactured with unique hardware fingerprint
   └─ Device ships with bootstrap certificate (short-lived)

2. Site Installation: Technician powers on device at site
   └─ Device connects to provisioning endpoint using bootstrap cert
   └─ Sends: hardware fingerprint + bootstrap cert

3. Cloud Provisioning Service:
   └─ Validates bootstrap cert (is it a legitimate device?)
   └─ Checks hardware fingerprint against purchase order
   └─ Issues: production mTLS certificate (long-lived, rotatable)
   └─ Assigns: deviceId, siteId, tenantId
   └─ Pushes: initial configuration, driver packages

4. Edge Controller:
   └─ Receives new device enrollment
   └─ Downloads and installs appropriate driver
   └─ Runs health check sequence
   └─ Reports device ONLINE status

5. Operator Verification:
   └─ Admin portal shows new device pending verification
   └─ Technician confirms physical installation via field app
   └─ Device status → ACTIVE
```

### 5.2 Certificate Rotation

```
Automated Certificate Rotation:
  - Production certificates valid for 1 year
  - Rotation starts 30 days before expiry
  - New certificate issued via CSR from device
  - Old certificate remains valid during overlap period (7 days)
  - Revocation via CRL distributed to MQTT broker and API gateway
  - Emergency rotation: cloud can force immediate rotation
```

---

## 6. OTA Update System

### 6.1 TUF-Based Update Architecture

```
┌─────────────────────────────────────────────────┐
│              UPDATE REPOSITORY (Cloud)           │
│                                                  │
│  ┌────────────┐  ┌────────────┐  ┌───────────┐  │
│  │   Root     │  │  Targets   │  │ Timestamp  │  │
│  │  Metadata  │  │  Metadata  │  │  Metadata  │  │
│  │ (offline   │  │ (signed    │  │ (freshness │  │
│  │  signing)  │  │  manifest) │  │  guarantee)│  │
│  └────────────┘  └────────────┘  └───────────┘  │
│                                                  │
│  ┌────────────┐  ┌────────────┐                 │
│  │  Snapshot  │  │  Target    │                 │
│  │  Metadata  │  │  Artifacts │                 │
│  │ (version   │  │ (container │                 │
│  │  manifest) │  │  images,   │                 │
│  │            │  │  configs)  │                 │
│  └────────────┘  └────────────┘                 │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│           UPDATE AGENT (Edge)                    │
│                                                  │
│  1. Check for updates (periodic + push trigger)  │
│  2. Download metadata, verify signatures         │
│  3. Check version constraints & rollback safety  │
│  4. Download target artifacts                    │
│  5. Verify artifact hashes                       │
│  6. Apply update (container image swap)          │
│  7. Run health checks                           │
│  8. Report success/failure to cloud              │
│  9. Rollback if health checks fail              │
│                                                  │
│  Protections:                                    │
│  - Freeze attack: timestamp metadata has expiry  │
│  - Rollback attack: version monotonicity check   │
│  - Mix-and-match: snapshot binds all metadata    │
│  - Arbitrary software: targets signed by role    │
│  - Key compromise: threshold signing (m-of-n)    │
└─────────────────────────────────────────────────┘
```

### 6.2 Update Rollout Strategy

```
Update Created (v2.3.1)
     │
     ▼
┌──────────────────┐
│  Internal Test   │  (dev/staging sites)
│  Sites (2-3)     │  Duration: 24h
│  Pass? ──────────┤
└───────┬──────────┘
        │ Yes
        ▼
┌──────────────────┐
│  Canary Sites    │  (5% of production fleet)
│  (selected)      │  Duration: 48h
│  Pass? ──────────┤  Monitor: error rate, latency, device health
└───────┬──────────┘
        │ Yes
        ▼
┌──────────────────┐
│  Progressive     │  10% → 25% → 50% → 100%
│  Rollout         │  Duration: 1 week total
│  Auto-pause on   │  Pause if error rate > threshold
│  anomaly         │
└──────────────────┘

Rollback:
  - Automatic if health checks fail within 5 minutes of update
  - Manual trigger from operator dashboard
  - Preserves previous container image for instant rollback
```

---

## 7. Site Type Configurations

### 7.1 Service Kiosk Site

```
┌─────────────────────────────────────────┐
│         SERVICE KIOSK SITE              │
│                                         │
│  Edge Controller: Raspberry Pi CM4      │
│                                         │
│  Devices:                               │
│  ├── 21.5" Touchscreen Display          │
│  ├── QR/Barcode Scanner (Honeywell)     │
│  ├── Receipt Printer (Epson TM-T88)     │
│  ├── Card Reader (PAX)                  │
│  ├── Fingerprint Scanner (Suprema)      │
│  ├── SIM Dispensing Motor               │
│  ├── Document Scanner (optional)        │
│  └── 4G/LTE Modem (failover)           │
│                                         │
│  Software:                              │
│  ├── Kiosk UI (Android kiosk mode)      │
│  ├── Edge Gateway                       │
│  ├── Device Drivers                     │
│  ├── Offline Queue                      │
│  ├── Telemetry Collector                │
│  └── Update Agent                       │
│                                         │
│  Network: Ethernet primary, 4G backup   │
│  Power: UPS (15-minute battery backup)  │
└─────────────────────────────────────────┘
```

### 7.2 Smart Locker Site

```
┌─────────────────────────────────────────┐
│         SMART LOCKER SITE               │
│                                         │
│  Edge Controller: Raspberry Pi CM4      │
│                                         │
│  Devices:                               │
│  ├── 10" Touchscreen (customer UI)      │
│  ├── QR Scanner (entry verification)    │
│  ├── Locker Controller Board (custom)   │
│  │   ├── 12-48 Electronic Locks         │
│  │   ├── Door Sensors (per compartment) │
│  │   └── Optional: Weight Sensors       │
│  ├── Camera (security + deposit verify) │
│  ├── Speaker (audio instructions)       │
│  └── 4G/LTE Modem (primary or backup)  │
│                                         │
│  Software:                              │
│  ├── Locker UI (compact touchscreen)    │
│  ├── Edge Gateway                       │
│  ├── Lock Controller Driver             │
│  ├── Compartment State Manager          │
│  ├── Access Code Cache                  │
│  ├── Offline Queue                      │
│  ├── Telemetry Collector                │
│  └── Update Agent                       │
│                                         │
│  Network: 4G/LTE primary (outdoor)      │
│  Power: Mains + UPS (30-min backup)     │
└─────────────────────────────────────────┘
```

### 7.3 Autonomous Store Site

```
┌─────────────────────────────────────────┐
│       AUTONOMOUS STORE SITE             │
│                                         │
│  Edge Controller: NVIDIA Jetson Orin    │
│                                         │
│  Devices:                               │
│  ├── Entry Gate Controller              │
│  │   ├── QR/NFC Reader                  │
│  │   ├── Turnstile/Gate Actuator        │
│  │   └── Entry Camera                   │
│  ├── Camera Array (8-16 ceiling cams)   │
│  ├── Shelf Sensor Array                 │
│  │   ├── Weight Sensors (per shelf)     │
│  │   └── Optional: RFID Readers         │
│  ├── Exit Gate Controller               │
│  │   ├── Exit Sensors                   │
│  │   └── Gate Actuator                  │
│  ├── Display Screens (pricing/promos)   │
│  └── 5G Modem (high-bandwidth backup)  │
│                                         │
│  Software:                              │
│  ├── Store Entry App (gate UI)          │
│  ├── Edge Gateway                       │
│  ├── AI Inference Pipeline              │
│  │   ├── Video Pipeline (GStreamer)     │
│  │   ├── Person Detection + Tracking    │
│  │   ├── Item Detection                 │
│  │   ├── Action Recognition             │
│  │   └── Sensor Fusion Engine           │
│  ├── Session Manager                    │
│  ├── Basket Reconciliation Engine       │
│  ├── Device Drivers (cameras, sensors)  │
│  ├── Offline Queue                      │
│  ├── Telemetry Collector                │
│  └── Update Agent                       │
│                                         │
│  Network: Ethernet (required, dedicated)│
│  Power: Mains + UPS (1-hour backup)     │
│  Storage: 256GB NVMe (video buffer)     │
└─────────────────────────────────────────┘
```
