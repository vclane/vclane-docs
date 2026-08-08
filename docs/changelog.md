# Changelog

This changelog tracks system-wide architectural modifications. Submodule developers should refer to this document to understand cross-component changes and follow their respective implementation checklists.

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
    - `durationSeconds <= 0` means indefinite. An active override pauses the schedule turn-loop and resumes the same turn at the same offset when it ends.
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

