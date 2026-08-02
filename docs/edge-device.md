# Raspberry Pi Edge Device

Each intersection contains an independent Raspberry Pi-based edge device acting as the intelligent edge computing unit. It performs video processing, AI inference, traffic analysis, device communication, and local decision-making.

## Video Processing

The edge device handles all camera-related operations.

Responsibilities:

- Capture live camera feeds
- Process frames using OpenCV
- Perform object detection using YOLO
- Detect vehicles and pedestrians
- Calculate traffic density
- Generate traffic statistics
- Publish live video streams

Data flow:

``` mermaid
graph TD
    Camera --> OpenCV
    OpenCV --> YOLO["YOLO Detection"]
    YOLO --> Analytics["Traffic Analytics"]
    YOLO --> RTSP["RTSP Stream"]
```

## AI-Based Traffic Analysis

The edge device performs local AI inference instead of sending raw video to the backend.

Advantages:

- Lower bandwidth usage
- Lower latency
- Faster response time
- Reduced cloud processing cost

Examples:

- Vehicle counting
- Lane occupancy detection
- Queue length estimation
- Traffic congestion detection

## Local Decision Making and Actuation

Under the grouped architecture, the Raspberry Pi Edge Device functions as the physical actuator for the traffic lights at its intersection. Rather than executing a standalone schedule or receiving direct signal overrides from the backend, it delegates coordination to the group's ESP-32 Traffic Controller.

Key behaviors:
- **LoRa Packet Reception:** The Raspberry Pi continuously listens for LoRa broadcast signals containing the target active phases.
- **Relay Actuation:** It parses the received command (e.g., active phase, green-yellow-red light configurations) and controls the physical relays to update the signal lamps.
- **Resilient Fallback:** If the LoRa broadcast signal is lost (e.g., ESP-32 hardware failure), the Raspberry Pi falls back to a pre-programmed local safety routine (such as flashing yellow for caution) to prevent unsafe signal states.

## Backend Communication

The edge device maintains a persistent MQTT connection to the backend when internet is available.

Responsibilities:

- Publish AI traffic telemetry (vehicle counts, lane occupancy, congestion levels)
- Send heartbeat updates and connection status
- Report device health and camera status

Example telemetry:

```json
{
  "deviceId": "intersection-01",
  "vehicleCount": 35,
  "trafficDensity": "HIGH",
  "timestamp": "2026-07-24T10:00:00Z"
}
```

## Device Monitoring

Each edge device periodically reports its system and operational status to the backend (excluding CPU/Memory metrics):

- Online/offline state
- Storage availability
- Network connectivity
- Camera status
- AI processing status
- LoRa receiver status (signal strength, packet loss)

## Technology Stack

| Component                   | Technology                     |
| --------------------------- | ------------------------------ |
| Hardware                    | Raspberry Pi                   |
| Operating System            | Raspberry Pi OS / Linux        |
| Programming Language        | Python                         |
| Video Processing            | OpenCV                         |
| AI Detection Model          | YOLO Object Detection          |
| AI Runtime                  | TensorFlow Lite / ONNX Runtime |
| Camera Interface            | CSI Camera / USB Camera        |
| Local Wireless Receiver     | LoRa SX1276 / SX1278 Module    |
| Local Control Protocol      | SPI (Pi to LoRa) / LoRa Radio  |
| Actuator Interface          | GPIO-controlled Relay Board    |
| MQTT Communication          | Eclipse Paho MQTT Client       |
| Communication Protocol      | MQTT over TLS (to Backend)     |
| Video Streaming             | RTSP Publisher                 |
| Containerization (Optional) | Docker                         |

## Key Features

**AI-Based Traffic Analysis** — Vehicle detection, pedestrian detection, traffic density estimation, lane occupancy detection, queue length estimation, congestion detection.

**Local Edge Processing** — Performs AI inference locally, reduces bandwidth consumption, provides faster response time, minimizes cloud processing requirements.

**LoRa-Driven Signal Actuation** — Receives synchronized traffic light phase commands from the group's ESP-32 Traffic Controller via LoRa, driving the physical light relays.

**Fault Tolerant Safety Fallback** — Reverts to flashing yellow/local caution mode if LoRa control signals are lost or corrupted, ensuring driver safety.

**Device Monitoring** — Storage availability, network connectivity, camera status, AI processing status, and LoRa receiver signal quality.

**Secure Backend Communication** — Device heartbeat reporting, telemetry publishing, remote configuration updates.