# Hardware guide

This document separates the **active wristband hardware** from the **standalone microphone diagnostic**. Confirm every pin and power connection against the exact board and module revision on your bench before applying power.

## 1. Active wristband hardware

The current active firmware controls one vibration motor through channel A of a TB6612FNG motor driver.

### 1.1 Main components

| Component | Role | Repository evidence |
|---|---|---|
| ESP32-C6 development board | Wi-Fi and haptic controller | `firmware/wristband/` |
| TB6612FNG dual H-bridge driver | Motor current switching | Active firmware and wiring table |
| Vibration motor | Haptic output | Connected to driver outputs AO1/AO2 |
| Host computer | Runs detector, backend, and dashboard | Root setup instructions |
| Wi-Fi access point | Connects host and ESP32-C6 | Device provisioning workflow |

Add the exact board manufacturer/model, motor part number, supply voltage/current, connector details, and photographs of the physical prototype before calling the bill of materials complete.

### 1.2 Active motor pin mapping

The current wristband firmware defines these pins in `firmware/wristband/main/hardware_pins.h`:

| TB6612FNG signal | ESP32-C6 GPIO | Purpose |
|---|---:|---|
| `PWMA` | GPIO10 | PWM motor intensity |
| `AIN1` | GPIO11 | Channel-A direction input |
| `AIN2` | GPIO18 | Channel-A direction input |
| `STBY` | GPIO21 | Driver standby control |
| `AO1`, `AO2` | Motor terminals | H-bridge motor output |

The firmware uses one direction for the vibration motor. Do not connect the motor directly to an ESP32 GPIO. Verify the motor supply and logic supply requirements against the TB6612FNG carrier-board schematic and the motor datasheet. Ensure the driver and ESP32 share a reference ground where required by the module design.

### 1.3 Power and handling

- Do not power a vibration motor from an ESP32 GPIO.
- Verify motor rated voltage, stall/start current, driver current capability, and supply polarity.
- Verify the TB6612FNG breakout's `VM`, `VCC`, and `GND` connections using its documentation.
- Keep the motor disabled while checking wiring continuity.
- Secure exposed conductors and prevent the motor terminals from shorting against the wearable frame.
- Do not wear the prototype during electrical debugging or while exposed wiring is present.
- Record the exact power source and measured voltage/current in the hardware test log.

No battery telemetry is currently reported by the active firmware or dashboard. Do not document a battery percentage feature until a voltage/current sensing circuit and firmware reporting path have been implemented and tested.

## 2. Separate INMP441 diagnostic hardware

The standalone project at `firmware/hardware_diagnostic/` reads an INMP441 digital microphone over I2S. It is not the microphone path used by the active YAMNet detector.

### 2.1 Proposed diagnostic I2S mapping

The current diagnostic `hardware_pins.h` labels these connections as proposed and asks the builder to verify them:

| INMP441 signal | ESP32-C6 GPIO |
|---|---:|
| `SCK` / `BCLK` | GPIO6 |
| `WS` / `LRCLK` | GPIO7 |
| `SD` | GPIO0 |

The diagnostic code configures I2S standard Philips mode, 32-bit stereo slots, and a 16 kHz sample rate. It shifts the received 32-bit slots to interpret the INMP441's signed 24-bit samples, then reports left/right peak and RMS levels.

Check the microphone's `L/R` channel-select pin, supply voltage, ground, and the board's pin availability before wiring. The source pin comments are a proposed wiring reference, not proof that every assembled prototype uses this exact arrangement.

### 2.2 Diagnostic motor test caution

The diagnostic firmware contains a compile-time motor-test enable flag in `firmware/hardware_diagnostic/main/hardware_pins.h`. The source currently sets `MOTOR_TEST_PINOUT_CONFIRMED` to `1`, while the diagnostic README describes the test as locked by default. Treat this as a documentation/configuration inconsistency: inspect the flag and physically verify all four motor-control connections before flashing or issuing the `M` command. Do not rely on the README wording alone.

## 3. Firmware projects

| Project | Purpose | Use |
|---|---|---|
| `firmware/wristband/` | Wi-Fi/WebSocket haptic application | Active end-to-end wristband firmware |
| `firmware/hardware_diagnostic/` | INMP441 I2S signal monitor and optional motor pulse | Bench diagnostics only |

Both projects target ESP32-C6 and use ESP-IDF 6.1 according to the repository documentation.

## 4. Hardware validation checklist

Before connecting the system end to end:

- [ ] Record the exact ESP32-C6 board and revision.
- [ ] Record the exact TB6612FNG module and motor part number.
- [ ] Verify the motor supply voltage and current capability.
- [ ] Verify GPIO10, GPIO11, GPIO18, and GPIO21 against the active firmware.
- [ ] Confirm the motor is connected to AO1/AO2, not directly to GPIO.
- [ ] Confirm motor outputs start disabled and return to standby.
- [ ] If using the diagnostic microphone, verify GPIO6/GPIO7/GPIO0 and INMP441 channel selection.
- [ ] Verify the diagnostic motor-test flag before running the test.
- [ ] Photograph the final wiring from more than one angle.
- [ ] Record measured supply voltage and current under motor load.

## 5. Recommended hardware evidence for the repository

Add original, well-lit photographs rather than stock images:

1. Complete wristband prototype.
2. ESP32-C6 and TB6612FNG wiring close-up.
3. Motor mounting and skin-contact arrangement.
4. Power source and regulation arrangement.
5. Separate INMP441 diagnostic setup, clearly labeled as diagnostic hardware.
6. A labeled pinout diagram created from the actual assembled wiring.

A hardware diagram should distinguish **verified connections** from **proposed or unverified connections**.
