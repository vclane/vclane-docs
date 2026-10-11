# Raspberry Pi Edge Device

Each intersection contains an independent Raspberry Pi-based edge device acting as the intelligent edge computing unit. It performs video processing, AI inference, traffic analysis, device communication, and local decision-making.

## Hardware Wiring

An intersection node connects three peripherals to the Raspberry Pi: the **LoRa receiver** (listens for phase broadcasts from the group's ESP-32 Traffic Controller), the **relay board** (drives the green/red signal lamps), and the **camera** (feeds the video/YOLO pipeline).

```mermaid
graph TD
    ESP32["ESP-32 Traffic Controller"] -->|"LoRa RF (group frequency)"| LORA["LoRa SX1278 Receiver"]
    CAM["Camera (CSI / USB)"] -->|"video frames"| PI["Raspberry Pi Edge Device<br/>(40-pin GPIO)"]
    LORA -->|"SPI"| PI
    PI -->|"GPIO"| RELAY["Relay Board<br/>(active-low, 2 channels)"]
    RELAY --> IN1["IN1 — GREEN"]
    RELAY --> IN2["IN2 — RED"]
```

### LoRa module (SX1276 / SX1278)

The LoRa module connects to the Raspberry Pi over SPI. The pins below match the `lora_cs_pin` / `lora_rst_pin` defaults.

| LoRa signal | Pi 40-pin header | BCM GPIO |
| ----------- | ---------------- | -------- |
| `3.3V`      | Pin 1            | —        |
| `GND`       | Any GND pin      | —        |
| `SCK`       | Pin 23           | 11       |
| `MOSI`      | Pin 19           | 10       |
| `MISO`      | Pin 21           | 9        |
| `CS` (NSS)  | Pin 22           | 25       |
| `RST`       | Pin 13           | 27       |

- Set `lora_frequency_mhz` to your region's band (433 / 868 / 915 MHz) and match the ESP-32 controller's spreading factor, bandwidth, and coding rate — otherwise packets will not decode.
- The driver uses the RadioHead-compatible sync word `0x12` (the RadioLib default on the ESP-32); the module's DIO0 pin is not required.

### Relay board

A 2-channel, active-low relay board drives the signal lamps. The pins below match the `relay_pins` defaults (`green`=17, `red`=22).

| Relay board | Signal lamp | Pi 40-pin header | BCM GPIO |
| ----------- | ----------- | ---------------- | -------- |
| `IN1`       | green       | Pin 11           | 17       |
| `IN2`       | red         | Pin 15           | 22       |
| `VCC`       | —           | Separate supply  | —        |
| `GND`       | —           | Any GND pin      | —        |

- Most relay boards are **active-low** (a LOW GPIO signal energizes the coil) — keep `relay_active_high: false`.
- Relays draw significant current: power the board from a dedicated 5 V / 12 V supply and connect only the signal pins to the Pi.

### Camera

- **USB camera**: plug into any USB port and set `video_source` to the device index (e.g. `"0"`).
- **Raspberry Pi Camera Module**: connect the ribbon cable to the CSI connector and enable the camera interface; set `video_source` to the camera device.

The default pins are chosen to avoid the SPI bus (GPIO 8–11) and the LoRa CS/RST pins (GPIO 25/27). If you rewire any peripheral, update `lora_cs_pin`, `lora_rst_pin`, or `relay_pins` to match.

## Video Processing

The edge device handles all camera-related operations.

Responsibilities:

- Capture live camera feeds
- Process frames using OpenCV
- Perform object detection using YOLO (detect-only) or YOLO + ByteTrack (tracking)
- Track vehicles across frames with persistent IDs (when tracking is enabled)
- Count vehicles via virtual counting lines (live occupancy + cumulative crossings)
- Calculate traffic density from live counts
- Generate traffic statistics
- Publish live annotated video streams (boxes + track IDs + counting lines)

Data flow (tracking enabled):

```mermaid
graph TD
    Camera --> OpenCV
    OpenCV --> TRACK["YOLO + ByteTrack (track, persist=True)"]
    TRACK --> Counter["VehicleCounter (live + line crossings)"]
    Counter --> Analytics["Traffic Analytics / Telemetry"]
    TRACK --> RTSP["RTSP Stream (annotated)"]
```

Data flow (detect-only, `tracker: null` / `--no-tracker`):

```mermaid
graph TD
    Camera --> OpenCV
    OpenCV --> YOLO["YOLO Detection (no IDs)"]
    YOLO --> Analytics["Traffic Analytics (mock telemetry)"]
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

- Vehicle counting (live occupancy + cumulative line crossings)
- Lane occupancy detection
- Queue length estimation
- Traffic congestion detection
- Per-direction and per-class breakdowns

### Vehicle Tracking (YOLO + ByteTrack)

Tracking is **opt-in** and disabled by default (`tracker: null`).

- **Detect-only (default):** `tracker: null` or `vclane-edge run --no-tracker` — plain `YOLO.detect()` per sampled frame, no IDs, no counting. Telemetry falls back to the legacy mock `random_walk()`.
- **Tracking:** set `tracker: "bytetrack.yaml"` (or `--tracker bytetrack.yaml`) — `YOLODetector.track()` calls `model.track(persist=True, tracker=...)`, preserving Kalman state across frames. Requires the `lap` dependency (`pip install lap` / `uv sync`).
- Annotated boxes show `{label} #{track_id} {conf:.2f}` (e.g. `car #7 0.82`); counting lines are drawn in cyan with their names.
- **Frame ordering matters:** tracking uses an ordered inference queue (`maxsize=1`, blocking `put(timeout=0.05)` instead of dropping middle frames). Detect-only uses `maxsize=2` and drops the oldest frame for throughput.
- **Keep `yolo_interval` at 1–2 when tracking.** Values `> 2` are forced back to `2` with a warning — skipping frames breaks the ByteTrack Kalman state and causes ID switches.
- **Stream gaps reset IDs:** on EOF / reconnect (`read_frame()` returns `None`), `reset_tracker()` clears ByteTrack state so stale tracks do not leak into the new sequence.
- Verify manually with `python scripts/test_tracking.py --source <video|0> [--save out.mp4]` (defaults: `--model models/best.pt --conf 0.4 --imgsz 416 --tracker bytetrack.yaml`).

CLI flags:

| Flag | Description |
| ---- | ----------- |
| `--tracker <yaml>` | Tracker config (e.g. `bytetrack.yaml`); empty string disables tracking |
| `--no-tracker` | Disable tracking (detect-only, no IDs/counting) |
| `--yolo-interval <N>` | Run YOLO every N frames (1–2 recommended for tracking) |

### Vehicle Counting (VehicleCounter)

`src/vclane_edge/counter.py` layers counting on top of ByteTrack IDs. It reports two complementary counts:

- **Live occupancy** — unique tracked IDs currently visible (drives `vehicleCount` and `trafficDensity` / congestion).
- **Cumulative line crossings** — each `(track_id, line)` pair counted at most once (drives `vehicleTotal` and `directionCounts`).

How it works:

- Counting lines are stored **normalized (0–1)** so they survive resolution changes, e.g. `{"name": "northbound", "p1": [0.05, 0.5], "p2": [0.95, 0.5]}`. Defaults to a single `northbound` mid-frame line; configure via `counting_lines`.
- A crossing registers when a track's **centroid changes signed side** of a directed line `p1 → p2` between consecutive updates (strict sign change; on-line `== 0` is ignored to avoid jitter counts).
- Only `count_classes` (default `["car", "truck", "bus", "motorcycle", "bicycle"]`) are counted — pedestrians and other labels are tracked/annotated but excluded from counts.
- Loitering / stop-and-go at a red light does not inflate totals: each ID counts once per line.
- Disappeared tracks have their centroid history pruned eagerly to bound memory (ByteTrack keeps them alive for `track_buffer` frames internally).
- Per-frame `by_class` in telemetry is the **live** per-class breakdown, not cumulative.

Density thresholds (applied to live count in `set_counts()`):

| Live count | Density |
| ---------- | ------- |
| `< 8` | `LOW` |
| `≤ 20` | `MODERATE` |
| `≤ 30` | `HIGH` |
| `> 30` | `HEAVY` |

## Local Decision Making and Actuation

Under the grouped architecture, the Raspberry Pi Edge Device functions as the physical actuator for the traffic lights at its intersection. Rather than executing a standalone schedule or receiving direct signal overrides from the backend, it delegates coordination to the group's ESP-32 Traffic Controller.

Key behaviors:

- **LoRa Packet Reception:** The Raspberry Pi continuously listens for LoRa broadcast signals containing the target active phases.
- **Relay Actuation:** It parses the received command (e.g., active phase, green/red light configurations) and controls the physical relays to update the signal lamps.
- **Resilient Fallback:** If the LoRa broadcast signal is lost (e.g., ESP-32 hardware failure), the Raspberry Pi falls back to a pre-programmed local safety routine (such as all relays off for caution) to prevent unsafe signal states.

## Backend Communication

The edge device maintains a persistent MQTT connection to the backend when internet is available.

Responsibilities:

- Publish AI traffic telemetry (live count, cumulative total, per-direction / per-class breakdowns, traffic density)
- Report the currently actuated traffic light phase (from LoRa) or caution state
- Send heartbeat updates (process uptime and timestamp)

Example telemetry (tracking enabled):

```json
{
  "deviceId": "intersection-01",
  "vehicleCount": 12,
  "vehicleTotal": 148,
  "directionCounts": { "northbound": 148 },
  "classCounts": { "car": 9, "truck": 2, "bus": 1 },
  "trafficDensity": "MODERATE",
  "currentTrafficLightSignal": "GREEN",
  "timestamp": "2026-07-24T10:00:00Z"
}
```

Detect-only mode (no tracker) omits real counts and falls back to the mock `vehicleCount` random walk; `vehicleTotal` stays `0` with empty `directionCounts` / `classCounts`.

## Device Monitoring

Each edge device periodically reports its status to the backend over MQTT:

- **Telemetry** (`traffic/device/{deviceId}/telemetry`): live vehicle count, cumulative total, per-direction counts, per-class counts, traffic density, and the active traffic light signal (or `CAUTION` during fallback)
- **Heartbeat** (`traffic/device/{deviceId}/heartbeat`): process uptime and timestamp

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
│   ├── yolo.py              # YOLO detection + ByteTrack wrapper (track/reset_tracker/annotate)
│   ├── counter.py           # VehicleCounter + CountingLine (live + line-crossing counts)
│   ├── telemetry.py         # Telemetry state (set_counts / random_walk fallback)
│   ├── lora.py              # LoRa receiver (ESP-32 phase broadcasts)
│   ├── relay.py             # Relay actuation + caution fallback
│   └── registration.py      # Provisioning flow
├── scripts/
│   └── test_tracking.py     # Manual ByteTrack verification (--source <video|0> [--save out.mp4])
├── pyproject.toml            # uv-managed deps (includes `lap` for ByteTrack association)
├── Dockerfile                # Multi-stage container
└── docker-compose.yml        # Edge device service
```

### CLI Commands

The edge device exposes three commands via the `vclane-edge` entry point:

| Command                  | Description                                        |
| ------------------------ | -------------------------------------------------- |
| `vclane-edge run`        | Run video pipeline, YOLO inference, MQTT telemetry |
| `vclane-edge register`   | Provision device with backend API                  |
| `vclane-edge deregister` | Remove device from backend                         |

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
  "api_url": "https://vclane.example.com",
  "tracker": "bytetrack.yaml",
  "yolo_interval": 1,
  "counting_lines": [
    { "name": "northbound", "p1": [0.05, 0.5], "p2": [0.95, 0.5] }
  ],
  "count_classes": ["car", "truck", "bus", "motorcycle", "bicycle"]
}
```

Set `"tracker": null` (or pass `--no-tracker`) for detect-only mode. An empty-string `tracker` is normalized to `null`.

### Video Pipeline Modes

| Mode                      | Description                                                            |
| ------------------------- | ---------------------------------------------------------------------- |
| `opencv+ffmpeg` (default) | OpenCV capture → YOLO detect → FFmpeg stdin → RTSP                     |
| `direct-ffmpeg`           | Source → FFmpeg → RTSP (no OpenCV/YOLO, pure passthrough)              |
| `opencv+gstreamer`        | OpenCV capture → YOLO → GStreamer → RTSP (falls back to opencv+ffmpeg) |

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
| AI Detection Model          | YOLO Object Detection + ByteTrack multi-object tracking |
| AI Runtime                  | Ultralytics YOLO (PyTorch) + `lap` (ByteTrack association) |
| Camera Interface            | CSI Camera / USB Camera        |
| Local Wireless Receiver     | LoRa SX1276 / SX1278 Module    |
| Local Control Protocol      | SPI (Pi to LoRa) / LoRa Radio  |
| Actuator Interface          | GPIO-controlled Relay Board    |
| MQTT Communication          | Eclipse Paho MQTT Client       |
| Communication Protocol      | MQTT over TLS (to Backend)     |
| Video Streaming             | RTSP Publisher                 |
| Containerization (Optional) | Docker                         |

## Key Features

**AI-Based Traffic Analysis** — Vehicle detection, ByteTrack multi-object tracking with persistent IDs, vehicle counting (live occupancy + cumulative line crossings), per-direction / per-class breakdowns, pedestrian detection, traffic density estimation, lane occupancy detection, queue length estimation, congestion detection.

**Local Edge Processing** — Performs AI inference locally, reduces bandwidth consumption, provides faster response time, minimizes cloud processing requirements.

**LoRa-Driven Signal Actuation** — Receives synchronized traffic light phase commands from the group's ESP-32 Traffic Controller via LoRa, driving the physical green/red relays.

**Fault Tolerant Safety Fallback** — Reverts to all relays off/local caution mode if LoRa control signals are lost or corrupted, ensuring driver safety.

**Device Monitoring** — Periodic telemetry (live count, cumulative total, per-direction / per-class counts, traffic density, active signal) and heartbeat reporting (uptime, timestamp).

**Secure Backend Communication** — Device heartbeat reporting, telemetry publishing, remote configuration updates.
