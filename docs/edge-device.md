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

## Local Decision Making

The Raspberry Pi can perform immediate actions without waiting for the backend.

Examples:

- Detect abnormal traffic conditions
- Apply fallback traffic rules
- Continue operation during temporary network outages
- Execute previously received traffic schedules

## Backend Communication

The edge device maintains a persistent MQTT connection.

Responsibilities:

- Publish telemetry
- Send heartbeat updates
- Report device health
- Receive commands
- Send execution acknowledgements

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

Each edge device periodically reports:

- Online/offline state
- CPU usage
- Memory usage
- Storage availability
- Network connectivity
- Camera status
- AI processing status

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
| MQTT Communication          | Eclipse Paho MQTT Client       |
| Communication Protocol      | MQTT over TLS                  |
| Video Streaming             | RTSP Publisher                 |
| System Monitoring           | psutil / Linux Metrics         |
| Containerization (Optional) | Docker                         |

## Key Features

**AI-Based Traffic Analysis** — Vehicle detection, pedestrian detection, traffic density estimation, lane occupancy detection, queue length estimation, congestion detection.

**Local Edge Processing** — Performs AI inference locally, reduces bandwidth consumption, provides faster response time, minimizes cloud processing requirements.

**Local Decision Making** — Executes traffic rules during network interruptions, performs fallback traffic control, maintains operation during backend downtime.

**Device Monitoring** — CPU utilization, memory usage, storage availability, network connectivity, camera status, AI processing status.

**Secure Backend Communication** — Device heartbeat reporting, telemetry publishing, remote command execution, command acknowledgements.