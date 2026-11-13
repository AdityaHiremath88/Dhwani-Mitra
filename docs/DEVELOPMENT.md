# Development setup

This guide describes the current Windows PowerShell-oriented local workflow. Commands are based on the repository's launch scripts and source configuration.

## 1. Prerequisites

| Tool | Version / requirement |
|---|---|
| Windows | Windows 10 or 11 for the supplied PowerShell workflow |
| Python | 3.11 is used in the documented commands |
| Node.js | 18 or newer |
| npm | Bundled with Node.js |
| ESP-IDF | 6.1 |
| Hardware | ESP32-C6, TB6612FNG, vibration motor |
| Network | Shared 2.4 GHz Wi-Fi/LAN for laptop and ESP32-C6 |

The first detector run downloads/caches YAMNet and AudioSet labels. An internet connection is required for that initial model/label retrieval.

## 2. Clone the repository

```powershell
git clone https://github.com/poorvik-44/Dhwani-Mitra.git
cd Dhwani-Mitra
```

## 3. Start the backend

Open a PowerShell terminal at the repository root:

```powershell
py -3.11 -m venv .venv-backend
.\.venv-backend\Scripts\Activate.ps1
pip install -r .\backend\requirements.txt
python -m uvicorn app.main:app --app-dir .\backend --host 0.0.0.0 --port 8000
```

The backend exposes REST and WebSocket interfaces on port `8000`. Binding to `0.0.0.0` allows the ESP32-C6 to reach the host over the LAN. Do not expose this development server to an untrusted network.

Check:

```powershell
Invoke-RestMethod http://localhost:8000/api/health
```

## 4. Start the dashboard

Open another terminal:

```powershell
cd .\web
npm install
npm run dev
```

Open `http://localhost:5173`.

The frontend uses `VITE_API_URL` if set; otherwise it defaults to `http://localhost:8000`. For a different host address, set the variable before starting Vite, for example:

```powershell
$env:VITE_API_URL = 'http://192.168.1.10:8000'
npm run dev
```

Use the actual host address for your network. The dashboard's backend CORS allow-list is currently configured for localhost development origins; update it deliberately if you use a different browser origin.

## 5. Start the detector

Open another terminal at the repository root:

```powershell
py -3.11 -m venv .venv-detector
.\.venv-detector\Scripts\Activate.ps1
pip install -r .\detector\requirements.txt
```

Select the correct Windows input device in **Settings → System → Sound → Input**. Then run:

```powershell
Remove-Item Env:DHVANI_MIC_DEVICE -ErrorAction SilentlyContinue
.\detector\run_laptop_mic.ps1
```

To select a particular device by a name substring:

```powershell
$env:DHVANI_MIC_DEVICE = 'AB13X USB Audio'
.\detector\run_laptop_mic.ps1
```

The detector terminal prints the selected input device and microphone RMS level. Confirm that the RMS changes when sound is made near the microphone. The detector connects to `ws://127.0.0.1:8000/ws/detector` unless `DHVANI_DETECTOR_WS` is overridden.

## 6. Build and provision the ESP32-C6 wristband

Open an ESP-IDF 6.1 PowerShell environment. Confirm that the motor is connected through the TB6612FNG driver, not directly to an ESP32 GPIO.

```powershell
idf.py -C .\firmware\wristband set-target esp32c6
idf.py -C .\firmware\wristband build
idf.py -C .\firmware\wristband -p COMx flash monitor
```

Replace `COMx` with the serial port shown in Windows Device Manager.

At the serial prompt, provision the 2.4 GHz network and the laptop's LAN IPv4 address:

```text
wifi <SSID>|<password>|<laptop IPv4 address>
```

The firmware stores these values in ESP32 NVS and reboots. The laptop IP must be reachable from the ESP32-C6; do not enter `localhost` or `127.0.0.1`.

Useful serial commands:

```text
status
motor
```

`status` reports Wi-Fi and bridge connection state. `motor` requests one local 150 ms test pulse. Use it only after verifying the motor driver wiring and supply.

## 7. Recommended startup order

1. Start FastAPI.
2. Flash and provision the ESP32-C6 if not already configured.
3. Confirm the wristband reports a connection to the backend.
4. Start the dashboard and confirm its WebSocket connection.
5. Start the detector.
6. Confirm detector and microphone status in the dashboard.
7. Test a supported sound near the selected laptop microphone.
8. Confirm the event appears and progresses through delivery states.

## 8. Environment variables

| Variable | Default | Component | Purpose |
|---|---|---|---|
| `DHVANI_MIC_DEVICE` | Windows default input | Detector | Input device name substring or PortAudio input index |
| `DHVANI_DETECTOR_WS` | `ws://127.0.0.1:8000/ws/detector` | Detector | Backend detector WebSocket |
| `VITE_API_URL` | `http://localhost:8000` | Dashboard | Backend HTTP/WebSocket base URL |

## 9. Local runtime data

The backend creates its data directory and stores:

- `backend/data/patterns.json`
- `backend/data/history.json`

The dashboard stores local pattern preferences and custom sound-profile audio in browser `localStorage`. Do not use real private recordings in shared/demo browser profiles.

## 10. Common setup problems

### Backend starts but ESP32-C6 cannot connect

- Confirm the laptop and ESP32-C6 are on the same reachable network.
- Confirm the ESP32-C6 uses the laptop's current LAN IPv4 address.
- Confirm FastAPI is bound to `0.0.0.0:8000`.
- Check Windows Firewall for inbound TCP port `8000`.
- Check the ESP32 serial `status` output.

### Dashboard says backend disconnected

- Confirm FastAPI is running.
- Check `VITE_API_URL`.
- Check the browser console for failed HTTP/WebSocket requests.
- Confirm the browser origin is included in backend CORS settings.

### Detector selects the wrong microphone

- Select the intended input in Windows Sound settings.
- Set `DHVANI_MIC_DEVICE` to a distinctive name substring.
- Restart the detector and verify the printed device name.
- Check that the RMS level changes when making sound near that microphone.

### No category is detected

- Confirm the microphone level is non-zero and not below the configured RMS gate.
- Use a clear recording of one of the five mapped sound categories.
- Inspect the detector's printed top YAMNet predictions and category scores.
- Remember that YAMNet is a general sound-event classifier, not a perfect template matcher.

### Motor does not vibrate

- Run the local `motor` test from the ESP32 serial console.
- Verify the TB6612FNG wiring and motor supply.
- Confirm the device is connected to the backend.
- Inspect serial logs for rejected commands or a full haptic queue.
