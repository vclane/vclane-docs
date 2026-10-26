# Flutter Mobile Application

The VCLane Flutter mobile app provides the user interface for monitoring and controlling traffic infrastructure. It enables operators to view live camera streams, monitor intersection status, manage edge devices, trigger signal overrides, and coordinate traffic control across multiple intersections. Administrators additionally get a role-gated tools screen for detection and override testing.

## Architecture

The app follows **Clean Architecture** with three layers:

```text
Presentation (Flutter UI + Riverpod)
    │  depends on domain via providers
    ▼
Domain (pure Dart — no Flutter imports)
    │  interfaces consumed by data layer
    ▼
Data (Firebase services, Dio HTTP, repository implementations)
```

### Layer Rules

- **Data** depends only on `core` and `domain` interfaces
- **Domain** has zero Flutter/dependency imports — pure Dart only
- **Presentation** depends on `domain` via providers; never imports `data` directly
- **Core** is cross-cutting: constants, networking, theme, error handling

## Project Structure

```text
lib/
├── app/                        # App entry point, router config
│   ├── app.dart                # MaterialApp.router widget
│   └── router.dart             # go_router route definitions
│
├── core/                       # Shared infrastructure
│   ├── constants/              # API, Firestore, RTDB constants
│   ├── errors/                 # Error handler, failure types
│   ├── network/                # Dio client, auth interceptor
│   ├── theme/                  # Light/dark theme (Material 3)
│   └── utils/                  # Validators, date formatters, stream helpers
│
├── data/                       # Data layer
│   ├── datasources/            # Remote data sources (Firebase Auth/Firestore/RTDB/Storage, Dio API)
│   ├── models/                 # JSON-serializable DTOs with fromFirestore/fromRtdb
│   └── repositories/           # Repository implementations
│
├── domain/                     # Domain layer (pure Dart)
│   ├── entities/               # Immutable domain objects
│   ├── repositories/           # Abstract repository interfaces
│   └── usecases/               # Single-responsibility business logic
│
├── presentation/               # UI layer
│   ├── auth/                   # Login screen
│   ├── admin/                  # Admin tools screen (role-gated)
│   ├── dashboard/              # Dashboard screen
│   ├── devices/                # Device detail screen
│   ├── profile/                # Profile screens
│   ├── providers/              # Riverpod state providers
│   ├── shared/                 # Shared/reusable widgets
│   └── stream/                 # Live stream screen
│
├── firebase_options.dart       # Generated Firebase config
└── main.dart                   # Entry point
```

## Technology Stack

| Component           | Technology                 |
| ------------------- | -------------------------- |
| Framework           | Flutter (Dart)             |
| State Management    | Riverpod                   |
| Routing             | go_router                  |
| Authentication      | Firebase Authentication    |
| Database            | Cloud Firestore            |
| Real-Time Telemetry | Firebase Realtime Database |
| File Storage        | Firebase Storage           |
| HTTP Client         | Dio                        |
| Video Playback      | `flutter_webrtc` (WHEP); `media_kit` RTSP fallback |
| Charts              | fl_chart                   |
| Image Picker        | image_picker               |
| Fonts               | google_fonts (Inter)       |

## Screens

| Screen             | Route                 | Purpose                                                      |
| ------------------ | --------------------- | ------------------------------------------------------------ |
| LoginScreen        | `/login`              | Email/password sign-in with VCLane branding                  |
| DashboardScreen    | `/dashboard`          | Device list grouped by status, quick stats, group breakdown  |
| DeviceDetailScreen | `/devices/:id`        | Full device info, telemetry, stream preview, signal override |
| LiveStreamScreen   | `/devices/:id/stream` | Full-screen WHEP/WebRTC playback (RTSP fallback)             |
| ProfileScreen      | `/profile`            | User profile, preferences, app info, sign out                |
| EditProfileScreen  | `/profile/edit`       | Edit display name, photo, address, phone                     |
| AdminToolsScreen   | `/admin/tools`        | Admin-only detection test + group signal override            |

Shared widgets include `StatusBadge`, `LoadingIndicator`, `ErrorDisplay`, `EmptyState`, and `ContextExtensions` for theme/media query access.

## Live Video Playback

