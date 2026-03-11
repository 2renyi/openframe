# OpenFrame Architecture

**Version:** 0.2-draft  
**Status:** Working Draft

---

## 1. Design Philosophy

OpenFrame is not a transport protocol. It is a **semantic and representation contract** for device-generated data, designed to make heterogeneous physical-world observations legible to AI systems.

### What OpenFrame does NOT do

- Define how devices connect to networks
- Replace MQTT, OPC-UA, Matter, or any transport/connectivity standard
- Mandate a specific AI model or embedding algorithm
- Require devices to be replaced or reconfigured at the hardware level

### What OpenFrame DOES do

- Define a standard observation envelope (L1) that any device data can be wrapped in
- Define a semantic object model (L2) for states, events, and context
- Define a representation contract (L3) for AI-consumable outputs with full lineage
- Define how L1 → L2 → L3 transformations should be documented and traceable

---

## 2. Layer Architecture

```
┌────────────────────────────────────────────────────────────┐
│  L3 · Representation Layer                                  │
│  AI-consumable features, embeddings, token sequences        │
│  with lineage, encoder metadata, and task contracts         │
└──────────────────────────────┬─────────────────────────────┘
                               │  derives from
┌──────────────────────────────▼─────────────────────────────┐
│  L2 · Semantic Layer                                        │
│  Entities, Observations, States, Events, Context           │
│  — the core semantic contract of OpenFrame —               │
└──────────────────────────────┬─────────────────────────────┘
                               │  aggregates from
┌──────────────────────────────▼─────────────────────────────┐
│  L1 · Observation Layer                                     │
│  Raw or lightly processed device observations               │
│  with provenance, units, quality, and temporal metadata     │
└──────────────────────────────┬─────────────────────────────┘
                               │  transported over
         ┌─────────────────────▼──────────────────────┐
         │  Existing transport protocols               │
         │  MQTT · OPC-UA · HTTP · Kafka · BLE · ...  │
         └────────────────────────────────────────────┘
```

---

## 3. Layer Definitions

### L1 · Observation Layer

**Purpose:** Standardize the envelope for raw device observations across heterogeneous sources.

**Core principle:** An observation answers *"what value did this sensor report, when, and with what reliability?"*

**Required metadata every L1 object must carry:**

| Field | Description |
|---|---|
| `openframe_version` | Protocol version string |
| `layer` | Always `1` |
| `observation_id` | UUID v4, globally unique |
| `device_id` | Stable identifier for the source device |
| `sensor_id` | Identifier for the specific sensor on the device |
| `event_time` | ISO 8601 — when the measurement occurred |
| `received_time` | ISO 8601 — when the observation was received by the pipeline |
| `clock_source` | `gps`, `ntp`, `rtc`, `estimated` |
| `data_type` | Controlled vocabulary (see Data Type Registry) |
| `value` | The raw measured value |
| `unit` | SI unit string or registered domain unit |
| `quality` | Quality object (see Quality Model) |
| `os_env` | Device environment: `bare_metal` · `rtos` · `embedded_linux` · `linux` · `macos` · `windows` · `ios` · `android` |
| `sampling_rate_hz` | Nominal sampling rate, if applicable |
| `raw` | Optional: original payload before any transformation |

**Quality object:**

```json
{
  "status": "good | degraded | uncertain | bad",
  "confidence": 0.95,
  "missing_reason": null,
  "interpolated": false,
  "calibration_state": "calibrated | uncalibrated | expired | unknown",
  "anomaly_flag": false
}
```

**What L1 does NOT define:**

