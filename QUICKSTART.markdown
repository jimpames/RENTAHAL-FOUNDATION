# QUICKSTART: Setting Up RENT-A-HAL with Realm-Based Architecture

This guide walks you through setting up the RENT-A-HAL system with the realm-based architecture for planetary-scale operation, as described in `MTOR-planetscale-code.pdf`. It assumes a local environment (Linux/MacOS/Windows with Python 3.8+) and focuses on running the system with minimal configuration.

## Prerequisites
- **Python 3.8+**: Install from [python.org](https://www.python.org/downloads/).
- **Git**: To clone the repository.
- **Docker**: Optional, for running model workers (e.g., Stable Diffusion).
- **pip**: Python package manager.

## Step 1: Clone the Repository
Clone the MTOR repository (replace `<repo-url>` with the actual URL):
```bash
git clone <repo-url>
cd mtor
```

## Step 2: Install Dependencies
Create a virtual environment and install required packages:
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```

**`requirements.txt`**:
```
fastapi==0.115.2
uvicorn==0.32.0
websockets==13.1
aiohttp==3.10.10
pydantic==2.9.2
pyyaml==6.0.2
sqlalchemy==2.0.36
sqlite3
```

## Step 3: Configure `realms.yml`
Create a `realms.yml` file in the project root to define realms and federation settings:
```yaml
federation_id: federation_12345678
federation_callback_url: http://localhost:5000
realms:
  $CHAT:
    primary_query_type: chat
    min_workers: 1
    max_workers: 5
  $VISION:
    primary_query_type: vision
    min_workers: 1
    max_workers: 3
  $IMAGINE:
    primary_query_type: imagine
    min_workers: 1
    max_workers: 3
discovery:
  bootstrap_nodes: []
  advertise_interval: 3600
database_sharding:
  enabled: false
  shards: []
  shard_map: {}
```

**Note**: This configuration disables database sharding for simplicity. To enable sharding, see `PERFORMANCE.md`.

## Step 4: Set Up Model Workers
RENT-A-HAL requires workers for query types (`chat`, `vision`, `imagine`). For testing, use mock workers or local model servers.

### Example: HuggingFace Worker for `$CHAT`
Run a local HuggingFace model server (e.g., GPT-2):
```bash
pip install transformers
python -m transformers.serve --model gpt2 --port 8000
```
Update `webgui.py` to point to this worker:
```python
default_worker_address = "http://localhost:8000"
```

### Example: Stable Diffusion Worker for `$IMAGINE`
Run Stable Diffusion via Docker:
```bash
docker run -p 8001:8000 stabilityai/stable-diffusion
```
Register the worker in `realms.yml` or via the Admin API.

**Note**: Worker setup details are pending. For production, configure real workers (e.g., HuggingFace, Stable Diffusion) as per your infrastructure.

## Step 5: Initialize the Database
Create the SQLite database (`llm_broker.db`):
```bash
python -c "from webgui import initialize_db; initialize_db()"
```

## Step 6: Run the Server
Start the FastAPI server with Uvicorn:
```bash
uvicorn webgui:app --host 0.0.0.0 --port 5000
```

The server initializes three realms (`$CHAT`, `$VISION`, `$IMAGINE`) and listens for WebSocket connections at `ws://localhost:5000/ws`.

## Step 7: Test the System
### Test WebSocket Query
Connect to the WebSocket endpoint using `wscat`:
```bash
wscat -c ws://localhost:5000/ws
```
Send a chat query:
```json
{
    "type": "submit_query",
    "query": {
        "prompt": "Hello, MTOR!",
        "query_type": "chat",
        "model_type": "huggingface",
        "model_name": "gpt2"
    }
}
```
**Expected**: The query is routed to the `$CHAT` realm, processed, and returns a response.

### Test Admin API
List realms:
```bash
curl http://localhost:5000/api/admin/realms
```
**Expected**:
```json
{
    "realms": [
        {"name": "$CHAT", "primary_query_type": "chat", "workers": 0, "pools": [], "metrics": {}},
        {"name": "$VISION", "primary_query_type": "vision", "workers": 0, "pools": [], "metrics": {}},
        {"name": "$IMAGINE", "primary_query_type": "imagine", "workers": 0, "pools": [], "metrics": {}}
    ]
}
```

## Step 8: Next Steps
- **Enable Sharding**: Configure `database_sharding` in `realms.yml` for scalable persistence.
- **Set Up Federation**: Add `bootstrap_nodes` to `realms.yml` and run multiple instances for cross-federation testing.
- **Monitor Performance**: See `PERFORMANCE.md` for stress testing and metrics.
- **Deploy to Production**: Use Docker Compose for multi-realm deployments.

## Troubleshooting
- **Worker Not Found**: Ensure workers are running and registered in `realms.yml` or via the Admin API.
- **Invalid `realms.yml`**: Validate YAML syntax using `yaml.safe_load`.
- **Database Errors**: Check `llm_broker.db` permissions and schema.

For detailed setup and performance tuning, refer to `PERFORMANCE.md` or contact the MTOR team.