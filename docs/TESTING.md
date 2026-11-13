# Testing and validation

A convincing engineering project should distinguish **implemented features**, **manual demonstrations**, and **measured performance**. This document provides a repeatable test plan. It intentionally does not invent accuracy, latency, range, or battery-life results.

## 1. Test philosophy

For every reported result, record:

- Hardware revision and wiring.
- Software commit or release version.
- Operating system and dependency versions.
- Microphone model, position, orientation, and input gain.
- Sound source and playback level.
- Distance and environment.
- Number of trials.
- Pass/fail criteria.
- Raw observations and calculated metrics.

Do not report a single successful demonstration as a general accuracy result.

## 2. Functional test matrix

| ID | Subsystem | Test | Expected observation | Evidence to save |
|---|---|---|---|---|
| T-01 | Backend | Start API and call `/api/health` | Health response returned | Terminal output / JSON |
| T-02 | Dashboard | Open UI with backend running | Dashboard connects to `/ws/web` | Screenshot + browser console |
| T-03 | Detector | Start with selected microphone | Correct input device is printed | Detector log |
| T-04 | Audio | Speak/play sound near microphone | RMS level changes | Detector log / dashboard |
| T-05 | Detection | Play each supported sound | Correct category event appears, if recognized | Audio test ID + event log |
| T-06 | Negative audio | Play unrelated sounds and silence | No unsupported category is emitted as a target event | Test log |
| T-07 | Pattern API | Update a category pattern | Backend validates and persists it | Request/response + JSON |
| T-08 | Device link | Provision ESP32-C6 | Device connects to `/ws/device` | Serial log + dashboard |
| T-09 | Motor | Run local motor test | Motor pulses and returns to standby | Video + supply measurements |
| T-10 | End-to-end | Trigger a supported sound | Event reaches backend and haptic command reaches device | Correlated event ID + video |
| T-11 | Acknowledgement | Observe event delivery states | `sent`, `acknowledged`, and `completed` appear in order when connection is healthy | Event history |
| T-12 | Recovery | Restart Wi-Fi/backend/device | Reconnection behavior is observable | Serial/backend logs |
| T-13 | Bounds | Send malformed or out-of-range haptic payload | Device rejects invalid command | Unit/integration test |
| T-14 | History | Generate events, restart backend | Persisted history reloads | Before/after API output |

## 3. Sound-detection evaluation

Build a labeled test set containing positive and negative examples. Avoid evaluating only on the exact recordings used during development.

### 3.1 Suggested test conditions

For each target category, collect varied examples across:

- Different source recordings and devices.
- Different distances and orientations.
- Quiet and noisy rooms.
- Background speech, music, traffic, and household sounds.
- Different playback volumes.
- Short, overlapping, and partially obscured events.

Include negative examples that resemble target sounds but should not trigger the category.

Record the ground-truth label before running the detector. Keep training/tuning examples separate from final evaluation examples.

### 3.2 Metrics

| Metric | Definition | Why it matters |
|---|---|---|
| Precision | `TP / (TP + FP)` | How many emitted detections are correct |
| Recall | `TP / (TP + FN)` | How many target events are detected |
| F1-score | Harmonic mean of precision and recall | Balances precision and recall |
| False alerts per hour | False positive events divided by test hours | Operational nuisance rate |
| Miss rate | `FN / (TP + FN)` | Fraction of target events missed |
| Detection latency | Time from ground-truth event onset to confirmed detector event | Responsiveness |
| End-to-end haptic latency | Time from event onset to motor activation | User-perceived responsiveness |

For multi-class evaluation, publish a confusion matrix and per-class precision/recall/F1. Define how repeated detections during one continuous sound are grouped into a single ground-truth event.

### 3.3 Confidence terminology

The detector currently reports an aggregated YAMNet category score. Do not label it as a calibrated probability or write statements such as "91% accurate" based on that score.

If probability calibration is later required, use a held-out labeled dataset and document the calibration method and calibration error.

## 4. Latency measurement

Use timestamps from a common clock where possible. For end-to-end timing, capture:

1. Ground-truth sound onset.
2. Detector confirmation timestamp.
3. Backend event receipt timestamp.
4. Device command receipt/acknowledgement.
5. Motor electrical activation or measured vibration onset.

A phone video can help demonstrate sequence, but is not a substitute for precise latency instrumentation. Report median, 95th percentile, sample count, and test conditions rather than only the fastest run.

## 5. Haptic pattern validation

Test every category pattern at low, medium, and high configured intensity, using the actual motor and power source.

Record:

- Requested intensity.
- Pulse count.
- On/off duration.
- Measured motor supply voltage.
- Peak or average current where measurement equipment is available.
- Whether the pattern was physically distinguishable.
- Whether the motor/driver overheated or reset.
- Whether the firmware returned to standby after the pattern.

A software `haptic_complete` event proves the firmware completed its sequence; it does not prove that a person perceived the vibration.

## 6. Results table template

Fill this table only with measured results.

| Category | Trials | TP | FP | FN | Precision | Recall | F1 |
|---|---:|---:|---:|---:|---:|---:|---:|
| Alarm | — | — | — | — | — | — | — |
| Vehicle horn | — | — | — | — | — | — | — |
| Doorbell | — | — | — | — | — | — | — |
| Knocking | — | — | — | — | — | — | — |
| Siren | — | — | — | — | — | — | — |

### Latency report template

| Measurement | Median | 95th percentile | Samples | Test conditions |
|---|---:|---:|---:|---|
| Sound onset → detector event | — | — | — | — |
| Detector event → backend receipt | — | — | — | — |
| Backend command → device acknowledgement | — | — | — | — |
| Sound onset → motor activation | — | — | — | — |

## 7. Reproducibility package

For a strong release, publish:

- A test protocol and sound-source IDs.
- A CSV of trial-level results (without private audio).
- A confusion matrix.
- A latency summary.
- Hardware photographs and wiring revision.
- Firmware/software commit hash.
- Dependency versions.
- Known failures and limitations.

Do not upload copyrighted sound recordings or private environmental recordings unless you have permission and a clear distribution basis.
