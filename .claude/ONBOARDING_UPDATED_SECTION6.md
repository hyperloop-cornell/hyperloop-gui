# SECTION 6 — Mini Project: End-to-End Health Metric Addition

## Purpose
Provide a practical, production-relevant task that teaches the full development workflow: modifying the Pi hub to collect a new metric, updating the cloud API to handle it, and displaying it in the React dashboard.

## Objective
Add a new health metric (e.g., `network_latency_ms`) that flows from the Pi hub → cloud API → React dashboard.

---

## Step 1 — Collect the Metric on Pi Hub

In [rpi-hub-server/src/health_reporter.py](rpi-hub-server/src/health_reporter.py), modify the health data collection to include your new metric:

```python
# In the collect_metrics() or report_health() method
def get_health_data(self) -> Dict[str, Any]:
    """Collect comprehensive health metrics."""
    health_data = {
        "uptime_seconds": int(time.time() - self._start_time),
        "system": self._collect_system_metrics(),
        "service": self._collect_service_metrics(),
        "errors": self._collect_error_metrics(),
        "network_latency_ms": self._measure_network_latency(),  # NEW FIELD
    }
    return health_data

def _measure_network_latency(self) -> float:
    """Measure network latency to cloud server."""
    import time
    start = time.time()
    # Ping the server endpoint or similar
    latency = (time.time() - start) * 1000  # Convert to ms
    return latency
```

---

## Step 2 — Update Cloud API Models

In [cloud-services/src/models.py](cloud-services/src/models.py), update the `HealthMessage` Pydantic model:

```python
class HealthMessage(BaseModel):
    """Health metrics from hub."""
    type: str = "health"
    hubId: str
    timestamp: str
    uptime_seconds: int
    network_latency_ms: float  # NEW FIELD
    system: Dict[str, Any]
    service: Dict[str, Any]
    errors: Dict[str, Any]
```

The CloudAPI automatically broadcasts this message to all subscribed clients via [cloud-services/src/websocket/hub_endpoint.py](cloud-services/src/websocket/hub_endpoint.py):

```python
# This already exists in handle_health()
client_message = {
    "type": "health_status",
    "hubId": hub_id,
    "timestamp": health.timestamp,
    "uptime_seconds": health.uptime_seconds,
    "network_latency_ms": health.network_latency_ms,  # NEW FIELD
    "system": health.system,
    "service": health.service,
    "errors": health.errors,
}
```

---

## Step 3 — Handle WebSocket Message in React

In [web-client/src/stores/healthStore.ts](web-client/src/stores/healthStore.ts) (or create if it doesn't exist), add handling for the new field:

```typescript
interface HealthMetrics {
  uptime_seconds: number;
  network_latency_ms: number;  // NEW FIELD
  system: Record<string, any>;
  service: Record<string, any>;
  errors: Record<string, any>;
}

export const useHealthStore = create<HealthStoreState>((set) => ({
  metrics: null,
  
  updateHealth: (message: WebSocketMessage) => {
    if (message.type === 'health_status') {
      set({
        metrics: {
          uptime_seconds: message.uptime_seconds,
          network_latency_ms: message.network_latency_ms,  // NEW FIELD
          system: message.system,
          service: message.service,
          errors: message.errors,
        },
        lastUpdate: Date.now(),
      });
    }
  },
}));
```

---

## Step 4 — Display in React UI

In [web-client/src/pages/Dashboard.tsx](web-client/src/pages/Dashboard.tsx) (or your health display component), render the new field:

```typescript
import { useHealthStore } from '@/stores/healthStore';

export function HealthDashboard() {
  const { metrics } = useHealthStore();

  if (!metrics) return <div>Loading health metrics...</div>;

  return (
    <div className="health-panel">
      <h2>System Health</h2>
      <p>Uptime: {metrics.uptime_seconds} seconds</p>
      <p className={metrics.network_latency_ms > 100 ? 'text-red-500' : 'text-green-500'}>
        Network Latency: {metrics.network_latency_ms.toFixed(2)}ms
      </p>
      {/* Other metrics... */}
    </div>
  );
}
```

---

## Step 5 — Test Locally

1. **Start the Pi hub simulator** (or actual hub):
   ```bash
   cd rpi-hub-server
   python -m pytest tests/mock_hub_simulator.py
   # OR run the actual hub
   python src/main.py
   ```

2. **Start the cloud API backend**:
   ```bash
   cd cloud-services
   python -m uvicorn src.main:app --reload
   ```

3. **Start the React frontend**:
   ```bash
   cd web-client
   npm run dev
   ```

4. **Verify the flow**:
   - Open browser DevTools → Network → WS tab
   - Look for `health_status` messages with the new `network_latency_ms` field
   - Confirm it displays in the UI

---

## Step 6 — Commit and Push

```bash
# Stage changes
git add rpi-hub-server/src/health_reporter.py cloud-services/src/models.py web-client/src/stores/ web-client/src/pages/

# Commit with a clear message
git commit -m "feat: add network latency health metric

- Measure network latency on Pi hub
- Include network_latency_ms in HealthMessage model
- Display in React health dashboard"

# Push to your branch
git push origin feature/network-latency-metric
```

---

## Step 7 — Open a Pull Request

1. Go to your repository on GitHub
2. Click **New Pull Request**
3. Select your branch → main
4. Add a clear description:
   - What the change does
   - Why it's useful
   - How to test it
5. Assign a reviewer and wait for feedback

---

## Key Differences from Generic Example

- **Your architecture uses base64-encoded serial data** (raw bytes) rather than discrete telemetry fields
- **Health metrics** (separate from telemetry) are a better fit for adding new scalar fields
- **WebSocket messages** flow through hub_endpoint.py with built-in broadcasting
- **Frontend parsing** happens in the telemetryStore or a dedicated health store
