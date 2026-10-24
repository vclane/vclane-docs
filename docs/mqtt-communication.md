# MQTT Communication

MQTT provides the real-time communication channel between the backend, the ESP-32 Traffic Controllers, and the Raspberry Pi Edge Devices, enabling bidirectional messaging.

## Responsibilities

- Deliver timing schedules (turns loop + scheduling mode) and manual overrides to ESP-32 Traffic Controllers
- Deliver per-device queue `enqueue` commands to ESP-32 Traffic Controllers (dynamic mode)
- Receive lean AI traffic telemetry from Raspberry Pi Edge Devices (active signal phase + emergency-vehicle flag; no counts) and heartbeats
- Receive queue-completion control notices from ESP-32 Traffic Controllers
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
            telemetry       # Publish lean telemetry (signal + emergency flag)
            heartbeat       # Publish device status
            status          # Retained online status + LWT (online: true/false)
            commands        # Subscribe to firmware updates/reboots
            acknowledgements# Publish command completion reports

    group/                  # ESP-32 Traffic Controllers
        {groupId}/
            commands        # Subscribe to overrides, cancel, and queue enqueue commands
            schedule        # Subscribe to schedule configs (turns loop + schedulingMode + dynamicConfig)
            schedule-request# Publish a request for the latest schedule on connect
            phase           # Controller publishes phase-sync frames (MQTT alt to LoRa); edge subscribes
            queue-done      # Publish queue-completion notices (controller does not publish telemetry)
            heartbeat       # Publish controller heartbeat
            status          # LWT online/offline
