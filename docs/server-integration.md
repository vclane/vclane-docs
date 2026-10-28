# Server Integration

The VCLane server is a Docker Compose stack of five services — Caddy, Backend (FastAPI), MediaMTX, go2rtc, and Mosquitto — that work together to manage edge devices, relay video streams, and handle real-time communication. MediaMTX ingests RTSP from edge devices; go2rtc pulls those streams on demand and delivers them to viewers over WebRTC/WHEP.

## Backend API

The backend acts as the centralized management and coordination layer between users, edge devices, and cloud services.

### Architecture

```mermaid
graph TB
    subgraph Server["VCLane Server (Docker Compose)"]
        Caddy["Reverse Proxy — Caddy<br/>(TLS termination)"]
        Backend["Backend API — FastAPI<br/>(Auth / Commands / Telemetry / WHEP proxy)"]
        Media["Media Server — MediaMTX<br/>(RTSP ingest / auth)"]
        RTC["WebRTC Delivery — go2rtc<br/>(on-demand WHEP / MSE)"]
        MQTT["MQTT Broker — Mosquitto<br/>(TLS + Dynamic Security)"]

        Caddy -->|"/api/* /ws/* /health"| Backend
        Backend -->|"Stream registration + SDP proxy"| RTC
        Backend -->|"Auth query"| Media
        RTC -->|"RTSP pull (on demand)"| Media
    end

    Clients["Operators<br/>(Mobile / Web)"] -->|"HTTPS /api/*"| Caddy
    Clients -->|"WebRTC media :8555"| RTC
    Backend -->|"Auth / DB"| Firebase["Firebase Services<br/>(Auth / Firestore / RTDB)"]
    MQTT -->|"Commands / Sync"| Edge["Raspberry Pi Edge Devices"]
    Edge -->|"Telemetry / Heartbeat / Ack"| MQTT
    Edge -->|"RTSP Stream"| Media
```

### Responsibilities

- User authentication
- Device registration
- Device management
- Traffic monitoring
- Telemetry processing
- Signal override handling
- Intersection grouping
- Traffic synchronization
- MQTT command publishing
- Firebase synchronization

### Technology Stack

| Component            | Technology                          |
| -------------------- | ----------------------------------- |
| Programming Language | Python                              |
| Backend Framework    | FastAPI                             |
| API Architecture     | REST API + WebSocket                |
| MQTT Integration     | Paho MQTT Client                    |
| MQTT Broker          | Mosquitto                           |
| Authentication       | Firebase Admin SDK + JWT + bcrypt   |
| Database             | Firestore                           |
| Real-Time Telemetry  | Realtime Database                   |
| Media Server         | MediaMTX (RTSP ingest) + go2rtc (WebRTC) |
| Reverse Proxy        | Caddy                               |
| API Documentation    | OpenAPI / Swagger                   |
| Containerization     | Docker + Docker Compose             |
| Deployment           | Linux Server / Cloud Infrastructure |

### Key Features

**Device Management** — Register edge devices, manage device configurations, track device status, monitor device health, assign devices to intersections.

**Traffic Monitoring** — Receive real-time telemetry, monitor traffic conditions, store traffic information, generate traffic statistics, detect traffic events.

**Traffic Control** — Send traffic signal commands, trigger manual overrides, manage intersection schedules, coordinate multiple intersections, process command responses.

**Real-Time Synchronization** — MQTT command publishing, device heartbeat processing, WebSocket client updates, event notifications.

## Backend ↔ MediaMTX Authentication

MediaMTX is configured to delegate authentication to the backend via HTTP callbacks.

### Configuration

```yaml
# provisions/mediamtx/mediamtx.dev.yml
authMethod: http
authHTTPAddress: http://backend:8000/api/mediamtx/auth
authHTTPExclude: []
```

On every RTSP publish or play action, MediaMTX issues an HTTP `POST` to the backend with a JSON body:

```json
{
  "user": "{user}",
  "password": "{password}",
  "action": "publish|read",
  "path": "{path}",
  "protocol": "rtsp",
  "ip": "{clientIp}"
}
```

The current backend model reads `user`, `password`, `action`, and `path`; `ip` and `protocol` are available if tighter authorization is added later.

### Auth Flow

The backend tries three strategies in order:

1. **go2rtc service account** — `user` equals `GO2RTC_MEDIA_USER`, `action` is `read`, and the password matches `GO2RTC_MEDIA_SECRET` (constant-time compare). Used by go2rtc pulling from MediaMTX.
2. **JWT session auth** — treats `user` as a session token, verified via `verify_session_token()`. Used by human operators reading RTSP directly (fallback/debug).
3. **Device secret auth** — looks up `devices/{deviceId}` in Firestore, retrieves `deviceSecretHash`, verifies the `password` field with bcrypt. Used by edge devices publishing streams.

All three strategies return `200 OK` on success or `403 Forbidden` on failure. No path-level or action-level authorization is enforced beyond credential verification.

### RTSP URL Format

When a device is provisioned, it receives an RTSP URL of the form:

```
rtsp://{deviceId}@mediamtx:8554/{deviceId}
```

- The `deviceId` serves as the RTSP username (sent as `user` in the auth callback).
- The `deviceSecret` serves as the RTSP password (sent as `password`).
- The path (same as `deviceId`) is used by MediaMTX to identify the stream.

This URL is **unchanged** by the go2rtc addition. The edge keeps publishing to MediaMTX; go2rtc reads from MediaMTX using a separate internal service-account URL (`rtsp://{GO2RTC_MEDIA_USER}:{GO2RTC_MEDIA_SECRET}@mediamtx:8554/{deviceId}`) that is never exposed to devices.

---

## Backend ↔ go2rtc (WebRTC Delivery)

go2rtc is the low-latency delivery plane. It pulls RTSP from MediaMTX **on demand** and serves WebRTC/WHEP to viewers. The backend never proxies media; it registers the stream and forwards the SDP offer/answer.

### Docker Compose

Add one service per profile, mirroring the MediaMTX pattern. The dev service is shown; `go2rtc-prod` uses `mediamtx-prod`, `backend-prod`, and the `prod` profile.

```yaml
go2rtc:
  image: alexxit/go2rtc:1.9.13        # pin a tested tag; never :latest
  profiles:
    - dev
  command:
    - "go2rtc"                       # image entrypoint is tini; include the binary
    - "-config"
    - "/config/runtime/go2rtc.yml"   # writable config; ConfigPath for PUT/DELETE
    - "-config"
    - "/config/go2rtc.yml"           # read-only base
  volumes:
    - ./provisions/go2rtc/go2rtc.yml:/config/go2rtc.yml:ro
    - go2rtc_runtime:/config/runtime
  environment:
    GO2RTC_API_USER: ${GO2RTC_API_USER}
    GO2RTC_API_PASS: ${GO2RTC_API_PASS}
    GO2RTC_MEDIA_USER: ${GO2RTC_MEDIA_USER}
    GO2RTC_MEDIA_SECRET: ${GO2RTC_MEDIA_SECRET}
    GO2RTC_CANDIDATE: ${GO2RTC_CANDIDATE}   # public media host:port
  ports:
    - "8555:8555/tcp"
    - "8555:8555/udp"
  depends_on:
    - mediamtx
  networks:
    - vclane
```

Add `go2rtc_runtime:` to the top-level `volumes:` block.

- The image entrypoint is `tini --`, so `command` **must** start with the `go2rtc` binary. Without it, tini tries to exec `-c` and the container exits `127` before go2rtc starts.
- Do **not** publish `1984`; the backend reaches it over the Docker network.
- `mediamtx` keeps only `8554:8554`.
- Add the five variables to `.env` and `.env.example`.
- go2rtc is a multi-arch image; no special build stage is required.
- The runtime config **must be writable**: `PUT`/`DELETE /api/streams` call `PatchConfig`, which writes to the first `-c` file. A read-only config makes those requests return `400` after mutating memory. The base config is mounted read-only and the runtime file lives in a named volume.

### go2rtc Configuration

```yaml
# provisions/go2rtc/go2rtc.yml
api:
  listen: ":1984"
  username: ${GO2RTC_API_USER}
  password: ${GO2RTC_API_PASS}
  local_auth: true

rtsp:
  listen: ""            # ingestion belongs to MediaMTX

webrtc:
  listen: ":8555"
  candidates:
    - ${GO2RTC_CANDIDATE}   # e.g. vclane.devistlab.com:8555
  ice_servers:
    - urls: [stun:stun.cloudflare.com:3478]
```

Notes:

- `rtsp.listen: ""` prevents a second, unauthenticated RTSP server from appearing.
- `candidates` must advertise an address viewers can reach. Set `GO2RTC_CANDIDATE` to the server's public media host (for this deployment, `vclane.devistlab.com:8555`) or its public IP. Do **not** use a placeholder web domain: a wrong value makes clients try an unreachable ICE host candidate and fall back to STUN or fail. Use `stun:8555` when the public IP is dynamic; symmetric NAT may require TURN.
- `local_auth: true` closes the loopback bypass.
- Do **not** define a `streams:` key in this base file. Runtime entries live in `/config/runtime/go2rtc.yml`, which is loaded first and is the file `PatchConfig` writes. A `streams:` map here would override the runtime entries on load.
- Runtime entries persist in the `go2rtc_runtime` volume across restarts. Delete stale entries via the API or remove the volume.
- The runtime config contains the media service-account secret; treat the `go2rtc_runtime` volume as sensitive.

### Backend Configuration

```python
# backend/app/config.py
go2rtc_url: str = "http://go2rtc:1984"
go2rtc_api_user: str = ""
go2rtc_api_pass: str = ""
go2rtc_media_user: str = "go2rtc"
go2rtc_media_secret: str = ""
```

### MediaMTX Service Account

The existing `/api/mediamtx/auth` callback gains a reader branch before the device-secret check:

```python
if (
    body.user == settings.go2rtc_media_user
    and body.action == "read"
    and secrets.compare_digest(body.password, settings.go2rtc_media_secret)
):
    return Response(status_code=200)
```

