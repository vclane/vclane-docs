# Changelog

This changelog tracks system-wide architectural modifications. Submodule developers should refer to this document to understand cross-component changes and follow their respective implementation checklists.

---

## [2.12.1] — 2026-10-24

### Backend MQTT Client Reconnects After Disconnects

#### Summary of Major Changes
1.  **Automatic reconnect:** the backend MQTT client now uses `connect_async()` + `loop_start()` with a 1–30 s exponential backoff. A broker restart, network drop, or a broker that is not ready when the backend starts no longer leaves the backend permanently deaf. Every successful (re)connect re-subscribes all inbound topics.
2.  **QoS 1 outbound:** commands, schedules, prepare, overrides, and `enqueue` publish at QoS 1. The paho return code is checked; while disconnected, QoS 1 messages are queued locally (bounded at 1000) and flushed after reconnect.
3.  **Visible failures:** `on_disconnect` / `on_connect_fail` log warnings, and `GET /health` reports `{"status": "ok", "mqtt": {"connected": bool}}` (always HTTP 200; this is a liveness probe, not readiness).
4.  **Truthful queue bookkeeping:** RTDB `has_queued_schedule` is set only when the broker accepted the `enqueue` publish; a failed publish leaves the detection `enqueued: false`.
5.  **Backend-only:** topics and payloads are unchanged; no controller or edge change is required.

---

## [2.12.0] — 2026-10-24

### Admin Force-Enqueue, Group Enqueue Switch, Non-Deduped FIFO, Optional Detection Image

#### Summary of Major Changes
1.  **Group enqueue switch:** `groups/{groupId}.enqueueEdgeDetections` (`bool`, default `true`; missing on legacy docs = `true`). When `false`, detections are still stored but the queue gate never publishes `enqueue`. Signal overrides are unaffected. Backend-only; it is not part of the MQTT schedule payload.
2.  **Admin force-enqueue:** `POST /api/devices/{id}/detections` accepts optional multipart `forceEnqueue` (default `false`). Requires `users/{uid}.role == "admin"` (403 otherwise, nothing written). Skips the allowlist, staleness, RED/CAUTION, `has_queued_schedule`, and `enqueueEdgeDetections` checks; still requires a group in `schedulingMode: "dynamic"`.
3.  **Controller FIFO no longer dedupes:** every `enqueue` appends one entry, including duplicates and the currently-GREEN device. A queue of `cam1 | cam1 | cam2` serves all three in order. Only empty IDs and a full queue (16 entries) are rejected.
4.  **Optional image:** `image` is optional; when absent no Storage upload occurs and `imageUrl` is `null`.
5.  **Detection audit fields:** `forceEnqueueRequested`, `enqueueEdgeDetections` (group snapshot), `enqueued`.
6.  **Admin Tools (client):** `/admin/tools`, role-gated; detection test (no image, optional force) plus group override / cancel with a full per-device phase map.
7.  **Gate corrections:** staleness is 180 s (not ~30 s); the gate accepts `RED` or `CAUTION`.
8.  **Group override spec corrections:** `durationSeconds` `0` = indefinite, negative rejected; overrides are RAM-only and survive MQTT loss until expiry; unlisted devices fall to caution/RED; the fixed turn and queue phase restart in full after an override; group commands remain fire-and-forget with no audit record.
9.  **Known caveat:** RTDB `has_queued_schedule` remains a per-device boolean; with repeated forced entries the first completion clears it while duplicates may still be pending.
10. **Docs-only:** other components implement from this entry.

---

## [2.11.0] — 2026-10-24

### Edge Online Status via Retained LWT (Timeout Checker Removed)

Edge devices now own their online/offline state over MQTT. The backend's periodic `lastSeen` timeout checker is removed; online transitions arrive as broker-published will messages and retained status messages.

