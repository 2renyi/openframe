# OpenFrame L2 · Semantic Layer Specification

**Version:** 0.1-draft  
**Status:** Working Draft

---

## Overview

The Semantic Layer (L2) is OpenFrame's most important layer. It defines how streams of raw device observations (L1) are transformed into discrete, semantically meaningful objects that AI systems can reason over without needing to understand underlying sensor physics.

**L2 answers:** *What happened? To what entity? In what context? With what confidence?*

Without L2, AI systems must either:
1. Understand domain-specific sensor physics to interpret raw values, or
2. Receive ad-hoc, non-standard semantic representations

L2 eliminates both problems by providing a common semantic contract.

---

## Core Object Model

L2 defines six first-class object types:

```
Entity          — what exists in the world
  │
  ├── Observation   — what was measured (from L1)
  │
  ├── State         — what condition an entity is in
  │
  ├── Event         — what discrete occurrence happened
  │
  └── Context       — what situational frame surrounds everything
       │
       └── Derivation   — how any L2 object was produced from sources
```

---

## 1. Entity

An Entity is a stable, identifiable real-world thing to which observations, states, and events are attributed.

### Schema

```json
{
  "openframe_version": "0.2",
  "layer": 2,
  "object_type": "entity",
  "entity_id": "uuid-v4",
  "entity_type": "string",
  "name": "string",
  "description": "string (optional)",
  "location": {
    "label": "string",
    "lat": "number (optional)",
    "lon": "number (optional)",
    "floor": "string (optional)",
    "zone_id": "string (optional)"
  },
  "properties": "object (domain-specific key-value)",
  "parent_entity_id": "uuid (optional — for hierarchical composition)",
  "created_at": "ISO 8601",
  "tags": ["string"]
}
```

### Entity Type Registry (Core)

| Type | Description |
|---|---|
| `device` | A physical computing/communication device |
| `sensor` | A measurement-producing component (may be part of a device) |
| `actuator` | A component that changes physical state |
| `asset` | An industrial or operational asset (machine, vehicle, infrastructure) |
| `person` | A human subject (requires `sensitivity_label: pii`) |
| `zone` | A defined physical space (room, area, geofence) |
| `vehicle` | A mobile asset |
| `product` | A tracked physical product or batch |
| `system` | A composed system of multiple entities |

Domain extensions add to this registry under namespaced prefixes (e.g., `openframe.industrial.cnc_machine`).

### Rules

- Every Entity must have a stable `entity_id` that does not change over the entity's lifetime
- `parent_entity_id` supports hierarchical composition (e.g., sensor → device → asset → system)
- Entity records should be maintained in a registry; L2 objects reference entities by ID

---

## 2. Observation (L1 bridge object)

Observations are L1 data surfaced at the L2 level for reference by States, Events, and Derivations. They are not new objects — they are L1 objects with their `entity_id` resolved via the entity registry.

L2 processing systems must resolve `device_id` and `sensor_id` from L1 to a known Entity before an observation can be referenced by L2 objects.

---

## 3. State

A State describes the condition of an Entity over a time interval.

### Schema

```json
{
  "openframe_version": "0.2",
  "layer": 2,
  "object_type": "state",
  "state_id": "uuid-v4",
  "entity_id": "ref → Entity.entity_id",
  "state_type": "string (from State Type Registry)",
  "value": "string | number | object",
  "timestamp_start": "ISO 8601",
  "timestamp_end": "ISO 8601 | null",
  "duration_seconds": "number | null",
  "confidence": "0.0–1.0",
  "quality": {
    "status": "good | degraded | uncertain | bad",
    "has_gaps": "boolean",
    "interpolation_used": "boolean"
  },
  "source_observation_ids": ["uuid (L1 observation IDs)"],
  "derivation_id": "uuid (ref → Derivation)",
  "context_ids": ["uuid (ref → Context, optional)"],
  "sensitivity_label": "public | internal | pii | phi | confidential",
  "tags": ["string"]
}
```

### State Type Registry (Core)

**Operational states:**
`running` · `idle` · `standby` · `starting` · `stopping` · `fault` · `maintenance` · `offline` · `unknown`

**Condition states:**
`normal` · `degraded` · `critical` · `overheating` · `overloaded` · `underperforming` · `calibrating`

**Occupancy/presence states:**
`occupied` · `vacant` · `in_transit` · `docked` · `charging`

