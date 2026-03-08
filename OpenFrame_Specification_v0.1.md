# OpenFrame Protocol Specification

**Version:** 0.1 Draft  
**Author:** Liwei Cong  
**License:** CC BY 4.0  
**Date:** 2026-03

---

## Abstract

OpenFrame is an open protocol that defines a standardized pipeline for collecting real-world data from any device or system and delivering it to AI models as structured, semantically meaningful input — including vector embeddings for long-term AI memory.

OpenFrame fills a gap that no existing protocol addresses: the end-to-end standard from physical device signal to AI-consumable vector representation.

---

## 1. Problem Statement

AI models are powerful reasoners, but they are blind to the real world. They do not know:

- What the user did today
- What is happening in the user's environment
- The user's physical state, habits, and history

Existing approaches are fragmented:

| Existing Protocol | What It Does | What It Misses |
|------------------|--------------|----------------|
| Matter / Zigbee / Z-Wave | Device communication | No AI output layer |
| Apple HealthKit / Google Fit | Health data aggregation | Proprietary, closed |
| MCP (Anthropic) | AI tool invocation | No real-world data ingestion |
| RAG frameworks | Document retrieval | No device/sensor integration |
| MQTT / CoAP | IoT messaging | No semantic layer |

**OpenFrame defines the missing link: a universal standard from device data → structured event → vector embedding → AI memory.**

---

## 2. Architecture Overview

OpenFrame defines a three-layer architecture. Capable devices may implement any layer directly. Incapable devices rely on a gateway to handle upper layers.

```
┌─────────────────────────────────────────────────────┐
│                    DEVICE LAYER (L1)                 │
│  Raw signals from any device or operating environment│
│  Linux / Embedded Linux / RTOS / Bare-metal /        │
│  Windows / macOS / iOS / Android / No-OS             │
└──────────────────────┬──────────────────────────────┘
                       │ raw data upload
                       ▼
┌─────────────────────────────────────────────────────┐
│                   GATEWAY LAYER (L2)                 │
│  Aggregates raw data into structured semantic events │
│  Runs on: Mac Mini / Raspberry Pi / NAS / Server     │
└──────────────────────┬──────────────────────────────┘
                       │ structured events
                       ▼
┌─────────────────────────────────────────────────────┐
│                   VECTOR LAYER (L3)                  │
│  Converts events to embeddings                       │
│  Stores in vector database                           │
│  Exposes retrieval API to AI models                  │
└──────────────────────┬──────────────────────────────┘
                       │ semantic retrieval
                       ▼
┌─────────────────────────────────────────────────────┐
│                     AI MODEL                         │
│  Any model: local or cloud                           │
│  Claude / GPT / Gemini / LLaMA / etc.                │
└─────────────────────────────────────────────────────┘
```

### Two-path Device Model

```
[ Capable Device ]                [ Incapable Device ]
Implements L1+L2 or L1+L2+L3      Implements L1 only
Outputs structured events    →    Outputs raw signals
or vectors directly               Gateway handles L2+L3
         └──────────────────────────────┘
                          ↓
                  OpenFrame Data Stream
```

---

## 3. Layer 1 — Data Collection (L1)

### 3.1 Purpose

L1 defines how any device captures and transmits raw data into the OpenFrame pipeline.

### 3.2 Device Environment Matrix

| Environment | Examples | Implementation Method |
|-------------|---------|----------------------|
| Full Linux | Raspberry Pi, NAS, OpenWrt router | Python or Go agent process |
| Embedded Linux | Smart camera, smart speaker firmware | Lightweight binary or MQTT client |
| Bare-metal / RTOS | ESP32, Arduino, industrial sensors | C library, serial or BLE output |
| Windows | PC, workstation | Background service (exe) |
| macOS | Mac Mini, MacBook | Background daemon or menu bar app |
| iOS | iPhone, iPad | Native app (sandboxed) |
| Android | Phone, tablet, car system | App or background service |
| No OS | Pure hardware sensors | Output raw signal only — gateway translates |

### 3.3 L1 Raw Data Envelope

Every L1 message MUST contain:

```json
{
  "openframe_version": "0.1",
  "layer": 1,
  "device_id": "unique-device-identifier",
  "device_type": "sensor_type_string",
  "os_env": "linux | embedded_linux | rtos | bare_metal | windows | macos | ios | android | none",
  "timestamp_utc": "2026-03-08T11:15:00Z",
  "data_type": "heart_rate | location | temperature | activity | audio_env | ...",
  "value": "<raw value — number, string, or object>",
  "unit": "bpm | celsius | meters | steps | ...",
  "raw": "<optional: original unprocessed payload>"
}
```

