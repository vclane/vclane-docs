# MQTT Communication

MQTT provides the real-time communication channel between the backend, the ESP-32 Traffic Controllers, and the Raspberry Pi Edge Devices, enabling bidirectional messaging.

## Responsibilities

- Deliver timing schedules and manual overrides to ESP-32 Traffic Controllers
- Receive AI traffic telemetry from Raspberry Pi Edge Devices (vehicle counts, congestion status, active traffic light phase)
- Receive fallback traffic light phase telemetry from ESP-32 Traffic Controllers when edge devices are offline
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
            commands        # Subscribe to manual override and cycle signals
            schedule        # Subscribe to daily schedule configurations
            telemetry       # Publish fallback traffic light active phases
            heartbeat       # Publish controller heartbeat
            status          # LWT online/offline
```

## Technology Stack

| Component      | Technology                        |
| -------------- | --------------------------------- |
| Protocol       | MQTT v5                           |
| Encryption     | TLS (external)                    |
| MQTT Broker    | Mosquitto                         |
| Authentication | Mosquitto Dynamic Security Plugin |
| Authorization  | Role-based ACL (Dynamic Security) |
| Message Format | JSON                              |

## Key Features

**Edge Telemetry Communication** — Raspberry Pi Edge Devices publish traffic statistics and AI analysis results.

```json
{
  "deviceId": "intersection-01",
  "vehicleCount": 35,
  "trafficDensity": "HIGH",
  "currentTrafficLightSignal": "GREEN",
  "timestamp": "2026-07-24T10:00:00Z"
}
```

**Fallback Telemetry Communication** — The ESP-32 Traffic Controller publishes the current state of traffic lights in its group as a fallback when edge devices lose connectivity.

```json
{
  "groupId": "group-01",
  "activePhases": [
    { "intersectionId": "intersection-01", "phase": "NORTH_SOUTH_GREEN" },
    { "intersectionId": "intersection-02", "phase": "EAST_WEST_RED" }
  ],
  "isFallback": true,
  "timestamp": "2026-07-24T10:00:05Z"
}
```

**Controller Command & Schedule Delivery** — The backend publishes traffic schedule configurations and manual overrides directly to the group's ESP-32 controller.

**Device Monitoring** — Online/offline tracking for all hardware via LWT (Last Will and Testament) topics and connection status reporting.

**Traffic Synchronization** — Coordinated intersection timing is managed locally within each group by the ESP-32 Traffic Controller via LoRa radio broadcasts, eliminating the need for complex, latency-sensitive multi-device MQTT coordination protocols.
