# Queue Server API Communication Flow

This document explains how the BOLT application sends requests to the Queue Server API, which then executes Bluesky plans and returns results.

## Complete Request Flow

### 1. **Application Layer** (`bolt_api.py`)

When you call `bolt_api.get_current_angle("DMC01:A")`, here's what happens:

```python
def get_current_angle(self, motor: str) -> CurrentAngleReading:
    # Constructs a curl command to POST to Queue Server API
    queue_url = f"http://{self.FASTAPI_URL}:8003/api/queue/item/execute"
    
    cmd = [
        "curl", "-X", "POST", queue_url,
        "-H", "Authorization: Apikey test",
        "-d", json.dumps({
            "item": {
                "name": "get_angle",           # ← Plan name
                "kwargs": {"motor": "rotation_motor"},  # ← Plan parameters
                "item_type": "plan",
                "user_group": "primary",
            }
        }),
    ]
    
    # Execute the curl command
    result = subprocess.run(cmd, capture_output=True, text=True)
    product = json.loads(result.stdout)
    item_uid = product["item"]["item_uid"]  # ← Get unique ID for tracking
```

### 2. **Queue Server HTTP API** (Port 8003)

The HTTP API receives the POST request at `/api/queue/item/execute`:

```http
POST http://localhost:8003/api/queue/item/execute
Authorization: Apikey test
Content-Type: application/json

{
  "item": {
    "name": "get_angle",
    "kwargs": {"motor": "rotation_motor"},
    "item_type": "plan"
  }
}
```