#### Summary of Major Changes
1.  **Retained device `status` topic + LWT:** on connect the edge registers a retained will `{"online": false}` on `traffic/device/{deviceId}/status`, publishes retained `{"online": true}` after a successful connect, and publishes retained `{"online": false}` before a graceful DISCONNECT (MQTT 3.1.1 does not fire the will on a clean disconnect). QoS 1, `retain: true`. This mirrors the ESP-32 controller's existing `traffic/group/{groupId}/status` contract.
2.  **Keepalive is the offline latency:** the broker publishes the will after ~1.5 × the MQTT keepalive. New `mqtt_keepalive` setting / `--mqtt-keepalive` CLI flag on the edge, default `60` s (~90 s worst-case detection). Lower it if faster detection is required.
3.  **Backend consumes `traffic/device/+/status`:** new `_handle_device_status` — `online: true` refreshes `online` + `lastSeen` without touching telemetry fields, `online: false` marks the device offline. The backend's `OnlineChecker` (30 s loop, marks stale at 60 s) is deleted along with its `main.py` wiring, for both `/telemetry` and `/controllers`.
4.  **Broker ACL:** the `device` role gains `publishClientSend traffic/device/%u/status`. Existing deployments must apply the ACL (reset the `mosquitto_data` Dynamic Security volume or issue `addRoleACL`) or the status/will publish is denied.
5.  **Telemetry/heartbeat unchanged:** both still inject `online: true` + `lastSeen` as a self-healing safety net and for backward compatibility. `lastSeen` becomes informational and no longer drives online state.
6.  **Legacy `goodbye` handler kept:** the `traffic/device/{id}/goodbye` handler stays for older firmware; new firmware marks itself offline via the retained `status` message instead.

---

## [2.10.0] — 2026-10-15

### Edge: Backend Login at Device Startup

The shared `BackendApiClient` now authenticates once when `vclane-edge run` starts instead of lazily on the first request.

#### Summary of Major Changes
1.  **Eager auth:** `BackendApiClient.ensure_login()` performs `POST /api/auth/login` at `VclaneEdgeDevice.run()` startup (before the video pipeline / uploader / override threads), so the first detection upload or emergency override does not pay the login round-trip.
2.  **Non-fatal fallback:** a failed startup login is logged and the existing lazy auth path is retained — `request()` / `request_multipart()` still log in on first use and re-authenticate once on a 401. Devices without admin credentials (`api_url` / `admin_email` / `admin_password`) skip it silently.

---

## [2.9.0] — 2026-10-14

### Edge-Only Telemetry

Traffic telemetry is now published exclusively by the Raspberry Pi Edge Devices. The ESP-32 Traffic Controller no longer publishes fallback phase telemetry.

#### Summary of Major Changes
1.  **Edge-only telemetry:** `traffic/device/{deviceId}/telemetry` (active signal + emergency flag) is the system's sole telemetry source. The controller's periodic `isFallback: true` group telemetry is removed; an offline edge device is now reported offline rather than back-filled by the controller.
2.  **Dedicated queue-completion topic:** queue completion moves from `traffic/group/{groupId}/telemetry` to `traffic/group/{groupId}/queue-done` (`queueDoneDeviceId`, `queueDepth` → clears `has_queued_schedule`). Controller `heartbeat` (30 s) and LWT `status` are unchanged.
3.  **Docs-only change:** the controller firmware (`vclane-controller`) and the backend still publish/subscribe the old `traffic/group/{groupId}/telemetry` topic (and its fallback telemetry) until the corresponding code changes land. This entry documents the target architecture.

---

## [2.8.0] — 2026-10-14

### Track-Only Edge (ByteTrack Default, Detect-Only Removed)

Single inference path: ByteTrack always on when `yolo_enabled`. Breaking removal of the detect-only mode.

#### Summary of Major Changes
1.  **`tracker` required (default `bytetrack.yaml`):** `tracker: null`, `--no-tracker`, `YOLO.detect()`, and `detect-only` emergency episodes removed. `lap` is mandatory.
2.  **`yolo_interval` validated 1–2** (config-level clamp; `1` accuracy, `2` perf). Larger skips break the Kalman state.
3.  **`yolo_enabled: false` retained** as the raw-passthrough escape hatch (no inference; or `stream_mode: direct-ffmpeg`). Migration: delete `tracker: null` / `--no-tracker` from configs.

---

## [2.7.0] — 2026-10-14

### No-Count Edge + Queue Trigger on Detections

Edge stops all aggregate counting; per-vehicle history lives in `detections` and the dynamic queue gate triggers off the REST endpoint.

#### Summary of Major Changes
1.  **Lean telemetry:** edge MQTT telemetry is signal + emergency only (`currentTrafficLightSignal`, `emergencyVehicleDetected/Class/Confidence`, `timestamp`). `vehicleCount/Total`, `directionCounts`, `classCounts`, `trafficDensity` removed; today-counts come from `detections` queries.
2.  **Unified detection module (edge):** `detection.py` (`DetectionTracker` + `DetectionUploader`) replaces counting + emergency trackers. One event per new ByteTrack ID (first sighting, any YOLO label) with JPEG crop → `POST /api/devices/{id}/detections` (bounded queue, retry offline). Detect-only sends none.
3.  **Queue gate moves to REST:** `POST /api/devices/{id}/detections` (new-doc only) runs the dynamic gate (vehicle allowlist, staleness guard, RED + `has_queued_schedule` dedup). Telemetry no longer triggers enqueue.

