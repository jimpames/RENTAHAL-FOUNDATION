# PERFORMANCE: Stress Testing and Optimizing RENT-A-HAL

This document describes how to evaluate and optimize the performance of the RENT-A-HAL system with the realm-based architecture, as implemented in `webgui.py` and `MTOR-planetscale-code.pdf`. It includes stress testing instructions, metric collection, and recommendations for planetary-scale operation.

## Metrics to Monitor
The system tracks several metrics via `RealmConnectionManager`, `RealmQueryProcessor`, and `FederationRouter`:

- **Connections** (`realm_stats["connections"]`): Number of active WebSocket connections per realm.
- **Peak Connections** (`realm_stats["peak_connections"]`): Maximum concurrent connections.
- **Processed Queries** (`realm_stats["processed_queries"]`): Total queries processed per realm.
- **Average Processing Time** (`realm_stats["avg_processing_time"]`): Mean query latency.
- **Error Count** (`realm_stats["error_count"]`): Number of failed queries.
- **Cross-Realm Queries** (`federation_stats["cross_realm_queries"]`): Queries routed between realms.
- **Cross-Federation Queries** (`federation_stats["cross_federation_queries"]`): Queries routed to remote federations.
- **Failed Routes** (`federation_stats["failed_routes"]`): Queries that couldn’t be routed.

## Setting Up Stress Testing
Use `locust` to simulate concurrent users and measure system performance.

### Install Locust
```bash
pip install locust
```

### Create `locustfile.py`
```python
from locust import HttpUser, task, between

class MTORUser(HttpUser):
    wait_time = between(1, 5)  # Wait 1-5 seconds between tasks

    @task
    def submit_chat_query(self):
        self.client.post("/api/chat", json={
            "prompt": "Test query",
            "query_type": "chat",
            "model_type": "huggingface",
            "model_name": "gpt2"
        })

    @task
    def submit_vision_query(self):
        self.client.post("/api/vision", json={
            "prompt": "Analyze image",
            "query_type": "vision",
            "model_type": "clip"
        })

    @task
    def submit_imagine_query(self):
        self.client.post("/api/imagine", json={
            "prompt": "Generate image",
            "query_type": "imagine",
            "model_type": "stable_diffusion"
        })
```

### Run Locust
Start the Locust web interface:
```bash
locust -f locustfile.py --host http://localhost:5000
```
Access `http://localhost:8089` in your browser, set the number of users (e.g., 1000) and spawn rate (e.g., 10 users/second), and start the test.

## Test Scenarios
1. **Single-Realm Load**:
   - Simulate 1000 users sending `$CHAT` queries.
   - **Expected**: `$CHAT` realm processes queries with low latency (<1s) and no errors.
   - **Metrics**: Monitor `processed_queries` and `avg_processing_time` for `$CHAT`.

2. **Multi-Realm Load**:
   - Simulate 500 users each for `$CHAT`, `$VISION`, and `$IMAGINE`.
   - **Expected**: Queries are routed to respective realms without cross-realm interference.
   - **Metrics**: Check `cross_realm_queries` (should be 0) and realm-specific `processed_queries`.

3. **Federation Load**:
   - Run two instances (ports 5000, 5001) with `bootstrap_nodes` configured:
     ```yaml
     discovery:
       bootstrap_nodes: ["http://localhost:5001"]
       advertise_interval: 3600
     ```
   - Simulate 1000 users, with 50% queries requiring cross-federation routing.
   - **Expected**: Queries are routed to the best worker across federations.
   - **Metrics**: Monitor `cross_federation_queries` and `failed_routes`.

4. **Sharding Performance**:
   - Enable sharding in `realms.yml`:
     ```yaml
     database_sharding:
       enabled: true
       shards:
         - name: shard1
           connection_string: sqlite:///shard1.db
         - name: shard2
           connection_string: sqlite:///shard2.db
       shard_map:
         "00": shard1
         "01": shard2
     ```
   - Simulate 1000 users with unique GUIDs.
   - **Expected**: User data is distributed across shards, reducing database contention.
   - **Metrics**: Check database query latency via logs.

## Interpreting Results
- **Latency**: `avg_processing_time` should be <1s for `$CHAT`, <5s for `$VISION`/`$IMAGINE`.
- **Throughput**: Aim for 1000 queries/minute per realm with 5 workers.
- **Error Rate**: `error_count` should be <1% of `processed_queries`.
- **Routing Efficiency**: `failed_routes` should be 0; `cross_federation_queries` should scale with federation size.

## Optimization Tips
1. **Increase Worker Capacity**:
   - Add more workers per realm in `realms.yml` (e.g., `max_workers: 10`).
   - Deploy GPU workers for `$IMAGINE` to reduce latency.

2. **Tune Load Balancing**:
   - Use `least_busy` strategy in `RealmWorkerManager` for high loads:
     ```python
     def select_worker_from_pool(self, pool_workers, strategy='least_busy'):
         return min(pool_workers, key=lambda w: w.active_connections)
     ```

3. **Enable Sharding**:
   - Configure multiple shards to distribute database load.
   - Use a production database (e.g., PostgreSQL) instead of SQLite.

4. **Monitor with Prometheus**:
   - Expose `/metrics` endpoint:
     ```python
     @app.get("/metrics")
     async def metrics():
         return {
             "realm_stats": {name: realm["conn_manager"].realm_stats for name, realm in realm_registry.realms.items()},
             "federation_stats": federation_router.federation_stats
         }
     ```
   - Integrate with Prometheus/Grafana for real-time dashboards.

5. **Scale Federation**:
   - Add more `bootstrap_nodes` to distribute queries globally.
   - Optimize `advertise_interval` (e.g., 1800s) for dynamic discovery.

## Benchmark Results
*Pending actual stress test results. Run the above scenarios and document here.*

Example (hypothetical):
- **Single-Realm Load**: 1000 users, 1200 queries/min, 0.8s avg latency, 0% errors.
- **Multi-Realm Load**: 1500 users, 1800 queries/min, 1.2s avg latency, 0.1% errors.
- **Federation Load**: 2 nodes, 1000 users, 50% cross-federation, 1.5s avg latency, 0% failed routes.

## Next Steps
- Run stress tests with production workers (e.g., Stable Diffusion, HuggingFace).
- Deploy a multi-node federation for real-world testing.
- Integrate Prometheus/Grafana for continuous monitoring.
- Update this file with benchmark results.

For setup details, see `QUICKSTART.md`.