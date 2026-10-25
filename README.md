# ColdChain Sentinel
**Tamper-Evident IoT Monitoring for Pharmaceutical Cold Chains** · *Every Reading Accounted For. Every Record Verifiable.*

> Educational prototype. It is **not** a certified, validated or regulator-approved medical monitoring system and makes no claim of guaranteed pharmaceutical safety.

New to this? Follow **[RUN_INSTRUCTIONS_WINDOWS.md](RUN_INSTRUCTIONS_WINDOWS.md)**. Judges: **[docs/DEMO_SCRIPT.md](docs/DEMO_SCRIPT.md)**.

## 1. Overview
An ESP32 with a DS18B20 sensor and OLED sends temperature readings over Wi-Fi to a Node.js backend. The backend authenticates each device
(HMAC-SHA-256), validates the data, links every reading to the previous one with SHA-256 in a per-device hash chain, stores it in SQLite,
raises temperature alerts, and exposes a React dashboard with live updates and a real verification service that detects edits, deletions,
reordering and gaps. Everything also runs without hardware through a software simulator that uses the same ingestion pipeline.

## 2. Problem statement
Temperature-sensitive medicines must stay within product-specific limits in transit. A conventional monitor can display readings, but
history can be lost, altered or deleted. ColdChain Sentinel makes unauthorized changes to stored history detectable and keeps *genuine
temperature excursions* clearly separate from *data-integrity violations*.

## 3. Objectives
Measure → authenticate → validate → hash-chain → store → display → verify. Detect modified, missing, reordered and gapped records.
Distinguish unsafe-but-genuine readings from tampering. Work end to end in simulation before hardware is attached.

## 4. Features
* **Landing page** with an interactive in-browser SHA-256 chain demo, and an authenticated app (collapsible sidebar, 9 pages + Tamper lab).
* **Overview, Live monitoring, Devices, Shipments, Integrity ledger, Security alerts (security + temperature), Audit logs, Reports, Settings.**
* **Hash-chain ledger** with canonical serialization, genesis value, full / per-device / time-range verification, per-record status, record drawer, copy-hash buttons.
* **Signed checkpoints** (HMAC-SHA-256) in an append-only file outside the database; detect a "perfectly re-hashed" rewrite.
* **Alerts:** TEMP_EXCURSION (peak, duration, auto-recovery after 2 in-range readings), TEMP_RECOVERED, SENSOR_DISCONNECTED, INVALID_MEASUREMENT, STALE_READINGS;
  HASH_MISMATCH, BROKEN_LINK, MISSING_RECORD, SEQUENCE_GAP, REORDERED_RECORDS, CHECKPOINT_MISMATCH, INVALID_DEVICE_AUTH, REPEATED_FAILED_REQUESTS, SEQUENCE_CONFLICT.
* **Tamper lab:** isolated sandbox database; modify / delete / reorder / "skilled attacker re-hash" / append; original-vs-modified table; reset to byte-identical data.
* **Simulator** (start/stop, interval, device, trigger excursion/disconnect, generate N readings, reset demo data). Records are labelled SIMULATOR, devices are separate from hardware.
* **Reports:** temperature CSV/JSON, ledger verification, incidents, shipment integrity summary (CSV formula-injection safe, with disclaimers).
* **Security:** scrypt passwords, httpOnly cookie sessions (hashed server side), CSRF header + origin check, login & ingestion rate limits, Helmet, Zod validation, 8 kB body limit, secrets AES-256-GCM encrypted and shown once, log redaction, admin/viewer roles.

## 5. Architecture
```
DS18B20 ─► ESP32 ──HMAC-signed JSON──► Express API ─► validate ─► BEGIN IMMEDIATE tx ─► readings (hash chain) ─► SQLite
 OLED ◄─┘                                  │                         └► temperature alerts   └► signed checkpoint file (outside DB)
                       Simulator ─────────►┘ (same ingestReading() service)
React dashboard ◄── REST + Socket.IO ◄──── Verifier (recomputes every hash, link, sequence, checkpoint) ─► security alerts
```
Hash of a record = `SHA256("coldchain-sentinel/record/v1\n" + canonicalJSON(fields incl. previousHash))`; canonical JSON has sorted keys,
no whitespace, and fixed-precision strings for numbers. First record's `previousHash` = `SHA256("coldchain-sentinel/genesis/v1:" + deviceId)`.
The verifier never trusts a stored hash: it recomputes it, and links each record to the *recomputed* hash of its predecessor.

## 6. Technology stack
Frontend: React 19, TypeScript, Vite, Tailwind CSS 4, Recharts, React Router, TanStack Query, Lucide icons, Socket.IO client (own component set instead of shadcn/ui).
Backend: Node.js ≥ 22.13, Express 5, TypeScript (run with tsx), Zod, Socket.IO, **SQLite via Node's built-in `node:sqlite`** (see Limitations for why not PostgreSQL/Prisma).
Firmware: Arduino C++ (ESP32 core, OneWire, DallasTemperature, Adafruit SSD1306/GFX, mbedTLS). Tests: Vitest + Supertest.