```

> Queue completion uses a dedicated `traffic/group/{groupId}/queue-done` topic; `enqueue` continues to reuse `traffic/group/{groupId}/commands` (see below).

> Per-vehicle detections are **not** sent over MQTT — they go via `POST /api/devices/{id}/detections` (see Backend API). Telemetry is the lean signal + emergency snapshot only; the dynamic queue gate triggers off the REST endpoint and honors the group's `enqueueEdgeDetections` switch (an admin `forceEnqueue` overrides it).

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

**Edge Telemetry Communication (sole telemetry source)** — Raspberry Pi Edge Devices publish lean status: `currentTrafficLightSignal` (or `RED` during fallback) plus `emergencyVehicleDetected` / `emergencyClass` / `emergencyConfidence` for visible emergency vehicles (`ambulance`, `fire truck`, `police car` by default). No counts, no density — per-vehicle history lives in `detections` via REST.

```json
{
  "deviceId": "intersection-01",
  "currentTrafficLightSignal": "GREEN",
  "emergencyVehicleDetected": false,
  "emergencyClass": "",
  "emergencyConfidence": 0.0,
  "timestamp": "2026-07-24T10:00:00Z"
}
```

**Controller Command & Schedule Delivery** — The backend publishes traffic schedule configurations and manual overrides directly to the group's ESP-32 controller.

**Group Command Envelope** — Commands on `traffic/group/{gid}/commands` use the `{command, payload}` envelope:

```json
{ "command": "signal_override", "payload": { "phases": { "device-001": "GREEN", "device-002": "RED" }, "durationSeconds": 30 } }
{ "command": "cancel_override", "payload": {} }
```

`durationSeconds` `0` is indefinite, a positive value is a fixed duration, and a negative value causes the controller to reject the command. While an override is active the controller pauses the schedule turn-loop (or dynamic queue) and uses the override phases in its place; when the override ends (expiry or cancel) it resumes the same turn/phase but restarts its full duration from the beginning — remaining time is not preserved.

Group commands are fire-and-forget: the backend does not validate `command`/`payload`, accepts any authenticated user, and writes no audit record. Only the device-level `POST /api/devices/{id}/signal` path writes `signalOverrides`.

**Schedule Sync on Connect** — On connection, the ESP-32 publishes an empty message to `traffic/group/{groupId}/schedule-request`. The backend responds on `traffic/group/{groupId}/schedule` with the stored schedule, keeping the controller's cached copy (and offline NVS fallback) in sync without periodic polling.

**Dynamic Queue Scheduling** — Applies when the group's `schedulingMode` is `"dynamic"`:

- **Detection (Edge → Backend):** `POST /api/devices/{deviceId}/detections` — one multipart request per new ByteTrack ID (first sighting, any YOLO label) with the full annotated frame (optional; `imageUrl` is `null` when omitted). New docs only (not `deduped:true`) with vehicle-allowlist + staleness guard run the queue gate, unless the group has `enqueueEdgeDetections: false` or an admin sends `forceEnqueue: true`.
- **Enqueue (Backend → Controller):** Reuses `traffic/group/{groupId}/commands` with the standard envelope:

```json
{ "command": "enqueue", "payload": { "deviceId": "intersection-01" } }
```

The controller appends one FIFO entry per command and does not dedupe. Duplicates and the currently-GREEN device are executed in order (`cam1 | cam1 | cam2` → all three), subject only to the 16-entry capacity.

- **Queue completion (Controller → Backend):** Publishes to `traffic/group/{groupId}/queue-done`:

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

`queueDoneDeviceId` tells the backend to clear `has_queued_schedule` for that device in RTDB. `queueDepth` is the remaining FIFO depth after completion, counting pending entries plus the one currently being served (duplicates included). When the queue is empty the controller broadcasts all-RED. Duplicate entries produce duplicate `queue-done` notices, and clearing the per-device `has_queued_schedule` flag more than once is harmless.

**Device Monitoring (Retained Status + LWT)** — Every edge device publishes its connection state to `traffic/device/{deviceId}/status` with **QoS 1 and the retain flag set**:

```json
{ "deviceId": "intersection-01", "online": true, "timestamp": "2026-07-24T10:00:00Z" }
```

- **On connect:** the device publishes `online: true`, replacing any stale retained state in the broker.
- **Graceful shutdown:** the device publishes `online: false` before it sends DISCONNECT (MQTT 3.1.1 does not fire the will on a clean disconnect).
- **Ungraceful loss:** the connect also registers a **retained** Last Will & Testament on the same topic with `{"online": false}`. The broker publishes it automatically when the connection drops (network loss, crash, power cut) after roughly 1.5 × the MQTT keepalive (60 s default → ~90 s detection).

The backend subscribes to `traffic/device/+/status`, mirrors `online` into RTDB `/telemetry/{deviceId}.online`, and refreshes `lastSeen`. There is no backend timeout checker. Because the messages are retained, a backend restart re-reads the last known state immediately.

> Broker ACL: the `device` role must allow `publishClientSend traffic/device/%u/status`; otherwise both the status publish and the will are denied.

**Traffic Synchronization** — Coordinated intersection timing is managed locally within each group by the ESP-32 Traffic Controller via LoRa radio broadcasts by default, eliminating the need for complex, latency-sensitive multi-device MQTT coordination protocols.

## Phase Sync via MQTT (LoRa Alternative)

When `sync_transport` is `mqtt` or `dual`, the controller publishes the same phase frame as the LoRa packet to `traffic/group/{groupId}/phase` with **QoS 1, retained** — 1/sec heartbeat plus a burst of 3 identical-`seq` frames on every phase change. Edge devices subscribe to their group's phase topic and feed frames into the same `seq`-dedup state as LoRa (first-valid wins in `dual` mode).

```json
{
  "gid": "group-01",
  "seq": 10542,
  "ts": 1782806400,
  "p": { "intersection-01": "GREEN", "intersection-02": "RED" },
  "ttl": 15,
  "ov": false
}
```

`ov` is `true` while an override is active (`ov = override.active()`), `false` otherwise — the edge maps it to 5 vs 3 relay blinks. Missing `ov` defaults to `false` for backward compatibility.

- Broker ACL: controller role may `publishClientSend traffic/group/%u/phase`; device role may `subscribePattern` / `publishClientReceive` / `unsubscribePattern traffic/group/+/phase`.
- The backend ignores this topic (no subscription change needed).
- Unlike LoRa, MQTT phase sync requires broker connectivity — if the broker is unreachable past the phase timeout, edge devices enter caution mode.
