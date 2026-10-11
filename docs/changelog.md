# Changelog

This changelog tracks system-wide architectural modifications. Submodule developers should refer to this document to understand cross-component changes and follow their respective implementation checklists.

---

## [Unreleased]

### Feature: Automatic Emergency-Vehicle Signal Override (edge-initiated)

Edge devices now request a signal override automatically when YOLO detects an emergency vehicle, reusing the existing group-commands endpoint so no server change is needed.

#### Summary of Major Changes
1.  **Telemetry:** new `emergencyVehicleDetected` (`bool`), `emergencyClass` (`string`), `emergencyConfidence` (`number`) fields on every edge telemetry payload (and RTDB `/telemetry/{deviceId}`). Set when an `emergency_classes` label is visible, cleared after `emergency_clear_s` without a sighting.
2.  **Detection (edge-only):** `emergency_classes` (`ambulance`, `fire truck`, `police car` default) above `emergency_confidence` for `emergency_min_frames` consecutive sightings. One request per distinct vehicle (ByteTrack ID; one per presence episode in detect-only mode).
3.  **Override request:** `POST /api/groups/{id}/commands` with `{"command": "signal_override", "payload": {"phases": {deviceId: "GREEN"}, "durationSeconds": 60}}`. Auth reuses `POST /api/auth/login` with `admin_email` / `admin_password` (env fallback `VCLANE_ADMIN_EMAIL` / `VCLANE_ADMIN_PASSWORD`). No `cancel_override` — the override expires naturally. Global `emergency_cooldown_s` (60 s default) suppresses redundant POSTs.

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
2.  **Phase Handling:** YELLOW, RED_YELLOW, FLASHING_YELLOW, and CAUTION phases are no longer actuated — they result in all relays being turned off (caution mode).
3.  **Safety Fallback:** When no valid LoRa signal is received, the device enters caution mode with all relays off (instead of flashing yellow).
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

