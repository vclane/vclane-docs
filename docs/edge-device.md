# Raspberry Pi Edge Device

Each intersection contains an independent Raspberry Pi-based edge device acting as the intelligent edge computing unit. It performs video processing, AI inference, traffic analysis, device communication, and local decision-making.

## Phase Sync Transport (`sync_transport`)

Phase frames (`{gid, seq, ts, p, ttl, ov}`) can arrive via LoRa radio (default) or via the MQTT `traffic/group/{groupId}/phase` topic (QoS 1, retained) — both feed one shared `seq`-dedup state (first-valid wins). `ov` is `true` while an override is active (5 relay blinks), `false` otherwise (3 blinks); missing `ov` defaults to `false`.

| `sync_transport` | Behavior |
| ---------------- | -------- |
| `lora` (default) | LoRa receiver only; needs SX1276/SX1278 wiring below |
| `mqtt` | MQTT phase subscription only; no LoRa hardware needed, but requires broker connectivity |
| `dual` | Accepts either transport; useful for migration |

- Set via config `sync_transport` or CLI `--sync-transport`.
- The caution fallback (`lora_timeout`, default 5 s without any valid phase packet) applies to both transports.
- In `mqtt`/`dual` modes the existing MQTT connection gains one extra subscription; no new credentials.

## Hardware Wiring

