# SECTION 3 — Local Development Environment Setup

## Purpose
Enable all developers to run the complete three-tier stack locally: Pi Hub Server, Cloud Services API, and React Web Client. This allows testing features end-to-end before pushing to production.

## Architecture Overview

Your system has three components that work together:

```
┌─────────────────────────┐
│   RPi Hub Server        │       Runs on Pi (or simulator locally)
│  - Serial management    │       WebSocket client to cloud
│  - Health monitoring    │       
│  - Task handling        │
└──────────────┬──────────┘
               │ WebSocket
               ↓
┌─────────────────────────┐
│ Cloud Services API      │       FastAPI backend
│  - Authentication       │       WebSocket hub (bi-directional)
│  - Telemetry storage    │       REST API for clients
│  - Message routing      │
└──────────────┬──────────┘
               │ WebSocket + REST
               ↓
┌─────────────────────────┐
│  React Web Client       │       Browser UI
│  - Live telemetry view  │       Real-time updates
│  - Device management    │       Zustand store
└─────────────────────────┘
```

---

## Prerequisites

Ensure you have the following installed:

### Required
- **Python 3.11+** (cloud services tests run on 3.11)
  ```bash
  python --version  # Should be Python 3.11 or higher
  ```
- **Node.js 20+** (web client build uses 20)
  ```bash
  node --version  # Should be v20.x or higher
  npm --version   # Usually comes with Node
  ```
- **Git** (for cloning and version control)
  ```bash
  git --version
  ```

### Recommended
- **VS Code** with these extensions installed:
  - Python (Microsoft)
  - Pylance
  - ESLint
  - Prettier
  - REST Client (for testing API endpoints)

- **GitHub Copilot** enabled in VS Code (if available)

### System Requirements
- **2GB+ RAM** (for running all three services simultaneously)
- **500MB+ disk space** for dependencies

---

## Step 1 — Clone the Repository (if not already)

```bash
# Navigate to your workspace
cd "C:\Users\Westo\Saved Games\electrical-gui"

# Or clone fresh:
git clone <your-repo-url> electrical-gui
cd electrical-gui
```

---

## Step 2A — Set Up Cloud Services (FastAPI Backend)

### Install Python Dependencies

```bash
cd cloud-services

# Create a virtual environment (optional but recommended)
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configure Environment

```bash
# Copy the example .env file
cp .env.example .env

# Edit .env and set values (for local development, defaults work fine):
#   HOST=0.0.0.0
#   PORT=8080
#   JWT_SECRET_KEY=dev-secret-key
#   ENVIRONMENT=development
```

### Start the Cloud Services Server

```bash
# From cloud-services directory
python -m uvicorn src.main:app --reload --host 0.0.0.0 --port 8080
```

**Expected output**:
```
INFO:     Uvicorn running on http://0.0.0.0:8080
INFO:     Application startup complete

# API docs available at:
#   http://localhost:8080/docs          (Swagger UI)
#   http://localhost:8080/redoc         (ReDoc)
```

**Verify it's running**:
```bash
# In another terminal
curl http://localhost:8080/
# Should return: {"service":"RPi Hub Cloud Service","version":"1.0.0","docs":"/docs"}
```

---

## Step 2B — Set Up Web Client (React + Vite)

### Install Node Dependencies

```bash
cd web-client

# Install npm packages
npm install
```

### Start the Development Server

```bash
# From web-client directory
npm run dev
```

**Expected output**:
```
VITE v4.x.x  ready in xxx ms

➜  Local:   http://localhost:5173/
➜  press h to show help
```

Open http://localhost:5173 in your browser. You should see the application UI.

---

## Step 2C — Set Up RPi Hub Server (Pi Simulator or Actual Pi)

### Option A: Run Locally (Simulator/Testing)

```bash
cd rpi-hub-server

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configure for Local Testing

Edit `rpi-hub-server/src/config.py` or set environment variables:

```bash
# Create .env in rpi-hub-server directory (optional)
cat > .env << EOF
ENVIRONMENT=development
SERVER_ENDPOINT=ws://localhost:8080/hubs/ws
DEVICE_TOKEN=dev-token-rpi-bridge-01
HUB_ID=local-hub-01
EOF
```

### Start the Pi Hub Server

```bash
# From rpi-hub-server directory
python -m src.main
```

**Expected output**:
```
[INFO] Hub Agent initialized
[INFO] Hub Agent started
[INFO] Attempting to connect to server...
[INFO] Connected to server
```

The hub will attempt to connect to your cloud services backend via WebSocket. If cloud services is running, you should see telemetry messages.

### Option B: Run the Mock Simulator (if available)

```bash
cd rpi-hub-server

# Run the test simulator
python -m pytest tests/mock_hub_simulator.py -v -s
```

---

## Step 3 — Verify Full Stack Communication

Once all three services are running, verify they communicate:

### Check Cloud Services Health

```bash
curl http://localhost:8080/health
# Should return health status
```

### Check Web Client Connection

1. Open http://localhost:5173 in your browser
2. Go to **DevTools → Network → WS tab**
3. You should see an active WebSocket connection to `ws://localhost:8080/...`
4. Look for `ping` messages and `subscriptions` being exchanged

### Check Pi Hub Connection

1. In the Pi Hub Server terminal, you should see:
   ```
   [INFO] Connected to server
   [INFO] Telemetry queued for port <port-id>
   ```

2. In the React UI, navigate to **Device Manager**
3. You should see "Hub Connections" showing your hub is online

---

## Step 4 — Run Tests Locally

Before committing, always run tests locally to catch issues early:

### Cloud Services Tests

```bash
cd cloud-services

# Run all tests
pytest tests/ -v

# Run specific test file
pytest tests/test_auth.py -v

# Run with coverage
pytest tests/ --cov=src/
```

### Web Client Tests

```bash
cd web-client

# Type checking
npm run type-check

# Linting
npm run lint

# Build (catches many errors)
npm run build
```

### RPi Hub Server Tests

```bash
cd rpi-hub-server

# Run all tests
pytest tests/ -v

# Run specific test
pytest tests/test_serial_manager.py -v
```

---

## Step 5A — Connecting a Real Pi (Optional)

If you have an actual Raspberry Pi with the hub service installed:

1. **Update the server endpoint** in the Pi's config to point to your machine:
   ```bash
   # On the Pi
   ENVIRONMENT=production
   SERVER_ENDPOINT=ws://<your-machine-ip>:8080/hubs/ws
   DEVICE_TOKEN=dev-token-rpi-bridge-01
   ```

2. **Ensure firewall allows connections**:
   - Windows: Allow Python through Windows Defender Firewall
   - Or open port 8080: `python -m http.server 8080`

3. **Test connectivity from Pi**:
   ```bash
   # On the Pi
   curl http://<your-machine-ip>:8080/
   ```

---

## Complete Multi-Terminal Setup (Quick Start)

Here's a complete flow to run everything:

### Terminal 1: Cloud Services

```bash
cd cloud-services
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
python -m uvicorn src.main:app --reload --host 0.0.0.0 --port 8080
```

### Terminal 2: Web Client

```bash
cd web-client
npm install
npm run dev
```

### Terminal 3: RPi Hub Server

```bash
cd rpi-hub-server
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
python -m src.main
```

### Terminal 4: Browser

```bash
# Open in your browser:
http://localhost:5173
```

After ~5 seconds, you should see the hub connected in the dashboard.

---

## Environment Variables Reference

### Cloud Services (cloud-services/.env)
```
HOST=0.0.0.0                      # Server host
PORT=8080                         # Server port
ENVIRONMENT=development           # Environment mode
JWT_SECRET_KEY=your-secret-key    # Authentication key
CORS_ORIGINS=http://localhost:5173  # Frontend URL
```

### RPi Hub Server (rpi-hub-server/.env)
```
SERVER_ENDPOINT=ws://localhost:8080/hubs/ws  # Cloud service WebSocket
DEVICE_TOKEN=dev-token-rpi-bridge-01         # Authentication token
HUB_ID=local-hub-01                          # Hub identifier
ENVIRONMENT=development                      # Development mode
```

