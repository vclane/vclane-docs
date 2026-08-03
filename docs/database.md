# Database

VCLane uses two Firebase database services: **Firestore** for persistent structured data and **Realtime Database (RTDB)** for real-time device telemetry and status.

## Firestore

Firestore stores long-lived documents in collections. It is used for device metadata, user profiles, groups, and signal overrides — data that changes infrequently.

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

| Field         | Type        | Description                     |
| ------------- | ----------- | ------------------------------- |
| `name`        | `string`    | Group display name              |
| `description` | `string`    | Group description               |
| `location`    | `string`    | Group location                  |
| `deviceIds`   | `string[]`  | List of device IDs in the group |
| `createdAt`   | `Timestamp` | Document creation time          |
| `updatedAt`   | `Timestamp` | Last document update time       |

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

## Realtime Database (RTDB)

RTDB stores ephemeral, rapidly-changing data. It is used for the latest telemetry snapshot and device online status.

### Paths

#### `/telemetry/{deviceId}`

This path is updated on every heartbeat and telemetry message from the edge device. The payload is an **arbitrary JSON object** from the device, with two fields injected by the backend:

| Field             | Type            | Source              | Description                                                                                       |
| ----------------- | --------------- | ------------------- | ------------------------------------------------------------------------------------------------- |
| _(device fields)_ | _various_       | Device payload      | Device-specific data (vehicleCount, trafficDensity, cpuUsage, etc.)                               |
| `timestamp`       | `string \| int` | Device payload      | Device-local timestamp                                                                            |
| `online`          | `bool`          | Injected by backend | `True` — set on every heartbeat or telemetry, `False` — set by goodbye message or timeout checker |
| `lastSeen`        | `int`           | Injected by backend | Server Unix timestamp of the most recent heartbeat or telemetry                                   |

Example document:

```json
{
  "deviceId": "intersection-01",
  "vehicleCount": 35,
  "trafficDensity": "HIGH",
  "timestamp": "2026-07-24T10:00:00Z",
  "online": true,
  "lastSeen": 1721815200
}
```

### Offline detection

Devices are marked offline through three mechanisms:

| Mechanism                 | Trigger                                                                                   | Latency   |
| ------------------------- | ----------------------------------------------------------------------------------------- | --------- |
| **Goodbye message**       | Device publishes to `traffic/device/{id}/goodbye` on graceful shutdown                    | Instant   |
| **Last Will & Testament** | Broker publishes device's pre-set LWT on ungraceful disconnect                            | Instant   |
| **Timeout checker**       | Backend background task runs every 30s, marks devices offline when `lastSeen` is >60s old | Up to 60s |
