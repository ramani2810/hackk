# ColdChain Sentinel – Windows 10/11 step-by-step guide

No prior experience needed. Every command below can be copied into the VS Code terminal (PowerShell).
You do **not** need Docker, PostgreSQL or any paid service. The database is SQLite, built into Node.js.

---

## Step 1 – Install the prerequisites

1. **Node.js 22.13 or newer (LTS)** – required.
   Download the Windows installer from https://nodejs.org, run it, keep all defaults (tick "Add to PATH").
   Check: open PowerShell and run `node -v`. It must print `v22.13.0` or higher (for example `v22.20.1` or `v24.x`).
2. **Visual Studio Code** – https://code.visualstudio.com (you said you already use it).
3. **Google Chrome** (or Edge) – for the website.
4. **Arduino IDE 2.x** – only needed in Step 12 for the ESP32 hardware. https://www.arduino.cc/en/software
5. **Git is NOT required.**

If PowerShell says scripts are disabled, run once: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` and answer `Y`.

## Step 2 – Extract the ZIP

1. Right-click `ColdChain_Sentinel_Final.zip` → **Extract All…**
2. Choose a normal writable folder such as `C:\Projects` (avoid `Program Files`, and avoid OneDrive-synced folders if you can).
   You will get `C:\Projects\coldchain-sentinel`.

## Step 3 – Open the project in VS Code

1. VS Code → **File → Open Folder…** → select `coldchain-sentinel` → *Yes, I trust the authors*.
2. Open a terminal: **Terminal → New Terminal**. It should show PowerShell and a path ending in `coldchain-sentinel`.
3. Check Node: `npm run check` → `Node.js v22.x OK`.

## Step 4 – Create the configuration (environment) files

```powershell
npm run setup
```
This creates `backend\.env` with two **random secret keys** and `frontend\.env`. You do not have to edit anything.
What the variables mean (all in `backend\.env`):

| Variable | Purpose |
|---|---|
| `PORT` | backend port, default 4000 |
| `FRONTEND_ORIGIN` | website address allowed to call the backend, default `http://localhost:5173` |
| `DEVICE_SECRET_ENC_KEY` | encrypts device secrets inside the database (generated for you) |
| `CHECKPOINT_SIGNING_KEY` | signs ledger checkpoints (generated for you). **Do not lose or share it.** |
| `CHECKPOINT_EVERY` | a signed checkpoint is created after this many new records per device (default 10) |
| `ENABLE_DEV_ENDPOINTS` | `true` enables the simulator and tamper lab (development only) |
| `SEED_*` | optional fixed passwords for the demo accounts; empty means random passwords |

Never commit `.env` files or paste their contents anywhere public.

## Step 5 – Install dependencies

```powershell
npm run install:all
```
Takes 1–3 minutes. Warnings about "deprecated" or "funding" are harmless. It must end without `ERR!` lines.

## Step 6 – Create and fill the database

```powershell
npm run db:migrate
npm run db:seed
```
`db:migrate` creates the SQLite files in `backend\data\`. `db:seed` creates demo data and prints **once**:

* the administrator login `admin@coldchain.local` and its random password,
* the viewer login `viewer@coldchain.local` and its password,
* the **device secret** for the physical ESP32 `COLD-BOX-01` (needed in Step 12).

**Copy these into a text file now.** They are not shown again. (Lost them? Start over with `npm run db:reset -- --yes`
then migrate and seed again; or, as admin, use *Devices → Rotate credential* for the device secret.)

## Step 7 – Start the backend and Step 8 – the website

The simple way (one terminal runs both):

```powershell
npm run dev
```
Expected output (shortened):
```
  ColdChain Sentinel backend is running
  - API:          http://localhost:4000/api/v1
  - Health check: http://localhost:4000/api/v1/health
  VITE v8.x  ready   ➜  Local: http://localhost:5173/