### 3.4 Transport Protocols (L1)

Devices may use any of the following to deliver L1 data:

| Transport | Use Case |
|-----------|---------|
| MQTT | Low-power sensors, IoT standard |
| HTTP/HTTPS | Any network-connected device |
| WebSocket | Real-time streaming devices |
| BLE Advertisement | Ultra-low-power beacons |
| Serial / UART | Bare-metal to gateway |
| Unix socket | Local process on same machine |
| File drop | Batch upload from offline devices |

### 3.5 Power Protocol Awareness

OpenFrame-compliant devices SHOULD declare their power source to allow the gateway to infer reliability and data continuity:

```json
"power_source": "usb_c_pd | usb_a | poe | battery_li | battery_aa | solar | mains_220v | unknown"
```

---

## 4. Layer 2 — Semantic Event Layer (L2)

### 4.1 Purpose

L2 transforms raw data streams into discrete, meaningful **events** that describe what happened in the real world.

A raw data point answers: *"what is the value?"*  
An L2 event answers: *"what happened?"*

### 4.2 Event Schema

```json
{
  "openframe_version": "0.1",
  "layer": 2,
  "event_id": "uuid-v4",
  "event_type": "physical_activity | health_metric | location_visit | environment_change | device_interaction | communication | sleep | ...",
  "timestamp_start": "2026-03-08T14:00:00Z",
  "timestamp_end": "2026-03-08T14:30:00Z",
  "duration_seconds": 1800,
  "subject_device_ids": ["device-id-1", "device-id-2"],
  "location": {
    "label": "park",
    "lat": 39.9042,
    "lon": 116.4074,
    "accuracy_meters": 10
  },
  "summary": "User ran 5km in Chaoyang Park. Average heart rate 142bpm. Duration 30 minutes.",
  "attributes": {
    "distance_meters": 5000,
    "avg_heart_rate": 142,
    "max_heart_rate": 168,
    "calories": 420
  },
  "confidence": 0.95,
  "source_l1_ids": ["raw-data-uuid-1", "raw-data-uuid-2"]
}
```

### 4.3 Gateway Aggregation Rules

The gateway aggregates L1 data into L2 events using:

- **Time windowing** — group data points within a time range
- **Semantic clustering** — combine related data types (heart rate + GPS + accelerometer → running event)
- **Threshold triggering** — significant value change triggers a new event
- **Device correlation** — multiple devices contributing to same event are merged

### 4.4 Conflict Resolution: Device-native AI

Many modern devices contain proprietary AI (Apple Watch health AI, camera face recognition, car voice assistant). These are closed systems.

**OpenFrame does not replace device-native AI. It extracts outputs.**

```
Device Native AI (closed)
         ↓
Device Official API / SDK   ← OpenFrame Adapter reads here
         ↓
OpenFrame L2 Event
```

| Situation | Example | OpenFrame Approach |
|-----------|---------|-------------------|
| Open API exists | Apple HealthKit, Google Fit | Write Adapter, pull data |
| App only, no API | Some wearable brands | Companion app extracts data |
| Fully closed | Some cameras | Add external capture at edge |

---

## 5. Layer 3 — Vector Layer (L3)

### 5.1 Purpose

L3 converts L2 events into vector embeddings and stores them in a vector database. This creates a **persistent, semantically searchable memory** that any AI model can query.

This is the layer that no existing IoT or AI protocol defines. It is the core innovation of OpenFrame.

### 5.2 Embedding Process

```
L2 Event (JSON)
      ↓
Text serialization of event summary + attributes
      ↓
Embedding model (local or cloud)
      ↓
Vector [float32 × N dimensions]
      ↓
Vector database (with metadata)
```

Recommended embedding models:

| Model | Dimensions | Where it runs |
|-------|-----------|---------------|
| text-embedding-3-small (OpenAI) | 1536 | Cloud |
| text-embedding-3-large (OpenAI) | 3072 | Cloud |
| nomic-embed-text | 768 | Local |
| mxbai-embed-large | 1024 | Local |
| all-MiniLM-L6-v2 | 384 | Local, ultra-light |

### 5.3 Vector Record Schema

