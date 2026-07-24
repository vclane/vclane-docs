# Flutter Mobile Application

The VC-LANE Flutter mobile app provides the user interface for monitoring and controlling traffic infrastructure. It enables operators to view live camera streams, monitor intersection status, manage edge devices, trigger signal overrides, and coordinate traffic control across multiple intersections.

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

| Component              | Technology                    |
| ---------------------- | ----------------------------- |
| Framework              | Flutter (Dart)                |
| State Management       | Riverpod                      |
| Routing                | go_router                     |
| Authentication        | Firebase Authentication       |
| Database               | Cloud Firestore               |
| Real-Time Sensor Data  | Firebase Realtime Database     |
| File Storage           | Firebase Storage              |
| HTTP Client            | Dio                           |
| Video Playback         | media_kit (RTSP/WebRTC)       |
| Charts                 | fl_chart                      |
| Image Picker           | image_picker                  |
| Fonts                  | google_fonts (Inter)          |

## Screens

| Screen             | Route                  | Purpose |
|--------------------|------------------------|---------|
| LoginScreen        | `/login`               | Email/password sign-in with VC-LANE branding |
| DashboardScreen    | `/dashboard`           | Device list grouped by status, quick stats, group breakdown |
| DeviceDetailScreen | `/devices/:id`         | Full device info, sensor readings, stream preview, signal override |
| LiveStreamScreen   | `/devices/:id/stream`  | Full-screen RTSP video playback |
| ProfileScreen      | `/profile`             | User profile, preferences, app info, sign out |
| EditProfileScreen  | `/profile/edit`        | Edit display name, photo, address, phone |

Shared widgets include `StatusBadge`, `LoadingIndicator`, `ErrorDisplay`, `EmptyState`, and `ContextExtensions` for theme/media query access.

## Data Flow

```text
Firebase Auth/Firestore/RTDB ──► DataSource ──► Repository ──► Provider ──► Widget
    (real-time streams)                                              (ref.watch)
                                                                         │
Dio HTTP (backend API) ──► ApiDataSource ──► Repository ──► FutureProvider ──► Widget
    (one-shot requests)                                           (ref.watch)
```

- **Real-time data** (devices, groups, users, sensor readings) flows through `StreamProvider` from Firestore/RTDB → widgets update automatically
- **One-shot requests** (stream URLs) use `FutureProvider.family` via the Dio-based API client
- **Derived state** (grouped device lists, effective readings with overrides) uses plain `Provider<T>`

## Routing & Auth Guard

The app uses go_router with 6 routes and an auth redirect guard:

- Unauthenticated users are redirected to `/login` from any protected route
- Authenticated users on `/login` are redirected to `/dashboard`
- All non-dashboard routes push full-screen using a root navigator key

```text
AuthState: Authenticated | Unauthenticated | AuthLoading | AuthError
```

## State Management (Riverpod)

All providers are handwritten (no code generation). Key patterns:

| Pattern | Example |
|---------|---------|
| `StreamProvider` | Real-time device list, group list |
| `StreamProvider.family` | Single device by ID, sensor readings by device |
| `FutureProvider.family` | Stream URL by device ID |
| `Provider<T>` (derived) | Auth state, grouped devices, effective readings |

No `autoDispose` is used — all providers are global singletons for the app lifetime.

## Domain Entities

| Entity | Key Fields |
|--------|------------|
| `User` | id, email, displayName, role ('operator') |
| `Device` | id, name, location, status (online/offline/warning), firmwareVersion, rtspUrl |
| `Group` | id, name, description, location |
| `SensorReading` | vehicleCount, currentTrafficLightSignal, timestamp |
| `SignalOverride` | deviceId, targetSignal, requestedAt |

> **Note:** `lastSeen` was previously stored in Firestore as a `Timestamp` on the Device document. It has been removed from Firestore. The "Last seen" timestamp is now derived from the `timestamp` field in `SensorReading`, which is stored in the Realtime Database at `sensors/{deviceId}`. This avoids excessive Firestore writes from frequent heartbeat updates.

## Implementation Notes

- **Signal overrides** use an in-memory mock map. TODO: replace with backend API call when the endpoint is available.
- **Video playback** with `media_kit` is stubbed. The `VideoPlayerWidget` shows a placeholder with the RTSP URL and a no-op connect button. Full integration is pending.
- **Use cases** exist in the domain layer (`LoginUseCase`, `GetDevicesUseCase`, etc.) but are not wired into providers — providers call repositories directly. This is a transitional pattern.
- **Charts** (`fl_chart`) is a dependency but no chart widgets have been implemented yet.
- **Firebase Cloud Messaging** is not yet wired in.

## Environment & Setup

- Dart SDK `^3.12.2`
- Flutter stable
- Firebase initialized via `firebase_options.dart` (Android + iOS)
- Google Fonts loads Inter at runtime (requires network)
