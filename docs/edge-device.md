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

## Software Architecture

### Project Layout

```
vclane-edge/
├── src/vclane_edge/
│   ├── __init__.py          # Version
│   ├── cli.py               # Typer CLI (run, register, deregister)
│   ├── config.py            # VclaneSettings + load_settings()
│   ├── device.py            # VclaneEdgeDevice orchestrator
│   ├── mqtt.py              # MQTT client (publish/subscribe)
│   ├── video.py             # Video pipeline (3 modes)
│   ├── yolo.py              # YOLO detection wrapper
│   ├── telemetry.py         # Mock telemetry state
│   └── registration.py      # Provisioning flow
├── pyproject.toml            # uv-managed deps
├── Dockerfile                # Multi-stage container
└── docker-compose.yml        # Edge device service
```

### CLI Commands

The edge device exposes three commands via the `vclane-edge` entry point:

| Command | Description |
|---------|-------------|
| `vclane-edge run` | Run video pipeline, YOLO inference, MQTT telemetry |
| `vclane-edge register` | Provision device with backend API |
| `vclane-edge deregister` | Remove device from backend |

All options are documented inline with `--help`.

### Configuration

Settings are resolved from two sources (highest priority wins):

1. **JSON config file** — passed via `--config / -c` (e.g. `vclane-edge run --config config.json`)
2. **CLI flags** — `--device-id`, `--mqtt-host`, etc.

Example `config.json`:

```json
{
  "device_id": "intersection-01",
  "device_secret": "…from registration…",
  "mqtt_host": "10.0.0.5",
  "mqtt_port": 8883,
  "mqtt_tls": true,
  "api_url": "https://vclane.example.com"
}
```

### Video Pipeline Modes

| Mode | Description |
|------|-------------|
| `opencv+ffmpeg` (default) | OpenCV capture → YOLO detect → FFmpeg stdin → RTSP |
| `direct-ffmpeg` | Source → FFmpeg → RTSP (no OpenCV/YOLO, pure passthrough) |
| `opencv+gstreamer` | OpenCV capture → YOLO → GStreamer → RTSP (falls back to opencv+ffmpeg) |

### Registration Flow

1. Admin runs `vclane-edge register --api-url <url>`
2. Prompts for Firebase email/password (or sets `VCLANE_ADMIN_EMAIL` / `VCLANE_ADMIN_PASSWORD`)
3. `POST /api/auth/login` → JWT
4. `POST /api/devices` → receives `{deviceId, deviceSecret, rtspUrl}`
5. With `--out <path>`, credentials saved as JSON; otherwise printed to stdout

Auto-registration at `vclane-edge run` start occurs when `device_id` or `device_secret` are empty (or `force_register` is `true`). Registered credentials are used in-memory for the session — the config file is not modified.

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
| CLI Framework               | Typer                          |
| Config Management           | Pydantic                       |
| Package Manager             | uv                             |
| Containerization (Optional) | Docker                         |

## Key Features

**AI-Based Traffic Analysis** — Vehicle detection, pedestrian detection, traffic density estimation, lane occupancy detection, queue length estimation, congestion detection.

**Local Edge Processing** — Performs AI inference locally, reduces bandwidth consumption, provides faster response time, minimizes cloud processing requirements.

**Local Decision Making** — Executes traffic rules during network interruptions, performs fallback traffic control, maintains operation during backend downtime.

**Device Monitoring** — CPU utilization, memory usage, storage availability, network connectivity, camera status, AI processing status.

**Secure Backend Communication** — Device heartbeat reporting, telemetry publishing, remote command execution, command acknowledgements.