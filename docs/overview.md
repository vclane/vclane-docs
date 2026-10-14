# VCLane — Overview

**VCLane** (Adaptive Traffic Monitoring System) is a distributed intelligent traffic monitoring and management platform designed to provide real-time traffic observation, live video streaming, AI-based traffic analysis, device monitoring, and coordinated traffic signal control across multiple intersections.

The system follows an **edge computing architecture**, where traffic intersections are organized into **intersection groups**. Each group functions as a resilient local network consisting of a dedicated **ESP-32 Traffic Controller** and multiple **Raspberry Pi Edge Devices**.

### Component Roles within an Intersection Group

- **ESP-32 Traffic Controller:** Acts as the local coordinator for the group. It connects to the backend over WiFi (MQTT) to receive traffic light schedules, synchronization rules, and manual override signals. It runs a scheduling loop — fixed turns-loop in `fixed` mode, FIFO queue-based on-demand with all-RED idle in `dynamic` mode — and broadcasts the current target traffic light states to all edge devices in its group using **LoRa (Long Range) radio wireless broadcasts**.
- **Raspberry Pi Edge Devices:** Co-located at each intersection within the group, equipped with a camera and physical relays connected to the traffic lights. They perform real-time video capture, AI object detection, and per-vehicle tracking (no aggregate counting). They receive the LoRa signal broadcasts from the ESP-32 and actuate the physical traffic light relays.

### Local Communication & Offline Resilience

- **Offline Coordinated Signaling:** If the internet connection is lost, the ESP-32 Traffic Controller enters an offline fallback mode. It runs the cached fixed turns loop using its internal real-time clock (RTC) — even for groups configured as `dynamic`, since queueing needs the server — and broadcasts phase changes to the edge devices via LoRa. This ensures that the traffic lights continue to operate in a synchronized, coordinated manner without any internet dependency.
- **Telemetry:** The Raspberry Pi Edge Devices are the sole source of traffic telemetry, publishing a lean status snapshot (active signal + emergency-vehicle flag) directly to the backend via MQTT. If an edge device loses connectivity it is reported offline; the ESP-32 Traffic Controller does not publish telemetry on its behalf.

The system enables authenticated users through a Flutter mobile application to:

- View live traffic camera streams
- Monitor intersection group status and device health
- View real-time traffic information (signal + emergency status)
- Trigger traffic signal overrides targeting the group controller
- Co-ordinate traffic operations across groups of intersections

Each connected device communicates with the backend using **MQTT over TLS**, providing low-latency bidirectional communication for:

- Telemetry publishing (traffic signal + emergency status from Raspberry Pi Edge Devices)
- Device heartbeats and online/offline status reporting
- Remote schedule configurations and manual override commands
- Command execution acknowledgements
- Group traffic synchronization and status updates
