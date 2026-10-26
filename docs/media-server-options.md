# Streaming Server Options (Free)

VCLane needs sub-second live video from Raspberry Pi cameras to mobile and web operators, while keeping the edge publish path simple and the server footprint small. This page compares the free and open-source options evaluated for that job and records why **go2rtc** was selected as the additional streaming server alongside MediaMTX.

The current stack keeps MediaMTX for RTSP ingest, authentication, and optional recording. go2rtc is added as an on-demand WebRTC delivery layer; see [Media Server](media-server.md) for the resulting topology and [Server Integration](server-integration.md) for the implementation spec.

## Evaluation Criteria

| Criterion | Why it matters here |
| --------- | ------------------- |
| License and cost | Must be free to self-host; no per-minute billing for 24/7 cameras |
| RTSP ingest | Edge devices already publish RTSP to MediaMTX; rewriting the publish path is expensive |
| Low-latency output | Operators need < 1 s glass-to-glass for live monitoring |
| On-demand pull | Cameras should not be re-streamed when nobody is watching |
| H.265 support | Optional today (edge publishes H.264), but useful for future camera models |
| Authentication | Viewer access must be JWT-gated; publishing stays behind MediaMTX |
| Mobile support | Flutter needs a reliable WebRTC player, not a browser-only solution |
| Operational weight | Small team; one container is preferable to a cluster |

## Self-Hosted Options

| Option | License | RTSP ingest | Low-latency output | On-demand | H.265 | Mobile SDK | Ops weight | Verdict |
| ------ | ------- | ----------- | ------------------ | --------- | ----- | ---------- | ---------- | ------- |
| **MediaMTX** (current) | MIT | ✅ push | WHEP/WebRTC, HLS | ✅ | ⚠️ WebRTC browser limits | ⚠️ no official Flutter SDK | Low | Keep for RTSP ingest, auth, recording |
| **go2rtc** | MIT | ✅ pull + push | WHEP/WS, MSE, HLS | ✅ | ✅ via MSE | ⚠️ use `flutter_webrtc` | Very low | **Chosen sidecar for WebRTC delivery** |
| **ZLMediaKit** | MIT | ✅ push/pull | WebRTC, HLS, HTTP-FLV | ✅ | ✅ | ⚠️ use `flutter_webrtc` | Medium | Alternative if replacing MediaMTX |
| **SRS** | MIT | ❌ native (FFmpeg pull only) | WebRTC, HLS, HTTP-FLV | ⚠️ | ⚠️ | ⚠️ use `flutter_webrtc` | Medium | Requires changing the edge publish protocol |
| **OvenMediaEngine** | AGPL-3.0 | ✅ pull | WebRTC, LL-HLS | ⚠️ | ✅ | ⚠️ use `flutter_webrtc` | High | Sub-second plus built-in transcode |
| **LiveKit** (self-hosted) | Apache-2.0 | ❌ WHIP only | WebRTC SFU | ✅ | ⚠️ | ✅ `livekit_client` | High | Best SDK/fan-out; largest rewrite |
| **Janus** | GPLv3 | ❌ plugin-dependent | WebRTC | ⚠️ | ⚠️ | ❌ | High | Building block, not a turnkey server |
| **mediasoup** | ISC | ❌ | WebRTC SFU | ✅ | ⚠️ | ❌ | High | Library, not a server |

### Why go2rtc won

- **No edge rewrite.** It pulls RTSP from MediaMTX, so the Raspberry Pi keeps publishing exactly as it does today.
- **On-demand.** A stream is only pulled when a viewer joins; idle cameras cost nothing.
- **One container.** MIT-licensed, ~10 MB Go binary, multi-arch image.
- **Useful fallbacks.** WHEP/WebRTC for mobile and web, MSE for browsers without WebRTC H.265, HLS as a last resort.
- **Private by default.** Its API can stay on the internal Docker network while only the WebRTC media port is public.

### Why the others were not selected