---

## [2.6.0] — 2026-10-14

### Feature: Per-Vehicle Detections in Firestore (via REST)

One Firestore document per detected vehicle for history/insights; mobile queries "vehicles today" directly from Firestore.

#### Summary of Major Changes
1.  **New collection `detections/{detectionId}`:** `{deviceId, groupId, timestamp (server), deviceTimestamp, trackId, class (any YOLO label), imageUrl (required)}`. Deterministic doc ID `{deviceId}_{trackId}_{deviceTimestamp}` enforces server-side `trackId + deviceTimestamp + deviceId` dedup.
2.  **Transport is REST, not MQTT:** `POST /api/devices/{id}/detections` (multipart `trackId`, `class`, `deviceTimestamp`, required `image`) with JWT auth → backend Storage upload + Firestore write. Detections bypass Mosquitto and RTDB.
3.  **RTDB cleanup:** `vehicleToday` removed from `/telemetry/{deviceId}`; today-counts come from `detections` queries. Indexes `(deviceId, timestamp)`, `(groupId, timestamp)`; rules read-only for mobile.

---

## [2.5.0] — 2026-10-14

### Feature: Automatic Emergency-Vehicle Signal Override (edge-initiated)

Edge devices now request a signal override automatically when YOLO detects an emergency vehicle, reusing the existing group-commands endpoint so no server change is needed.

#### Summary of Major Changes
1.  **Telemetry:** new `emergencyVehicleDetected` (`bool`), `emergencyClass` (`string`), `emergencyConfidence` (`number`) fields on every edge telemetry payload (and RTDB `/telemetry/{deviceId}`). Set when an `emergency_classes` label is visible, cleared after `emergency_clear_s` without a sighting.
2.  **Detection (edge-only):** `emergency_classes` (`ambulance`, `fire truck`, `police car` default) above `emergency_confidence` for `emergency_min_frames` consecutive sightings. One request per distinct vehicle (ByteTrack ID; one per presence episode in detect-only mode).
3.  **Override request:** `POST /api/groups/{id}/commands` with `{"command": "signal_override", "payload": {"phases": {deviceId: "GREEN"}, "durationSeconds": 60}}`. Auth reuses `POST /api/auth/login` with `admin_email` / `admin_password` (env fallback `VCLANE_ADMIN_EMAIL` / `VCLANE_ADMIN_PASSWORD`). No `cancel_override` — the override expires naturally. Global `emergency_cooldown_s` (60 s default) suppresses redundant POSTs.

### Feature: Relay Blink Before Transition (3x normal / 5x override)

Edge relays blink the upcoming phase before latching solid: 3 blinks on normal schedule transitions, 5 on override-driven transitions.

#### Summary of Major Changes
1.  **Wire format:** LoRa/MQTT phase frame gains `ov: bool` (controller sets `ov = override.active()`). `ov: true` → 5 blinks, `ov: false` → 3. Defaults to `false` for backward compatibility.
2.  **Edge actuation:** `apply_phase(phase, is_override)` ignores duplicate phases (1 s heartbeat re-broadcasts), otherwise blinks the new lamp on/off × N on a relay worker thread under the relay lock, then latches solid — so the 1 s signal/telemetry loop is not blocked. Entering caution aborts any in-progress blink and goes all-off with no blink; the first valid phase after boot goes straight solid.
3.  **Config:** `relay_blink_normal=3`, `relay_blink_override=5`, `relay_blink_interval=0.5`.

---

## [2.4.0] — 2026-10-11

### Feature: Dynamic Queue-Based Scheduling (per-group, opt-in)

Groups can now switch from the fixed-interval turns loop to on-demand queue scheduling: the controller idles at all-RED and serves GREEN only when a vehicle is detected.

