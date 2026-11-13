# Security and privacy

Dhvani Mitra is currently a local-network prototype. The security posture described here is intentionally conservative.

## 1. Audio handling

The active detector captures audio from the host computer and runs YAMNet locally. It sends category events, confidence-like scores, timestamps, microphone metadata, and RMS levels to the FastAPI backend. The raw microphone waveform is not sent to the backend by the current detector implementation.

The dashboard's custom sound-profile feature is separate: it records or uploads audio in the browser and stores it as a data URL in browser `localStorage` so the user can replay it. These custom recordings are not uploaded to the detector, do not train YAMNet, and do not create automatically recognized sound classes.

Before recording or testing in shared spaces:

- Obtain consent where required.
- Avoid capturing conversations or identifiable private audio.
- Do not commit recordings to the repository.
- Clear browser-local custom recordings when no longer needed.
- Do not use real private recordings in public demos or screenshots.

## 2. Network exposure

The backend is launched with `--host 0.0.0.0` so the ESP32-C6 can connect over the local network. The current API and WebSocket implementation does not provide user authentication or authorization. The device uses `ws://` rather than TLS-protected `wss://`.

Therefore:

- Run the prototype only on a trusted, controlled network.
- Do not port-forward port `8000` to the public internet.
- Do not deploy the backend on a public server without adding authentication, authorization, TLS, rate limiting, and input hardening.
- Restrict firewall access to the host and expected device where practical.
- Treat WebSocket messages as untrusted input even on a local network.

The frontend's CORS allow-list is configured for local development origins. CORS is not authentication and does not prevent non-browser clients from connecting.

## 3. Credentials

The ESP32 firmware stores the Wi-Fi SSID, password, and backend host in NVS. Do not commit credentials, screenshots containing passwords, private environment files, or serial logs that expose secrets.

The repository should not contain:

- Real Wi-Fi credentials.
- Personal access tokens.
- Private `.env` files.
- Private microphone recordings.
- Personal device identifiers unless intentionally documented.

NVS storage alone should not be described as encrypted credential storage unless NVS encryption and the associated secure-boot/flash-encryption configuration are explicitly enabled and verified.

## 4. Data retention

The backend stores up to 500 detection-history events in a local JSON file. These events include event identifiers, categories, confidence-like scores, timestamps, pattern names, and delivery status. They do not include the raw microphone waveform in the current detector flow.

The dashboard can clear history through the backend API. The backend JSON files and browser `localStorage` are local runtime data; they should be excluded from version control.

## 5. Production-readiness checklist

Before any deployment beyond a trusted local prototype:

- [ ] Add authenticated users and device identity.
- [ ] Authorize every device-control operation.
- [ ] Use TLS for HTTP and WebSocket traffic.
- [ ] Store credentials using an appropriate protected mechanism.
- [ ] Add request size limits, rate limits, and strict message validation.
- [ ] Add audit logs without retaining unnecessary audio.
- [ ] Define retention and deletion behavior.
- [ ] Threat-model malicious or compromised local-network clients.
- [ ] Test firmware behavior for malformed, replayed, duplicated, and stale commands.
- [ ] Publish a privacy notice explaining what is captured, processed, transmitted, and retained.

## 6. Safety statement

Dhvani Mitra is an experimental assistive-technology prototype. It is not certified, clinically validated, or safety-rated. Do not rely on it as the only means of receiving alarms, detecting hazards, or responding to emergencies.
