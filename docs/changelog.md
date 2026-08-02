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

---

## Submodule Implementation Checklists

Select your submodule below to see what changes are required on your side:

### 1. Backend API (FastAPI)
- [ ] **Database Schema Updates:**
  - Create a group-mapping schema linking multiple Edge Devices (`deviceId`) to a single Group Traffic Controller (`groupId`).
  - Add registration fields for ESP-32 controllers (WiFi MAC address, LoRa Group ID, firmware version, status).
- [ ] **MQTT Topic Adjustments:**
  - Stop publishing direct traffic light commands to `traffic/device/{deviceId}/commands`.
  - Implement schedule configuration publishing to `traffic/group/{groupId}/schedule` (JSON payloads).
  - Implement manual override publishing to `traffic/group/{groupId}/commands`.
- [ ] **Fallback Telemetry Integration:**
  - Add a consumer for `traffic/group/{groupId}/telemetry` to parse and store fallback traffic light states.
  - Update API endpoints to merge active traffic light states from Raspberry Pi reports (when online) and ESP-32 fallback reports (when Pi is offline).

### 2. Raspberry Pi Edge Device (Python)
- [ ] **Hardware Integration:**
  - Interface an SX1276/SX1278 LoRa transceiver module with the Raspberry Pi GPIO header via SPI.
  - Wire the physical traffic light relays to the Raspberry Pi GPIO pins.
- [ ] **Software Adjustments:**
  - Integrate a Python LoRa library (e.g., `pyLoRa` or `Adafruit-CircuitPython-RFM9x`).
  - Implement a background listener thread to receive, verify, and parse LoRa broadcast packets matching the local Group ID.
  - Update the control loop to actuate relays based on LoRa packet instructions.
  - Implement a safety timeout fallback: if no valid LoRa packet is received for 5 seconds, command the relays to enter caution mode (flashing yellow).
- [ ] **Telemetry Cleanup:**
  - Remove all code reporting CPU/memory usage metrics via `psutil`.
  - Continue publishing AI traffic telemetry (vehicleCount, trafficDensity) to `traffic/device/{deviceId}/telemetry` over MQTT.

### 3. ESP-32 Traffic Controller (Firmware)
- [ ] **Hardware Setup:**
  - Wire an SX1276/SX1278 LoRa transceiver module (SPI) and a DS3231 high-precision RTC module (I2C) with battery backup.
- [ ] **Core Software Tasks:**
  - Implement WiFi/Ethernet network drivers and MQTT client (over TLS v1.3).
  - Implement a local schedule scheduler engine that parses and executes JSON-formatted daily traffic timing plans.
  - Save received schedules to Non-Volatile Storage (NVS) to survive power cycles.
  - Code the LoRa broadcast loop to transmit active phase maps and sequence numbers every 1 second (and on state change).
- [ ] **Resilience Logic:**
  - Implement background connection retries to prevent WiFi dropouts from freezing the scheduler thread.
  - Implement NTP time sync when online, updating the DS3231 RTC.
  - Implement fallback telemetry publishing to `traffic/group/{groupId}/telemetry` over MQTT when connected.

### 4. Flutter Mobile App
- [ ] **UI Updates:**
  - Update the dashboard to group intersections visually by their respective Group Traffic Controller.
  - Add status indicators representing the ESP-32 Controller state (Online, Offline, Fallback Mode).
- [ ] **Control Logic:**
  - Update manual signal override actions to send commands targeting the group's ESP-32 controller rather than individual Raspberry Pi devices.
  - Display the fallback traffic light phase states on the dashboard if the individual edge device is offline but the controller is online.
