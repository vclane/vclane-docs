# Changelog

This changelog tracks system-wide architectural modifications. Submodule developers should refer to this document to understand cross-component changes and follow their respective implementation checklists.

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

