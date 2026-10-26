# Media Server (MediaMTX + go2rtc)

Live video is handled by two cooperating servers:

- **MediaMTX** is the single ingest point. Every Raspberry Pi publishes one RTSP stream to MediaMTX, which handles authentication, per-device paths, and optional recording.
- **go2rtc** is the low-latency delivery layer. It pulls a device's RTSP stream from MediaMTX **on demand** and serves it to mobile and web viewers over WebRTC/WHEP (with MSE/HLS fallbacks).

MediaMTX still owns publishing and access control; go2rtc never sees the edge devices directly. See [Streaming Server Options](media-server-options.md) for the comparison that led to this split.

## Responsibilities

**MediaMTX**

- Receive RTSP publishes from edge devices
- Authenticate publishers and readers via the backend HTTP callback
- Expose one path per device (`/{deviceId}`)
- Relay/record when configured
- Stay reachable from edge devices over port `8554`

**go2rtc**

- Pull `rtsp://.../{deviceId}` from MediaMTX only when a viewer is active
- Exchange WHEP SDP offers/answers with the backend
- Fan out WebRTC media to multiple viewers
- Provide MSE/HLS fallbacks for browsers without WebRTC support
- Close the upstream producer when the last viewer leaves

## Architecture

```mermaid
graph LR
    Pi["Raspberry Pi Edge<br/>(RTSP publisher)"] -->|"RTSP publish :8554"| MTX["MediaMTX<br/>ingest / auth / recording"]
    MTX -->|"RTSP pull (on demand)"| RTC["go2rtc<br/>WHEP / MSE / HLS"]
    RTC -->|"WebRTC media :8555"| Mobile["Flutter App"]
    RTC -->|"WebRTC / MSE"| Web["Web Dashboard"]
    Backend["Backend API"] -->|"auth callback"| MTX
    Backend -->|"register stream + proxy WHEP SDP"| RTC
    Mobile -->|"JWT + SDP"| Backend
    Web -->|"JWT + SDP"| Backend
```

Media never flows through Caddy or the backend. Caddy terminates TLS for `/api/*`; the WebRTC media path connects directly to go2rtc on `8555/udp` or `8555/tcp`.

## Technology Stack

| Component | Technology |
| --------- | ---------- |
| Edge Stream Publisher | RTSP Publisher (H.264, low-delay) |
| RTSP Ingest | MediaMTX |
| WebRTC Delivery | go2rtc |
| Ingest Protocol | RTSP over TCP |
| Playback Protocols | WHEP/WebRTC (primary), MSE, HLS, RTSP (fallback) |
| Video Codec | H.264 (edge output); H.265 supported by go2rtc via MSE |
| Mobile Playback | `flutter_webrtc` (WHEP) |
| Web Playback | `RTCPeerConnection` (WHEP) |
| Authentication | MediaMTX HTTP callback + backend JWT + go2rtc service account |
| Transport Security | HTTPS (signaling) + DTLS/SRTP (media) |

## Ports

| Port | Server | Exposure | Purpose |
| ---- | ------ | -------- | ------- |
| `8554/tcp` | MediaMTX | Edge devices / VPN | RTSP publish |
| `8555/tcp` + `8555/udp` | go2rtc | Public | WebRTC media |
| `1984/tcp` | go2rtc | Internal only | HTTP API, stream registration, WHEP signaling |
| `8889/tcp` | MediaMTX | Internal only | Native MediaMTX WebRTC (legacy/debug) |
| `443/tcp` | Caddy | Public | `/api/*`, `/ws/*`, `/health` |

Do not publish `1984`. The backend is the only client of the go2rtc API.

## Stream Lifecycle

1. The edge publishes `rtsp://{deviceId}:{deviceSecret}@mediamtx:8554/{deviceId}` as before.
2. A viewer requests `POST /api/devices/{deviceId}/whep` with a JWT and an SDP offer.
3. The backend ensures a go2rtc stream exists:
   `PUT http://go2rtc:1984/api/streams?name={deviceId}&src=rtsp://go2rtc:{secret}@mediamtx:8554/{deviceId}`
