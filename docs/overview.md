# VCLane — Overview

**VCLane** (Adaptive Traffic Monitoring System) is a distributed intelligent traffic monitoring and management platform designed to provide real-time traffic observation, live video streaming, AI-based traffic analysis, device monitoring, and coordinated traffic signal control across multiple intersections.

The system follows an **edge computing architecture**, where traffic intersections are organized into **intersection groups**. Each group functions as a resilient local network consisting of a dedicated **ESP-32 Traffic Controller** and multiple **Raspberry Pi Edge Devices**.

### Component Roles within an Intersection Group

- **ESP-32 Traffic Controller:** Acts as the local coordinator for the group. It connects to the backend over WiFi (MQTT) to receive traffic light schedules, synchronization rules, and manual override signals. It runs a scheduling loop and broadcasts the current target traffic light states to all edge devices in its group using **LoRa (Long Range) radio wireless broadcasts**.
- **Raspberry Pi Edge Devices:** Co-located at each intersection within the group, equipped with a camera and physical relays connected to the traffic lights. They perform real-time video capture, AI object detection, and local traffic density analysis. They receive the LoRa signal broadcasts from the ESP-32 and actuate the physical traffic light relays.

### Local Communication & Offline Resilience

- **Offline Coordinated Signaling:** If the internet connection is lost, the ESP-32 Traffic Controller enters an offline fallback mode. It continues executing coordination schedules using its internal real-time clock (RTC) and broadcasting phase changes to the edge devices via LoRa. This ensures that the traffic lights continue to operate in a synchronized, coordinated manner without any internet dependency.
- **Fallback Telemetry:** Under normal conditions, the Raspberry Pi Edge Devices publish real-time traffic statistics (vehicle counts, lane occupancy) directly to the backend via MQTT. If a Raspberry Pi loses internet connectivity but the group's ESP-32 Traffic Controller remains online, the ESP-32 publishes the active traffic light signal states for the group to the backend as a fallback.

The system enables authenticated users through a Flutter mobile application to:

- View live traffic camera streams
- Monitor intersection group status and device health
- View real-time traffic information and fallback signal states
- Trigger traffic signal overrides targeting the group controller
- Co-ordinate traffic operations across groups of intersections

Each connected device communicates with the backend using **MQTT over TLS**, providing low-latency bidirectional communication for:

- Telemetry publishing (traffic stats from Raspberry Pis, fallback signal states from ESP-32s)
- Device heartbeats and online/offline status reporting
- Remote schedule configurations and manual override commands
- Command execution acknowledgements
- Group traffic synchronization and status updates
