# VC-LANE Documentation

**VC-LANE** (Adaptive Traffic Monitoring System) is a distributed intelligent traffic monitoring and management platform providing real-time traffic observation, live video streaming, AI-based traffic analysis, device monitoring, and coordinated traffic signal control across multiple intersections.

The system follows an **edge computing architecture** — every traffic intersection operates as an independent AI-enabled Raspberry Pi edge device communicating with a centralized backend over MQTT over TLS.

## Documentation Sections

| # | Page | Description |
|---|------|-------------|
| 1 | [Overview](overview.md) | System overview |
| 2 | [Architecture](architecture.md) | System architecture and component interaction |
| 3 | [Edge Device](edge-device.md) | Raspberry Pi edge device — video processing, AI, local control, monitoring |
| 4 | [Backend API](backend-api.md) | Backend — FastAPI, device management, traffic coordination |
| 5 | [MQTT Communication](mqtt-communication.md) | MQTT layer — broker, security, topic structure, telemetry |
| 6 | [Media Server](media-server.md) | Video streaming — MediaMTX, RTSP, WebRTC |
| 7 | [Mobile App](mobile-app.md) | Flutter application — auth, dashboard, video playback, device control |
| 8 | [Features](features.md) | System-wide key features |
| 9 | [Architectural Advantages](architectural-advantages.md) | Low latency, scalability, resilience, security |
| 10 | [Database](database.md) | Firestore and RTDB schemas — collections, paths, and field definitions |