### Web Client
The web client reads the API URL from [web-client/src/services/api.ts](web-client/src/services/api.ts):
- Default backend: `http://localhost:8080`
- Default WebSocket: `ws://localhost:8080/...`

---

## Troubleshooting

### Cloud Services Won't Start

**Issue**: `Address already in use`
```bash
# Another process is using port 8080
lsof -i :8080  # Linux/Mac
netstat -ano | findstr :8080  # Windows

# Kill the process or use a different port:
python -m uvicorn src.main:app --reload --port 8081
```

**Issue**: `ModuleNotFoundError: No module named 'fastapi'`
```bash
# Didn't install dependencies
pip install -r requirements.txt

# Or wrong virtual environment activated
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate  # Windows
```

### Web Client Won't Start

**Issue**: `npm: command not found`
```bash
# Node.js not installed
node --version
# If not installed, download from https://nodejs.org/
```

**Issue**: Cannot reach backend API
```bash
# Check if cloud services is running on port 8080
curl http://localhost:8080/

# If using different port, update src/services/api.ts:
# const API_URL = 'http://localhost:8081';  // Changed from 8080
```

### Pi Hub Server Can't Connect to Cloud

**Issue**: `ConnectionRefusedError: [Errno 111] Connection refused`
```bash
# Cloud services not running or wrong endpoint
# 1. Verify cloud services is running: curl http://localhost:8080/
# 2. Check SERVER_ENDPOINT in .env or config.py
# 3. Check firewall: Windows might block Python
```

**Issue**: `WebSocket connection failed`
```bash
# This is normal during startup — the hub retries automatically
# Wait up to 30 seconds for connection to establish
# Check logs for "Connected to server" message
```

### Tests Fail Locally But Pass on GitHub

**Issue**: Python version mismatch
```bash
python --version  # Compare with .github/workflows/test-cloud-services.yml
# Should be Python 3.11
```

**Issue**: Different dependency versions
```bash
# Ensure you're in the correct virtual environment
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate  # Windows

# Reinstall fresh
pip install --upgrade -r requirements.txt
```

### Port Conflicts

If you already have services running on 5173 or 8080:

**Option 1**: Kill existing processes
```bash
# Windows
taskkill /pid <PID> /f

# Linux/Mac
kill -9 <PID>
```

**Option 2**: Use different ports
```bash
# Cloud Services on 8081
python -m uvicorn src.main:app --port 8081

# Web Client on 5174
npm run dev -- --port 5174
```

### Still Having Issues?

1. **Check for error messages** in the terminal output
2. **Verify all prerequisites** are installed with correct versions
3. **Try a fresh clone** if files seem corrupted:
   ```bash
   cd ..
   rm -rf electrical-gui
   git clone <repo> electrical-gui
   ```
4. **Ask for help** with error logs and terminal output

---

## Expected Result

When everything is running correctly:

✅ **Cloud Services** 
- Terminal shows: `Uvicorn running on http://0.0.0.0:8080`
- `curl http://localhost:8080/` returns JSON

✅ **Web Client**
- Terminal shows: `Local: http://localhost:5173/`
- Browser loads dashboard UI without errors

✅ **RPi Hub Server**
- Terminal shows: `[INFO] Connected to server`
- Hub appears in DevTools WebSocket tab on frontend

✅ **Full Stack**
- DevTools Network tab shows active WebSocket connection
- When hub is running, telemetry appears in the React dashboard
- Health metrics update every 30 seconds

---

## Next Steps

Once you have the full stack running locally:

1. **Explore the codebase**: Open each service in VS Code
2. **Read the README** files in each directory for component-specific details
3. **Run the tests**: Ensure your environment passes the test suite
4. **Create a feature branch**: Start implementing your first feature (see SECTION 4)
5. **Test your changes**: Run tests locally before pushing

Great work setting up your development environment! You're now ready to contribute. 🚀