4. The backend forwards the offer to `POST http://go2rtc:1984/api/webrtc?src={deviceId}` and returns the SDP answer.
5. go2rtc pulls the RTSP stream from MediaMTX on first consumer and fans out WebRTC media on `8555`.
6. When the last viewer leaves, go2rtc closes the upstream producer. The stream registration stays in go2rtc's runtime config so the next viewer can reuse it.

The registered stream is written to a writable runtime config volume. A go2rtc restart does not affect MediaMTX or the edge; the runtime entry is reloaded.

## Authentication

Publishing and playing through MediaMTX still use the existing HTTP callback. go2rtc is a **reader**, so the backend callback gains a service-account branch:

- `user = GO2RTC_MEDIA_USER` and `password = GO2RTC_MEDIA_SECRET` with `action = read` → `200`.
- Everything else falls through to the existing JWT/device-secret checks.

Viewer access is JWT-gated at the backend. go2rtc's own API uses HTTP Basic auth with `GO2RTC_API_USER` / `GO2RTC_API_PASS`, which never leaves the server.

MediaMTX cannot combine HTTP auth with internal auth (bluenviron/mediamtx#4659), and `authHTTPExclude: [{action: read}]` would open reads to everyone. The shared reader account is the supported workaround.

## Latency Budget

Server-side contribution is small; the edge publisher and the player dominate.

| Stage | Current RTSP path | Target WHEP path |
| ----- | ----------------- | ---------------- |
| Camera capture | 30–60 ms | 30–60 ms |
| YOLO + annotate | 50–300 ms (affects annotation freshness only after the edge fix) | 50–300 ms |
| x264 encode | 20–50 ms | 20–50 ms |
| RTSP mux buffer | ~700 ms (default `muxdelay`) | 0 ms (`-muxdelay 0`) |
| MediaMTX relay | 30–80 ms | 30–80 ms |
| go2rtc relay | — | 50–150 ms |
| Client jitter/demux buffer | 1–8 s (`media_kit` RTSP) | 100–300 ms (WebRTC) |
| **Typical total** | **2–8 s+** | **300–800 ms** |

> **Known issue:** the current edge build only sends frames that pass through YOLO, while telling FFmpeg the pipe runs at 24 fps. If inference is slower, the media clock drifts behind wall time and the lag grows. go2rtc cannot correct this. The fix is specified in [Edge Device → Stream Publishing Pipeline](edge-device.md#stream-publishing-pipeline).

## Playback Paths

| Client | Primary | Fallback |
| ------ | ------- | -------- |
| Flutter mobile | WHEP via `flutter_webrtc` | `media_kit` RTSP |
| Web dashboard | WHEP via `RTCPeerConnection` | MSE, then HLS |
| Debug / VLC | Direct RTSP `rtsp://...:8554/{deviceId}` | — |

## Key Features

**Live Video Monitoring** — Real-time traffic camera feeds, multiple viewer support, sub-second WebRTC delivery, mobile-compatible playback.

**On-Demand Relay** — One RTSP uplink per camera; go2rtc only pulls while someone is watching, so idle cameras do not consume origin bandwidth.

**Centralized Stream Management** — MediaMTX keeps the per-device path and authentication model; go2rtc adds the low-latency delivery plane without changing the edge contract.

**Protocol Flexibility** — WebRTC for latency, MSE for H.265/browser compatibility, HLS for restrictive networks, RTSP for debugging and legacy clients.

## Related

- [Streaming Server Options](media-server-options.md) — comparison and cost math
- [Server Integration](server-integration.md) — compose, go2rtc config, auth, WHEP endpoint
- [Edge Device](edge-device.md) — publish-rate fix
- [Mobile App](mobile-app.md) — WHEP player