## 7. Folder structure
```
coldchain-sentinel/
  backend/   src/{routes,services,middleware,db,lib,scripts}  migrations/001_init.sql  tests/  .env.example
  frontend/  src/{pages,components,app,lib}  index.html  vite.config.ts
  firmware/coldchain_esp32/  coldchain_esp32.ino  config.example.h
  docs/      WIRING.md  API.md  DEMO_SCRIPT.md
  scripts/   setup.mjs  dev.mjs  start-single.mjs  check-node.mjs
  README.md  RUN_INSTRUCTIONS_WINDOWS.md  package.json
```

## 8–13. Prerequisites, install, configure, database, run
Node.js ≥ 22.13. From the project root:
```bash
npm run check          # Node version
npm run setup          # creates backend/.env with random keys and frontend/.env
npm run install:all    # installs backend + frontend dependencies
npm run db:migrate     # creates SQLite databases in backend/data
npm run db:seed        # demo users (random passwords printed ONCE), simulated devices, COLD-BOX-01 + its secret, sandbox
npm run dev            # backend :4000 + frontend :5173   (or npm run start:backend / npm run start:frontend separately)
```
Open http://localhost:5173. Health: http://localhost:4000/api/v1/health. Single process: `npm run start:prod` → http://localhost:4000.
Reset local data: `npm run db:reset -- --yes`. Register another ESP32: `npm run device:register --prefix backend -- COLD-BOX-02 "Name"` or *Devices* page.

Environment variables: see `backend/.env.example` (every variable is commented).

## 14. Simulation mode
*Live monitoring → Simulator → Start simulator.* The mode badge shows **SIMULATION** unless a physical device reported within the stale limit (60 s by default), then **LIVE HARDWARE**.

## 15. Tampering demonstration
*Tamper lab* (administrator, development only). See the script in docs/DEMO_SCRIPT.md. The endpoints are `404` when `NODE_ENV=production` or `ENABLE_DEV_ENDPOINTS=false`.

## 16–17. ESP32 integration and wiring
See **docs/WIRING.md** and Step 12 of the Windows guide. Pins: DS18B20 DATA → GPIO 4 (4.7 kΩ to 3V3), OLED SDA → GPIO 21, SCL → GPIO 22, all at 3.3 V.
The firmware reads every 5 s, flags `-127` (disconnected) and `85.0` (invalid), persists sequence numbers across reboots by reserving blocks in flash,
queues up to 40 unsent readings and retries with backoff using the identical signed body, syncs time with NTP (omits `recordedAt` until synced), and shows temperature, Wi-Fi and server status on the OLED.

## 18. Automated tests
`npm test` → 97 tests (hash vectors, canonicalization, chain creation, modified/deleted/reordered/gap detection, checkpoints, duplicate and conflicting ingestion,
invalid HMAC, 40- and 100-way concurrent ingestion, temperature-vs-integrity alert separation, RBAC/CSRF/rate limit, validation, simulator, end-to-end sandbox tampering, reports).

## 19. API endpoints
See **docs/API.md**.

## 20. Troubleshooting
See Step 13 of RUN_INSTRUCTIONS_WINDOWS.md.

## 21. Security limitations (please read)
* The hash chain protects the integrity of the **server-held history**. It does **not** prove a sensor measured the true temperature: a compromised, miscalibrated or spoofed sensor with a valid secret can send false values.
  The backend computes the authoritative hashes after authenticating the device (beginner-friendly design); device-side signing with a secure element is future work.
* Someone who controls the whole server (database **and** `.env` **and** checkpoint files) can rewrite everything. Checkpoints only give independent protection once copied to separate storage (WORM bucket, second server, e-mail, transparency log) or signed with a key in a KMS/HSM. In this prototype they are in `backend/data/checkpoints` and the signing key is in `backend/.env`.
* Records written after the newest checkpoint are protected only by the chain (a checkpoint is created every 10 records by default).
* The ESP32 talks plain HTTP on your LAN in this prototype. Request bodies are authenticated (HMAC), but not encrypted; put the backend behind HTTPS (reverse proxy) and add TLS to the firmware for any real deployment. Replay of a captured request is harmless: it is de-duplicated by (device, sequence).
* Sessions use cookies without `Secure` in development; set `NODE_ENV=production` and serve over HTTPS for deployment.
* Not calibrated, validated or certified. Reports say so.

## 22. Future improvements
Device-side signing / secure element, external checkpoint publication, TLS pinning on ESP32, OTA updates, PostgreSQL + Prisma for multi-user scale, humidity sensor (DHT22/SHT31) UI, GPS and door sensors, multi-factor authentication, e-mail/SMS alerts.

## Honest deviations from the original brief
* **SQLite (`node:sqlite`) instead of PostgreSQL + Prisma**: the brief allows SQLite for a simpler local setup; it needs no install on Windows. Migrations are plain SQL (`backend/migrations`). Because it is Node's built-in module it is still marked "experimental" in Node 22 (the warning is suppressed).
* **No shadcn/ui**: a small consistent component set is hand-built with Tailwind.
* **No Docker**: not needed.
* **Express/TypeScript runs through `tsx`** (no separate compile step).
* **Humidity**: accepted, stored and shown on the Live page when a device sends it; the DS18B20 itself measures only temperature, so the supplied firmware sends none.
