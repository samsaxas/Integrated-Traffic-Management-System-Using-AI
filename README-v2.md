# ITMSAI — Integrated Traffic Management System Using AI

AI traffic monitoring with adaptive signals, number-plate recognition and a citizen crime-reporting
hub, in one web application. Based on the AMC Engineering College major-project report (2025-26).

## About Version 2

Version 2 is a simulation and prototype of the proposed AI-based Integrated Traffic Management System. It demonstrates the expected system behaviour through a web dashboard, vehicle detection, lane-wise traffic analysis, adaptive traffic signals and a Wokwi-based Raspberry Pi Pico W simulation.

The project is designed to demonstrate how the proposed system would operate in a real-world traffic environment. **Version 3 will build upon this prototype to develop the next stage of the integrated system.**


| Module (report §5.2) | What it does here |
|---|---|
| Vehicle detection | YOLOv8n (ONNX Runtime by default, Ultralytics optional) on video files, RTSP/HTTP IP cameras, a phone/laptop browser camera, or the built-in junction simulation |
| Lane congestion analysis | Vehicles are tracked and counted inside up to 4 lane polygons (drawn in the browser); counts are smoothed and graded NONE / LOW / MEDIUM / HIGH |
| Number-plate recognition | OpenCV plate localisation + **Tesseract OCR**, Indian-format correction, images + records in MongoDB |
| Traffic control | Adaptive controller: busiest lane next, green time sized to its queue, gap-out, anti-starvation, manual override |
| Traffic-light automation | Raspberry Pi Pico W MicroPython firmware (USB serial, Wi-Fi, or Wokwi) + live in-browser Wokwi-style board |
| Control-centre dashboard | Live annotated video, lane cards, signal phase, serial monitor, history chart, timing editor |
| Reporting (Cyber Crime Hub) | Registration, login, incident reports with evidence uploads and map location, your reports, authority moderation, emergency numbers, location sharing, city news |

Stack: **FastAPI + MongoDB** (pymongo async), vanilla HTML/CSS/JS front-end (no build step), Docker Compose.

![Architecture](docs/architecture.svg)

---

## Quick start (Docker — Windows, macOS, Linux)

Requirements: [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker Engine + Compose v2) and Python 3 (only to generate secrets).

```bash
git clone <your-repo-url> itmsai
cd itmsai
python scripts/init_env.py          # creates .env with random secrets and prints the admin password
docker compose up --build -d        # first build downloads base images + Python packages (a few minutes)
```

Open **http://localhost:8000** (from a phone/tablet on the same Wi-Fi: `http://<computer-ip>:8000`).

* Log in as the traffic authority with the `admin` password printed by `init_env.py`
  (also in `.env` as `ADMIN_PASSWORD`) to control sources/signals and see ANPR records.
* Citizens register themselves in the Cyber Crime Hub.
* The dashboard starts with the **simulated junction** so the adaptive signals work immediately;
  switch to a sample video, upload your own, add an IP-camera URL or use the phone camera.

Useful commands:

```bash
docker compose logs -f app                         # application log
docker compose ps                                  # health status
docker compose --profile test run --rm tests       # run the test suite inside the container
docker compose down                                # stop (data is kept in volumes)
docker compose down -v                             # stop and DELETE all data
```

> On Windows use `py scripts\init_env.py` if `python` is not on PATH. Without Python you can
> `copy .env.example .env` and replace every `CHANGE_ME` value by hand.

## Option B — run without Docker

