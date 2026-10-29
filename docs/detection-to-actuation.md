# Detection to Actuation

This page follows a single vehicle detection through the whole control loop: an **edge device** detects and uploads it over REST, the **backend** decides whether it may queue it, the **ESP-32 controller** schedules the phase, and the **edge devices** actuate their relays. It covers the normal dynamic-queue path, emergency overrides, fixed-mode behavior, and what remains when the server is unreachable.

The loop has four hops:

| Hop | Transport | Payload |
| --- | --------- | ------- |
| Edge → Server | `POST /api/devices/{id}/detections` (HTTPS, JWT multipart) | `trackId`, `class`, `deviceTimestamp`, annotated frame |
| Server → Controller | MQTT `traffic/group/{gid}/commands` (QoS 1) | `enqueue` or `signal_override` |
| Controller → Edge | LoRa broadcast (default) or MQTT `traffic/group/{gid}/phase` | phase frame `{gid, seq, ts, p, ttl, ov}` |
| Edge → Server | MQTT telemetry / `queue-done` | lean signal + emergency flag; queue completion |

Two invariants matter throughout:

- Detections travel over **REST**, not MQTT, and only **new** documents run the queue gate. Telemetry never triggers an enqueue.
- The edge device does not know group mode, queue state, or `enqueueEdgeDetections`. It detects, uploads, and actuates whatever phase frame it receives.

## End-to-End Flowchart

```mermaid
flowchart TB
    subgraph DETECT["1-2 · Edge device — detection"]
        CAM["Camera frames"] --> TRACK["YOLO + ByteTrack<br/>persistent track IDs"]
        TRACK --> NEW{"New ByteTrack ID?"}
        NEW -->|no| TRACK
        NEW -->|yes| EVT["DetectionEvent<br/>trackId · class · annotated frame"]
        TRACK --> EMERG{"Emergency class debounced<br/>over N frames?"}
        EVT --> POST["POST /api/devices/{id}/detections<br/>multipart + JWT"]
        EMERG -->|yes| EOVR["POST /api/groups/{gid}/commands<br/>signal_override (edge-initiated)"]
    end

    subgraph SERVER["3-5 · Server — decision (FastAPI + Firestore/RTDB + Mosquitto)"]
        AUTH["Authenticate JWT<br/>forceEnqueue requires admin"]
        DUP{"Detection doc exists?<br/>trackId + deviceTimestamp"}
        DUPED["Return 200 deduped: true"]
        SAVE["Upload image · write detection doc<br/>audit: enqueued=false · signalOverrideSent=false"]
        ESTALE{"Emergency class<br/>and ≤ 180 s old?"}
        OVR["Publish signal_override<br/>detector GREEN · siblings RED · 60 s"]
        DYN{"Group schedulingMode<br/>dynamic?"}
        FORCE{"forceEnqueue<br/>from admin?"}
        SW{"enqueueEdgeDetections<br/>enabled?"}
        ALLOW{"Vehicle class<br/>allowlisted?"}
        FRESH{"deviceTimestamp<br/>≤ 180 s old?"}
        SIG{"Signal is RED<br/>or CAUTION?"}
        QD{"has_queued_schedule<br/>false?"}
        ENQ["Publish enqueue(deviceId) — QoS 1<br/>RTDB has_queued_schedule = true"]
        DROP["Store only<br/>no enqueue · no override"]
        CLR["Clear RTDB<br/>has_queued_schedule = false"]
    end

    subgraph SCHED["6-9 · ESP-32 controller — scheduling"]
        FIFO["Append FIFO entry<br/>no dedupe · max 16"]
        IDLE["All-RED idle"]
        SERVE["Serve head: GREEN(device)<br/>RED(siblings) · greenSec"]
        CLEAR["All-RED clearance · allRedSec"]
        QDONE["Publish queue-done<br/>queueDoneDeviceId · queueDepth"]
        OVRC["Pause schedule/queue<br/>apply override phases"]
        LOSS["Broker / Wi-Fi lost"] --> OFFLINE
        OFFLINE["Offline fallback<br/>cached fixed turns loop on RTC"]
    end

    subgraph ACT["10-12 · Edge devices — actuation"]
        FRAME["Phase frame — LoRa (default) or MQTT phase topic<br/>gid · seq · ts · p · ttl · ov"]
        SEQ{"seq newer?"}
        STALE["Ignore duplicate/stale frame"]
        APPLY["apply_phase(phase, ov)"]
        BLINK["Relay blink ×3 (ov=false)<br/>or ×5 (ov=true), then latch solid"]
        RELAY["GPIO relays drive GREEN / RED lamps"]
        CAUTION["Caution: all relays off"]
        TELEM["Publish telemetry — signal + emergency flag (1/s)"]
        RTDB["RTDB /telemetry/{id}"]
    end

    POST --> AUTH --> DUP
    DUP -->|yes| DUPED
    DUP -->|no| SAVE --> ESTALE
    ESTALE -->|yes| OVR
    ESTALE -->|no| DYN
    DYN -->|no| DROP
    DYN -->|yes| FORCE
    FORCE -->|yes| ENQ
    FORCE -->|no| SW
    SW -->|no| DROP
    SW -->|yes| ALLOW
    ALLOW -->|no| DROP
    ALLOW -->|yes| FRESH
    FRESH -->|no| DROP
    FRESH -->|yes| SIG
    SIG -->|no| DROP
    SIG -->|yes| QD
    QD -->|yes| DROP
    QD -->|no| ENQ
    ENQ --> FIFO
    EOVR -.->|same override command| OVRC
    OVR -->|MQTT| OVRC
    FIFO --> SERVE --> CLEAR --> QDONE
    QDONE -->|queue not empty| SERVE
    QDONE -->|queue empty| IDLE
    QDONE -->|MQTT| CLR
    CLR -.->|next detection| QD
    IDLE --> FRAME
    SERVE --> FRAME
    CLEAR --> FRAME
    OVRC --> FRAME
    OFFLINE -.->|LoRa only| FRAME
    FRAME --> SEQ
    SEQ -->|no| STALE
    SEQ -->|yes| APPLY --> BLINK --> RELAY
    FRAME -.->|no valid phase for 5 s| CAUTION
    RELAY --> TELEM
    CAUTION --> TELEM
    TELEM -->|MQTT| RTDB
```