An intersection node connects up to three peripherals to the Raspberry Pi: the **LoRa receiver** (listens for phase broadcasts from the group's ESP-32 Traffic Controller — only needed when `sync_transport` is `lora`/`dual`), the **relay board** (drives the green/red signal lamps), and the **camera** (feeds the video/YOLO pipeline).

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
- Run YOLO + ByteTrack tracking with persistent IDs
- Emit one per-vehicle detection event per new tracked ID (first sighting, any YOLO label) with the full annotated frame
- Generate traffic statistics (signal + emergency flag only; no aggregate counts)
- Publish live annotated video streams (boxes + track IDs)

Data flow:

```mermaid
graph TD
    Camera --> Capture["Capture loop (fps)"]
    Capture --> Slot["Latest-frame slot"]
    Slot --> TRACK["YOLO + ByteTrack (track, persist=True)"]
    TRACK --> Annotate["Annotated frame"]
    Annotate --> Slot
    TRACK --> Detect["DetectionTracker (first-seen per ID + annotated frame)"]
    Detect --> Upload["DetectionUploader (POST /detections, retry)"]
    Slot --> PUB["Publisher (publish_fps)"]
    PUB --> FFMPEG["FFmpeg (H.264, low-delay)"]
    FFMPEG --> RTSP["RTSP → MediaMTX"]
```

Set `yolo_enabled: false` for raw passthrough with no inference (or `stream_mode: direct-ffmpeg`).

### Stream Publishing Pipeline

The publisher must run on a **wall-clock cadence**, independent of how fast YOLO inference completes. The current build does not, which is the primary cause of growing lag and stutter.

#### Current behavior (to be replaced)

- When `yolo_enabled` is true, only frames selected for inference are sent (`device.py:414-424`); every other frame is read and discarded.
- The only `send_frame` call is inside the inference worker (`device.py:337-340`).
- FFmpeg is told the pipe runs at `settings.fps` (24) regardless of the actual inference rate (`video.py:210`).
- The writer queue holds two frames and drops on overflow (`video.py:61-93`).
- The RTSP muxer keeps its default ~0.7 s delay.

If inference yields 12–15 fps while FFmpeg timestamps at 24 fps, the media clock runs at roughly half wall-clock speed. The stream falls further behind live and eventually stalls.

#### Target design

Three decoupled loops:

1. **Capture** at `fps` (24).
   - USB cameras: set `cv2.CAP_PROP_BUFFERSIZE = 1`; prefer `grab()`/`retrieve()` so the loop always reads the newest frame.
   - Picamera2: `capture_array()` already returns the current frame.
   - Always overwrite the latest raw frame slot; never block on inference.

2. **Inference** worker.
   - Takes the newest raw frame when idle. The capture loop only offers every `yolo_interval`-th frame, so the worker samples without blocking.
   - Runs `track()` → `annotate()` and writes the annotated frame into the slot.
   - A slow inference pass reduces annotation freshness only; it must not affect publish cadence.
   - Keep a single worker so ByteTrack still sees frames in order. `yolo_interval` remains 1–2 to limit ID switches.

3. **Publisher** at `publish_fps` (default 15).
   - On every tick, take the newest annotated frame; fall back to the newest raw frame; if neither is new, repeat the last frame.
   - Write exactly `publish_fps` frames per wall-clock second to FFmpeg.
   - If the FFmpeg pipe backs up, drop the oldest pending frame and increment a counter; never block the capture loop.

```python
def publisher_loop():
    frame_interval = 1.0 / settings.publish_fps
    next_tick = time.monotonic()
    last = None
    while running:
        frame = annotated_slot.get() or raw_slot.get() or last
        if frame is not None:
            writer.write(frame)
            last = frame
        next_tick += frame_interval
        time.sleep(max(0.0, next_tick - time.monotonic()))
```

#### Configuration

| Field | Default | Meaning |
| ----- | ------- | ------- |
| `fps` | `24` | Camera capture rate (unchanged) |
| `publish_fps` | `15` | Actual FFmpeg/RTSP output rate |
| `yolo_interval` | `1` | Inference sampling; affects annotation freshness, **not** publish rate |
| `hw_encoder` | `false` | Use `h264_v4l2m2m` when available, with `libx264` fallback |
| `video_bitrate` | `1M` | H.264 target/max bitrate (`600k`, `1M`, ...); lower it for constrained uplinks |

Picamera2 is opened at `fps` with `buffer_count=2`, so capture does not run at the sensor's default 30 fps.

`rtsp_url`, `mediamtx_host`, and `mediamtx_port` are **unchanged**. The edge still publishes to MediaMTX; go2rtc is a server-side consumer.

> Provisioning caveat: the backend builds `rtspUrl` from `request.url.hostname`. Provisioning through `https://vclane.com` yields a public host that works from a remote Pi; provisioning through `http://backend:8000` yields an internal host that does not. `--rtsp-url` remains the override. This is a pre-existing provisioning concern, not part of the go2rtc change.

#### FFmpeg arguments

Use these for all pipelines (`opencv+ffmpeg`, `picamera2`, and the direct/GStreamer variants where applicable):

```python
"-r", str(settings.publish_fps),
"-an",
"-c:v", "h264_v4l2m2m" if settings.hw_encoder else "libx264",
"-preset", "ultrafast",
"-tune", "zerolatency",
"-profile:v", "main",
"-pix_fmt", "yuv420p",
"-g", str(settings.publish_fps),
"-keyint_min", str(settings.publish_fps),
"-sc_threshold", "0",
"-bf", "0",
"-b:v", settings.video_bitrate,
"-maxrate", settings.video_bitrate,
"-bufsize", settings.video_bitrate,
"-muxdelay", "0",
"-muxpreload", "0",
"-rtsp_transport", "tcp",
"-f", "rtsp",
url,
```

- 1 s GOP (`-g = publish_fps`) gives fast joins and clean WebRTC keyframes.
- `-muxdelay 0 -muxpreload 0` removes the ~0.7 s RTSP mux buffer.
- A 1 s `-bufsize` reduces encoder buffering.
- If `h264_v4l2m2m` fails to initialize, fall back to `libx264` once and log it.

#### Observability

Log at the existing telemetry cadence:

- `publish_fps_actual` — measured output rate
- `frames_dropped` — writer backpressure
- `inference_skipped` — frames not annotated
- `ffmpeg_restarts` — pipe recoveries

#### Acceptance Criteria

- `ffprobe` on the MediaMTX path reports `avg_frame_rate == publish_fps`, not `fps`.
- Clock-in-frame test: stream lag vs wall clock stays within ±0.5 s over a 10-minute run.
- With artificially slow inference (stub the model or add a sleep), the stream still publishes at `publish_fps` with no drift.
- Killing the backend or inference worker does not stop the publisher or trigger an FFmpeg restart loop.
- Time-to-first-frame on LAN is under 2 s after a viewer joins.
- A Pi 4 at 720p/`publish_fps` stays within the agreed CPU budget; otherwise enable `hw_encoder`.

## AI-Based Traffic Analysis

The edge device performs local AI inference instead of sending raw video to the backend.

Advantages:

- Lower bandwidth usage
- Lower latency
- Faster response time
- Reduced cloud processing cost

Examples:

- Vehicle detection events (one per new tracked ID, with the full annotated frame)
- Lane occupancy detection
- Queue length estimation
- Traffic congestion detection
- Per-direction and per-class breakdowns

### Vehicle Tracking (YOLO + ByteTrack, default)

Tracking runs with `tracker: "bytetrack.yaml"` out of the box — `YOLODetector.track()` calls `model.track(persist=True, tracker=...)`, preserving Kalman state across frames. Requires the `lap` dependency (`pip install lap` / `uv sync`).

- Annotated boxes show `{label} #{track_id} {conf:.2f}` (e.g. `car #7 0.82`).
- **Frame ordering matters:** a single inference worker consumes frames in order, preserving sequence for the Kalman filter. Capture does not block on it: a latest-frame slot drops intermediate frames only under overload.
- **`yolo_interval` is 1 or 2** (config-validated; `1` = every frame accuracy, `2` = every 2nd frame perf). Larger skips break the Kalman state and cause ID switches. It controls annotation freshness, not the published frame rate — the publisher runs at `publish_fps` regardless.
- **Stream gaps reset IDs:** on EOF / reconnect (`read_frame()` returns `None`), `reset_tracker()` clears ByteTrack state so stale tracks do not leak into the new sequence.
- Verify manually with `python scripts/test_tracking.py --source <video|0> [--save out.mp4]` (defaults: `--model models/best.pt --conf 0.4 --imgsz 416 --tracker bytetrack.yaml`).

CLI flags:

| Flag | Description |
| ---- | ----------- |
| `--tracker <yaml>` | Tracker config (default `bytetrack.yaml`) |
| `--yolo-interval <N>` | Run YOLO every N frames (`1` accuracy, `2` perf) |

### Per-Vehicle Detection Events (`detection.py`)

`src/vclane_edge/detection.py` layers eventing on top of ByteTrack IDs (no aggregate counts):

- **Trigger** — first sighting of each ByteTrack ID emits one `DetectionEvent` (any YOLO label, unfiltered) with the full annotated frame. Seen IDs are remembered until a stream gap resets them.
- **Upload** — `DetectionUploader` POSTs each event immediately via `POST /api/devices/{id}/detections` (bounded queue, drop-oldest on overflow, retry offline with backoff).
- **Emergency** — the same tracker debounces `emergency_classes` over `emergency_min_frames` consecutive sightings per vehicle for override requests.

## Local Decision Making and Actuation

Under the grouped architecture, the Raspberry Pi Edge Device functions as the physical actuator for the traffic lights at its intersection. Rather than executing a standalone schedule or receiving direct signal overrides from the backend, it delegates coordination to the group's ESP-32 Traffic Controller.

Key behaviors:

- **Phase Reception:** The Raspberry Pi listens for phase broadcasts via the configured `sync_transport` — LoRa packets and/or MQTT `traffic/group/{groupId}/phase` frames containing the target active phases plus the `ov` override flag.
- **Relay Actuation:** It parses the received command (e.g., active phase, green/red light configurations) and controls the physical relays to update the signal lamps. Via `apply_phase(phase, is_override)`: duplicate phases are ignored (the 1 s heartbeat re-broadcasts the same `seq`); otherwise the new lamp blinks on/off × N — 3 blinks when `ov` is `false`, 5 when `ov` is `true` — on a relay worker thread under the relay lock, then latches solid so the 1 s signal/telemetry loop is not blocked. Entering caution aborts any in-progress blink and goes all-off with no blink; the first valid phase after boot goes straight solid.
- **Resilient Fallback:** If the phase signal is lost (LoRa out of range / ESP-32 failure, or MQTT broker unreachable in `mqtt` mode), the Raspberry Pi falls back to a pre-programmed local safety routine (such as all relays off for caution) to prevent unsafe signal states.

## Backend Communication

The edge device maintains a persistent MQTT connection to the backend when internet is available.

Responsibilities:

- Publish lean AI traffic telemetry — the sole telemetry source in the system (emergency-vehicle flag + active signal only; no counts — history lives in `detections`)
- Report the currently actuated traffic light phase (from LoRa) or caution state
- Send heartbeat updates (process uptime and timestamp)
- Maintain the retained `status` topic (`online: true` on connect, `online: false` on graceful shutdown) and register a Last Will & Testament so the broker reports ungraceful disconnects
- Submit per-vehicle detections via `POST /api/devices/{id}/detections` (one multipart request per new ByteTrack ID: `trackId`, `class`, `deviceTimestamp`, optional `image` full annotated frame — the edge always includes one; bounded queue, hold/retry offline).

> Dynamic queue gate triggers off `POST /api/devices/{id}/detections` (new-doc only), not telemetry. Dedup is server-side via doc-ID existence + RTDB `has_queued_schedule`, and the gate also honors the group's `enqueueEdgeDetections` switch; an admin-only `forceEnqueue` request can override both. The edge keeps uploading normally and never tracks queue or group settings itself.

Example telemetry:

```json
{
  "deviceId": "intersection-01",
  "currentTrafficLightSignal": "GREEN",
  "emergencyVehicleDetected": false,
  "emergencyClass": "",
  "emergencyConfidence": 0.0,
  "timestamp": "2026-07-24T10:00:00Z"
}
```

When an emergency vehicle (`ambulance`, `fire truck`, `police car` by default) is visible, `emergencyVehicleDetected` is `true` with the top-match `emergencyClass` / `emergencyConfidence` (e.g. `"ambulance"`, `0.91`).

### Emergency Vehicle Override

When YOLO reports an `emergency_classes` label (`ambulance`, `fire truck`, `police car` by default) above `emergency_confidence` for `emergency_min_frames` consecutive sightings, the edge sets the telemetry emergency flag and sends one `signal_override` request per distinct vehicle (ByteTrack ID) via the existing `POST /api/groups/{id}/commands` endpoint (`durationSeconds: 60` by default, no cancel — the override expires naturally). Auth reuses `POST /api/auth/login` with `admin_email` / `admin_password` (or `VCLANE_ADMIN_EMAIL` / `VCLANE_ADMIN_PASSWORD`). Requires `api_url` + `group_id` + admin credentials; without them the device sets the telemetry flag only. A global `emergency_cooldown_s` (default 60 s) suppresses redundant POSTs while an override is active.

> Backend 2.13.0 also reacts to emergency-class entries in `POST /api/devices/{id}/detections` by publishing the same 60 s group override (or a device-level fallback). The edge-initiated request remains supported and its cooldown still applies; if both paths fire, the controller simply receives a duplicate identical override.

## Device Monitoring

Each edge device periodically reports its status to the backend over MQTT:

- **Telemetry** (`traffic/device/{deviceId}/telemetry`): active signal (or `RED` during fallback) + emergency-vehicle flag (`emergencyVehicleDetected` / `emergencyClass` / `emergencyConfidence`). No counts. Dynamic queue trigger moved to `POST /api/devices/{id}/detections`.
- **Heartbeat** (`traffic/device/{deviceId}/heartbeat`): process uptime and timestamp
- **Status / LWT** (`traffic/device/{deviceId}/status`, QoS 1, retained): `{"online": true}` published on every successful connect, `{"online": false}` published before a graceful disconnect, and a retained will of `{"online": false}` published by the broker when the connection drops without a clean DISCONNECT. Detection of an ungraceful loss takes up to ~1.5 × the MQTT keepalive (`mqtt_keepalive`, 60 s default).

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
│   ├── detection.py         # DetectionTracker (per-vehicle events) + DetectionUploader (REST queue)
│   ├── telemetry.py         # Telemetry state (signal + emergency flag only)
│   ├── backend_api.py       # Backend HTTP client (auth, provisioning, commands, detections)
│   ├── signal_override.py   # Override scheduler (cooldown + worker thread)
│   ├── lora.py              # LoRa receiver (ESP-32 phase broadcasts)
│   ├── relay.py             # Relay actuation + caution fallback
│   ├── backend_api.py       # Backend HTTP client (auth, provisioning, commands)
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
  "publish_fps": 15,
  "hw_encoder": false,
  "detections_enabled": true,
  "detections_max_queue": 200,
  "relay_blink_normal": 3,
  "relay_blink_override": 5,
  "relay_blink_interval": 0.5
}
```

`tracker` is required (default `bytetrack.yaml`); `yolo_interval` is validated to 1–2 (`1` accuracy, `2` perf). `publish_fps` (default `15`) is the actual RTSP output rate and is independent of both `fps` and `yolo_interval`. `hw_encoder` enables `h264_v4l2m2m` on supported Pis with an automatic `libx264` fallback. Set `yolo_enabled: false` (or `stream_mode: direct-ffmpeg`) for raw passthrough with no inference.

`relay_blink_normal` / `relay_blink_override` select the blink count before latching solid (3 on schedule transitions, 5 when the phase frame has `ov: true`); `relay_blink_interval` is the seconds per on/off step.

### Video Pipeline Modes

| Mode                      | Description                                                            |
| ------------------------- | ---------------------------------------------------------------------- |
| `opencv+ffmpeg` (default) | OpenCV capture → latest-frame slot → YOLO annotate → fixed-rate publisher → FFmpeg → RTSP |
| `direct-ffmpeg`           | Source → FFmpeg → RTSP (no OpenCV/YOLO, pure passthrough)                                  |
| `opencv+gstreamer`        | OpenCV capture → latest-frame slot → YOLO → publisher → GStreamer → RTSP (falls back to opencv+ffmpeg) |

### Registration Flow

1. Admin runs `vclane-edge register --api-url <url>`
2. Prompts for Firebase email/password (or sets `VCLANE_ADMIN_EMAIL` / `VCLANE_ADMIN_PASSWORD`)
3. `POST /api/auth/login` → JWT
4. `POST /api/devices` → receives `{deviceId, deviceSecret, rtspUrl}`
5. With `--out <path>`, credentials saved as JSON; otherwise printed to stdout

Auto-registration at `vclane-edge run` start occurs when `device_id` or `device_secret` are empty (or `force_register` is `true`). Registered credentials are used in-memory for the session — the config file is not modified.

At `run` start the backend client authenticates once (`POST /api/auth/login`) before the video pipeline starts, so the first detection upload or emergency override does not pay the login round-trip. A failed startup login is logged and retried lazily on the first request (with a single re-auth on any 401).

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
| Video Streaming             | RTSP Publisher (H.264, fixed `publish_fps`) |
| Containerization (Optional) | Docker                         |

## Key Features

**AI-Based Traffic Analysis** — Vehicle detection, ByteTrack multi-object tracking with persistent IDs, per-vehicle detection events with full annotated frames, pedestrian detection, lane occupancy detection, queue length estimation, congestion detection.

**Local Edge Processing** — Performs AI inference locally, reduces bandwidth consumption, provides faster response time, minimizes cloud processing requirements.

**LoRa-Driven Signal Actuation** — Receives synchronized traffic light phase commands from the group's ESP-32 Traffic Controller via LoRa (default) or MQTT phase topic, driving the physical green/red relays.

**Fault Tolerant Safety Fallback** — Reverts to all relays off/local caution mode if phase control signals are lost or corrupted, ensuring driver safety.

**Device Monitoring** — Periodic lean telemetry (active signal, emergency-vehicle flag) and heartbeat reporting (uptime, timestamp).

**Secure Backend Communication** — Device heartbeat reporting, telemetry publishing, remote configuration updates.

**Low-Latency Publishing** — A wall-clock publisher decouples the RTSP frame rate from YOLO inference, and low-delay FFmpeg mux flags keep the stream close to real time.
