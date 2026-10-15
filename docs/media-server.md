# Media Server (LiveKit Cloud)

LiveKit Cloud acts as the managed WebRTC SFU for live video. Each Raspberry Pi publishes one annotated WHIP stream to a per-device room, which LiveKit fans out to operators. Replaces the self-hosted MediaMTX RTSP relay (removed for slow single-origin routing).

## Responsibilities

- Receive annotated WHIP streams (boxes + track IDs burned in on the Pi)
- Fan out to multiple viewers over WebRTC (<500 ms)
- Reduce edge device bandwidth usage (one uplink per camera, on-demand publish)
- Provide low-latency playback with global edge PoPs
- Token-gated room access (no shared RTSP credentials)

## Technology Stack

| Component             | Technology                |
| --------------------- | ------------------------- |
| Edge Stream Publisher | WHIP Publisher (FFmpeg)   |
| Media Server          | LiveKit Cloud (SFU)       |
| Ingest Protocol       | WHIP (H.264 baseline)     |
| Playback Protocol     | WebRTC                    |
| Video Codec           | H.264 (transcode-free)    |
| Mobile Playback       | livekit_client (Flutter)  |
| Security              | LiveKit join tokens (JWT) |

Target encode: 720p, ~1–1.5 Mbps, 2 s GOP. Transcode-less WHIP ingest avoids transcode fees.

## Key Features

**Live Video Monitoring** — Real-time traffic camera feeds with edge-burned annotations, multiple viewer support, sub-second streaming, mobile-compatible playback.

**Managed Fan-Out** — Room-per-device (`room=deviceId`), publisher `deviceId`, subscribers by operator `userId`. Backend mints join tokens; LiveKit handles SFU, TURN, and edge delivery.

**On-Demand Publishing** — Instead of 24/7 RTSP publish, the Pi idles and only starts FFmpeg/WHIP on `start_publish` (viewer joined) and stops on `stop_publish`:

``` mermaid
graph TD
    RPi["Raspberry Pi Camera + YOLO annotate"] -->|WHIP on demand| LiveKit["LiveKit Cloud room=deviceId"]
    LiveKit --> Mobile["Mobile App (livekit_client)"]
    LiveKit --> Web["Web Dashboard (WHEP)"]
    Backend["Backend API"] -->|join tokens + webhooks| LiveKit
    Backend -->|"start_publish / stop_publish"| RPi
```

Benefits:

- Sub-second latency via managed edge network
- Protects edge devices (single uplink)
- Supports many viewers without origin load
- Stays in free tier for pilots (Build: 5k min + 50GB)
- Simplifies auth (short-lived tokens, no shared secrets)

> Legacy note: MediaMTX (`RTSP/WebRTC relay`, `GET /api/mediamtx/auth`, `rtsp://{deviceId}@mediamtx:8554/{deviceId}`) is deprecated since 2.11.0 but kept for backward compat. Dual-publish (RTSP + WHIP) is supported during migration.