# Media Server (MediaMTX)

MediaMTX acts as the centralized video streaming server. Each Raspberry Pi publishes one stream to MediaMTX, which distributes it to multiple clients.

## Responsibilities

- Receive RTSP streams
- Relay video streams
- Support multiple viewers
- Reduce edge device bandwidth usage
- Provide low-latency playback

## Technology Stack

| Component             | Technology           |
| --------------------- | -------------------- |
| Edge Stream Publisher | RTSP Publisher       |
| Media Server          | MediaMTX             |
| Video Protocol        | RTSP / WebRTC        |
| Video Codec           | H.264 / H.265        |
| Mobile Playback       | WebRTC / RTSP Player |
| Security              | HTTPS / TLS          |

## Key Features

**Live Video Monitoring** — Real-time traffic camera feeds, multiple viewer support, low-latency streaming, mobile-compatible playback.

**Centralized Stream Relay** — Stream aggregation, client distribution, reduced edge bandwidth usage, multiple protocol support.

**Scalable Video Access** — Instead of clients connecting directly to Raspberry Pi:

``` mermaid
graph TD
    RPi["Raspberry Pi Camera"] --> MediaMTX
    MediaMTX --> Mobile["Mobile App"]
    MediaMTX --> Web["Web Dashboard"]
```

Benefits:

- Protects edge devices
- Supports many viewers
- Simplifies stream management