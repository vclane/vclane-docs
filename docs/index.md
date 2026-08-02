# VC-LANE Documentation

**VC-LANE** (Adaptive Traffic Monitoring System) is a distributed intelligent traffic monitoring and management platform providing real-time traffic observation, live video streaming, AI-based traffic analysis, device monitoring, and coordinated traffic signal control across multiple intersections.

The system follows an **edge computing architecture**, where every traffic intersection operates as an independent AI-enabled Raspberry Pi edge device for local camera processing and traffic light actuation. Intersections are grouped, and each group is coordinated locally by an **ESP-32 Traffic Controller** via **LoRa** wireless broadcasts, ensuring fault-tolerant signaling even during internet outages. Remote management and schedules are handled by the backend over MQTT over TLS.

## Documentation Sections

| # | Page | Description |
|---|------|-------------|
| 1 | [Overview](overview.md) | System overview |
| 2 | [Architecture](architecture.md) | System architecture and component interaction |
| 3 | [Edge Device](edge-device.md) | Raspberry Pi edge device — video processing, AI, and traffic light actuation |
| 4 | [Traffic Controller](traffic-controller.md) | ESP-32 Traffic Controller — local group scheduling, LoRa broadcasts, and fallback telemetry |
| 5 | [Backend API](backend-api.md) | Backend — FastAPI, device management, traffic coordination |
| 6 | [MQTT Communication](mqtt-communication.md) | MQTT layer — broker, security, topic structure, telemetry |
| 7 | [Media Server](media-server.md) | Video streaming — MediaMTX, RTSP, WebRTC |
| 8 | [Mobile App](mobile-app.md) | Flutter application — auth, dashboard, video playback, device control |
| 9 | [Features](features.md) | System-wide key features |
| 10 | [Architectural Advantages](architectural-advantages.md) | Low latency, scalability, resilience, security |
| 11 | [Changelog](changelog.md) | System and submodule-specific change log |