```json
{
  "vector_id": "uuid-v4",
  "openframe_version": "0.1",
  "layer": 3,
  "event_id": "reference to L2 event_id",
  "embedding": [0.23, -0.87, 0.44, "... N floats"],
  "embedding_model": "nomic-embed-text",
  "embedding_dimensions": 768,
  "timestamp_utc": "2026-03-08T14:30:00Z",
  "metadata": {
    "event_type": "physical_activity",
    "device_types": ["smartwatch", "phone"],
    "location_label": "park",
    "duration_seconds": 1800
  },
  "text_representation": "User ran 5km in Chaoyang Park. Average heart rate 142bpm. Duration 30 minutes."
}
```

### 5.4 Retrieval API

OpenFrame L3 exposes a standard retrieval interface to AI models:

```
GET /openframe/query
{
  "query_text": "How has my exercise been lately?",
  "top_k": 10,
  "filter": {
    "event_type": "physical_activity",
    "timestamp_after": "2026-02-01T00:00:00Z"
  }
}

Response:
{
  "results": [
    {
      "score": 0.94,
      "event_id": "...",
      "text_representation": "...",
      "metadata": { ... }
    }
  ]
}
```

### 5.5 Recommended Vector Databases

| Database | Deployment | Notes |
|----------|-----------|-------|
| Chroma | Local | Simple, Python-native, best for start |
| Qdrant | Local / Cloud | Production-grade, Rust-based |
| Weaviate | Local / Cloud | Built-in embedding support |
| pgvector | Local | PostgreSQL extension |
| Pinecone | Cloud only | Managed, easy to start |

---

## 6. OpenFrame Adapter Specification

An **Adapter** is a software component that connects a specific device or platform to the OpenFrame pipeline.

### 6.1 Adapter Interface

Every Adapter MUST implement:

```python
class OpenFrameAdapter:

    def device_id(self) -> str:
        """Return unique identifier for this device"""

    def device_type(self) -> str:
        """Return device type string"""

    def os_env(self) -> str:
        """Return operating environment"""

    def collect(self) -> List[L1Message]:
        """Collect raw data, return list of L1 messages"""

    def is_available(self) -> bool:
        """Return whether device/API is reachable"""
```

### 6.2 Adapter Registry

Adapters are registered by device type. The gateway auto-discovers and loads available adapters.

```
/openframe/adapters/
    apple_healthkit/
    google_fit/
    matter_devices/
    mqtt_generic/
    windows_activity/
    linux_system/
    ...
```

---

## 7. Privacy and Data Control

OpenFrame is designed with the principle that **the data owner controls everything**.

- All data is processed locally by default
- Cloud embedding/storage is opt-in only
- Each device and data type can be individually enabled or disabled
- Data retention period is user-configurable
- AI models receive only the data the user explicitly permits

---

## 8. Relationship to Existing Standards

| Standard | Relationship to OpenFrame |
|----------|--------------------------|
| Matter | OpenFrame should support Matter as an L1 transport for smart home devices |
| MCP (Anthropic) | OpenFrame L3 retrieval API can be exposed as an MCP tool |
| HL7 FHIR | Health data from medical devices should map to FHIR where applicable |
| W3C WoT | Web of Things device descriptions are compatible with OpenFrame device registry |
| OpenAPI | L3 Retrieval API is defined as an OpenAPI 3.0 spec |

---

## 9. Roadmap

| Milestone | Description | Status |
|-----------|------------|--------|
| v0.1 | Protocol specification draft | ✅ This document |
| v0.2 | Reference gateway implementation (Python) | Planned |
| v0.3 | First Adapters: Apple HealthKit, Linux system, MQTT generic | Planned |
| v0.4 | L3 vector layer reference implementation (Chroma) | Planned |
| v0.5 | MCP integration — expose OpenFrame as MCP tool | Planned |
| v1.0 | Stable protocol, multi-language SDK | Future |

---

## 10. Glossary

| Term | Definition |
|------|-----------|
| Adapter | Software component connecting a device to OpenFrame |
| Device Layer (L1) | Raw data collection layer |
| Semantic Layer (L2) | Structured event layer |
| Vector Layer (L3) | Embedding and retrieval layer |
| Gateway | Central node that runs L2 and L3 processing |
| Event | A discrete real-world occurrence derived from raw data |
| Embedding | A vector representation of an event in semantic space |

---

*OpenFrame Protocol Specification v0.1*  
*© 2026 Liwei Cong — Licensed under CC BY 4.0*  
*https://github.com/2renyi/openframe*
