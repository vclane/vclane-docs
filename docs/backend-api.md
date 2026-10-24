# Backend API

The backend acts as the centralized management and coordination layer between users, edge devices, and cloud services.

## Architecture

```mermaid
graph TB
    subgraph Server["VCLane Server (Docker Compose)"]
        Caddy["Reverse Proxy — Caddy<br/>(TLS termination)"]
        Backend["Backend API — FastAPI<br/>(Auth / Commands / Telemetry)"]
        Media["Media Server — MediaMTX<br/>(RTSP / WebRTC relay)"]
        MQTT["MQTT Broker — Mosquitto<br/>(TLS + password auth)"]

        Caddy -->|"/api/* /ws/* /health"| Backend
        Caddy -->|"/stream/*"| Media
        Backend -->|"Auth query"| Media
    end

    Backend -->|"Auth / DB"| Firebase["Firebase Services<br/>(Auth / Firestore / RTDB)"]
    MQTT -->|"Commands / Sync"| Edge["Raspberry Pi Edge Devices"]
    Edge -->|"Telemetry / Heartbeat / Ack"| MQTT
    Edge -->|"RTSP Stream"| Media
```

## Responsibilities

- User authentication
- Device registration
- Device management
- Traffic monitoring
- Telemetry processing
- Signal override handling
- Intersection grouping
- Traffic synchronization
- MQTT command publishing
- Firebase synchronization

## Technology Stack

| Component            | Technology                          |
| -------------------- | ----------------------------------- |
| Programming Language | Python                              |
| Backend Framework    | FastAPI                             |
| API Architecture     | REST API + WebSocket                |
| MQTT Integration     | Paho MQTT Client                    |
| MQTT Broker          | Mosquitto                           |
| Authentication       | Firebase Admin SDK + JWT + bcrypt   |
| Database             | Firestore                           |
| Real-Time Telemetry  | Realtime Database                   |
| Media Server         | MediaMTX (RTSP / WebRTC)            |
| Reverse Proxy        | Caddy                               |
| API Documentation    | OpenAPI / Swagger                   |
| Containerization     | Docker + Docker Compose             |
| Deployment           | Linux Server / Cloud Infrastructure |

## Key Features

**Device Management** — Register edge devices, manage device configurations, track device status, monitor device health, assign devices to intersections.

**Traffic Monitoring** — Receive real-time telemetry, monitor traffic conditions, store traffic information, generate traffic statistics, detect traffic events.

**Traffic Control** — Send traffic signal commands, trigger manual overrides, manage intersection schedules (turns loop + per-group `schedulingMode: fixed | dynamic`), coordinate multiple intersections, process command responses. In `dynamic` mode the backend runs a queue gate on new `POST /api/devices/{id}/detections` docs (vehicle allowlist + staleness guard → `enqueue` when RED/CAUTION + not already queued) and clears RTDB `has_queued_schedule` on controller queue-completion. Each group has an `enqueueEdgeDetections` switch (default `true`); when `false`, detections are stored but never enqueued. An admin may send `forceEnqueue=true` to skip the gate and the group switch.

**Real-Time Synchronization** — MQTT command publishing, device heartbeat processing, WebSocket client updates, event notifications.

## REST API Endpoints

| Method   | Path                         | Auth | Purpose                                      |
| -------- | ---------------------------- | ---- | -------------------------------------------- |
| `GET`    | `/health`                    | None | Health check                                 |
| `POST`   | `/api/auth/verify`           | None | Verify Firebase ID token, return session JWT |
| `GET`    | `/api/devices`               | JWT  | List all devices                             |
| `POST`   | `/api/devices`               | JWT  | Create device + provision secret             |
| `GET`    | `/api/devices/{id}`          | JWT  | Get device details                           |
| `PATCH`  | `/api/devices/{id}`          | JWT  | Update device                                |
| `DELETE` | `/api/devices/{id}`          | JWT  | Delete device                                |
| `POST`   | `/api/devices/{id}/commands` | JWT  | Send arbitrary command                       |
| `POST`   | `/api/devices/{id}/signal`   | JWT  | Override traffic signal                      |
| `POST`   | `/api/devices/{id}/detections` | JWT | Submit one per-vehicle detection (multipart `trackId`, `class`, `deviceTimestamp`, optional `image`, optional admin `forceEnqueue`) → optional Storage upload + Firestore `detections` write |
| `GET`    | `/api/groups`                | JWT  | List all groups                              |
| `POST`   | `/api/groups`                | JWT  | Create group                                 |
| `GET`    | `/api/groups/{id}`           | JWT  | Get group details                            |
| `PATCH`  | `/api/groups/{id}`           | JWT  | Update group                                 |
| `DELETE` | `/api/groups/{id}`           | JWT  | Delete group                                 |
| `POST`   | `/api/groups/{id}/sync`      | JWT  | Trigger group prepare                        |
| `POST`   | `/api/groups/{id}/schedule`  | JWT  | Publish + persist group schedule             |
| `GET`    | `/api/groups/{id}/schedule`  | JWT  | Get the stored group schedule                |
| `POST`   | `/api/groups/{id}/commands`  | JWT  | Send command to the group controller         |
| `GET`    | `/api/mediamtx/auth`         | None | MediaMTX auth callback                       |

See [Server Integration](server-integration.md) for details on authentication flows, MQTT communication, and component integration.

## Detections via REST

Per-vehicle detections are sent over HTTP REST, **not** MQTT. The edge posts one request per new ByteTrack ID (first sighting):