The live stream screen uses WebRTC/WHEP as the primary path, with RTSP retained as a fallback.

### Flow

1. The operator opens `/devices/:id/stream`.
2. The app creates an `RTCPeerConnection` with `flutter_webrtc`.
3. It adds a receive-only video transceiver and creates an SDP offer.
4. It posts the offer to `POST /api/devices/{id}/whep` with the session JWT and `Content-Type: application/sdp`.
5. The backend registers the device with go2rtc, pulls the RTSP feed from MediaMTX on demand, and returns the SDP answer.
6. The app applies the answer, waits for `onTrack`, and renders the stream in `RTCVideoRenderer`.

```dart
final pc = await createPeerConnection({
  'iceServers': [
    {'urls': 'stun:stun.cloudflare.com:3478'},
  ],
});

final transceiver = await pc.addTransceiver(
  kind: RTCRtpMediaType.RTCRtpMediaTypeVideo,
  init: RTCRtpTransceiverInit(direction: TransceiverDirection.RecvOnly),
);

final offer = await pc.createOffer();
await pc.setLocalDescription(offer);

final answer = await api.postSdp('/api/devices/$deviceId/whep', offer.sdp!);
await pc.setRemoteDescription(RTCSessionDescription(answer, 'answer'));

pc.onTrack = (event) {
  if (event.streams.isNotEmpty) {
    renderer.srcObject = event.streams[0];
  }
};
```

### Notes

- Media flows directly from go2rtc on `8555/udp` or `8555/tcp`; it never traverses Caddy or the backend.
- The backend endpoint is JWT-gated, so no go2rtc credentials ship with the app.
- The WHEP exchange is one-shot; if the connection drops, recreate the peer connection and repeat the flow.
- H.264 is published by the edge, so no codec negotiation fallback is required. go2rtc also exposes MSE/HLS for web clients.
- RTSP playback (`media_kit`) remains available for debugging and as a fallback when WebRTC/ICE is blocked.

## Admin Tools

A single admin-only screen at `/admin/tools` (`AdminToolsScreen`) provides operational test controls. It is visible in the profile menu and routable only when `user.role == 'admin'`; the router redirects every other authenticated role to `/dashboard`. The backend independently enforces the role for `forceEnqueue`, so hiding the UI is not the security boundary.

### Detection Test

| Field | Behavior |
| ----- | -------- |
| Device | Dropdown of devices (Firestore stream); the device determines `groupId` |
| Detected class | YOLO label suggestions plus free text |
| Track ID | Generated per send (`admin-{epochMs}`) so each request creates a new doc |
| Device timestamp | Current time (ISO-8601 UTC) |
| Image | Not sent (`null`) — the endpoint accepts an omitted image |
| Force enqueue | Toggle, default off; sent as multipart `forceEnqueue=true` |

The tool posts `multipart/form-data` to `/api/devices/{id}/detections` with `trackId`, `class`, `deviceTimestamp`, and optionally `forceEnqueue`. It then reads the deterministic detection doc to display `enqueued`, `forceEnqueueRequested`, and the group's `enqueueEdgeDetections` snapshot. Repeated sends with Force enqueue on deliberately create duplicate FIFO entries; reusing a track ID/timestamp returns `deduped: true` and does not re-run the gate.

### Override Signal

| Field | Behavior |
| ----- | -------- |
| Group | Dropdown of groups; shows `schedulingMode` and controller availability |
| Phases | One row per `group.deviceIds`, prefilled `RED`; pick any device for `GREEN` |
| Duration | Seconds, default 30; `0` = indefinite; negative rejected by the form |
| Apply | `POST /api/groups/{id}/commands` with `signal_override` |
| Cancel | Same endpoint with `cancel_override` |

The app sends a full phase map for every device in the group so no device falls to caution/RED from omission. The backend returns `202 Accepted` and the controller applies the override asynchronously, so the UI reports "Accepted", not "Applied". A group without a bound controller returns `409`. `YELLOW` is not offered because the 2-channel relay hardware actuates it as RED.

## Data Flow

```text
Firebase Auth/Firestore/RTDB ──► DataSource ──► Repository ──► Provider ──► Widget
    (real-time streams)                                              (ref.watch)
                                                                         │
Dio HTTP (backend API) ──► ApiDataSource ──► Repository ──► FutureProvider ──► Widget
    (one-shot requests)                                           (ref.watch)
```