1. Install **Python 3.10+**, **Tesseract OCR** (Windows: [UB-Mannheim build](https://github.com/UB-Mannheim/tesseract/wiki) · macOS: `brew install tesseract` · Linux: `sudo apt install tesseract-ocr`) and a **MongoDB** server
   (simplest: `docker run -d --name itmsai-mongo -p 127.0.0.1:27017:27017 mongo:7.0`, or MongoDB Community Server).
2. Set up and run:

   ```bash
   # Linux / macOS
   ./scripts/setup_local.sh
   source .venv/bin/activate
   python run.py
   ```
   ```powershell
   # Windows PowerShell
   powershell -ExecutionPolicy Bypass -File scripts\setup_local.ps1
   .\.venv\Scripts\Activate.ps1
   python run.py
   ```
   `run.py` checks Python packages, Tesseract and MongoDB first and tells you exactly what is missing.
   For a webcam attached to the computer set `ALLOW_DEVICE_SOURCES=true` in `.env`.

## Database setup and migrations

Nothing to do by hand: on start-up the app waits for MongoDB, applies versioned migrations
(indexes, emergency-contact seed data, default signal settings — recorded in `schema_migrations`)
and creates/updates the authority account from `ADMIN_USERNAME` / `ADMIN_PASSWORD`. In Docker,
`docker/mongo-init/01-create-app-user.sh` creates a least-privilege `readWrite` user on first start.

```bash
cd backend
python -m app.migrations --status          # list applied migrations
python -m app.migrations                   # apply pending ones manually
python scripts/import_legacy_sqlite.py ../legacy/flask_sqlite_app/vehicle_records.db   # import the old SQLite ANPR records
# Docker:  docker compose exec app python -m app.migrations --status
```

## Hardware (Raspberry Pi Pico W) and Wokwi

The signal state is exported over a small text protocol (`S,G,R,R,R`). Three ways to drive real LEDs:

| Setup | How |
|---|---|
| Pico W on USB (any OS) | flash MicroPython, copy `hardware/pico/*.py`, run `python hardware/bridge/pico_bridge.py --port auto --token <DEVICE_TOKEN>` |
| Pico W on Wi-Fi | set `TRANSPORT="wifi"`, SSID, `BACKEND_URL`, `DEVICE_TOKEN` in `hardware/pico/config.py` |
| Wokwi simulator | `hardware/wokwi/diagram.json` (4 lanes × R/Y/G) in Wokwi for VS Code + bridge on `rfc2217://localhost:4000` |
| No hardware | the dashboard's built-in board renders the same circuit live |

Wiring, flashing, and failure behaviour: **[docs/HARDWARE.md](docs/HARDWARE.md)**.

## Documentation

* [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — components, data flow, database collections, algorithms
* [docs/API.md](docs/API.md) — REST endpoints (interactive docs at `/api/docs`)
* [docs/HARDWARE.md](docs/HARDWARE.md) — Pico W, Wokwi, bridge, simulation alternatives
* [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) — production/HTTPS, Raspberry Pi, backups, configuration reference
* [docs/TESTING.md](docs/TESTING.md) — how to run tests and the recorded results
* [legacy/README.md](legacy/README.md) — the original code and the bugs that were fixed

## Directory structure

```
itmsai/
├── backend/
│   ├── app/
│   │   ├── main.py               FastAPI app, security headers, static front-end
│   │   ├── config.py             all settings from environment / .env
│   │   ├── db.py                 MongoDB clients (async for API, sync for video threads)
│   │   ├── migrations.py         versioned migrations (indexes, seed data)  python -m app.migrations
│   │   ├── security.py           scrypt hashing, JWT cookie sessions, roles, rate limiting
│   │   ├── uploads.py            streamed, size-limited, magic-byte-checked uploads
│   │   ├── signal_logic.py       adaptive signal controller (shared with the Pico firmware)
│   │   ├── routers/              auth, crime (reports/contacts/news), traffic, anpr, system
│   │   ├── services/             traffic engine (threads, sources, logging), news
│   │   └── vision/               detector (ONNX/Ultralytics), tracker, lanes, ANPR, simulation
│   ├── scripts/import_legacy_sqlite.py
│   ├── tests/                    unit, integration (HTTP + MongoDB), browser end-to-end
│   ├── Dockerfile
│   └── requirements*.txt
├── frontend/                     index, traffic, anpr, crime/* pages + css/js (+ vendored Leaflet)
├── hardware/
│   ├── pico/                     MicroPython firmware: main.py, config.py, signal_logic.py
│   ├── wokwi/                    diagram.json (Pico W + 12 LEDs + resistors), wokwi.toml
│   └── bridge/pico_bridge.py     backend <-> USB serial / Wokwi RFC2217 bridge
├── models/                       yolov8n.onnx, yolov8n.pt, license_plate_detector.pt
├── samples/videos/               CC-BY sample clips (see ATTRIBUTION.md)
├── docker/mongo-init/            creates the least-privilege MongoDB app user
├── scripts/                      init_env.py, setup_local.sh/.ps1, run_tests.py
├── docs/
├── legacy/                       original Flask/SQLite + PlatformIO files (reference)
├── docker-compose.yml
├── .env.example
└── run.py                        local launcher with pre-flight checks
```

## Security notes

* Passwords: scrypt (stdlib). Sessions: HS256 JWT in an `HttpOnly`, `SameSite=Strict` cookie (Bearer tokens also accepted).
* Roles: `citizen` (own reports only) and `authority` (control centre, ANPR, all reports). Login rate-limited.
* Uploads: streamed with size caps, type detected from magic bytes, random file names, served only to the owner/authority.
* MongoDB is not exposed outside the Docker network; the app uses a `readWrite`-only user.
* Strict Content-Security-Policy (no inline scripts), `nosniff`, frame and referrer policies.
* The original archive's `.env` contained a Google API key — it is **not** included; revoke it.
