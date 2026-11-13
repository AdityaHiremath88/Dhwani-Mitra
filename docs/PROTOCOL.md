# Communication protocol

This document describes the interfaces implemented by the current repository. It is intended as a practical reference for developers integrating another detector, dashboard, or compatible device.

## 1. Network topology

The backend is the central bridge. The host computer runs FastAPI on TCP port `8000`. The browser, detector, and ESP32-C6 connect to separate interfaces.

| Client | Interface | Direction |
|---|---|---|
| Dashboard | REST + `/ws/web` | Reads status/history/patterns; sends UI commands |
| Detector | `/ws/detector` | Sends microphone status, audio level, and confirmed sound events |
| ESP32-C6 | `/ws/device` | Receives haptic commands/configuration; sends acknowledgements and heartbeats |

The ESP32-C6 must connect to the host's LAN IPv4 address, not `localhost` or `127.0.0.1`.

## 2. REST API

Base URL in the local development setup: `http://localhost:8000`.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/health` | Backend health and connection summary |
| `GET` | `/api/status` | Detector, microphone, and device status |
| `GET` | `/api/patterns` | Read saved vibration patterns |
| `PUT` | `/api/patterns/{category}` | Validate and save one category's pattern |
| `GET` | `/api/history` | Read detection history |
| `DELETE` | `/api/history` | Clear persisted detection history |

### 2.1 Category identifiers

The backend accepts these category IDs:

- `ALARM`
- `VEHICLE_HORN`
- `DOORBELL`
- `KNOCKING`
- `SIREN`

The category in a pattern update path must match the category in its payload after normalization.

### 2.2 Pattern update example

```json
{
  "type": "update_pattern",
  "category": "DOORBELL",
  "intensity": 60,
  "repeat": 1,
  "pulses": [
    {
      "on_ms": 450,
      "off_ms": 0
    }
  ],
  "preset": "Single Pulse"
}
```

Backend validation currently constrains intensity to 0–100, repeat to 1–8, the pulse list to 1–8 entries, and each pulse duration to the schema's allowed range. The active firmware accepts intensity 1–100, so intensity `0` is a schema/device validation mismatch and should not be used for a physical command.

## 3. Detector WebSocket: `/ws/detector`

The detector connects to `ws://127.0.0.1:8000/ws/detector` by default. Override with `DHVANI_DETECTOR_WS`.

### 3.1 Detector status

```json
{
  "type": "detector_status",
  "microphone_device": "Microphone device name",
  "sample_rate": 44100
}
```

The sample rate field describes the selected capture device's input sample rate. The detector resamples audio to 16 kHz for YAMNet when required.

### 3.2 Audio level

```json
{
  "type": "audio_level",
  "rms": 0.0182
}
```

The backend accepts finite RMS values from 0 through 1.

### 3.3 Confirmed sound event

```json
{
  "type": "sound_detected",
  "category": "DOORBELL",
  "confidence": 0.72,
  "timestamp": "2026-11-13T12:00:00+00:00"
}
```

`confidence` is the detector's aggregated category score, not a calibrated probability. The timestamp is ISO 8601. If omitted, the backend schema supplies a UTC timestamp.

The backend creates an event ID and routes the category's saved haptic pattern to the connected wristband.

## 4. Device WebSocket: `/ws/device`

The ESP32-C6 connects to:

```text
ws://<HOST_LAN_IPV4>:8000/ws/device
```

The backend sends the saved pattern configuration after the device connects. The firmware currently logs `sync_config`; each subsequent haptic command carries the pulse data used for playback.

### 4.1 Haptic command

```json
{
  "type": "haptic_command",
  "event_id": "d7d3a9c4-2b5a-4f13-9d2b-4d22d7d5d2f1",
  "category": "DOORBELL",
  "preset": "Single Pulse",
  "intensity": 60,
  "repeat": 1,
  "pulses": [
    {
      "on_ms": 450,
      "off_ms": 0
    }
  ]
}
```

The firmware validates the message before queuing it. Current firmware constraints:

| Field | Constraint |
|---|---|
| `type` | `haptic_command` or `haptic_test` |
| `category` | String |
| `intensity` | 1–100 |
| `pulses` | Array containing 1–8 entries |
| `on_ms` | 20–5000 ms |
| `off_ms` | 0–5000 ms |
| `event_id` | Optional string for event correlation |

The firmware executes the pulse list. The `repeat` field is part of the dashboard/backend pattern representation; playback duration is represented by the `pulses` array.

### 4.2 Acknowledgement and completion

After accepting a command into the FreeRTOS queue, the device sends:

```json
{
  "type": "haptic_ack",
  "event_id": "d7d3a9c4-2b5a-4f13-9d2b-4d22d7d5d2f1",
  "category": "DOORBELL"
}
```

After playback finishes:

```json
{
  "type": "haptic_complete",
  "event_id": "d7d3a9c4-2b5a-4f13-9d2b-4d22d7d5d2f1",
  "category": "DOORBELL"
}
```

The backend maps these messages to `acknowledged` and `completed` event delivery states.

### 4.3 Device status and heartbeat

The firmware reports its IP address on connection and sends a heartbeat every 15 seconds:

```json
{
  "type": "heartbeat",
  "ip": "192.168.1.42"
}
```

The WebSocket client is configured with a 20-second ping interval and reconnect behavior. The backend updates device last-seen status when it receives device messages.

## 5. Dashboard WebSocket: `/ws/web`

The dashboard connects to `ws://<API_HOST>/ws/web`.

The backend sends messages including:

- `status` — current bridge status.
- `history` — current stored detection history.
- `sound_detected` — a new detector event.
- `haptic_status` — delivery progress for an event.
- `patterns_updated` — saved pattern configuration changed.
- `notice` — a user-facing validation or connectivity message.

The dashboard sends messages including:

- `update_pattern` — save a category's pattern.
- `test_haptic` — request a physical wristband test.
- `sync_config` — request a configuration sync.

## 6. Delivery-state semantics

| State | Meaning |
|---|---|
| `pending` | Event recorded; device delivery not yet resolved |
| `sent` | Backend WebSocket send call succeeded |
| `unavailable` | Wristband was not connected or send failed |
| `acknowledged` | Firmware accepted the command into its queue |
| `completed` | Firmware finished the pulse sequence |

These are software-reported states. They do not verify skin contact, user perception, or physical motor condition.

## 7. Compatibility guidance

When extending the protocol:

- Preserve category IDs or introduce explicit versioning.
- Validate all untrusted payload fields at the backend and device.
- Add message schema/version fields before making breaking changes.
- Consider event expiry, duplicate suppression, and sequence numbers.
- Keep haptic command bounds consistent between Pydantic schemas and firmware validation.
- Add tests for malformed JSON, unknown categories, invalid durations, disconnected devices, and queue saturation.