**What the HTTP API does:**
- Validates the API key (`test`)
- Converts the HTTP request into a ZMQ message
- Sends ZMQ message to the Queue Server
- Returns immediately with an `item_uid` (doesn't wait for completion!)

### 3. **Queue Server Core** (ZMQ communication)

The Queue Server:
- Receives the ZMQ message
- Adds the plan to its execution queue
- When it's the plan's turn, executes: `get_angle(motor="rotation_motor")`

### 4. **Bluesky Plan Execution** (`startup_bolt/15_plans.py`)

```python
def get_angle(motor, *, md=None):
    from bluesky import plan_stubs as bps
    
    # Read the motor position
    reading = yield from bps.read(motor)  # ← Reads from EPICS motor
    
    # Store metadata
    md = {
        "motor_angle": reading[motor.name]["value"],
        "angle_degrees": reading[motor.name]["value"] * 2.8125,
        "timestamp": datetime.now().isoformat()
    }
    
    # Create a Bluesky run with this metadata
    yield from bps.open_run(md=md)
    yield from bps.create()
    yield from bps.save()
    yield from bps.close_run()
    
    return reading
```

**What happens during plan execution:**
1. `bps.read(motor)` → Communicates with EPICS motor over Channel Access
2. Motor returns current position
3. Data is packaged into a Bluesky "run"
4. Run metadata/data is sent to **TiledWriter** subscriber
5. TiledWriter uploads data to Tiled server

### 5. **Polling for Completion** (Back in `bolt_api.py`)

Since the API returns immediately, your code **polls** to wait for completion:

```python
# Wait until the plan finishes running
current_pos = 0
while(current_pos != "run_list"):
    # Check active runs
    queue_url = f"http://{self.FASTAPI_URL}:8003/api/re/runs/active"
    result = subprocess.run(["curl", "-X", "GET", queue_url, ...])
    history_data = json.loads(result.stdout)
    time.sleep(1)  # Poll every second
```

### 6. **Retrieve Results from History**

Once complete, fetch the run ID from Queue Server history:

```python
queue_url = f"http://{self.FASTAPI_URL}:8003/api/history/get"
result = subprocess.run(["curl", "-X", "GET", queue_url, ...])
history_data = json.loads(result.stdout)

# Find the run_uid associated with our item_uid
for item in history_data["items"]:
    if item["item_uid"] == item_uid:
        run_id = item["result"]["run_uids"][0]
        break
```

### 7. **Fetch Data from Tiled**

Finally, connect to Tiled to retrieve the actual motor reading:

```python
from tiled.client import from_uri
tiled_client = from_uri("http://localhost:8000", api_key="...")

# Access the run data by run_id
run_data = tiled_client[run_id]
angle = run_data.metadata['start']['angle_degrees']

return CurrentAngleReading(
    motor=motor,
    angle=float(angle),
    condition="Remote Angle Reading",
    timestamp=datetime.now()
)
```

## Complete Architecture Diagram

```
┌─────────────────┐
│  bolt_api.py    │  Application Layer
│  get_angle()    │
└────────┬────────┘
         │ HTTP POST /api/queue/item/execute
         │ {"name": "get_angle", "kwargs": {...}}
         ↓
┌─────────────────────────┐
│  Queue Server HTTP API  │  Port 8003
│  (bluesky-httpserver)   │
└────────┬────────────────┘
         │ ZMQ message
         ↓
┌─────────────────────────┐
│   Queue Server Core     │  ZMQ communication
│   (bluesky-queueserver) │
└────────┬────────────────┘
         │ Execute plan
         ↓
┌─────────────────────────┐
│  Bluesky Run Engine     │  startup_bolt/15_plans.py
│  get_angle(motor=...)   │
└────────┬────────────────┘
         │ bps.read(motor)
         ↓
┌─────────────────────────┐
│   EPICS Motor           │  DMC01:A via Channel Access
│   rotation_motor        │
└────────┬────────────────┘
         │ Returns position
         ↓
┌─────────────────────────┐
│  TiledWriter Subscriber │  00_base.py: RE.subscribe(tw)
└────────┬────────────────┘
         │ Upload run data
         ↓
┌─────────────────────────┐
│   Tiled Server          │  Port 8000
│   Stores run metadata   │
└─────────────────────────┘
         ↑
         │ Fetch results
┌────────┴────────┐
│  bolt_api.py    │  tiled_client[run_id]
│  Return results │
└─────────────────┘
```

## Key Communication Endpoints

### Queue Server API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/queue/item/execute` | POST | Submit a plan to execute |
| `/api/re/runs/active` | GET | Check currently running plans |
| `/api/history/get` | GET | Retrieve execution history |

### Request/Response Format

**Execute Plan Request:**
```json
{
  "item": {
    "name": "get_angle",
    "args": [],
    "kwargs": {"motor": "rotation_motor"},
    "item_type": "plan",
    "user": "UNAUTHENTICATED_SINGLE_USER",
    "user_group": "primary"
  }
}
```

**Execute Plan Response:**
```json
{
  "success": true,
  "msg": "",
  "item": {
    "name": "get_angle",
    "item_uid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    ...
  }
}
```

## Key Insights

1. **Asynchronous Pattern**: The API returns immediately with an `item_uid`, not waiting for plan completion

2. **Polling Required**: Your code must poll `/api/re/runs/active` to detect when the plan finishes

3. **Two-Step Data Retrieval**: 
   - First get `run_uid` from Queue Server history
   - Then fetch actual data from Tiled using that `run_uid`

4. **ZMQ Bridge**: The HTTP API exists solely to translate HTTP ↔ ZMQ since browsers can't speak ZMQ directly

5. **Data Storage**: All experimental data flows through Bluesky → TiledWriter → Tiled server, not directly back through the API

## Why This Architecture?

This architecture enables:

- **Multi-user Access**: Multiple users/applications can submit plans to the same Queue Server
- **Asynchronous Operation**: Plans can take seconds to hours - the API doesn't block
- **Persistent Data**: All experimental data is stored in Tiled for later analysis
- **Single Run Engine**: Only one plan executes at a time, preventing conflicts
- **Web Browser Support**: HTTP API allows web-based interfaces (browsers can't use ZMQ directly)

## Authentication

The system uses a simple API key authentication:
- API Key: `test` (configured in environment variable)
- Header: `Authorization: Apikey test`
- This is a development setup - production would use more secure authentication

## Port Configuration

| Service | Port | Purpose |
|---------|------|---------|
| Tiled Server | 8000 | Data storage and retrieval |
| Queue Server HTTP API | 8003 | HTTP interface to Queue Server |
| Queue Server (ZMQ) | 60615 | Internal ZMQ communication |

## Common Workflow Example

```python
# 1. Submit plan
bolt_api.move_motor("DMC01:A", "45.0")
  ↓
# 2. Queue Server executes move_motor plan
# 3. Plan moves EPICS motor to position
# 4. Plan stores result in Tiled
# 5. Application polls for completion
# 6. Application retrieves result from Tiled
  ↓
# Returns: CurrentMoveMotorReading(motor="DMC01:A", angle=45.0, ...)
```

## Troubleshooting

**Plan not executing?**
- Check Queue Server is running
- Check HTTP API is running on port 8003
- Verify API key is correct

**Can't retrieve data?**
- Ensure Tiled server is accessible
- Check API key for Tiled
- Verify run_id exists in Tiled

**Polling timeout?**
- Plan may still be running
- Check Queue Server logs
- Verify EPICS devices are accessible
