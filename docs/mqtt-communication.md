# MQTT Communication

MQTT provides the real-time communication channel between the backend and edge devices over the Mosquitto broker, enabling bidirectional messaging for telemetry, commands, and synchronization.

## Broker

The VC-LANE server uses **Mosquitto** as its MQTT broker, running inside the Docker Compose stack.

- Internal port: `1883` (no TLS — backend and broker are on the same Docker network)
- External port (when enabled): `8883` (TLS, for edge devices)
- Authentication: password file managed via `mosquitto_passwd`

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
| Device → Backend | `traffic/device/{id}/telemetry` | Edge device | Backend (→RTDB) | Arbitrary JSON |
| Device → Backend | `traffic/device/{id}/heartbeat` | Edge device | Backend (→RTDB) | Arbitrary JSON |
| Device → Backend | `traffic/device/{id}/acknowledgements` | Edge device | Backend (logged) | Arbitrary JSON |
| Device → Backend | `traffic/group/{id}/prepare-response` | Edge device | Backend (unhandled) | Arbitrary JSON |
| Backend → Device | `traffic/device/{id}/commands` | Backend | Edge device | `{"command": str, "payload": {}}` |
| Backend → Devices | `traffic/group/{id}/prepare` | Backend | Edge devices in group | `{"deviceIds": [str]}` |

## Mosquitto ACL

The ACL file controls topic-level access for each connecting client:

```
user vclane-backend
  pattern rw traffic/device/+/#
  pattern rw traffic/group/+/#

pattern read write traffic/device/%u/#
pattern read write traffic/group/+/#
```

- `vclane-backend` (the backend's internal MQTT client) has read/write on all device and group topics.
- `%u` expands to the connecting device's username (= `deviceId`), scoping each device to its own topics.
- All devices can read/write group topics for synchronization.

## Security

- MQTT over TLS on the external listener (port 8883) for edge device connections
- Username/password authentication via `mosquitto_passwd` file
- Topic-based access control via ACL
- Devices are provisioned with a unique device secret used as both MQTT and RTSP password

## Technology Stack

| Component      | Technology                         |
| -------------- | ---------------------------------- |
| Protocol       | MQTT v5                            |
| Encryption     | TLS (external)                     |
| MQTT Broker    | Mosquitto                          |
| Authentication | Password file + `mosquitto_passwd` |
| Authorization  | Topic-based ACL                    |
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
