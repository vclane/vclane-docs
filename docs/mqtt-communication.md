# MQTT Communication

MQTT provides the real-time communication channel between the backend, the ESP-32 Traffic Controllers, and the Raspberry Pi Edge Devices, enabling bidirectional messaging.

## Responsibilities

- Deliver timing schedules (turns loop + scheduling mode) and manual overrides to ESP-32 Traffic Controllers
- Deliver per-device queue `enqueue` commands to ESP-32 Traffic Controllers (dynamic mode)
- Receive AI traffic telemetry from Raspberry Pi Edge Devices (live + cumulative vehicle counts, per-direction / per-class breakdowns, congestion status, active traffic light phase) — live `vehicleCount >= 1` doubles as the vehicle-detection trigger in dynamic mode
- Receive fallback traffic light phase telemetry + queue-completion notices from ESP-32 Traffic Controllers
- Receive connection heartbeats and device health stats from all devices
- Maintain persistent, low-latency connections over TLS

## Recommended Brokers

- EMQX
- Mosquitto

## Security

- MQTT over TLS
- Device authentication (X.509 Certificate or username/password)
- Topic authorization (Restricting devices to their own topics)

## MQTT Topic Structure

```text
traffic/
    device/                 # Raspberry Pi Edge Devices
        {deviceId}/
            telemetry       # Publish vehicle count, lane density
            heartbeat       # Publish device status
            status          # LWT (Last Will & Testament) online/offline
            commands        # Subscribe to firmware updates/reboots
            acknowledgements# Publish command completion reports

    group/                  # ESP-32 Traffic Controllers
        {groupId}/
            commands        # Subscribe to overrides, cancel, and queue enqueue commands
            schedule        # Subscribe to schedule configs (turns loop + schedulingMode + dynamicConfig)
            schedule-request# Publish a request for the latest schedule on connect
            telemetry       # Publish fallback phases + queue-done completion notices
            heartbeat       # Publish controller heartbeat
            status          # LWT online/offline
```

> No new topics are used for dynamic scheduling — existing telemetry/command topics are reused with extended payloads (see below).

## Technology Stack

| Component      | Technology                        |
| -------------- | --------------------------------- |
| Protocol       | MQTT v3.1.1 (controller) / v5-capable broker |
| Encryption     | TLS (external)                    |
| MQTT Broker    | Mosquitto                         |
| Authentication | Mosquitto Dynamic Security Plugin |
| Authorization  | Role-based ACL (Dynamic Security) |
| Message Format | JSON                              |

## Key Features

**Edge Telemetry Communication** — Raspberry Pi Edge Devices publish traffic statistics and AI analysis results. With tracking enabled (`tracker: bytetrack.yaml`), `vehicleCount` is the live occupancy, `vehicleTotal` is the cumulative line-crossing total, and `directionCounts` / `classCounts` carry the breakdowns. In detect-only mode the device falls back to a mock `vehicleCount` random walk with `vehicleTotal: 0` and empty breakdowns.

```json
{
  "deviceId": "intersection-01",
  "vehicleCount": 12,
  "vehicleTotal": 148,
  "directionCounts": { "northbound": 148 },
  "classCounts": { "car": 9, "truck": 2, "bus": 1 },
  "trafficDensity": "MODERATE",
  "currentTrafficLightSignal": "GREEN",
  "timestamp": "2026-07-24T10:00:00Z"
}
```

**Fallback Telemetry Communication** — The ESP-32 Traffic Controller publishes the current state of traffic lights in its group as a fallback when edge devices lose connectivity.

```json
{
  "groupId": "group-01",
  "activePhases": [
    { "deviceId": "intersection-01", "phase": "GREEN" },
    { "deviceId": "intersection-02", "phase": "RED" }
  ],
  "isFallback": true,
  "timestamp": "2026-07-24T10:00:05Z"
}
```

**Controller Command & Schedule Delivery** — The backend publishes traffic schedule configurations and manual overrides directly to the group's ESP-32 controller.

**Group Command Envelope** — Commands on `traffic/group/{gid}/commands` use the `{command, payload}` envelope:

```json
{ "command": "signal_override", "payload": { "phases": { "device-001": "GREEN", "device-002": "RED" }, "durationSeconds": 30 } }
{ "command": "cancel_override", "payload": {} }
```

`durationSeconds <= 0` is indefinite. While an override is active the controller pauses the schedule turn-loop and uses the override phases in its place; when the override ends (expiry or cancel) it resumes the same turn it paused on but restarts that turn's full duration from the beginning — remaining time is not preserved.

**Schedule Sync on Connect** — On connection, the ESP-32 publishes an empty message to `traffic/group/{groupId}/schedule-request`. The backend responds on `traffic/group/{groupId}/schedule` with the stored schedule, keeping the controller's cached copy (and offline NVS fallback) in sync without periodic polling.

**Dynamic Queue Scheduling (reused topics, no new topics)** — Applies when the group's `schedulingMode` is `"dynamic"`:

- **Detection (Edge → Backend):** Reuses `traffic/device/{deviceId}/telemetry`. Telemetry with live `vehicleCount >= 1` (threshold = 1 vehicle) is a detection event. The edge publishes immediately on the `0 → >=1` rising edge, plus its normal periodic publish.
- **Enqueue (Backend → Controller):** Reuses `traffic/group/{groupId}/commands` with the standard envelope:

```json
{ "command": "enqueue", "payload": { "deviceId": "intersection-01" } }
```

- **Queue completion (Controller → Backend):** Reuses `traffic/group/{groupId}/telemetry` with two extra fields:

```json
{
  "groupId": "group-01",
  "activePhases": [
    { "deviceId": "intersection-01", "phase": "RED" },
    { "deviceId": "intersection-02", "phase": "RED" }
  ],
  "queueDoneDeviceId": "intersection-01",
  "queueDepth": 1,
  "timestamp": "2026-07-24T10:00:20Z"
}
```

`queueDoneDeviceId` tells the backend to clear `has_queued_schedule` for that device in RTDB. `queueDepth` is the remaining FIFO depth after completion. When the queue is empty the controller broadcasts all-RED.

**Device Monitoring** — Online/offline tracking for all hardware via LWT (Last Will and Testament) topics and connection status reporting.

**Traffic Synchronization** — Coordinated intersection timing is managed locally within each group by the ESP-32 Traffic Controller via LoRa radio broadcasts, eliminating the need for complex, latency-sensitive multi-device MQTT coordination protocols.