**Reading the stages**

1. **Detection (1-2):** ByteTrack assigns a persistent ID; the first sighting emits one detection event with the annotated frame. Separately, an emergency-class vehicle debounced over consecutive frames can trigger an edge-initiated override request.
2. **Decision (3-5):** The backend authenticates, dedupes, stores the detection, and either publishes an emergency override, publishes an `enqueue`, or stores only.
3. **Scheduling (6-9):** The controller appends the entry to its FIFO, serves it as GREEN for `greenSec`, clears with all-RED for `allRedSec`, then reports `queue-done`.
4. **Actuation (10-12):** Every edge in the group receives the same phase frame, dedupes it by `seq`, blinks the requested lamps (3 normally, 5 during an override), latches solid, and reports lean telemetry.

## End-to-End Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant A as Edge A (detector)
    participant B as Edge B (sibling)
    participant API as Backend API
    participant DB as Firestore / RTDB
    participant MQ as MQTT Broker
    participant C as ESP-32 Controller

    Note over C: dynamic mode · FIFO empty · all-RED idle
    A->>A: YOLO + ByteTrack detects new track ID
    A->>API: POST /api/devices/A/detections<br/>(trackId, class, deviceTimestamp, annotated frame)
    API->>API: verify JWT · dedupe on trackId + deviceTimestamp
    API->>DB: upload image · write detections/{...}
    API->>API: queue gate — dynamic · admin forceEnqueue · enqueueEdgeDetections ·<br/>vehicle allowlist · ≤180 s · RED/CAUTION · not already queued
    API->>MQ: traffic/group/G/commands<br/>enqueue(deviceId=A) — QoS 1
    API->>DB: RTDB /telemetry/A: has_queued_schedule = true
    MQ->>C: enqueue(A)
    C->>C: append one FIFO entry (max 16, no dedupe)
    C->>A: phase frame — GREEN(A), RED(B) — greenSec [ov=false]
    Note over C,A: LoRa broadcast (default), or MQTT phase topic when sync_transport = mqtt/dual
    Note over A: seq dedupe → apply_phase → 3 blinks → latch solid
    A->>MQ: telemetry — signal=GREEN, emergency flag
    B->>MQ: telemetry — signal=RED
    MQ->>API: traffic/device/+/telemetry
    API->>DB: RTDB /telemetry/{id} + lastSeen
    C->>A: phase frame — all-RED — allRedSec
    C->>MQ: traffic/group/G/queue-done<br/>(queueDoneDeviceId=A, queueDepth)
    MQ->>API: queue-done
    API->>DB: RTDB /telemetry/A: has_queued_schedule = false
    alt queue has more entries
        C->>B: phase frame — GREEN(B), RED(A) — greenSec
        B->>MQ: telemetry — signal=GREEN
        C->>A: phase frame — all-RED — allRedSec
        C->>MQ: queue-done (B, depth 0)
        MQ->>API: queue-done
        API->>DB: clear has_queued_schedule(B)
    else queue empty
        C->>A: all-RED idle phase heartbeat (1/s)
    end
    opt emergency class detected (fixed or dynamic)
        A->>API: POST /detections — class = emergency vehicle
        API->>MQ: commands signal_override<br/>{A:"GREEN", B:"RED", durationSeconds:60}
        MQ->>C: signal_override
        Note over C: override pauses queue/fixed loop · ov=true
        C->>A: override phase — GREEN(A) [60 s, ov=true]
        Note over A: 5 blinks → latch solid
        Note over C: on expiry/cancel resume same turn/phase<br/>full duration restarts
    end
    opt backend / broker unreachable
        A->>API: detection upload retried with backoff
        Note over A,API: no gate, no enqueue — upload queue holds
        C->>C: offline fallback — cached fixed loop on RTC
        C->>A: LoRa phase frames only (no server in the path)
    end
