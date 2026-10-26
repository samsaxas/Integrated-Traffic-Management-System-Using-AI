# ITMSAI - Integrated Traffic Management System Using AI

ITMSAI is a secure web application with three AI-assisted services behind a login:

* **Traffic Control (administrators)**: upload a junction or road video. YOLOv8 finds and counts vehicles,
  and the system proposes the lanes by itself. The operator confirms or redraws them. The lane counts drive
  a signal controller (`main.py`) running on a simulated Raspberry Pi Pico in **Wokwi**. The dashboard shows
  the annotated video next to **four 3D traffic lights** that light exactly the LEDs Wokwi reports.
* **ANPR (everyone)**: upload a photo or video. The project's YOLOv8 licence-plate model
  (`license_plate_detector.pt`) finds the plates, and **Google Gemini** (`gemini-3.5-flash`, backup
  `gemini-3.5-flash-lite`) reads them. The text appears under the image or video, with boxes drawn on it.
* **Cyber Crime Hub**: citizens file complaints with evidence and location, follow their status, and get
  emergency numbers, nearby police on a map, location sharing and city news. Administrators get a case
  dashboard (pending, solved, rejected), the full workflow with remarks, a map, and the complete history of
  every case.

> How to install and run it, step by step with every command: **[README_RUN.md](README_RUN.md)**.

---

## Version 3 — Integrated System

Version 3 is the advanced integrated implementation of the ITMSAI project. Building upon the earlier versions, it brings AI-assisted traffic monitoring, adaptive traffic signal control, Automatic Number Plate Recognition (ANPR), and the Cyber Crime Hub together in a secure web application.

The traffic controller operates through a Raspberry Pi Pico simulation in Wokwi, replicating the intended real-world hardware setup. Physical hardware deployment is not part of this implementation.


## 1. Who sees what

| Page | Visitor | User | Administrator |
|---|:-:|:-:|:-:|
| Landing page (log in / register only) | ✔ | | |
| Home with role-based tiles | | ✔ | ✔ + system overview |
| ANPR - number-plate check | | own scans | all scans + plate search |
| Cyber Crime Hub (report, my complaints, map & police, emergency & location, news) | | ✔ | |
| Cyber Crime Admin (statistics, cases, workflow, evidence, map, history) | | | ✔ |
| Traffic Control (video + lanes + Wokwi + 3D lights, run history) | | | ✔ |
| Users & Audit (accounts, lock-outs, audit trail) | | | ✔ |
| My account (profile, password, active sessions) | | ✔ | ✔ |

Access is enforced on the server for every page and every API call, not just hidden in the menu. Only the
administrator in `.env` (`ADMIN_EMAIL` / `ADMIN_PASSWORD`) is an admin at first. An admin can promote other
accounts on the Users page.

## 2. Architecture

```
 Browser (any OS / phone)                     ITMSAI server (FastAPI, Python)                 MongoDB
 ─────────────────────────                    ───────────────────────────────                 ───────
 HTML + JS (no framework)  ── HTTPS/JSON ──▶  auth · sessions · CSRF · rate limits  ───────▶  users, sessions,
 three.js 3D signals       ◀── SSE events ──  traffic engine (YOLO, lanes, overlay)           complaints (encrypted),
 Leaflet maps              ◀── MJPEG video ──  ANPR (plate YOLO + Gemini)                       anpr_scans, traffic_runs,
                                               cyber crime hub · audit · mailer                 audit_log, GridFS files
                                                   │                 │
                                     RFC 2217 serial                 SMTP (Gmail etc.)   Gemini API
                                                   ▼
                              VS Code + Wokwi: simulated Pico runs wokwi/main.py
                              (12 LEDs = 4 signals; prints "Lane 1: GREEN | Lane 2: RED | ...")
```

* **One process, one origin.** FastAPI serves the API and the static front-end, so there is no CORS to
  configure. Background threads run the video pipeline, ANPR video jobs and the Wokwi link.
* **Wokwi link.** Through the simulated Pico's serial port (`rfc2217://localhost:4000`), the server does
  what `mpremote run main.py` did in the original project, over a single connection. It enters
  MicroPython's raw REPL, soft-resets, uploads `wokwi/main.py` and starts it. Then it sends `CONFIG,<lanes>`,
  plus the lane counts `c1,c2,c3,c4` once a second, like the original `vehicle_detector.py`. Every line
  `main.py` prints is read back. Light-state lines become the state of the 3D lights. The dashboard
  **never invents a light state**: if Wokwi is not running, the lights are dark and the page says so.
* **Models run with ONNX Runtime**: no PyTorch, small, and the same on Windows, macOS, Linux, x86-64 and
  ARM64. `license_plate_detector.pt` was converted with the included dependency-free converter
  (`backend/app/vision/pt2onnx.py`). Its output was checked against Ultralytics' own export of yolov8n, with
  a maximum box difference under 0.001 px. If the `.onnx` file is missing, the server converts the `.pt` at
  start-up.