```
POST /api/devices/{id}/detections
Content-Type: multipart/form-data
```

| Part | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| `trackId` | `int \| string` | Yes | ByteTrack ID (part of dedup key) |
| `class` | `string` | Yes | YOLO label, unfiltered |
| `deviceTimestamp` | `string \| int` | Yes | Edge crossing time (part of dedup key) |
| `image` | `file (JPEG)` | No | Full annotated frame. When omitted, no Storage upload occurs and `imageUrl` is `null`. |
| `forceEnqueue` | `bool` | No (`false`) | Admin-only. Skips the queue gate **and** the group `enqueueEdgeDetections` switch. |

**Authorization:** the session JWT carries only the user ID (`sub`), so the backend resolves `users/{uid}.role`. A request with `forceEnqueue=true` from a user whose role is not `admin` returns `403` before anything is written.

Backend flow:

1. Authenticate via operator JWT; reject `forceEnqueue` unless the user's role is `admin`.
2. Validate the required parts (`trackId`, `class`, `deviceTimestamp`). `image` is optional.
3. Resolve `groupId` from `devices/{id}.groupId`.
4. Compute doc ID `{deviceId}_{trackId}_{deviceTimestamp}`; if it exists, return `200` with `deduped: true` (no rewrite and no queue gate).
5. When `image` is present, upload it to Storage (`detections/{deviceId}/{trackId}_{deviceTimestamp}.jpg`); otherwise leave `imageUrl` `null`.
6. Write the `detections` doc with server `timestamp` plus the audit fields `forceEnqueueRequested`, `enqueueEdgeDetections`, and `enqueued: false`.
7. Run the queue decision below, then update `enqueued`.
8. Return `201` with the doc ID.

Queue decision (a gate failure is logged and never changes the `201`):

```text
if no group or group.schedulingMode != "dynamic":
    store only
elif request.forceEnqueue:
    publish enqueue(deviceId)        # wins over enqueueEdgeDetections
    RTDB: has_queued_schedule = true, queuedAt = now
elif group.enqueueEdgeDetections is false:
    store only                       # audit-only
else:
    class in vehicle allowlist
    deviceTimestamp age <= 180 s
    currentTrafficLightSignal in {RED, CAUTION}
    has_queued_schedule != true
    -> publish enqueue + set RTDB flags
```

`forceEnqueue` still requires a group in `dynamic` mode: against a `fixed` group the detection is stored only (`enqueued: false`), because the controller ignores `enqueue` outside dynamic mode. It does not bypass detection doc dedupe — reusing `trackId` and `deviceTimestamp` returns `200 deduped:true` and never runs the gate.

Detections bypass Mosquitto entirely and never touch RTDB themselves.

> The gate dedups normal edge detections via RTDB `has_queued_schedule` (per device, cleared on controller `queue-done`). `forceEnqueue` bypasses that dedup intentionally and may create duplicate FIFO entries; the controller appends every `enqueue` it receives. See Server Integration.

## Schedule Synchronization

Schedules are configured per intersection group and are executed locally by the group's ESP-32 Traffic Controller. The backend stores the schedule as the source of truth and delivers it to the controller over MQTT.

### Schedule Model

A schedule is a repeating loop of turns. Turn order is significant: the controller runs turn 1, then turn 2, and so on, wrapping back to turn 1 after the last turn. Phase values are simple signal lamp states: `RED`, `GREEN`, `YELLOW`.

```json
{
  "turns": [
    { "durationSeconds": 90, "phases": { "device-001": "RED",   "device-002": "GREEN", "device-003": "RED" } },
    { "durationSeconds": 90, "phases": { "device-001": "GREEN", "device-002": "RED",   "device-003": "RED" } }
  ]
}
```

Validation rules:

- `turns` must be a non-empty array
- `durationSeconds` must be a positive integer (>= 1)
- `phases` must map device IDs to `RED`, `GREEN`, or `YELLOW`

### Flow

1. **Configure:** `POST /api/groups/{id}/schedule` validates the schedule, persists it to Firestore (`groups/{id}.schedule`), and immediately publishes it to `traffic/group/{id}/schedule`.
2. **Read:** `GET /api/groups/{id}/schedule` returns the stored schedule (404 if none configured).
3. **On-connect pull:** When a controller connects, it publishes an empty message to `traffic/group/{id}/schedule-request`. The backend replies on `traffic/group/{id}/schedule` with the stored schedule, keeping the controller's cached copy (and its offline NVS fallback) in sync without periodic polling.
4. **Execution:** In `fixed` mode (default) the ESP-32 holds each turn for `durationSeconds`, advancing through the loop continuously, and broadcasts the active phase set over LoRa to the group's edge devices. In `dynamic` mode it instead runs the FIFO queue executor (all-RED idle, GREEN on demand per `dynamicConfig`) — see [Traffic Controller](traffic-controller.md).

### Scheduling Mode

Each group has a `schedulingMode` (`"fixed"` default | `"dynamic"`) stored on the group document alongside the turns-loop `schedule` and the `dynamicConfig {greenSec, allRedSec}` timings. The mode travels inside the `traffic/group/{groupId}/schedule` payload so the controller can switch executors without reflashing; offline it always falls back to the cached fixed turns loop.

The group document also carries `enqueueEdgeDetections` (`bool`, default `true`). It is backend-only configuration for the queue gate: when `false`, new detections are still stored but never enqueued, and an admin `forceEnqueue` request is the only way to queue one. It is **not** part of the MQTT schedule payload, so the controller does not need to know it.