- Keep `authHTTPExclude: []`.
- Do **not** exclude `action: read` globally — that would allow anonymous reads.
- MediaMTX does not support HTTP and internal auth simultaneously (bluenviron/mediamtx#4659), so a shared reader account is the supported approach.
- The comparison uses `secrets.compare_digest` to avoid timing leaks.

### WHEP Endpoint

`POST /api/devices/{deviceId}/whep`

| Aspect | Value |
| ------ | ----- |
| Auth | Bearer session JWT (`get_current_user`) |
| Request | `Content-Type: application/sdp`, raw WHEP offer |
| Success | `201`, `Content-Type: application/sdp`, raw SDP answer |
| Errors | `404` unknown device · `502` go2rtc failure · `401/403` auth |

Flow:

1. Verify the session and that the device exists.
2. Ensure the go2rtc stream exists (idempotent):
   ```
   PUT {GO2RTC_URL}/api/streams
       ?name={deviceId}
       &src=rtsp://{GO2RTC_MEDIA_USER}:{urlquote(GO2RTC_MEDIA_SECRET)}@mediamtx:8554/{deviceId}
   ```
   `PUT` creates or replaces the named stream, so repeating the call is safe.
3. Forward the offer with Basic auth:
   ```
   POST {GO2RTC_URL}/api/webrtc?src={deviceId}
   Content-Type: application/sdp
   Authorization: Basic base64(GO2RTC_API_USER:GO2RTC_API_PASS)
   ```
4. Return go2rtc's `201` SDP answer unchanged.

Cleanup: `DELETE /api/devices/{id}` also calls `DELETE {GO2RTC_URL}/api/streams?src={id}` best-effort. No token table or TTL is required because the endpoint is JWT-gated and the go2rtc API is private.

Rejected alternative: exposing `1984` and using Caddy `forward_auth`. It either hands clients the go2rtc Basic-auth credentials or requires path rewriting plus an exposed stream list.

### Ports and Firewall

| Port | Service | Exposure | Purpose |
| ---- | ------- | -------- | ------- |
| `8554/tcp` | MediaMTX | Edge devices / VPN | RTSP publish |
| `8555/tcp` + `8555/udp` | go2rtc | Public | WebRTC media |
| `1984/tcp` | go2rtc | Internal only | API, stream registration, WHEP signaling |
| `8889/tcp` | MediaMTX | Internal only | Native WebRTC (legacy/debug) |
| `443/tcp` | Caddy | Public | `/api/*`, `/ws/*`, `/health` |

Caddy must not be used for media. Only the SDP exchange traverses `/api/*`.

### Acceptance Criteria

- Two clients watch the same device concurrently through `/api/devices/{id}/whep`.
- ICE selects the `8555` candidate; no media appears in Caddy logs.
- `GET go2rtc:1984/api/streams` shows consumers return to `0` within 10 s of the last viewer leaving.
- Restarting go2rtc reloads the runtime streams; no backend restart is required.
- A 24 h join/leave soak shows no leaked producers/consumers and no file-descriptor growth.
- `1984` is unreachable from outside the Docker network.
- `authHTTPExclude` remains `[]`, and a read with wrong credentials still returns `403`.

### Failure Modes

| Symptom | Likely cause | Action |
| ------- | ------------ | ------ |
| WHEP `502` | go2rtc down or stream registration failed | Check `go2rtc` logs and `GET /api/streams` |
| ICE connects, no video | `8555` blocked or wrong `candidates` | Verify UDP/TCP firewall and public host |
| Read `403` from MediaMTX | service-account secret mismatch | Compare `.env` values and the callback branch |
| Stream disappears after restart | runtime config missing or not writable | Check the `go2rtc_runtime` volume and `-c` order |
| High latency despite WHEP | edge publish drift | Fix the edge publisher (see [Edge Device](edge-device.md#stream-publishing-pipeline)) |

---

## Backend ↔ MQTT (Mosquitto)

The backend connects to Mosquitto as an internal MQTT client to receive telemetry from devices and publish commands.

### Connection

- Client ID: `vclane-backend-{pid}-{random}` (one client per uvicorn worker)
- Host: `mosquitto:1883` (internal Docker network, no TLS)
- Password: `vclane-backend` — matches the admin client bootstrapped in the Dynamic Security config
- The `admin` role grants access to all traffic topics plus `$CONTROL/dynamic-security/#` for Dynamic Security administration
- Connect is asynchronous (`connect_async` + `loop_start`) with a 1–30 s exponential reconnect backoff; a broker restart, network drop, or slow broker start is retried indefinitely
- On every successful (re)connect the client re-subscribes to all topics listed below
- Outbound publishes use QoS 1, are queued locally (max 1000 messages) while disconnected, and their return code is checked and logged on failure
- `GET /health` includes `mqtt.connected` so a deaf backend is visible

### Subscriptions

On connect, the backend subscribes to the wildcard topics below plus the Dynamic Security response topic:

| Topic Pattern                       | Handler                   | Destination                                                                                                |
| ----------------------------------- | ------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `traffic/device/+/telemetry`        | `_handle_telemetry`       | Writes to RTDB at `/telemetry/{deviceId}` — injects `online: true` and `lastSeen` (no queue gate)          |
| `traffic/device/+/heartbeat`        | `_handle_heartbeat`       | Same as telemetry (no queue gate)                                                                          |
| `traffic/device/+/status`           | `_handle_device_status`   | Retained LWT/status topic — `online: true` refreshes `online` + `lastSeen`; `online: false` marks the device offline |
| `traffic/device/+/goodbye`          | `_handle_goodbye`         | Sets `online: false` on `/telemetry/{deviceId}` (legacy firmware path)                                     |
| `traffic/device/+/acknowledgements` | `_handle_ack`             | Logs the payload — no persistence or callback                                                              |
| `traffic/group/+/queue-done`        | `_handle_queue_done`      | Processes controller `queueDoneDeviceId` → clears `has_queued_schedule`                                    |
| `traffic/group/+/prepare-response`  | _(no handler)_            | Subscribed but not processed                                                                               |

All inbound handlers parse the topic to extract `deviceId` (third segment) and route accordingly.

### Publish (Outbound)

| Trigger                           | Topic                          | Payload                                                                              |
| --------------------------------- | ------------------------------ | ------------------------------------------------------------------------------------ |
| `POST /api/devices/{id}/commands` | `traffic/device/{id}/commands` | `{"command": "...", "payload": {...}}`                                               |
| `POST /api/devices/{id}/signal`   | `traffic/device/{id}/commands` | `{"command": "signal_override", "payload": {"targetSignal": "..."}}`                 |
| `POST /api/groups/{id}/commands`  | `traffic/group/{id}/commands`  | `{"command": "signal_override", "payload": {"phases": {...}, "durationSeconds": N}}` |
| `POST /api/groups/{id}/sync`      | `traffic/group/{id}/prepare`   | `{"deviceIds": ["id1", "id2", ...]}`                                                 |
| Dynamic queue gate (auto)         | `traffic/group/{gid}/commands` | `{"command": "enqueue", "payload": {"deviceId": "..."}}` — admin `forceEnqueue` bypasses the gate and `enqueueEdgeDetections` |

---

## Backend ↔ Edge Devices (via MQTT)

### Device Provisioning Flow

1. Admin calls `POST /api/devices` (JWT required) with `{deviceId, name, location}`.
2. Backend generates a device secret: `vcl-{32 hex chars}`.
3. Secret is bcrypt-hashed and stored in Firestore at `devices/{deviceId}.deviceSecretHash`.
4. RTSP URL is constructed: `rtsp://{deviceId}@mediamtx:8554/{deviceId}`.
5. Mosquitto user is created via two Dynamic Security RPCs: `publish_dynsec("createClient", ...)` to create the MQTT client, then `publish_dynsec("addClientRole", ...)` to assign the `device` role.
6. Response returns `{deviceId, deviceSecret, rtspUrl}` — the plaintext secret is only returned once.

### Device Authentication (End-to-End)

```mermaid
sequenceDiagram
    participant Edge as Edge Device
    participant MQTT as Mosquitto
    participant MTX as MediaMTX
    participant API as Backend API
    participant DB as Firestore

    Edge->>+MQTT: MQTT connect<br/>(deviceId, deviceSecret)
    MQTT->>MQTT: verify via Dynamic Security Plugin
    MQTT-->>-Edge: Role-based ACL granted<br/>(own device topics + group topics)

    Edge->>+MTX: RTSP publish<br/>rtsp://deviceId@mediamtx:8554/deviceId
    MTX->>+API: POST /api/mediamtx/auth<br/>{user, password, action:"publish", path}
    Note over API: go2rtc account? no → JWT? no → device auth

    API->>+DB: read devices/{deviceId}.deviceSecretHash
    DB-->>-API: bcrypt hash
    API->>API: bcrypt.checkpw(secret, hash)

    alt valid credentials
        API-->>-MTX: 200 OK
        MTX-->>-Edge: Stream accepted

        loop Device lifecycle
            Edge->>MQTT: telemetry/heartbeat
            MQTT->>API: forward to handler
            API->>DB: write to RTDB /telemetry/{id}
            API->>MQTT: publish commands to topic
            MQTT->>Edge: deliver command
            Edge->>MQTT: acknowledgement
        end
    else invalid credentials
        API-->>MTX: 403 Forbidden
        MTX-->>Edge: Stream rejected
    end
```

### Command Formats

**Signal override** (published on device creation or via `POST /api/devices/{id}/signal`):

```json
{
  "command": "signal_override",
  "payload": {
    "targetSignal": "green"
  }
}
```

The backend also writes a Firestore document to `signalOverrides/{id}` with `{deviceId, targetSignal, requestedBy, status: "active", requestedAt, expiresAt}` (expires in 1 hour). Expiration is not enforced by the backend — the edge device is expected to honor it.

**Group command (signal override / cancel)** — published on `POST /api/groups/{id}/commands` to `traffic/group/{id}/commands`:

```json
{ "command": "signal_override", "payload": { "phases": { "device-001": "GREEN", "device-002": "RED" }, "durationSeconds": 30 } }
{ "command": "cancel_override", "payload": {} }
```

`durationSeconds` `0` is indefinite; a positive value is a fixed duration; a negative value causes the controller to reject the whole command. The ESP-32 pauses its schedule turn-loop (or the dynamic queue) while an override is active and uses the override phases in its place. Device IDs omitted from `phases` are not broadcast, so those edge devices enter caution/solid RED; the device-level `POST /api/devices/{id}/signal` path fills the rest of the group with the inverse signal. On expiry/cancel the controller resumes the same turn (or the interrupted queue phase) but restarts its full duration — remaining time is not preserved.

Group commands are fire-and-forget: any authenticated user may send them, the backend returns `202 {"status": "sent", "groupId": ...}` without validating `command`/`payload`, and no `signalOverrides` audit document is written (only the device-level `/signal` endpoint writes one). Detection-triggered emergency overrides are recorded only as `signalOverrideSent` on the detection doc.

**Queue enqueue (dynamic mode)** — published automatically by the queue gate to `traffic/group/{groupId}/commands`:

```json
{ "command": "enqueue", "payload": { "deviceId": "intersection-01" } }
```

The controller appends exactly one FIFO entry per `enqueue` and does **not** dedupe: duplicates and the currently-GREEN device are accepted, so a queue of `cam1 | cam1 | cam2` serves all three in order. Only an empty `deviceId` or a full queue (16 entries) is rejected. Normal edge detections are deduped upstream via `has_queued_schedule`; an admin `forceEnqueue` bypasses that dedup. Emergency-class detections never enter the queue — the backend publishes `signal_override` instead (detecting device `GREEN`, siblings `RED`, 60 s) with a device-level `targetSignal: "GREEN"` fallback when no controller is bound.

**Group sync** (published on `POST /api/groups/{id}/sync`):

```json
{
  "deviceIds": ["intersection-01", "intersection-02"]
}
```

Edge devices in the group respond on `traffic/group/{groupId}/prepare-response`.

### Dynamic Queue Gate (per-group `schedulingMode: "dynamic"`)

No new topics — the gate triggers off `POST /api/devices/{id}/detections` (new docs only). Telemetry never triggers enqueue.

**Authorization:** `forceEnqueue=true` requires `users/{uid}.role == "admin"` (the JWT carries only `sub`); otherwise the endpoint returns `403` before writing anything.

**On `POST /api/devices/{deviceId}/detections` (new doc, not `deduped:true`):**

```
group = Firestore lookup via devices/{deviceId}.groupId (already resolved)
if class in emergency allowlist:
    if now - deviceTimestamp > 180s: return     # stale detection → store only
    publish signal_override                     # GREEN here, RED siblings, 60 s
    return
if group is None or group.schedulingMode != "dynamic": return  # store only
if request.forceEnqueue:
    # wins over enqueueEdgeDetections and skips every check below
    enqueue
elif group.enqueueEdgeDetections is false:
    return                              # audit-only: store the detection
else:
    if class not in vehicle allowlist: return   # pedestrians must not queue
    if now - deviceTimestamp > 180s: return     # stale offline retry → store only
    if currentTrafficLightSignal not in {RED, CAUTION}: return  # already served
    if has_queued_schedule == true: return      # dedup a pending/ongoing serve
    enqueue

override (emergency):
    if group has controllerId:
        publish {"command": "signal_override", "payload": {"phases": {deviceId: "GREEN", <siblings>: "RED"}, "durationSeconds": 60}}
            to traffic/group/{groupId}/commands
    else:
        publish {"command": "signal_override", "payload": {"targetSignal": "GREEN", "durationSeconds": 60}}
            to traffic/device/{deviceId}/commands
    no RTDB write

enqueue:
    publish {"command": "enqueue", "payload": {"deviceId"}} to traffic/group/{groupId}/commands
    RTDB update /telemetry/{deviceId}: {has_queued_schedule: true, queuedAt: now}
```

A gate failure is logged and never changes the `201`; the detection's `enqueued` / `signalOverrideSent` audit fields record the outcome. `forceEnqueue` still requires `schedulingMode: "dynamic"` — against a `fixed` group it stores only, because the controller ignores `enqueue` outside dynamic mode. Emergency overrides are mode-independent and still require a fresh `deviceTimestamp`.

**On `traffic/group/{groupId}/queue-done` with `queueDoneDeviceId`:**

```
RTDB update /telemetry/{queueDoneDeviceId}: {has_queued_schedule: false, queuedAt: null}
```

When the controller's FIFO empties it broadcasts all-RED; no backend action is needed for the idle state. `has_queued_schedule` is a per-device boolean, not a per-entry counter: with duplicate forced entries the first completion clears it while another duplicate may still be pending.

---

## MQTT Topic Reference

| Direction            | Topic                                  | Publisher         | Consumer                      | Payload                                                                               |
| -------------------- | -------------------------------------- | ----------------- | ----------------------------- | ------------------------------------------------------------------------------------- |
| Device → Backend     | `traffic/device/{id}/telemetry`        | Edge device       | Backend (→RTDB)               | Lean telemetry JSON (signal + emergency flag; no counts, no queue trigger) |
| Device → Backend     | `traffic/device/{id}/heartbeat`        | Edge device       | Backend (→RTDB)               | Arbitrary JSON                                                                        |
| Device → Backend     | `traffic/device/{id}/acknowledgements` | Edge device       | Backend (logged)              | Arbitrary JSON                                                                        |
| Device → Backend     | `traffic/group/{id}/prepare-response`  | Edge device       | Backend (unhandled)           | Arbitrary JSON                                                                        |
| Controller → Backend | `traffic/group/{id}/schedule-request`  | ESP-32 controller | Backend                       | Empty (on connect)                                                                    |
| Controller → Backend | `traffic/group/{id}/queue-done`        | ESP-32 controller | Backend (queue clear)         | `queueDoneDeviceId`, `queueDepth`                                                     |
| Controller → Backend | `traffic/group/{id}/heartbeat`         | ESP-32 controller | Backend                       | Uptime + timestamp JSON                                                               |
| Backend → Controller | `traffic/group/{id}/schedule`          | Backend           | ESP-32 controller             | Schedule JSON (turns loop + schedulingMode + dynamicConfig)                           |
| Backend → Controller | `traffic/group/{id}/commands`          | Backend           | ESP-32 controller             | `{"command": str, "payload": {}}` (`signal_override` / `cancel_override` / `enqueue`) |
| Backend → Device     | `traffic/device/{id}/commands`         | Backend           | Edge device                   | `{"command": str, "payload": {}}`                                                     |
| Backend → Devices    | `traffic/group/{id}/prepare`           | Backend           | Edge devices in group         | `{"deviceIds": [str]}`                                                                |

### Mosquitto Dynamic Security

Mosquitto uses the **Dynamic Security Plugin** instead of static ACL files. Two roles are bootstrapped at image build:

- **`admin`** — assigned to `vclane-backend`, grants read/write on all `traffic/device/+/#` and `traffic/group/+/#` topics plus CONTROL topics.
- **`device`** — applied per device, grants read/write on `traffic/device/%u/#` and `traffic/group/+/#` topics.

#### RPC Format

Dynamic Security commands are sent as MQTT RPCs on `$CONTROL/dynamic-security/v1`. The payload must wrap the command in a `commands` array:

```json
{
  "commands": [
    {
      "command": "createClient",
      "username": "intersection-01",
      "password": "vcl-abc123...",
      "textname": "intersection-01"
    }
  ]
}
```

A single `"command"` key at the top level (without the `commands` array) will cause an `"Unknown command"` / `"Invalid/missing commands"` error on Mosquitto 2.1.2.

Success responses omit both `"success"` and `"error"` keys — only the response fields (e.g., `"clients"` for `listClients`). Failure responses include an `"error"` key.

#### Provisioning Flow

When a device is provisioned via `POST /api/devices`, the backend sends **two** RPCs:

1. `createClient` — registers the MQTT client with username/password
2. `addClientRole` — assigns the `device` role to the newly created client

On deprovisioning (`DELETE /api/devices/{id}`), a single `deleteClient` RPC removes the client and its role assignments.

#### Docker Volume Persistence

The `mosquitto_data` Docker volume at `/mosquitto/data` in the container persists the Dynamic Security database file (`dynamic-security.json`) across container restarts. To apply a fresh image config (e.g., after rebuilding with new roles/ACLs), the volume must be deleted:

```bash
docker compose down mosquitto
docker volume rm vclane-server_mosquitto_data
docker compose up -d mosquitto
```

---

## Integration API Endpoints

| Method   | Path                           | Auth     | Purpose                          | Side-Effect                                                      |
| -------- | ------------------------------ | -------- | -------------------------------- | ---------------------------------------------------------------- |
| `POST`   | `/api/devices`                 | JWT      | Create device + provision secret | `add_mosquitto_user()` via Dynamic Security RPC                  |
| `DELETE` | `/api/devices/{id}`            | JWT      | Delete device                    | `remove_mosquitto_user()` via Dynamic Security RPC               |
| `POST`   | `/api/devices/{id}/commands`   | JWT      | Send arbitrary command           | MQTT publish to `traffic/device/{id}/commands`                   |
| `POST`   | `/api/devices/{id}/signal`     | JWT      | Override traffic signal          | MQTT publish + Firestore write                                   |
| `POST`   | `/api/devices/{id}/detections` | JWT      | Submit per-vehicle detection (admin-only `forceEnqueue`) | Optional Storage upload + Firestore `detections` write; may publish `enqueue` and set RTDB queue flags |
| `POST`   | `/api/groups/{id}/sync`        | JWT      | Trigger group prepare            | MQTT publish to `traffic/group/{id}/prepare`                     |
| `POST`   | `/api/devices/{id}/whep`       | JWT      | Exchange WHEP offer for an SDP answer | Registers the go2rtc stream on demand; returns SDP; no MQTT/RPC side-effect |
| `POST`   | `/api/mediamtx/auth`           | _(none)_ | MediaMTX auth callback           | Reads Firestore, bcrypt verify; allows the go2rtc reader account |