**Environmental states:**
`within_spec` · `threshold_exceeded` · `out_of_range`

**Health/physiological states** (`openframe.health.*`):
`awake` · `sleep_light` · `sleep_deep` · `sleep_rem` · `active` · `resting` · `elevated_hr` · `elevated_stress`

### Rules

- `timestamp_end: null` means the state is ongoing at the time of record creation
- `confidence` must not exceed the minimum confidence of source observations
- State transitions should be represented as sequential State records, not mutations

---

## 4. Event

An Event is a discrete, time-bounded occurrence associated with an Entity.

### Schema

```json
{
  "openframe_version": "0.2",
  "layer": 2,
  "object_type": "event",
  "event_id": "uuid-v4",
  "entity_id": "ref → Entity.entity_id",
  "event_type": "string (from Event Type Registry)",
  "timestamp": "ISO 8601",
  "timestamp_end": "ISO 8601 | null",
  "duration_seconds": "number | null",
  "severity": "info | warning | critical",
  "summary": "string (natural language description)",
  "attributes": "object (event-specific structured data)",
  "confidence": "0.0–1.0",
  "quality": {
    "status": "good | degraded | uncertain | bad"
  },
  "source_observation_ids": ["uuid"],
  "source_state_ids": ["uuid"],
  "derivation_id": "uuid (ref → Derivation)",
  "context_ids": ["uuid"],
  "related_event_ids": ["uuid (causal or correlated events, optional)"],
  "sensitivity_label": "public | internal | pii | phi | confidential",
  "tags": ["string"]
}
```

### Event Type Registry (Core)

**Lifecycle events:**
`started` · `stopped` · `paused` · `resumed` · `created` · `destroyed` · `connected` · `disconnected`

**Threshold / anomaly events:**
`threshold_exceeded` · `threshold_cleared` · `anomaly_detected` · `anomaly_resolved` · `spike_detected` · `drift_detected`

**Spatial events:**
`entered_zone` · `exited_zone` · `arrived` · `departed` · `position_updated`

**Fault / alert events:**
`fault_detected` · `fault_cleared` · `alert_triggered` · `alert_resolved` · `maintenance_required`

**Interaction events:**
`activated` · `deactivated` · `command_received` · `command_executed` · `command_failed`

**Industrial events** (`openframe.industrial.*`):
`production_started` · `production_completed` · `batch_started` · `batch_completed` · `quality_check_failed` · `tool_change` · `unplanned_downtime`

**Building events** (`openframe.building.*`):
`hvac_mode_changed` · `occupancy_changed` · `energy_spike` · `access_granted` · `access_denied`

### Rules

- `summary` is required and must be human-readable without referencing raw sensor values
- `attributes` contains structured event-specific data (e.g., peak amplitude, threshold delta)
- `confidence` reflects the reliability of event detection, not just data quality
- Events with `severity: critical` should propagate immediately to consuming systems

---

## 5. Context

A Context object provides situational framing that modifies how observations, states, and events should be interpreted.

### Schema

```json
{
  "openframe_version": "0.2",
  "layer": 2,
  "object_type": "context",
  "context_id": "uuid-v4",
  "context_type": "string (from Context Type Registry)",
  "timestamp": "ISO 8601",
  "valid_until": "ISO 8601 | null",
  "spatial_scope": {
    "zone_id": "string (optional)",
    "entity_ids": ["uuid (entities this context applies to)"]
  },
  "attributes": "object (context-type-specific key-value pairs)",
  "source": "scheduled | measured | manual | inferred",
  "confidence": "0.0–1.0 (for inferred context)"
}
```

### Context Type Registry (Core)

| Type | Example Attributes |
|---|---|
| `environment` | `ambient_temp_c`, `humidity_pct`, `air_quality_index`, `lighting_lux` |
| `schedule` | `shift`, `production_run_id`, `planned_downtime`, `maintenance_window` |
| `task` | `task_id`, `task_type`, `operator_id`, `target_asset_id` |
| `topology` | `upstream_entity_ids`, `downstream_entity_ids`, `network_segment` |
| `weather` | `condition`, `temp_c`, `wind_kph`, `precipitation_mm` |
| `occupancy` | `headcount`, `zone_id`, `occupancy_method` |
| `operational_mode` | `mode`, `setpoint`, `target_output`, `constraint_active` |

### Rules

