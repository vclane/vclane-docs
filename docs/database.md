# Database

VCLane uses two Firebase database services: **Firestore** for persistent structured data and **Realtime Database (RTDB)** for real-time device telemetry and status.

## Firestore

Firestore stores long-lived documents in collections. It is used for device metadata, user profiles, groups, signal overrides, and per-vehicle detections — data that changes infrequently (detections are append-only history written via REST).

### Collections

#### `devices/{deviceId}`

| Field              | Type             | Description                                         |
| ------------------ | ---------------- | --------------------------------------------------- |
| `name`             | `string`         | Human-readable intersection name                    |
| `location`         | `string`         | Intersection location description                   |
| `status`           | `string`         | Static seed-time status (`"online"` or `"offline"`) |
| `firmwareVersion`  | `string`         | Edge device firmware version                        |
| `rtspUrl`          | `string`         | RTSP stream URL for video                           |
| `deviceSecretHash` | `string`         | bcrypt hash of the device secret                    |
| `groupId`          | `string \| null` | Assigned group, if any                              |
| `createdAt`        | `Timestamp`      | Document creation time                              |
| `updatedAt`        | `Timestamp`      | Last document update time                           |

> **Note:** The `status` field in Firestore is set at provisioning/seed time and is **not** updated by runtime heartbeat or telemetry. Real-time online/offline state is tracked in RTDB (see below).

#### `groups/{groupId}`

| Field            | Type        | Description                                                              |
| ---------------- | ----------- | ------------------------------------------------------------------------ |
| `name`           | `string`    | Group display name                                                       |
| `description`    | `string`    | Group description                                                        |
| `location`       | `string`    | Group location                                                           |
| `deviceIds`      | `string[]`  | List of device IDs in the group                                          |
| `schedule`       | `map`       | Repeating turns loop `{turns: [{durationSeconds, phases}]}` (see Backend API) |
| `schedulingMode` | `string`    | Per-group executor: `"fixed"` (turns loop, default) or `"dynamic"` (FIFO queue) |
| `dynamicConfig`  | `map`       | Dynamic queue timings: `{greenSec, allRedSec}` (2-channel GREEN/RED relays) |
| `createdAt`      | `Timestamp` | Document creation time                                                   |
| `updatedAt`      | `Timestamp` | Last document update time                                                |

> **Note:** `schedulingMode` is per-group. `fixed` runs the stored turns loop; `dynamic` runs queue-based on-demand scheduling (see [Traffic Controller](traffic-controller.md)). The mode and timings are delivered to the controller via `traffic/group/{groupId}/schedule` and cached in NVS.

#### `users/{uid}`

| Field         | Type        | Description                   |
| ------------- | ----------- | ----------------------------- |
| `email`       | `string`    | User email address            |
| `displayName` | `string`    | User display name             |
| `role`        | `string`    | User role (e.g. `"operator"`) |
| `createdAt`   | `Timestamp` | Document creation time        |
| `updatedAt`   | `Timestamp` | Last document update time     |

#### `signalOverrides/{overrideId}`

| Field          | Type        | Description                       |
| -------------- | ----------- | --------------------------------- |
| `deviceId`     | `string`    | Target device ID                  |
| `targetSignal` | `string`    | Requested signal state            |
| `requestedBy`  | `string`    | User who requested the override   |
| `status`       | `string`    | Override status (e.g. `"active"`) |
| `requestedAt`  | `Timestamp` | When the override was requested   |
| `expiresAt`    | `Timestamp` | When the override expires         |

#### `detections/{detectionId}`

One document per detected vehicle, written by the backend when the edge posts to `POST /api/devices/{id}/detections`. Doc ID is deterministic (`{deviceId}_{trackId}_{deviceTimestamp}`) so retries are idempotent and the server-side dedup key (`trackId` + `deviceTimestamp` + `deviceId`) is enforced via existence check.

| Field             | Type        | Description                                              |
| ----------------- | ----------- | -------------------------------------------------------- |
| `deviceId`        | `string`    | Source device ID (from REST path)                        |
| `groupId`         | `string \| null` | Owning group (backend lookup of `devices/{id}.groupId`) |
| `timestamp`       | `Timestamp` | Server write time — canonical key for "today" queries    |
| `deviceTimestamp` | `string \| int` | Edge crossing time passthrough (part of dedup key)   |
| `trackId`         | `int \| string` | ByteTrack ID (part of dedup key)                     |
| `class`           | `string`    | YOLO label, unfiltered (any class)                       |
| `imageUrl`        | `string`    | Firebase Storage download URL of the detection frame — **required** |

