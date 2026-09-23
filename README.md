<!-- Top-level project README for electrical-gui -->
# electrical-gui

[![Status](https://img.shields.io/badge/status-active-brightgreen)](https://gui.cornellhyperloop.com/)
![Python](https://img.shields.io/badge/python-%3E%3D3.11-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-%5E0.115.0-informational)
![Frontend](https://img.shields.io/badge/frontend-React%20%2B%20Vite-61DAFB)

A modular suite for hub management, telemetry, and a modern web GUI for Hyperloop team operations. This repository contains the cloud services, Raspberry Pi hub server, web client, and related utilities used to run, test, and develop the system.

**Quick links**
- Cloud API: [cloud-services](cloud-services)
- Hub agent & server: [rpi-hub-server](rpi-hub-server)
- Browser UI: [web-client](web-client)
- Uplink manager (cellular hub): [wifi-fallback-relay](wifi-fallback-relay)

Anyone is welcome to view the live GUI at: https://gui.cornellhyperloop.com/

**Repository layout**
- `cloud-services/` — FastAPI backend for authentication, websockets, hub coordination, and telemetry storage.
- `rpi-hub-server/` — Edge service that runs on every Raspberry Pi hub (same code, per-Pi config profile): USB serial boards (Arduino Uno R3/R4, Mega, Nano, STM32F407G-DISC1), flashing, and the cloud uplink. Includes a simulated-board mode for development without hardware.
- `web-client/` — React + TypeScript single-page app (Vite) for live telemetry, device control, and charting.
- `wifi-fallback-relay/` — Uplink manager for the one hub with a cellular HAT: prefers Wi-Fi, falls back to cellular, and reports the active link.

## Architecture

The system is split into three primary tiers:

- Cloud / API: central FastAPI application exposing REST endpoints and WebSocket hubs for realtime telemetry and command routing.
- Edge / Hub: Raspberry Pi agent that manages local serial devices, performs device tasks, and streams telemetry back to the cloud.
- Client: A TypeScript React app (Vite) that connects to the cloud WebSocket endpoints for realtime dashboards and control.

![Cloud service diagram](res/readme/cloud_service_diagram.png)

![RPI hub diagram](res/readme/rpi_hub_diagram.png)

![GUI diagram](res/readme/gui_diagram.png)

A short video of the GUI being used to operate our team's minipod is attached [here](res/readme/loophyper.mp4).

## Technical specifics

- Backend: `FastAPI` (>=0.115.0) + `uvicorn` for ASGI serving. WebSocket hubs use the `websockets` package and integrate with Pydantic models for typed messages.
- Auth: NetID allowlist plus one shared team password (bcrypt), JWT sessions, and a view-only mode; hubs authenticate with per-hub device tokens (see `cloud-services/src/auth/`).
- Data models: `pydantic` / `pydantic-settings` for config and runtime validation.
- Edge: `pyserial` for serial connections, arduino-cli and OpenOCD for flashing, `psutil` for health metrics; components are wired in `rpi-hub-server/src/runtime.py`.
- Contract: `cloud-services/contracts/openapi.json` (REST + WebSocket messages) is generated from the cloud models; the web client's types are generated from it (`npm run gen:types`).
- Frontend: React 19 + TypeScript, built with Vite; key libs include `recharts` for charts and `zustand` for state.

## Local development

Prerequisites: Python 3.11+, Node 18+ (or matching versions used by your environment), Git.

Backend (example)

1. Create and activate a Python virtual environment in `cloud-services`:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

2. Run the cloud API locally (development):

```powershell
cd cloud-services
set ENV_FILE=.env.development
python -m uvicorn src.main:app --reload --host 0.0.0.0 --port 8000
```

Frontend

```bash
cd web-client
npm install
npm run dev
```

Edge / RPi

1. Install dependencies in a venv under `rpi-hub-server`.
2. Without hardware: `HUB_PROFILE=dev-sim python -m src.main` runs simulated boards against a local cloud-services.
3. On a Pi: follow [.claude/rpi-hub-setup.md](.claude/rpi-hub-setup.md) (profiles `lab-hub` / `cellular-hub`, systemd unit in `rpi-hub-server/deploy/systemd/`).

The web client also has a browser-only mock: `npm run dev:mock` in `web-client`.

## Setup guides

- [.claude/auth-setup.md](.claude/auth-setup.md): production login (NetID allowlist + team password), hub device tokens, rotation.
- [.claude/rpi-hub-setup.md](.claude/rpi-hub-setup.md): setting up a Raspberry Pi hub, board checks, and the cellular hub's Wi-Fi/cellular failover.

## Tests & CI

- Python: `pytest` and `pytest-asyncio` in `cloud-services/tests`, `rpi-hub-server/tests` and `wifi-fallback-relay/tests`; CI runs all three.
- Frontend: TypeScript type checks, ESLint, a production build, and a check that `src/types/api.gen.ts` matches the cloud contract.

## Assets and visuals

Project visuals and diagrams used above live in `res/readme/`:

- ![res/readme/gui_telemetry_screenshot.png](res/readme/gui_telemetry_screenshot.png)
- ![res/readme/rpi_hub_server_real_photo.jpg](res/readme/rpi_hub_server_real_photo.jpg)
- [res/readme/cloud_service_diagram.png](res/readme/cloud_service_diagram.png)
- [res/readme/rpi_hub_diagram.png](res/readme/rpi_hub_diagram.png)
- [res/readme/gui_diagram.png](res/readme/gui_diagram.png)

## License

This project is licensed under the MIT License — see [LICENSE.md](LICENSE.md) for details.