## 3. Traffic Control in detail

1. **Upload** any MP4/AVI/MOV/MKV/WEBM (up to 500 MB). Each run uses a newly uploaded video. The
   previous video is deleted when a new one is uploaded.
2. **Automatic lanes** (`backend/app/vision/lane_auto.py`). YOLO and a tracker watch the first 20 s and
   record where each vehicle touches the road. Two cases are handled:
   * **Road view** (all flows roughly parallel). Vehicle paths are grouped by direction, so opposite
     carriageways are separated. The number of lanes is measured using **the vehicles as a ruler**: a lane
     is about 1.9 car-widths, so the spread of the paths ÷ (1.9 × median car width) + 1 gives the lane
     count. The paths are then split by optimal 1-D k-means. Each lane becomes a perspective-correct
     quadrilateral, numbered left to right.
   * **Junction view** (flows crossing at large angles, like the original aerial-junction video). Each flow
     becomes one **approach region**: its incoming road up to where it meets crossing traffic. Queues of
     stopped vehicles that are not covered yet become regions too.
   * At most 4 lanes are proposed, one per signal. If there was too little traffic, the system says so and
     asks for manual drawing.
3. **Confirm**. The proposal is drawn on the first frame. The operator accepts it, drags corners, deletes
   lanes, or draws new ones (mouse, pen or touch). The run records whether the lanes were automatic,
   edited or manual.
4. **Live detection**. Each frame is analysed in real time, dropping late frames so the video never falls
   behind. The overlay matches the original "Smart Traffic Analyzer" window: green boxes with `car 0.83`,
   yellow lanes with `Lane 1: 8 vehicles`, and a black box with yellow `Lane 1: 8` lines. Counts are smoothed
   over 5 frames. Unique vehicles per lane and the peak queue are recorded.
5. **Wokwi**. Counts go to `main.py` every second. `main.py` runs the adaptive cycle from the original
   project:
   * all lanes empty → all red;
   * light traffic everywhere → the normal cycle;
   * heavy lanes present → busiest first, with extended green.

   Lanes that a video does not use are switched **OFF** (`CONFIG,n`), so a 3-lane video lights 3 signals.
6. **History**. Every run (video name, operator, lanes and their source, final totals) is stored in
   `traffic_runs` and listed on the page.

## 4. ANPR in detail

* Plates are found with `license_plate_detector` (YOLOv8n, 1 class), keeping plate-shaped boxes only.
  A padded, upscaled crop of each plate is sent to Gemini with a strict JSON schema
  (`plate`, `readable`, `confidence`) and a prompt for Indian formats such as `KA01AB1234` or `22BH1234AA`.
  Gemini is told not to guess unreadable characters. `thinkingLevel: minimal` keeps it fast and cheap.
* **Fallback chain**: `GEMINI_MODEL`, then `GEMINI_FALLBACK_MODEL` (on quota/overload/404/timeout/bad
  answer), then Tesseract (offline, if installed). Each plate shows which reader produced it. Note:
  Google lists `gemini-3.5-flash` as deprecated and forwards its requests to `gemini-3.6-flash`; both model
  names are settings in `.env`.
* **Images**: boxes are drawn on the photo, and the plate text, crop, confidence and model are listed below.
* **Videos**: frames are sampled 5 times a second, and plates are tracked across frames. Only the
  **sharpest crop of each plate** is read, so a one-minute clip costs a handful of Gemini calls. The browser
  plays the original video and draws the boxes and text from the stored timeline. Clicking a plate jumps to
  the frame where it was read. Videos in codecs browsers cannot play (AVI, MKV, HEVC, MPEG-4 Part 2) are
  converted to H.264 when ffmpeg is installed (it is in the Docker image).
* Media and crops are stored in MongoDB GridFS. Users see only their own scans. Administrators can search
  every scan by plate number.

## 5. Cyber Crime Hub in detail

**Citizen**
* **Report a crime**:
  * 14 categories modelled on India's national cyber-crime portal (UPI fraud, sextortion, loan-app
    harassment, ...).
  * Title and description, incident time, money lost, suspect details (phone/UPI/handles), contact number.
  * Location picked on a map or taken from the device's GPS, plus an address.
  * Up to 5 evidence files of 25 MB each (images, video, PDF, text, DOCX), and a declaration.
  * Each complaint gets an acknowledgement number `CCH-2026-000123` and a confirmation e-mail.
  * Complaints of ₹1 lakh or more are flagged high priority.