#### Summary of Major Changes
1.  **Mode (per-group, `fixed` default):** `groups/{groupId}` gains `schedulingMode: "fixed" | "dynamic"` plus `dynamicConfig {greenSec, allRedSec}`. `fixed` runs the stored turns-loop schedule unchanged; `dynamic` runs the FIFO queue executor. Delivered to the controller via `traffic/group/{groupId}/schedule` and cached in NVS.
2.  **Detection reuses telemetry (threshold = 1 vehicle):** Any `traffic/device/{deviceId}/telemetry` with live `vehicleCount >= 1` is a detection event. The edge publishes immediately on the `0 → >=1` rising edge plus its normal periodic publish. No new topics.
3.  **Backend queue gate:** On telemetry, if the group is `dynamic` and the device is RED with `has_queued_schedule == false`, the backend publishes `{"command": "enqueue", "payload": {"deviceId"}}` to `traffic/group/{groupId}/commands` and sets RTDB `/telemetry/{deviceId} {has_queued_schedule: true, queuedAt}`. Non-RED or already-queued telemetry is dropped (dedup). Only the backend writes these flags.
4.  **Controller FIFO executor:** `enqueue` entries are served FIFO as GREEN (`greenSec`) → all-RED clearance (`allRedSec`) via LoRa (2-channel GREEN/RED relays, no YELLOW). Completion is reported via `traffic/group/{groupId}/telemetry` (`queueDoneDeviceId`, `queueDepth`) so the backend clears `has_queued_schedule`. Empty queue = all-RED idle. Manual overrides still preempt.
5.  **Offline fallback:** On MQTT loss the controller runs the cached fixed turns-loop via RTC (even for `dynamic` groups, since queueing needs the server) and discards the queue; on reconnect it resyncs and resumes the configured mode.

---

## [2.3.0] — 2026-10-08

### Feature: ByteTrack Vehicle Tracking + Line Counting (opt-in)

Edge devices can now track vehicles across frames with persistent IDs and report real counts instead of mock telemetry.

#### Summary of Major Changes
1.  **Tracking (opt-in, off by default):** `YOLODetector.track()` runs `model.track(persist=True, tracker=...)` with `bytetrack.yaml`. Enabled via `tracker: "bytetrack.yaml"` (or `--tracker`); disabled via `tracker: null` / `--no-tracker` (detect-only fallback). New `lap` dependency provides ByteTrack association.
2.  **Counting:** New `VehicleCounter` reports live occupancy (unique IDs visible) plus cumulative line crossings (each `(track_id, line)` counted once on centroid side-change). Lines are normalized 0–1 coords (`counting_lines`, default single `northbound` line); only `count_classes` (default car/truck/bus/motorcycle/bicycle) are counted.
3.  **Telemetry:** `vehicleCount` = live, plus new `vehicleTotal`, `directionCounts`, `classCounts`. Density thresholds: `LOW <8`, `MODERATE ≤20`, `HIGH ≤30`, `HEAVY >30`. Legacy `random_walk()` mock only applies in detect-only mode. Payload is thread-safe (`set_counts()` / `snapshot()` under lock).
4.  **Pipeline constraints:** Tracking forces `yolo_interval ≤ 2` (warn + clamp), uses an ordered inference queue (`maxsize=1`, blocking put) to preserve frame sequence, and calls `reset_tracker()` on stream gaps (EOF/reconnect). Annotated RTSP frames show `{label} #{id} {conf}` plus cyan counting lines.
5.  **Tooling:** `python scripts/test_tracking.py --source <video|0> [--save out.mp4]` for manual ByteTrack verification.

---

## [2.2.0] — 2026-09-17

### Feature: 2-Channel Relay Module (GREEN + RED only)

Simplified relay hardware to a 2-channel module supporting only GREEN and RED signal lamps, removing the YELLOW lamp entirely.

#### Summary of Major Changes
1.  **Relay Hardware:** The edge device now supports a 2-channel relay board (GREEN and RED) instead of a 3-channel board. The yellow lamp relay has been removed.
2.  **Phase Handling:** YELLOW, RED_YELLOW, FLASHING_YELLOW, and CAUTION phases are no longer actuated — they result in solid RED (caution mode).
3.  **Safety Fallback:** When no valid LoRa signal is received, the device enters caution mode with solid RED (instead of flashing yellow).
4.  **Default GPIO Pins:** `relay_pins` defaults updated to `{"green": 17, "red": 22}` (GPIO 18 for yellow removed).

---

## [2.1.0] — 2026-08-05

### Feature: Schedule Configuration & Controller On-Connect Sync

Schedules are now a first-class, persisted concept instead of fire-and-forget MQTT publishes.