- **Real-time data** (devices, groups, users, telemetry) flows through `StreamProvider` from Firestore/RTDB → widgets update automatically
- **One-shot requests** (WHEP sessions, admin detection tests, group commands) use `FutureProvider.family`/controller calls via the Dio-based API client
- **Derived state** (grouped device lists, effective readings with overrides) uses plain `Provider<T>`

## Routing & Auth Guard

The app uses go_router with 7 routes and an auth redirect guard:

- Unauthenticated users are redirected to `/login` from any protected route
- Authenticated users on `/login` are redirected to `/dashboard`
- `/admin/*` routes additionally require `user.role == 'admin'`; other authenticated roles are redirected to `/dashboard`
- All non-dashboard routes push full-screen using a root navigator key

```text
AuthState: Authenticated | Unauthenticated | AuthLoading | AuthError
```

## State Management (Riverpod)

All providers are handwritten (no code generation). Key patterns:

| Pattern                 | Example                                         |
| ----------------------- | ----------------------------------------------- |
| `StreamProvider`        | Real-time device list, group list               |
| `StreamProvider.family` | Single device by ID, telemetry by device        |
| `FutureProvider.family` | WHEP session by device ID                        |
| `Provider<T>` (derived) | Auth state, grouped devices, effective readings |

No `autoDispose` is used — all providers are global singletons for the app lifetime.

## Domain Entities

| Entity           | Key Fields                                                                    |
| ---------------- | ----------------------------------------------------------------------------- |
| `User`           | id, email, displayName, role (`admin` \| `operator` \| `viewer`, default `operator`) |
| `Device`         | id, name, location, status (online/offline/warning), firmwareVersion, rtspUrl |
| `Group`          | id, name, description, location, `enqueueEdgeDetections`                      |
| `Telemetry`      | currentTrafficLightSignal, emergencyVehicleDetected, emergencyClass, emergencyConfidence, timestamp (no counts — history via `Detection`) |
| `SignalOverride` | deviceId, targetSignal, requestedAt                                           |
| `Detection`      | deviceId, groupId, timestamp (server), deviceTimestamp, trackId, class (any YOLO label), imageUrl (`null` when no image), enqueued |

> **Note:** Online/offline state and last-seen timestamps are tracked in the Realtime Database at `/telemetry/{deviceId}`, not in Firestore. The `online` field is set to `true` on every heartbeat, telemetry, or retained `status` online message, and to `false` via the retained device `status` topic (graceful shutdown or broker-published LWT). There is no backend timeout checker. This avoids excessive Firestore writes from frequent updates.

> **Note:** `vehicleToday` was removed from RTDB telemetry. "Vehicles detected today" is a direct Firestore query on `detections` (no REST read path):
>
> ```dart
> detections
>   .where('deviceId', isEqualTo: id)
>   .where('timestamp', isGreaterThanOrEqualTo: startOfDay)
>   .orderBy('timestamp', descending: true)
> ```
>
> Count = number of docs; filter on `class` for breakdowns, `imageUrl` for thumbnails.

## Implementation Notes

- **Signal overrides** are executed through the backend: device-detail overrides use `POST /api/devices/{id}/signal`, and the admin tool uses `POST /api/groups/{id}/commands`. The former in-memory mock map is no longer used.
- **Video playback** uses WHEP via `flutter_webrtc` (see [Live Video Playback](#live-video-playback)). `media_kit` RTSP remains as a fallback/debug path; `rtspUrl` stays on the `Device` entity for that purpose.
- **Use cases** exist in the domain layer (`LoginUseCase`, `GetDevicesUseCase`, etc.) but are not wired into providers — providers call repositories directly. This is a transitional pattern.
- **Charts** (`fl_chart`) is a dependency but no chart widgets have been implemented yet.
- **Firebase Cloud Messaging** is not yet wired in.

## Environment & Setup

- Dart SDK `^3.12.2`
- Flutter stable
- Firebase initialized via `firebase_options.dart` (Android + iOS)
- Google Fonts loads Inter at runtime (requires network)