```
Health check: open http://localhost:4000/api/v1/health in Chrome. You should see `{"status":"ok", ...}`.

If you prefer two terminals: terminal 1 `npm run start:backend`; open a second terminal with the **+** button, then `npm run start:frontend`.

Open **http://localhost:5173** in Chrome. Click **Sign in**, use the admin e-mail and password from Step 6.

*Alternative single-process mode:* `npm run start:prod` builds the website and serves everything from http://localhost:4000.

## Step 9 – Check that everything works

1. Top bar shows **SIMULATION** (amber), **Backend connected** (green).
2. Go to **Live monitoring** → scroll to the **Simulator** card → **Start simulator** (or **Generate 30 readings now**).
3. Within seconds the chart, the readings table and the counters change. They come from the backend and are stored in SQLite
   (press F5 or restart the backend: the records are still there).
4. Open **Integrity ledger** → **Verify chain**: result **Verified**, with the number of records checked.
   Records show **Simulated** badges, never **Hardware**.

## Step 10 – Demonstrate tamper detection

1. Sidebar → **Tamper lab** (administrator only; works on a separate disposable database, your real ledger is never touched).
2. **Verify sandbox ledger** → "All hashes match".
3. Record sequence `12`, new temperature `3.3`, click **Change temperature**, then **Verify sandbox ledger** →
   record 12 *Invalid hash*, record 13 *Broken link*; the table shows original versus modified values.
4. **Reset sandbox**. Click **Edit and re-hash the whole chain**, **Verify** → *Checkpoint mismatch* (the signed checkpoint exposes the rewrite).
5. Try **Delete record** and **Swap with next record** (reset in between). **Reset sandbox** restores everything.
6. On *Live monitoring* click **Trigger temperature excursion**: you get an amber **temperature** alert, while **Verify chain** on the real ledger stays **Verified**.
   A warm reading is not tampering.

## Step 11 – Run the automated tests

```powershell
npm test
```
Success looks like:
```
 Test Files  4 passed (4)
      Tests  97 passed (97)
```
Also: `npm run typecheck` (no output = OK) and `npm run build` (frontend production build).

## Step 12 – Connect the physical ESP32

Hardware list and wiring: **docs\WIRING.md** (DS18B20 DATA → GPIO 4 with 4.7 kΩ pull-up to 3V3, OLED SDA → GPIO 21, SCL → GPIO 22, power 3V3/GND).

1. **Find your laptop's LAN IP:** `ipconfig` → under your Wi-Fi adapter note **IPv4 Address**, e.g. `192.168.1.50`.
   The ESP32 and laptop must be on the **same Wi-Fi network** (2.4 GHz for the ESP32). "Guest" networks with client isolation will not work.
2. **Allow the port through Windows Firewall** (PowerShell opened as Administrator, once):
   ```powershell
   New-NetFirewallRule -DisplayName "ColdChain Sentinel 4000" -Direction Inbound -Protocol TCP -LocalPort 4000 -Action Allow -Profile Private
   ```
   (Set your Wi-Fi network profile to *Private* in Windows Settings → Network, or use `-Profile Any` on a trusted network.)
3. **Arduino IDE:** *File → Preferences → Additional boards manager URLs*: add
   `https://espressif.github.io/arduino-esp32/package_esp32_index.json`. *Tools → Board → Boards Manager* → install **esp32 by Espressif Systems**.
4. **Libraries:** *Tools → Manage Libraries* → install **OneWire**, **DallasTemperature**, **Adafruit SSD1306**, **Adafruit GFX Library** (accept dependencies).
5. Open `firmware\coldchain_esp32\coldchain_esp32.ino`. In the same folder **copy `config.example.h` to `config.h`**
   and set `WIFI_SSID`, `WIFI_PASSWORD`, `BACKEND_URL` (`http://<your-laptop-IP>:4000/api/v1/readings`, **never localhost**),
   `DEVICE_ID` (`COLD-BOX-01`) and `DEVICE_SECRET` (the 64 characters from Step 6).
