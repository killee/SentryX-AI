# SentryX AI  
Smart ESP32-based Door Access & Presence System  

<img src="pic/logo_SentryX_AI.png" width="260" alt="SentryX AI Logo">

---

## Overview

**SentryX AI** is a fully local, ESP32-based smart entry system for front doors.  
It combines:

- **HLK-FM22x (FM225) face recognition**
- **LD2410 mmWave radar presence detection**
- **8×8 WS2812 LED matrix** with icons, animations and greetings
- **Tight Home Assistant integration** via ESPHome API
- A small state machine that reacts to presence and face events

The goal is a **fast, privacy-friendly and hackable** access system that runs entirely on your own hardware.

---

## Features (Current)

### 🔍 Face Recognition (HLK-FM22x / FM225)

- Real-time face scan, triggered by radar or manual button
- Actions exposed via ESPHome API:
  - `enroll(name, direction)`
  - `scan`
  - `delete(face_id)`
  - `delete_all`
  - `reset`
  - `reset_enroll_status`
  - `show_enroll_result`
- Events forwarded to Home Assistant:
  - Face matched (with `face_id` and `name`)
  - Face unmatched
  - Invalid scan (with error code)
  - Enrollment done / failed
  - Face tracking info (yaw, pitch, roll, bounding box)
- Text sensors for scan status, enrollment status, firmware, last face name
- Visual feedback on the matrix with happy/sad faces during enrollment

---

### 📡 Radar Presence (LD2410)

- LD2410 mmWave radar connected via UART
- Presence and distance values:
  - Moving distance
  - Still distance
  - Detection distance
- Binary presence sensor (`has_target`)
- Template switch to **enable/disable radar-triggered logic**
- LD2410 buttons and numbers for:
  - Factory reset, restart, query parameters
  - Timeout, light threshold, max distance gates
  - Engineering mode and Bluetooth

The radar drives the **display state machine**:

- **0 – Blank**: nobody in range  
- **1 – Pulse**: person detected in 0.4–4 m range  
- **2 – Happy**: person very close (≤ 35 cm) → triggers face scan  
- **3 – Greeting**: “Hi <Name>” scrolls on matrix  
- **4 – Sad**: unknown face or error

---

### 🟩 LED Matrix (8×8 WS2812)

The LED matrix is driven using the `addressable_light` display platform and a set of templates:

- 8×8 WS2812 strip configured with a custom pixel mapper
- Matrix **mode** is controlled by a `select` entity:
  - Text
  - Arrow Up / Down / Left / Right
  - Smiley Happy / Sad / Wink / Wow
  - Center Dot
  - Sand Uhr (animated hourglass)
  - Tuer (animated door opening)
  - House
  - X
  - Big DOT
  - Pulse
  - Blank
- Scrollable text (`matrix_text`) with horizontal scrolling
- Matrix color and brightness controlled via the `light` entity
- Multiple animations implemented in the `lambda`:
  - Pulse ring from the center
  - Hourglass with falling sand
  - Door that opens over multiple frames

A small global state machine (`display_state` + `matrix_locked`) makes sure:

- Radar cannot overwrite a greeting (“Hi Name”) while it is active
- Unknown faces briefly show a red sad face, then return to blank
- Face enrollment directly drives happy/sad feedback

---

### 🧠 Logic / State Machine

Core logic is implemented using:

- `globals` for:
  - `matrix_locked`
  - `display_state`
  - `enroll_any_failed`
- `script`s for:
  - Triggering a face scan from LD2410
  - Showing greeting text for known faces (`matrix_show_greeting`)
  - Setting the matrix into various states (blank, pulse, happy, sad)
- `on_face_scan_*` and `on_enrollment_*` hooks on the `hlk_fm22x` component
- `on_value` for LD2410 detection distance to update display state and trigger scans

Result: The device behaves as a **self-contained frontdoor “brain”** reacting to presence and faces, and only exposes high-level entities/events to Home Assistant.

---

## Planned Features (Roadmap)

### 🔊 Audio & Voice (planned)

- I²S microphone (e.g. INMP441) for:
  - Knock / noise detection
  - Optional short audio snippets to HA
- I²S amplifier (e.g. MAX98357A) + sealed speaker module for:
  - Boot / error / access-granted sounds
  - Short voice prompts (“Access granted”, “Unknown visitor”)
  - Optional TTS from Home Assistant routed to SentryX AI