- Context is not an observation — it is situational metadata, often from schedules, external APIs, or explicit operator input
- AI systems MUST be able to access relevant Context objects when processing L2 States and Events
- Context without a `valid_until` is considered to persist until superseded by a new Context of the same type in the same scope

---

## 6. Derivation Record

A Derivation Record documents exactly how an L2 object was produced from source data. Required for traceability.

### Schema

```json
{
  "openframe_version": "0.2",
  "layer": 2,
  "object_type": "derivation",
  "derivation_id": "uuid-v4",
  "output_object_id": "ref → State.state_id or Event.event_id",
  "output_object_type": "state | event",
  "source_observation_ids": ["uuid (L1 IDs used as input)"],
  "source_state_ids": ["uuid (L2 State IDs used as input, optional)"],
  "transformation_type": "threshold_rule | aggregation | ml_inference | rule_engine | manual | adapter",
  "model_or_rule_id": "string",
  "model_version": "string",
  "processing_time": "ISO 8601",
  "processing_duration_ms": "number",
  "parameters": "object (transformation-specific configuration)",
  "input_window": {
    "start": "ISO 8601",
    "end": "ISO 8601"
  },
  "notes": "string (optional)"
}
```

### Rules

- Every L2 State or Event produced by automated processing MUST have an associated Derivation Record
- Manually created States/Events use `transformation_type: manual` with `model_or_rule_id: null`
- Derivation Records must not be mutated after creation — corrections require new output objects with new Derivation Records

---

## 7. Temporal Model

L2 inherits OpenFrame's three-time model:

| Time | Meaning | Required On |
|---|---|---|
| `event_time` | When the thing actually happened | L1 observations |
| `timestamp` / `timestamp_start` | Canonical semantic time | All L2 objects |
| `processing_time` | When the L2 object was derived | Derivation Records |

### Time Window Rules

- State duration = `timestamp_end - timestamp_start`
- If `timestamp_end` is null, duration is unknown (state is ongoing)
- Events with duration use both `timestamp` (onset) and `timestamp_end` (resolution)
- All times are ISO 8601 with explicit timezone offset

---

## 8. Semantic Namespace and Extension

OpenFrame uses a hierarchical namespace for object types:

```
openframe.core.*       — base types, valid in all domains
openframe.industrial.* — manufacturing, process, equipment
openframe.building.*   — building systems, energy, facilities
openframe.health.*     — physiological, clinical, wellness
openframe.robotics.*   — motion, manipulation, navigation, embodied
openframe.logistics.*  — supply chain, fleet, cold chain
vendor.<name>.*        — vendor-specific (non-normative)
```

**Extension rules:**

1. Domain extensions may add new Entity types, State types, Event types, and Context types under their namespace
2. Extensions must not redefine core types
3. Core-compatible systems must pass unknown namespaced types through without error
4. A conformance profile specifies which namespaces a system is required to support

---

## 9. Quality Propagation

Quality degrades downstream but never silently improves:

- L2 State `confidence` ≤ minimum `quality.confidence` of all source L1 observations
- L2 Event `confidence` ≤ minimum `confidence` of all source States and observations
- L3 Representation inherits minimum quality of all source L2 objects
- A representation derived from `quality.status: bad` observations is itself `quality.status: bad` unless the derivation explicitly documents a quality recovery method

---

## 10. Sensitivity Labels

All L2 objects carry a `sensitivity_label`. Inheritance rules:

1. Objects without explicit labels inherit the highest label of their source objects
2. Redacted data must be documented in the Derivation Record
3. Processing systems must enforce label-appropriate access controls

---

## Implementation Notes

### Minimum conformant L2 processor

A conformant L2 processor must:

1. Accept L1 Observation objects as input
2. Resolve `device_id` / `sensor_id` to Entity records
3. Produce at minimum one State or Event object per processed window
4. Attach a Derivation Record to every produced object
5. Propagate quality and sensitivity labels correctly
6. Support the `openframe.core.*` namespace

### Recommended starting point

For new implementations, the recommended minimum L2 object set is:

- Entity (device + sensor)
- State (operational: running / idle / fault)
- Event (started / stopped / threshold_exceeded / fault_detected)
- Context (environment: ambient conditions)
- Derivation (for all produced objects)

---

*OpenFrame L2 Semantic Layer Specification v0.1-draft · github.com/2renyi/openframe*
