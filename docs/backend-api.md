# Backend API

The backend acts as the centralized management and coordination layer between users, edge devices, and cloud services.

## Architecture

```mermaid
graph TB
    subgraph Server["VCLane Server (Docker Compose)"]
        Caddy["Reverse Proxy — Caddy<br/>(TLS termination)"]
        Backend["Backend API — FastAPI<br/>(Auth / Commands / Telemetry / LiveKit tokens)"]
        MQTT["MQTT Broker — Mosquitto<br/>(TLS + password auth)"]

        Caddy -->|"/api/* /ws/* /health"| Backend
        Backend -->|"join tokens + webhooks"| LiveKit["LiveKit Cloud<br/>(WHIP in / WebRTC out)"]
    end

    Backend -->|"Auth / DB"| Firebase["Firebase Services<br/>(Auth / Firestore / RTDB)"]
    MQTT -->|"Commands / Sync / Publish control"| Edge["Raspberry Pi Edge Devices"]
    Edge -->|"Telemetry / Heartbeat / Ack"| MQTT
    Edge -->|"WHIP Stream (annotated, on demand)"| LiveKit
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
| Media Server         | LiveKit Cloud (WHIP / WebRTC)            |
| Reverse Proxy        | Caddy                               |
| API Documentation    | OpenAPI / Swagger                   |
| Containerization     | Docker + Docker Compose             |
| Deployment           | Linux Server / Cloud Infrastructure |

## Key Features

**Device Management** — Register edge devices, manage device configurations, track device status, monitor device health, assign devices to intersections.

**Traffic Monitoring** — Receive real-time telemetry, monitor traffic conditions, store traffic information, generate traffic statistics, detect traffic events.

**Traffic Control** — Send traffic signal commands, trigger manual overrides, manage intersection schedules (turns loop + per-group `schedulingMode: fixed | dynamic`), coordinate multiple intersections, process command responses. In `dynamic` mode the backend runs a queue gate on new `POST /api/devices/{id}/detections` docs (vehicle allowlist + staleness guard → `enqueue` when RED + not already queued) and clears RTDB `has_queued_schedule` on controller queue-completion.

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
| `POST`   | `/api/devices/{id}/detections` | JWT | Submit one per-vehicle detection (multipart `trackId`, `class`, `deviceTimestamp`, `image` required) → Storage upload + Firestore `detections` write |
| `GET`    | `/api/groups`                | JWT  | List all groups                              |
| `POST`   | `/api/groups`                | JWT  | Create group                                 |
| `GET`    | `/api/groups/{id}`           | JWT  | Get group details                            |
| `PATCH`  | `/api/groups/{id}`           | JWT  | Update group                                 |
| `DELETE` | `/api/groups/{id}`           | JWT  | Delete group                                 |
| `POST`   | `/api/groups/{id}/sync`      | JWT  | Trigger group prepare                        |
| `POST`   | `/api/groups/{id}/schedule`  | JWT  | Publish + persist group schedule             |
| `GET`    | `/api/groups/{id}/schedule`  | JWT  | Get the stored group schedule                |
| `POST`   | `/api/groups/{id}/commands`  | JWT  | Send command to the group controller         |
| `POST`   | `/api/devices/{id}/livekit-token` | JWT | Mint LiveKit join token (`room=deviceId`)  |
| `POST`   | `/api/livekit/webhook`       | LiveKit | LiveKit webhook (join/leave → `start/stop_publish`) |
| `GET`    | `/api/mediamtx/auth`         | None | MediaMTX auth callback (deprecated since 2.11.0, kept for backward compat) |

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
| `image` | `file (JPEG)` | Yes | Full annotated frame — rejected if missing |

Backend flow:

1. Authenticate via operator JWT.
2. Validate all parts present and `image` non-empty — else `400`.
3. Resolve `groupId` from `devices/{id}.groupId`.
4. Compute doc ID `{deviceId}_{trackId}_{deviceTimestamp}`; if it exists, return `200` with `deduped: true` (no rewrite).
5. Upload image to Storage (`detections/{deviceId}/{trackId}_{deviceTimestamp}.jpg`), write the `detections` doc with server `timestamp`, return `201` with the doc ID.

Detections bypass Mosquitto entirely and never touch RTDB.

> Dynamic queue gate (per-group `schedulingMode: "dynamic"`) triggers here on new docs only (not `deduped:true`): vehicle allowlist + staleness guard (`deviceTimestamp` older than ~30 s still stores but skips queue) + device RED + `has_queued_schedule == false` → `enqueue` to `traffic/group/{gid}/commands`. See Server Integration.

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