### 🔓 Door Unlock (planned)

- Dedicated **unlock output** (relay or MOSFET) to control:
  - Electric strike
  - Motor lock
- Configurable unlock logic in HA:
  - Face match + radar presence + time conditions
  - “Child mode” profiles
- Door status integration:
  - Lock/bolt contacts
  - Door open/closed sensor

### 🔔 Interaction & UX (planned)

- Integration with Shelly BLE buttons for local unlock triggers
- NFC/RFID option for fallback access (RC522/PN532)
- Additional matrix icons for:
  - “Do not disturb”
  - Alarm / warning
  - Call support / live view

### 📶 Smart Home / Integrations (planned)

- More detailed HA blueprints for:
  - Enrollment workflows
  - Notification flows (known/unknown visitor)
  - Audio snippet handling
- Optional multi-device setups:
  - Outside radar + inside radar
  - Shared events across multiple SentryX AI nodes

---

## Hardware (Reference Design)

A typical SentryX AI node consists of:

- **ESP32 DevKit** (ESP-IDF based ESPHome build)
- **HLK-FM22x / FM225** face recognition module (UART)
- **LD2410** mmWave radar (UART)
- **8×8 WS2812 LED panel** (or strip arranged as 8×8)
- Optional:
  - I²S microphone (INMP441)
  - I²S amplifier (MAX98357A)
  - 1–3 W rectangular speaker in a sealed chamber
  - Door unlock output (relay / MOSFET)
  - Door and lock contact switches

---

## ESPHome Configuration

The full ESPHome configuration lives in this repository, e.g.:

```text
esphome/frontdoor.yaml
```

It includes:

- Wi-Fi and OTA setup
- external_components for hlk_fm22x
- UART config for FM22x and LD2410
- Global variables and scripts for the state machine
- All sensors, switches, buttons and text/select entities
- LED matrix setup with custom pixel mapping and drawing logic

You can flash it directly via ESPHome and adapt pins/substitutions as needed.

---

## Home Assistant Integration

SentryX AI integrates with Home Assistant via the ESPHome native API.

### Exposed Entities (examples)

Sensors

- FM22x Face Count
- FM22x Last Face ID
- FM22x Status
- LD2410 Moving Distance
- LD2410 Still Distance
- LD2410 Detection Distance

Text Sensors

- FM22x Firmware
- FM22x Last Face Name
- FM22x Scan Status
- FM22x Enrollment Status

Switches

- FM22x Dauer-Scan (continuous scan)
- LD2410 Radar aktiv
- LD2410 Engineering Mode
- LD2410 Bluetooth

Binary Sensors

- LD2410 Presence (occupancy)

Buttons

- FM22x Scan
- LD2410 Factory reset, Restart, Query params

Matrix Controls

- LED Matrix (light component, color/brightness)
- LED Matrix Mode (select)
- LED Matrix Text (text input)

### Events

The node emits HA events via homeassistant.event:

- esphome.test_node_face_scan_matched  
  payload: face_id, name

- esphome.test_node_face_scan_unmatched

- esphome.test_node_face_scan_invalid  
  payload: error

- esphome.test_node_enrollment_done  
  payload: face_id, direction

- esphome.test_node_enrollment_failed  
  payload: error

- esphome.test_node_face_info  
  payload: status, left, top, right, bottom, yaw, pitch, roll

These can be used in HA automations to:

- Unlock doors
- Send notifications
- Play sounds
- Log access events

---

## Project Goals

- 100% local processing (no cloud, no vendor lock-in)
- Hackable: behavior controlled by YAML, not closed firmware
- Designed for 3D-printed housings and modular add-on boards

Usable as:

- DIY smart doorbell  
- Privacy-friendly access control  
- Frontdoor sensor hub

---

## License

Unless stated otherwise, this project is intended to be released under the MIT License.

You can freely use, modify and integrate it in your own projects, both private and commercial, at your own risk.

## Contributing

Contributions are welcome:

- New LED matrix icons and animations
- Improved face recognition workflows
- Audio support / I²S configs
- Door lock integration examples
- Home Assistant blueprints and dashboards

Please open an issue or pull request with your ideas and changes.

## Status

🚧 Early development — Core functionality is working (FM22x + LD2410 + matrix + HA events). Audio, door unlock and advanced UX are in active design.

SentryX AI – Local. Fast. Private.
Your front door, upgraded.