> **Note:** `imageUrl` is required — events without an image are rejected. Mobile queries this collection directly (no REST read path). Required composite indexes: `(deviceId, timestamp DESC)`, `(groupId, timestamp DESC)`. Security rules: signed-in operators may read; writes are backend-only (`write: false`, Admin SDK bypasses rules).

## Realtime Database (RTDB)

RTDB stores ephemeral, rapidly-changing data. It is used for the latest telemetry snapshot and device online status.

### Paths

#### `/telemetry/{deviceId}`

This path is updated on every heartbeat and telemetry message from the edge device. Only edge devices write here — the ESP-32 controller's group topic carries queue-completion notices only and does not populate this path. The payload is an **arbitrary JSON object** from the device, with four fields injected/maintained by the backend:

| Field                  | Type            | Source              | Description                                                                                       |
| ---------------------- | --------------- | ------------------- | ------------------------------------------------------------------------------------------------- |
| _(device fields)_      | _various_       | Device payload      | Device-specific data (currentTrafficLightSignal, emergencyVehicleDetected, emergencyClass, emergencyConfidence, etc. — no counts) |
| `timestamp`            | `string \| int` | Device payload      | Device-local timestamp                                                                            |
| `online`               | `bool`          | Injected by backend | `True` — set on every heartbeat, telemetry, or retained `status` online message; `False` — set by the retained `status` offline message or the device LWT |
| `lastSeen`             | `int`           | Injected by backend | Server Unix timestamp of the most recent heartbeat, telemetry, or `status` message (informational only — never used to derive `online`) |
| `has_queued_schedule`  | `bool`          | Backend queue gate  | `True` — device has a pending/ongoing queued GREEN in dynamic mode; `False` — default/idle        |
| `queuedAt`             | `int \| null`   | Backend queue gate  | Server Unix timestamp of the last `enqueue`, cleared (`null`) on `queue-done`                     |

> **Note:** `has_queued_schedule` / `queuedAt` are only used when the device's group has `schedulingMode: "dynamic"`. They are written **only by the backend** (set on `enqueue`, cleared on controller `queue-done`); edge devices never write them. They provide dedup: while `has_queued_schedule` is `true`, further detections for that device do not trigger another enqueue.

> **Note:** `vehicleToday` has been **removed** from `/telemetry/{deviceId}` and must not be written. "Vehicles detected today" is derived by counting `detections` documents with `timestamp >= startOfDay` — see `detections/{detectionId}` above.

Example document (tracking enabled, dynamically queued):

```json
{
  "deviceId": "intersection-01",
  "currentTrafficLightSignal": "RED",
  "emergencyVehicleDetected": false,
  "emergencyClass": "",
  "emergencyConfidence": 0.0,
  "timestamp": "2026-07-24T10:00:00Z",
  "online": true,
  "lastSeen": 1721815200,
  "has_queued_schedule": true,
  "queuedAt": 1721815201
}
```

### Offline detection

Edge devices publish their connection state to `traffic/device/{deviceId}/status` (QoS 1, retained):
`{"online": true}` on connect and `{"online": false}` on graceful shutdown. The backend mirrors that state into `/telemetry/{deviceId}.online`. A connection that drops without a clean DISCONNECT is covered by the Last Will & Testament, which is configured as a **retained** `{"online": false}` message on the same topic.

There is no background timeout checker — `online` is event-driven. `lastSeen` is still refreshed for display purposes but no longer drives the online flag.

| Mechanism                            | Trigger                                                                                              | Latency                                            |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| **Retained status (graceful)**       | Device publishes `{"online": false}` (retained) to `traffic/device/{id}/status` before disconnecting | Instant                                            |
| **Last Will & Testament**            | Broker publishes the device's pre-set retained will (`{"online": false}`) on ungraceful disconnect   | ≤ 1.5 × MQTT keepalive (~90 s at the 60 s default) |
| **Heartbeat / telemetry safety net** | Any telemetry or heartbeat sets `online: true` again                                                 | One publish interval (≤ 30 s)                      |

> **Note:** Because the status messages are retained, a backend restart immediately re-reads the last known state for every device instead of waiting for a timeout. Devices running firmware without LWT support have no offline transition of their own — they must be upgraded, or be marked offline explicitly (e.g. `goodbye`, or the controller offline cascade).

