# MQTT Communication Layer

MQTT provides the real-time communication channel between the backend and edge devices, enabling bidirectional messaging.

## Responsibilities

- Deliver commands to edge devices
- Receive telemetry
- Receive heartbeat updates
- Handle acknowledgements
- Maintain persistent connections

## Recommended Brokers

- EMQX
- Mosquitto

## Security

- MQTT over TLS
- Device authentication
- Topic authorization

## MQTT Topic Structure

```text
traffic/
    device/
        {deviceId}/
            telemetry
            heartbeat
            status
            commands
            acknowledgements

    group/
        {groupId}/
            prepare
            prepare-response
            commit
            abort
```

## Technology Stack

| Component      | Technology                                  |
| -------------- | ------------------------------------------- |
| Protocol       | MQTT v5                                     |
| Encryption     | TLS                                         |
| MQTT Broker    | EMQX / Mosquitto                            |
| Authentication | X.509 Certificate / Username Authentication |
| Authorization  | Topic-Based Access Control                  |
| Message Format | JSON                                        |

## Key Features

**Telemetry Communication** — Edge devices publish traffic statistics, device health information, sensor data, and AI analysis results.

```json
{
  "deviceId": "intersection-01",
  "vehicleCount": 35,
  "trafficDensity": "HIGH",
  "timestamp": "2026-07-24T10:00:00Z"
}
```

**Device Control** — Backend can send traffic light commands, configuration updates, restart commands, and AI model updates.

**Device Monitoring** — Online/offline detection, heartbeat tracking, connection status reporting.

> **Note on Last-Seen:** The "last seen" timestamp is no longer stored in Firestore. Instead, it is derived from the `timestamp` field in RTDB sensor data (`sensors/{deviceId}`), which is updated whenever the edge device publishes a telemetry payload. This avoids excessive Firestore writes from frequent heartbeat updates.

**Traffic Synchronization** — Multi-intersection synchronization via group topics:

- `traffic/group/{groupId}/prepare`
- `traffic/group/{groupId}/prepare-response`
- `traffic/group/{groupId}/commit`
- `traffic/group/{groupId}/abort`

Features coordinated intersection control with reliable command delivery.