6. *Tools → Board → ESP32 Arduino → ESP32 Dev Module*, *Tools → Port* → the COM port that appears when you plug in the ESP32
   (install the CP210x or CH340 USB driver if no port appears). Click **Upload**. If it stalls at "Connecting…", hold the **BOOT** button on the board until upload starts.
7. *Tools → Serial Monitor*, **115200 baud**. You should see `[sample] seq=…` lines. The OLED shows temperature, `WiFi: OK`, `Server: OK`.
8. In the website: top bar changes to **LIVE HARDWARE**, **Devices and sensors** shows `COLD-BOX-01` online, readings appear with a **Hardware** badge.
   Hold the probe in your hand to see the temperature rise; run **Verify chain** afterwards.

Software tests cover the signed-request path; the firmware itself needs your physical hardware to be tested.

## Step 13 – Troubleshooting

| Problem | Fix |
|---|---|
| `Port 4000 is already in use` / `EADDRINUSE` | An older server is still running. Close other terminals, or find it: `netstat -ano \| findstr :4000` then `taskkill /PID <number> /F`. Or change `PORT` in `backend\.env` and `VITE_BACKEND_URL` in `frontend\.env`. |
| `npm install` fails / `ERR!` | Check internet; make sure `node -v` ≥ 22.13; delete the `node_modules` folders and run `npm run install:all` again; avoid OneDrive/very long paths. |
| `Missing environment variable DEVICE_SECRET_ENC_KEY` | You skipped Step 4: run `npm run setup`. |
| `SQLite`/database errors, "no such table" | Run `npm run db:migrate`. For a clean start: `npm run db:reset -- --yes`, then migrate and seed. |
| Website shows "Backend unreachable" or login page says backend not reachable | Backend not running or wrong port. Open http://localhost:4000/api/v1/health. Restart with `npm run dev`. |
| CORS error in the browser console | Open the site at exactly `http://localhost:5173`; if you use another address, set `FRONTEND_ORIGIN` in `backend\.env` to it and restart. |
| "Too many failed login attempts" | Wait 15 minutes or restart the backend. |
| Lost admin password | `npm run db:reset -- --yes`, `npm run db:migrate`, `npm run db:seed` (this wipes local data). |
| ESP32 cannot join Wi-Fi | 2.4 GHz only, check SSID/password spelling (case-sensitive), move closer. |
| OLED stays blank | Swap SDA/SCL, try `OLED_ADDRESS 0x3D`, check 3V3 power. |
| Sensor shows SENSOR ERR / -127 / 85 | DATA on GPIO 4? 4.7 kΩ resistor between DATA and 3V3? Loose breadboard wire? |
| OLED says `Server: retry`, Serial shows `http=-1` | Wrong `BACKEND_URL`, backend not running, or Windows Firewall blocking port 4000 (Step 12.2). Test from your phone on the same Wi-Fi: `http://<laptop-IP>:4000/api/v1/health`. |
| OLED says `AUTH FAIL`, Serial `http=401` | `DEVICE_ID`/`DEVICE_SECRET` wrong, or credential revoked/rotated. Use *Devices → Rotate credential* and paste the new secret. |
| Ledger verification **fails** | Expected after the tamper lab on the *sandbox*. On the **real** ledger, a failure means stored records were changed outside the app (e.g. someone edited the database). Open the failure details; records are preserved, not repaired. Use *Reset demo data* only for simulated data. |
| "Verified" never appears | Nothing has been verified yet: click **Verify chain**. Records show *Not yet verified* until a run covers them. |

## Step 14 – Stop the application

* Click in the terminal running `npm run dev` and press **Ctrl + C** (answer `Y` if PowerShell asks to terminate the batch job).
* The simulator stops with the backend. To stop only the simulator: *Live monitoring → Stop simulator*.
* Closing VS Code with the terminal open also ends the processes. Check nothing is left: `netstat -ano | findstr :4000` should print nothing.
* To start again later: open the folder in VS Code and run `npm run dev`.