- How data is transported (use any protocol)
- Semantic meaning of the value (that is L2's job)
- How the value should be encoded for AI (that is L3's job)

---

### L2 · Semantic Layer

**Purpose:** Transform L1 observation streams into discrete, semantically meaningful objects that AI systems can reason over without understanding raw sensor physics.

**Core principle:** L2 answers *"what happened, to what entity, in what context?"*

L2 is the most important layer of OpenFrame. Without it, L3 representation is ungrounded.

**L2 object types:**

#### 2.1 Entity

A real-world entity that observations are attributed to.

```json
{
  "entity_id": "uuid",
  "entity_type": "device | sensor | asset | person | zone | vehicle | product | ...",
  "name": "Pump #3",
  "location": { "label": "Building A, Floor 2, Bay 7" },
  "properties": { "manufacturer": "...", "model": "...", "installed": "2024-03-01" }
}
```

#### 2.2 State

The condition of an entity over a time interval.

```json
{
  "state_id": "uuid",
  "entity_id": "ref → Entity",
  "state_type": "running | idle | overheating | occupied | fault | degraded | ...",
  "value": "overheating",
  "timestamp_start": "ISO 8601",
  "timestamp_end": "ISO 8601 | null (ongoing)",
  "confidence": 0.92,
  "source_observation_ids": ["uuid", "uuid"],
  "derivation": { "method": "threshold_rule | ml_model | ...", "model_id": "..." }
}
```

#### 2.3 Event

A discrete occurrence with a defined time point or interval.

```json
{
  "event_id": "uuid",
  "entity_id": "ref → Entity",
  "event_type": "started | stopped | fault_detected | threshold_exceeded | entry_detected | ...",
  "timestamp": "ISO 8601",
  "duration_seconds": 240,
  "severity": "info | warning | critical",
  "summary": "Pump #3 vibration anomaly detected — bearing wear signature",
  "attributes": { "peak_g": 2.3, "dominant_freq_hz": 47.5 },
  "confidence": 0.88,
  "source_observation_ids": ["uuid"],
  "source_state_ids": ["uuid"],
  "derivation": { "method": "ml_model", "model_id": "vibration-classifier-v2.1" }
}
```

#### 2.4 Context

Environmental or situational information that frames how observations and events should be interpreted.

```json
{
  "context_id": "uuid",
  "timestamp": "ISO 8601",
  "valid_until": "ISO 8601",
  "context_type": "environment | task | schedule | topology | operator | weather | ...",
  "attributes": {
    "ambient_temp_c": 22.4,
    "production_mode": "batch_run_47",
    "shift": "day",
    "operator_id": "op_042"
  },
  "spatial_scope": { "zone_id": "...", "label": "Building A Production Floor" }
}
```

#### 2.5 Derivation Record

Documents the transformation from L1 observations to L2 objects. Required for traceability.

```json
{
  "derivation_id": "uuid",
  "output_object_id": "ref → State or Event",
  "source_observation_ids": ["uuid"],
  "transformation_type": "threshold_rule | aggregation | ml_inference | rule_engine",
  "model_or_rule_id": "...",
  "model_version": "...",
  "processing_time": "ISO 8601",
  "parameters": { "window_seconds": 60, "threshold": 1.8 }
}
```

**L2 Semantic Namespace:**

OpenFrame defines a Core namespace for common entity types, state types, and event types. Domain-specific extensions use namespaced prefixes:

- `openframe.core.*` — universal base types
- `openframe.industrial.*` — manufacturing, process, equipment
- `openframe.building.*` — HVAC, energy, occupancy
- `openframe.health.*` — physiological, clinical
- `openframe.robotics.*` — motion, manipulation, navigation
- `vendor.*` — vendor-specific extensions (non-normative)

---

### L3 · Representation Layer

**Purpose:** Define the contract for AI-consumable representations derived from L2 semantic objects, with full lineage and task-context metadata.

**Core principle:** L3 does not define the representation format. It defines what metadata must accompany any representation, and what properties a compliant representation must guarantee.

**What L3 is NOT:**

- A specific embedding algorithm
- A specific vector dimension
- A mandate to use any particular AI model

**What L3 IS:**

A metadata contract ensuring any derived representation is traceable, task-annotated, and version-aware.

**L3 Representation object:**

```json
{
  "representation_id": "uuid",
  "source_semantic_ids": ["uuid (L2 State/Event/Context)"],
  "source_observation_ids": ["uuid (L1, optional)"],
  "representation_type": "embedding | token_sequence | feature_tensor | graph_features | multimodal_bundle",
  "intended_tasks": ["retrieval", "classification", "forecasting", "control"],
  "time_window": {
    "start": "ISO 8601",
    "end": "ISO 8601",
    "aggregation": "raw | mean | max | summary"
  },
  "encoder": {
    "id": "...",
    "version": "...",
    "modality": "timeseries | text | image | multimodal",
    "dimensions": 768,
    "normalization": "l2 | none | ..."
  },
  "quality_propagated": {
    "min_confidence": 0.88,
    "has_gaps": false,
    "interpolation_used": false
  },
  "created_at": "ISO 8601",
  "lineage_chain": ["derivation_id_1", "derivation_id_2"],
  "data": "[ float array | token ids | ... ]"
}
```

**Representation granularity levels:**

OpenFrame supports multi-granularity representations. The same physical asset may have representations at:

- `sample` — single observation window
- `event` — one discrete event
- `session` — one operational session or shift
- `asset` — aggregated asset health state
- `site` — facility-level aggregate

---

## 4. Time Model

OpenFrame distinguishes three time concepts that must be explicitly tracked:

| Concept | Field | Description |
|---|---|---|
| **Event time** | `event_time` | When the measurement/event actually occurred |
| **Received time** | `received_time` | When the system received the data |
| **Processing time** | `processing_time` | When the L2/L3 derivation was computed |

All timestamps are ISO 8601 with timezone. Clock source must be declared in L1 objects.

---

## 5. Quality Model

Quality propagates through all three layers. Downstream layers must not silently improve quality — if source observations have low confidence, derived L2 and L3 objects must reflect that.

Quality levels: `good` · `degraded` · `uncertain` · `bad`

---

## 6. Privacy and Data Sensitivity

L1 objects may carry a `sensitivity_label` field:

- `public` — no restrictions
- `internal` — organization-internal only
- `pii` — contains personally identifiable information
- `phi` — protected health information
- `confidential` — business-sensitive

L2 and L3 objects inherit the highest sensitivity label of their source objects unless explicit redaction is documented in the derivation record.

---

## 7. Versioning and Compatibility

- `openframe_version` field is required on all objects
- Backwards-incompatible changes increment the major version
- Field additions are backwards-compatible (minor version)
- Deprecated fields are announced one major version before removal
- L3 encoder versions must be pinned — changing encoder version requires new `representation_id`

---

## 8. Relationship to Existing Standards

| Standard | Relationship |
|---|---|
| **MQTT** | Recommended transport for L1 payloads in IoT deployments |
| **OPC-UA** | L2 entity/state model is compatible with OPC-UA information model; L1 can wrap OPC-UA tag reads |
| **Matter** | L1 observations can be generated from Matter device attribute reports |
| **HL7 FHIR** | L2 health domain events can be mapped to FHIR Observation/Condition resources |
| **MCP** | L3 retrieval query interface is expressible as an MCP tool definition |
| **OpenAPI 3.0** | All L3 query and ingest APIs are defined as OpenAPI specs |
| **JSON Schema** | All object schemas are normatively defined in JSON Schema |

---

*OpenFrame Architecture v0.2-draft · github.com/2renyi/openframe*
