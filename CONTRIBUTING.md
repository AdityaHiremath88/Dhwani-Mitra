# Contributing to Dhvani Mitra

Thank you for your interest in improving Dhvani Mitra. Contributions should make the project easier to reproduce, safer to operate, and more reliable for people experimenting with assistive technology.

## 1. Before opening a pull request

1. Search existing issues and pull requests.
2. Describe the problem and the expected behavior.
3. Identify which component is affected: detector, backend, dashboard, wristband firmware, or hardware documentation.
4. For hardware-related work, state the exact board/module revision and wiring.
5. Avoid including private audio, credentials, or personal data.

## 2. Development principles

- Keep detector, backend, dashboard, and firmware responsibilities clearly separated.
- Validate messages at trust boundaries.
- Keep backend schemas and firmware command limits consistent.
- Prefer explicit error handling over silently swallowing failures.
- Do not claim measured accuracy, latency, range, or battery life without test evidence.
- Document whether a feature is implemented, experimental, or planned.
- Preserve the distinction between the active laptop-based detector and the separate ESP32-C6 I2S diagnostic firmware.

## 3. Pull request checklist

- [ ] The change has a clear problem statement.
- [ ] Setup or usage documentation is updated.
- [ ] Tests or a reproducible manual test procedure are included.
- [ ] Hardware changes include a wiring/power note.
- [ ] Protocol changes update `docs/PROTOCOL.md`.
- [ ] Detection changes include the test audio conditions and evaluation method.
- [ ] No credentials, private audio, generated runtime data, or personal information are included.
- [ ] Limitations and known failure cases are documented.

## 4. Reporting detection issues

When reporting a false positive or missed detection, include:

- Category expected and category emitted (if any).
- Detector version/commit.
- Microphone model and selected input device.
- Approximate distance and orientation.
- Environment and background noise.
- Input RMS and relevant category scores from the detector log.
- Whether the sound was continuous, repeated, or overlapping.

Do not upload copyrighted or private audio without permission. A short description and a reproducible source reference may be sufficient.

## 5. Reporting hardware issues

Include:

- ESP32-C6 board and revision.
- TB6612FNG module and motor part number.
- Wiring diagram or clear photograph.
- Motor and logic supply details.
- Serial log and exact reproduction steps.
- Whether the issue occurs with the standalone motor test or only with network-delivered commands.

## 6. Scope of contributions

Good first contributions include documentation corrections, setup reproducibility, protocol tests, schema/firmware validation alignment, detector evaluation tooling, and hardware test records.

A contribution should not imply that the prototype is a certified accessibility or emergency-warning device.