- **ZLMediaKit** is the strongest alternative if MediaMTX is ever replaced: native RTSP push, WebRTC, H.265, single UDP port, on-demand. It is a larger migration and duplicates MediaMTX's auth/recording role.
- **SRS** has no native RTSP ingest, so every Pi would need to publish RTMP, SRT, or WHIP. That is a bigger change than adding a sidecar.
- **OvenMediaEngine** is excellent for sub-second delivery and transcoding, but AGPL-3.0 and a heavier configuration surface are hard to justify for a pilot.
- **LiveKit self-hosted** is the right upgrade if VCLane later needs SFU fan-out, simulcast, or the polished Flutter SDK. It requires WHIP publishing, a token service, and Redis — the rewrite that was already reverted once (see below).
- **Janus / mediasoup** are building blocks; they would require custom signaling and media handling.

## Hosted Free Tiers

Hosted services are attractive for a pilot, but free allowances are measured in **participant-minutes**. A single camera publishing 24/7 consumes 43,200 minutes per month:

| Service | Free allowance | One 24/7 camera lasts | Typical latency | Notes |
| ------- | -------------- | --------------------- | --------------- | ----- |
| **LiveKit Cloud Build** | 5,000 WebRTC min + 50 GB/month | ~3.5 days | < 500 ms | Hard cap; 100 concurrent participants |
| **Agora** | 10,000 RTC min/month | ~7 days | < 500 ms | Per-minute pricing after allowance |
| **Daily** | 10,000 participant min/month | ~7 days | < 500 ms | Per-participant-minute billing |
| **Mux** | 100,000 delivery min/month | ~69 days | 2–10 s | Live is HLS; not low-latency |
| **Cloudflare Stream** | No free live tier | ~$43/month per viewer | < 1 s WebRTC / 2–10 s HLS | $1 per 1,000 delivered minutes |
| **api.video** | Sandbox, watermarked 30 s | Not usable | — | Evaluation only |
| **YouTube Live** | Free | Unlimited | 5–30 s | Public; wrong for private traffic cameras |

Free tiers are only viable with **on-demand publishing** (the Pi idles until a viewer joins). For 24/7 monitoring, self-hosted is the only sensible free option.

### Bandwidth math

At the current 1 Mbps edge encode:

- 1 Mbps × 86,400 s/day = 86,400 Mb/day ≈ **10.8 GB/day**
- ≈ **324 GB/month per viewer** watching continuously

That is the real cost of a "free" hosted tier: egress. Self-hosting keeps the media on the VPS's included bandwidth, and on-demand pull prevents paying for idle cameras.

## Previous LiveKit Cloud Evaluation

Version 2.11.0 documented a migration to LiveKit Cloud (WHIP ingest, WebRTC rooms, `livekit_client`) and kept MediaMTX for backward compatibility. It was reverted shortly after. The reasons remain valid:

- The Build free tier (5,000 minutes) covers roughly 3.5 days of one 24/7 camera.
- 24/7 publishing would need on-demand `start_publish` / `stop_publish` control over MQTT.
- The migration touched the edge publisher, backend token minting, webhooks, and the Flutter player at once.
- Traffic camera video leaves the self-hosted infrastructure.

go2rtc avoids all four: no edge rewrite, no per-minute billing, no data offload, and a smaller client change.

## Operational Notes

- **Pin the image.** go2rtc is effectively single-maintainer; use a tested tag (e.g. `alexxit/go2rtc:1.9.13`), never `:latest`.
- **Public media port.** WebRTC media needs `8555/udp` and `8555/tcp` reachable. Caddy only carries signaling; it cannot proxy media.
- **NAT and TURN.** With a static public IP, set `webrtc.candidates: [host:8555]`. Behind NAT, use `stun:8555`; symmetric NAT may require a TURN server.
- **Auth boundary.** Keep the go2rtc API private. The backend proxies the WHEP exchange and mints/validates the JWT.
- **Codec.** The edge currently publishes H.264, which every WebRTC client supports. MSE exists for H.265 but is not needed yet.

## Related

- [Media Server](media-server.md) — resulting MediaMTX + go2rtc topology
- [Server Integration](server-integration.md) — compose, config, auth, and endpoint spec
- [Edge Device](edge-device.md) — publish-rate fix required for low latency
- [Mobile App](mobile-app.md) — WHEP playback
