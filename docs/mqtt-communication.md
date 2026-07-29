# MQTT Communication

MQTT provides the real-time communication channel between the backend and edge devices over the Mosquitto broker, enabling bidirectional messaging for telemetry, commands, and synchronization.

## Broker

The VC-LANE server uses **Mosquitto** as its MQTT broker, running inside the Docker Compose stack.

- Internal port: `1883` (no TLS — backend and broker are on the same Docker network)
- External port (when enabled): `8883` (TLS, for edge devices)
- Authentication: Mosquitto Dynamic Security Plugin

## Topic Structure

```text
traffic/
    device/
        {deviceId}/
            telemetry
            heartbeat
            acknowledgements
            commands

    group/
        {groupId}/
            prepare
            prepare-response
```

## Topic Reference

| Direction | Topic | Publisher | Consumer | Payload |
|---|---|---|---|---|
| Device → Backend | `traffic/device/{id}/telemetry` | Edge device | Backend (→RTDB) | Arbitrary JSON — `online: true` and `lastSeen` injected by backend |
| Device → Backend | `traffic/device/{id}/heartbeat` | Edge device | Backend (→RTDB) | Same as telemetry |
| Device → Backend | `traffic/device/{id}/goodbye` | Edge device (or LWT) | Backend (→RTDB) | Arbitrary JSON — sets `online: false` |
| Device → Backend | `traffic/device/{id}/acknowledgements` | Edge device | Backend (logged) | Arbitrary JSON |
| Device → Backend | `traffic/group/{id}/prepare-response` | Edge device | Backend (unhandled) | Arbitrary JSON |
| Backend → Device | `traffic/device/{id}/commands` | Backend | Edge device | `{"command": str, "payload": {}}` |
| Backend → Devices | `traffic/group/{id}/prepare` | Backend | Edge devices in group | `{"deviceIds": [str]}` |

The `online` field in RTDB at `/telemetry/{deviceId}` is set to `true` on every heartbeat or telemetry. It is set to `false` by a goodbye message (graceful shutdown), a Last Will & Testament (crash), or a background timeout checker (stale detection).

## Mosquitto Dynamic Security

Mosquitto uses the **Dynamic Security Plugin**, which manages clients, roles, and ACLs via MQTT RPC on `$CONTROL/dynamic-security/v1`.

### RPC Format

Commands must be wrapped in a `commands` array:

```json
{
  "commands": [
    {
      "command": "createClient",
      "username": "intersection-01",
      "password": "vcl-abc123...",
      "textname": "intersection-01"
    }
  ]
}
```

A top-level `"command"` key (without the `commands` array) causes an `"Unknown command"` error on Mosquitto 2.1.2. Success responses omit both `"success"` and `"error"` keys — failure responses include an `"error"` key.

### Roles

Two roles are defined at image build time:

- **`admin`** — assigned to `vclane-backend`, grants read/write on all `traffic/device/+/#` and `traffic/group/+/#` topics plus CONTROL topics.
- **`device`** — applied per device, grants read/write on `traffic/device/%u/#` and `traffic/group/+/#` topics (`%u` expands to the connecting device's username).

### Provisioning

When a new device is provisioned via `POST /api/devices`, the backend sends **two** RPCs:
1. `createClient` — registers the MQTT client with username/password
2. `addClientRole` — assigns the `device` role

On deprovisioning (`DELETE /api/devices/{id}`), a `deleteClient` RPC removes the client and its role assignments.

## Security

- MQTT over TLS on the external listener (port 8883) for edge device connections
- Username/password authentication via Mosquitto Dynamic Security Plugin
- Default ACL set to `deny`; role-based ACLs grant only required topic access
- Devices are provisioned with a unique device secret used as both MQTT and RTSP password

## Technology Stack

| Component      | Technology                         |
| -------------- | ---------------------------------- |
| Protocol       | MQTT v5                            |
| Encryption     | TLS (external)                     |
| MQTT Broker    | Mosquitto                          |
| Authentication | Mosquitto Dynamic Security Plugin |
| Authorization  | Role-based ACL (Dynamic Security) |
| Message Format | JSON                               |

## Key Features

**Telemetry Communication** — Edge devices publish traffic statistics, device health information, telemetry, and AI analysis results.

```json
{
  "deviceId": "intersection-01",
  "vehicleCount": 35,
  "trafficDensity": "HIGH",
  "timestamp": "2026-07-24T10:00:00Z"
}
```

**Device Control** — Backend can send traffic light commands, configuration updates, restart commands, and AI model updates via MQTT publish.

**Device Monitoring** — Online/offline detection, heartbeat tracking, connection status reporting.

**Traffic Synchronization** — Multi-intersection synchronization via group topics:
- `traffic/group/{groupId}/prepare`
- `traffic/group/{groupId}/prepare-response`

See [Server Integration](server-integration.md) for end-to-end authentication flows and device provisioning details.
