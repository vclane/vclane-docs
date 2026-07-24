# VC-LANE — Overview

**VC-LANE** (Adaptive Traffic Monitoring System) is a distributed intelligent traffic monitoring and management platform designed to provide real-time traffic observation, live video streaming, AI-based traffic analysis, device monitoring, and coordinated traffic signal control across multiple intersections.

The system follows an **edge computing architecture**, where every traffic intersection operates as an independent AI-enabled edge device.

Each edge device consists of a Raspberry Pi equipped with a camera and capable of:

- Video capture and streaming
- AI-based object detection
- Local traffic analysis
- Device monitoring
- Bidirectional communication with the backend
- Local traffic control decision-making

The system enables authenticated users through a Flutter mobile application to:

- View live traffic camera streams
- Monitor intersection status
- View real-time traffic information
- Monitor connected edge devices
- Trigger traffic signal overrides
- Manage groups of intersections
- Perform coordinated traffic control operations

Each Raspberry Pi edge device communicates with the backend using **MQTT over TLS**, providing low-latency bidirectional communication for:

- Telemetry
- Device heartbeat
- Remote commands
- Command acknowledgements
- Traffic synchronization