```

## Path Variants

| Path | Trigger | Server | Controller | Edge |
| ---- | ------- | ------ | ---------- | ---- |
| Dynamic queue | New allowlisted vehicle detection, group `dynamic`, signal RED/CAUTION, not already queued | Publish `enqueue`, set RTDB flag | Serve FIFO head GREEN for `greenSec`, then all-RED for `allRedSec`, publish `queue-done` | 3 blinks, latch solid |
| Admin force-enqueue | `forceEnqueue=true` from an admin, group `dynamic` | Skips `enqueueEdgeDetections` and the allowlist/staleness/signal/already-queued checks | Same FIFO serve (duplicates allowed) | 3 blinks |
| Detections disabled | Group `enqueueEdgeDetections: false` | Store only | Untouched | Unchanged |
| Fixed mode | `schedulingMode: "fixed"` | Store only; `enqueue` is never sent | Runs the cached turns loop | 3 blinks per turn |
| Emergency | `class` matches the emergency allowlist and detection is ≤ 180 s old | Publish `signal_override` (detector GREEN, siblings RED, 60 s); ignores mode and `enqueueEdgeDetections` | Pauses queue/turn loop, broadcasts override phases with `ov=true` | 5 blinks, latch solid |
| No bound controller | Emergency on a device with no group or no `controllerId` | Publish device-level `signal_override` with `targetSignal: "GREEN"` | Not involved | Applies the device command |
| Server / broker offline | Internet loss | No gate, no enqueue; edge uploads retry | Cached fixed turns loop on RTC, LoRa only | Receives phases over LoRa; MQTT-only groups enter caution |
| Phase loss | No valid frame for 5 s | — | — | All relays off (caution) |

## Timing and Safety Notes

- **Staleness:** 180 s for both the emergency and queue gates. Stale offline retries are stored but never queue or override.
- **Queue capacity:** 16 entries. The controller appends every `enqueue` exactly once; duplicates and the currently-GREEN device are served in order.
- **Queue dedupe:** `has_queued_schedule` is a per-device boolean, not a per-entry counter, so a duplicate forced entry can still be pending when the first `queue-done` clears it.
- **Override duration:** `durationSeconds` `0` is indefinite, positive is a fixed duration, negative is rejected. Overrides live in RAM only and are lost on controller reboot.
- **Resume:** after expiry or cancel, the controller resumes the same turn or interrupted queue phase, but restarts its full duration.
- **Phase frames:** the controller sends a 1 s heartbeat and a burst of 3 packets with the same `seq` on every phase change. Edge devices dedupe by `seq`; `ov` selects 3 vs 5 blinks (missing `ov` defaults to `false`).
- **Offline caveat:** with `sync_transport: mqtt`, there is no LoRa fallback. Without broker connectivity the edge sees no valid phase and enters caution after the 5 s timeout.
- **Emergency dual paths:** the backend reaction above is primary. The edge may also POST `signal_override` to `/api/groups/{id}/commands` directly (admin credentials, 60 s cooldown); if both fire, the controller receives an identical duplicate override.

## Related Pages

- [Architecture](architecture.md) — components and communication modes
- [Edge Device](edge-device.md) — video pipeline, tracking, and relay actuation
- [Traffic Controller](traffic-controller.md) — FIFO, overrides, and offline state machine
- [Backend API](backend-api.md) — detection endpoint and gate pseudocode
- [MQTT Communication](mqtt-communication.md) — topics and phase sync transport