#### Summary of Major Changes
1.  **Schedule Model:** A group schedule is a repeating loop of turns. Each turn holds a per-device phase set (`RED` / `GREEN` / `YELLOW`) for a fixed `durationSeconds`, then advances to the next turn and wraps after the last. No `groupId` is embedded — the MQTT topic and Firestore document provide scoping.
2.  **Persisted Source of Truth:** Schedules are stored on the group document in Firestore (`groups/{groupId}.schedule`) as the authoritative copy.
3.  **REST Endpoints:** `POST /api/groups/{id}/schedule` validates and persists the schedule then publishes it immediately; `GET /api/groups/{id}/schedule` retrieves the stored schedule (404 if none). Validation rejects empty `turns`, non-positive `durationSeconds`, and phase values outside `RED`/`GREEN`/`YELLOW`.
4.  **On-Connect Sync (replaces periodic polling):** The ESP-32 controller publishes an empty message to `traffic/group/{groupId}/schedule-request` on connect. The backend replies on `traffic/group/{groupId}/schedule` with the stored schedule, keeping the controller's NVS-cached offline fallback in sync without a background sync service.
5.  **Phase Vocabulary Simplification:** Phase names were normalized to the simple lamp states `RED`, `GREEN`, and `YELLOW`, replacing legacy directional/compound names.

## [2.1.1] — 2026-08-08

### Controller Implementation Notes (drift resolutions)

Resolved ambiguities from the 2.1.0 spec against the reference ESP-32 controller implementation.

#### Summary of Major Changes

1.  **Group Command Envelope:** Commands on `traffic/group/{groupId}/commands` reuse the backend's `{command, payload}` envelope:
    - `{"command": "signal_override", "payload": {"phases": { "device-001": "GREEN" }, "durationSeconds": 30}}`
    - `{"command": "cancel_override", "payload": {}}`
    - `durationSeconds <= 0` means indefinite. An active override pauses the schedule turn-loop and uses the override phases in its place. When the override ends (expiry or cancel) the controller resumes the same turn it paused on but restarts that turn's full duration from the beginning — remaining time is not preserved.
2.  **MQTT Version:** The ESP-32 controller speaks **MQTT 3.1.1** over TLS (via PubSubClient). The broker remains v5-capable; the controller uses no v5-only features.
3.  **Fallback Telemetry Trigger:** The controller cannot directly observe edge-device connectivity. It publishes group telemetry periodically (default **10 s**) with `isFallback: true`; the backend keeps this visible regardless of edge state.
4.  **Timing Intervals:** LoRa heartbeat **1 s**; phase-change burst **3 packets**; MQTT heartbeat **30 s**; fallback telemetry **10 s**.
5.  **LoRa Burst Semantics:** A 3-packet burst retransmits the same logical packet (identical `seq`); receivers deduplicate by `seq`. `seq` increments once per phase change and is persisted to NVS so stale/duplicate filtering survives controller reboots.
6.  **Provisioning:** The controller is configured over its serial console (WiFi, MQTT broker + CA cert, group ID, LoRa frequency) into NVS.

---

## [2.0.0] — 2026-07-31

### Architectural Refactor: Grouped Coordinated Control with LoRa Fallback

We have updated the system architecture to transition from individual intersection control to **Intersection Groups**. This change introduces the **ESP-32 Traffic Controller** and **LoRa wireless communication** to resolve signaling vulnerabilities during internet outages.

#### Summary of Major Changes
1.  **Group Coordination:** Intersections are now grouped. Coordinated signaling within a group is handled locally by a master **ESP-32 Traffic Controller** rather than via direct backend-to-edge MQTT command synchronization.
2.  **LoRa Broadcast Layer:** The ESP-32 Traffic Controller broadcasts active phase target signals to group **Raspberry Pi Edge Devices** via local LoRa radio.
3.  **Edge Actuator Role:** The Raspberry Pi Edge Devices act as passive actuators for the traffic lights, driving physical relays based on LoRa signals received from the ESP-32.
4.  **Resilience & Offline Fallback:** If internet is lost, the ESP-32 runs schedules locally using an external Real-Time Clock (RTC) and continues LoRa broadcasts.
5.  **Fallback Telemetry:** If a Raspberry Pi Edge Device goes offline but the ESP-32 remains online, the ESP-32 publishes fallback telemetry reporting the active phase of the intersections.
6.  **Telemetry Cleanup:** Removed CPU and memory usage reporting from Raspberry Pi telemetry.