* **My complaints**: status, full timeline with the cyber cell's remarks, evidence list, "add information".
* **Map & nearby police**: police stations within 5 km from OpenStreetMap (Overpass), with directions.
* **Emergency & location**: tap-to-call national numbers (112, 100, 1930, 101, 102, 108, 181, 1098,
  14567, 1073, 1070). "Share my location" creates a map link and opens the phone's share sheet.
* **City news**: Google News RSS by default; NewsAPI or GNews keys are optional.

**Administrator**
* Statistics: total, pending, investigating, solved, rejected, open high-priority cases, money reported
  lost, average days to resolve, complaints by category, new complaints per day (30 days).
* Case list with filters (status, category, text search) and a map coloured by status.
* Case file: decrypted details, evidence download with SHA-256 integrity check, complete history.
* Workflow: Submitted → Under review → Investigation → Resolved / Rejected → Closed. Reopening is allowed
  where it makes sense, and invalid jumps are refused. A remark is required to resolve or reject. Every
  change e-mails the citizen and is written to the audit trail.

## 6. Security

| Area | Protection |
|---|---|
| Registration | An account is created **only after** the 6-digit code e-mailed to the address is entered. The address must be valid, must not be a disposable-mailbox domain, and its domain must publish MX records; addresses the mail server refuses are rejected. Pending registrations expire after 10 minutes. A hidden honeypot field stops simple bots. |
| Account enumeration | Login, registration and password reset give identical answers whether or not an e-mail exists. Unknown accounts take the same time to check (a dummy hash is verified). |
| Passwords | scrypt (N=2^15, r=8, p=1, per-password salt). Policy: at least 10 characters, upper case, lower case, digit and symbol; common passwords and passwords containing your name or e-mail are refused. Changing or resetting a password signs out other devices and sends an e-mail. |
| Brute force | Rate limits per IP and per account (login, registration, codes, reset, uploads, complaints). Lock-out after 5 wrong passwords for 15 minutes, with an e-mail to the owner. One-time codes allow 5 attempts. |
| Sessions | Random 256-bit token in an **HttpOnly, SameSite=Strict** cookie (Secure over HTTPS); only an HMAC of it is stored. **30-minute inactivity timeout** with a 2-minute warning, an 8-hour absolute limit, server-side revocation, a list of active sessions and "sign out everywhere". Background video and event streams do not keep a session alive, and they stop when it expires. |
| CSRF / clickjacking | Per-session CSRF token plus Origin check on every change. `frame-ancestors 'none'` and `X-Frame-Options: DENY`. |
| XSS / injection | Strict Content-Security-Policy (no inline script except the hashed three.js import map, no `unsafe-inline` styles). The UI only inserts text with `textContent`. Pydantic validation with type and length limits, so `{"$ne": ...}`-style NoSQL injection is impossible. Unicode is normalised and control/bidi characters are stripped. |
| Uploads | Size checked before and while streaming. The real type is read from the file's bytes, not its name. Random storage names. Served with `nosniff`; non-images are served as attachments in a sandbox CSP. |
| Data at rest | Complaint description, suspect details and phone number, and every evidence file, are encrypted with **AES-256-GCM** bound to the case number. Evidence keeps its SHA-256 for chain of custody, and every download is checked and audited. |
| Authorization | Role checks on every endpoint, ownership checks on every complaint and scan (another user's ID returns "not found"), and pages redirected on the server. |
| Audit trail | Every login, failure, lock, registration, password change, admin action, complaint view/update, evidence download, ANPR scan and traffic run is recorded with time, user, IP and browser. |
| Transport & headers | HSTS when on HTTPS, `Referrer-Policy: no-referrer`, a restrictive Permissions-Policy, COOP/CORP, no server banner, `Cache-Control: no-store` on all data. An optional Caddy service gets Let's Encrypt certificates automatically. |
| Secrets & infrastructure | All secrets are in `.env`, generated randomly by `scripts/init_env.py` and never committed; production refuses to start without them. MongoDB requires authentication. The app uses a least-privilege database user, and the database port is bound to localhost only. The container runs as a non-root user with a read-only filesystem, all Linux capabilities dropped and `no-new-privileges`. API documentation is disabled in production. |

## 7. Directory structure

```
itmsai/
├── README.md                    this file (what the system is)
├── README_RUN.md                how to install, run, test and deploy - every command
├── run.py                       start the server on any OS (checks Python, packages, .env, MongoDB)
├── docker-compose.yml           MongoDB + app (+ optional HTTPS proxy)
├── .env                         your configuration (created by scripts/init_env.py, never commit)
├── backend/
│   ├── requirements.txt         runtime dependencies (no PyTorch)
│   ├── Dockerfile
│   ├── app/
│   │   ├── main.py              app, security headers, page access control
│   │   ├── config.py            all settings (.env)
│   │   ├── db.py · migrations.py · audit.py · mailer.py · uploads.py
│   │   ├── security/            passwords, sessions, crypto, rate limits, validators
│   │   ├── routers/             auth, admin, traffic, anpr, crime, system
│   │   ├── services/            traffic engine, wokwi_link, anpr, gemini, news
│   │   └── vision/              detector, tracker, lane_auto, lane_learn, overlay, plate_text, pt2onnx, models
│   └── tests/                   unit tests, API tests, stand-ins (mail, Gemini, Pico)
├── frontend/                    landing, login, register, forgot, app/*.html, js/, css/, vendor/ (three.js, Leaflet)
├── models/                      yolov8n.onnx, license_plate_detector.onnx (+ the original .pt files)
├── wokwi/                       diagram.json (Pico + 12 LEDs), wokwi.toml, main.py
├── scripts/                     init_env.py, run_tests.py, wokwi_relay.py
├── docker/                      MongoDB init script, Caddyfile
└── docs/                        HARDWARE.md, SECURITY.md, API.md, TESTING.md
```

## 8. Database (MongoDB)

| Collection | Contents |
|---|---|
| `users` | name, e-mail (unique), phone, role, scrypt hash, lock-out counters, known devices |
| `sessions` | HMAC of the session token, CSRF token, expiry times, IP, browser (TTL index) |
| `pending_registrations`, `otp_codes` | registrations awaiting the e-mailed code, password-reset codes (hashed, TTL) |
| `complaints` | case data (sensitive fields encrypted), evidence metadata and SHA-256, location, status, history |
| `evidence.files/.chunks` | GridFS, encrypted evidence |
| `anpr_scans`, `anpr.files/.chunks` | scans, plates, video timelines; media and crops in GridFS |
| `traffic_runs` | every traffic run: video, operator, lanes (auto/edited/manual), totals |
| `audit_log` | append-only audit trail |
| `counters` | acknowledgement-number sequence per year |

Indexes are created automatically at start-up (`backend/app/migrations.py`), and the admin account is
created or updated from `.env`.

## 9. Hardware and simulation

The signal controller runs in the **Wokwi simulator for VS Code** (plain Raspberry Pi Pico, MicroPython
v1.29). Each lane has 3 LEDs:

| Lane | Red | Yellow | Green |
|---|---|---|---|
| 1 | GP13 | GP14 | GP15 |
| 2 | GP16 | GP17 | GP18 |
| 3 | GP19 | GP20 | GP21 |
| 4 | GP10 | GP11 | GP12 |

The same `main.py` runs unchanged on a **real Pico** wired the same way (220 Ω resistors) and connected by
USB. Set `WOKWI_SERIAL_URL=COM5` (Windows) or `/dev/ttyACM0` (Linux) to the Pico's port.
See [docs/HARDWARE.md](docs/HARDWARE.md).

## 10. Configuration

All settings are in `.env` (see the comments in the generated file). The most important:
* `ADMIN_EMAIL`, `ADMIN_PASSWORD`
* `SMTP_*` (Gmail needs an App Password)
* `GEMINI_API_KEY`, `GEMINI_MODEL`, `GEMINI_FALLBACK_MODEL`
* `WOKWI_SERIAL_URL`
* `SESSION_IDLE_MINUTES`
* the upload limits
* the HTTPS settings (`COOKIE_SECURE`, `TRUST_PROXY`, `PUBLIC_URL`, `ALLOWED_HOSTS`, `DOMAIN`)

## 11. Tested

The results and exactly what was and was not verified are in [docs/TESTING.md](docs/TESTING.md).

## 12. Limits and notes

* **Automatic lanes are a proposal.** They need moving traffic in the first 20 seconds, and a fixed camera.
  The operator always confirms them.
* **Detection speed** depends on the CPU (yolov8n on ONNX Runtime). The video stays in real time by
  skipping frames on slow computers.
* **One traffic run at a time.** There is one junction and one simulated controller.
* **Rate limits live in memory** and are reset by a restart. Several app replicas would need a shared store
  such as Redis.
* **E-mail must be configured** (SMTP) for anyone to register. `MAIL_MODE=console` prints the codes in the
  server window, for a demo on one computer only.

## 13. Third-party components

* YOLOv8 / Ultralytics model weights (AGPL-3.0)
* three.js r186 (MIT)
* Leaflet (BSD-2)
* OpenStreetMap data (ODbL; tiles and Overpass are used live from the browser)
* MicroPython (MIT)
* FastAPI, PyMongo, OpenCV, ONNX Runtime, cryptography (permissive licences)

## 14. Screenshots

| | |
|---|---|
| ![Traffic control running](docs/screenshots/16_traffic_running.jpg) | ![Lane review](docs/screenshots/15_traffic_review.jpg) |
| ![ANPR](docs/screenshots/06_anpr_image.jpg) | ![Cyber crime case](docs/screenshots/19_crime_case.jpg) |
