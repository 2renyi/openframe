# OpenFrame

> **An AI-facing semantic contract for device-generated data.**  
> Not a transport protocol. Not a replacement for MQTT, OPC-UA, or Matter.  
> The layer that sits above them — turning connected device data into something AI can actually understand.

---

## The Problem

IoT protocols are good at moving data. MQTT routes messages. OPC-UA models industrial assets. Matter connects smart home devices. They answer: *can the data be transmitted?*

But AI systems need a different question answered: *what does this data mean, and can I reason with it?*

A raw temperature reading is not the same as "the server room exceeded thermal threshold for 4 minutes at 14:32." A vibration waveform is not the same as "Pump #3 shows early-stage bearing wear." An occupancy sensor ping is not the same as "the workspace has been empty since 09:15."

The gap between *transmitted* and *AI-consumable* is large, and no existing standard closes it.

**OpenFrame closes it.**

---

## What OpenFrame Is

OpenFrame is an open specification that defines:

1. **How device observations should be structured** — with provenance, units, quality, and temporal metadata
2. **How raw observations become semantic states and events** — the mapping from signal to meaning
3. **How semantic objects become AI-representable inputs** — with lineage, task context, and model-agnostic encoding contracts

It is designed to sit on top of existing transport and connectivity protocols, not replace them.

```
[ Physical World ]
       │
       │  MQTT / OPC-UA / Matter / HTTP / BLE / ...
       ▼
┌─────────────────────────────────────────┐
│           OpenFrame                     │
│                                         │
│  L1 Observation  →  structured signal   │
│  L2 Semantic     →  state / event       │
│  L3 Representation → AI-ready input     │
└─────────────────────────────────────────┘
       │
       ▼
[ AI Model / Agent / Pipeline ]
```

---

## The Three Layers

### L1 · Observation Layer

Defines the standard envelope for raw device observations. Every observation carries:

- **Source identity** — device, sensor, calibration state
- **Temporal metadata** — event time, clock source, sampling rate
- **Physical metadata** — unit, precision, measurement subject
- **Quality metadata** — confidence, missing data flags, interpolation markers
- **Transport agnosticism** — same schema over MQTT, HTTP, Kafka, or file batch

L1 does not replace your transport protocol. It defines what the payload should look like.

### L2 · Semantic Layer

Transforms streams of L1 observations into discrete, meaningful objects:

| L1 (raw) | L2 (semantic) |
|---|---|
| accelerometer: 2.3g peak | Event: `vibration_anomaly` on Asset `pump_03` |
| temp sensor: 87°C | State: `thermal_threshold_exceeded` — duration 4 min |
| door sensor: open/close sequence | Event: `entry_detected` — Zone `warehouse_bay_2` |
| HR + SpO2 stream | State: `sleep_phase` — deep, confidence 0.87 |

L2 is where OpenFrame's core value lives. This is the semantic contract that makes device data legible to AI without requiring the AI to understand raw sensor physics.

### L3 · Representation Layer

Defines how L2 semantic objects are packaged for AI consumption. Deliberately model-agnostic — OpenFrame does not mandate a specific embedding model or vector format.

Instead, L3 defines the **representation contract**: metadata that must accompany any derived representation, including:

- Source semantic object reference
- Encoder identity and version
- Time window and aggregation method
- Intended task type (retrieval / classification / forecasting / control)
- Lineage and provenance chain
- Confidence and quality propagation

The actual representation format (embedding vector, token sequence, graph features) is a profile-level concern, not a core spec constraint.

---

## How It Relates to Existing Standards

| Standard | Role | OpenFrame Relationship |
|---|---|---|
| MQTT | Transport | OpenFrame payloads delivered over MQTT |
| OPC-UA | Industrial information model + transport | OpenFrame L2 objects map to OPC-UA nodes; L1 can wrap OPC-UA tag data |
| Matter | Smart home device interop | OpenFrame L2 consumes Matter device state as observation input |
| HL7 FHIR | Clinical data exchange | OpenFrame L2 health events can align with FHIR resource types |
| MCP (Anthropic) | AI tool invocation | OpenFrame L3 retrieval API is exposable as an MCP tool |
| RAG / Vector DBs | Retrieval infrastructure | OpenFrame L3 feeds vector stores with semantically grounded, traceable inputs |

---

## Target Scenarios

OpenFrame is most valuable where **device data is heterogeneous** and **AI must reason over it continuously**:

- **Industrial predictive maintenance** — multi-sensor fusion → semantic fault events → AI diagnosis
- **Smart building energy optimization** — HVAC + occupancy + schedules → scene state → AI control policy
- **Logistics and cold chain** — location + environment + asset state → supply chain events → AI exception handling
- **Robotics / embodied AI** — sensor streams → world state representation → policy input
- **Health monitoring** — physiological signals → clinical semantic events → AI-assisted assessment

---

## Project Status

🚧 **Pre-alpha — specification drafting in progress**

- [x] Concept and architecture
- [x] L1 Observation schema (v0.1 draft)
- [x] L2 Semantic object model (v0.1 draft)  
- [ ] L3 Representation contract spec
- [ ] Domain profiles (industrial / building / health)
- [ ] Reference implementation
- [ ] Validator tooling
- [ ] SDK (Python)

---

## Specification

Full specification documents are in [`/spec`](./spec/):

- [`spec/ARCHITECTURE.md`](./spec/ARCHITECTURE.md) — layer architecture and design principles
- [`spec/L1-observation.md`](./spec/L1-observation.md) — observation schema
- [`spec/L2-semantic.md`](./spec/L2-semantic.md) — semantic object model
- [`spec/L3-representation.md`](./spec/L3-representation.md) — representation contract

---

## Get Involved

This project is in early design phase. The most useful contributions right now are:

- **Use case descriptions** — what device+AI scenario do you need this for?
- **Protocol design feedback** — open an Issue tagged `spec`
- **Domain expertise** — industrial, medical, robotics, building automation

---

## License

MIT License — see [LICENSE](./LICENSE)

---

*OpenFrame — the semantic bridge between connected devices and AI systems.*
