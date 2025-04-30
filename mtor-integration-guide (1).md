# Technical Guide: Integrating Your API with MTOR

## Introduction

The Multi-Tronic Operating Realm (MTOR) represents a new paradigm for AI-first, event-driven computing. This technical guide will walk you through the process of integrating your existing API service with MTOR's universal JSON WebSockets bus, enabling your service to participate in MTOR's stateless, non-blocking, event-driven architecture.

By the end of this guide, you'll understand:
- The foundational principles of MTOR integration
- How to create an event-driven adapter for your existing API
- Implementing proper stateless, non-blocking patterns
- Testing and deploying your MTOR-compatible service

## Prerequisites

- Basic understanding of asynchronous programming
- Familiarity with Python and FastAPI
- Your existing API service (REST, GraphQL, or other)
- Python 3.8+
- Access to the MTOR WebSocket endpoint

## 1. Understanding MTOR's Event-Driven Architecture

MTOR follows a pure event-driven architecture where:
- Everything is an event
- No process should block
- The realm stays in continuous motion

This differs from traditional request-response patterns by treating all interactions as discrete events flowing through the system. Your API integration will need to adopt this model.

### Key Concepts

- **Events**: Self-contained messages with all necessary context
- **Event Bus**: The WebSocket-based communication channel
- **Handlers**: Asynchronous functions that process specific event types
- **Non-blocking Operations**: All operations must be asynchronous

## 2. Creating an MTOR Adapter

We'll create an adapter class that bridges your API with MTOR. This adapter will:
1. Connect to the MTOR WebSocket bus
2. Subscribe to relevant event types
3. Translate incoming events to API calls
4. Convert API responses to MTOR events

### Basic Adapter Structure

```python
import asyncio
import json
import logging
import websockets
from typing import Dict, Any, Callable, Optional, List, Set

class MTORAdapter:
    """
    Adapter for connecting your API service to the MTOR event bus.
    """
    
    def __init__(self, mtor_websocket_url: str, service_name: str, api_base_url: str):
        self.mtor_websocket_url = mtor_websocket_url
        self.service_name = service_name
        self.api_base_url = api_base_url
        self.websocket = None
        self.event_handlers: Dict[str, Callable] = {}
        self.running = False
        self.logger = logging.getLogger(f"mtor_adapter.{service_name}")
        self.subscribed_events: Set[str] = set()
        
    async def connect(self):
        """Establish connection to the MTOR WebSocket bus."""
        self.logger.info(f"Connecting to MTOR at {self.mtor_websocket_url}")
        self.websocket = await websockets.connect(self.mtor_websocket_url)
        self.running = True
        
        # Register with MTOR
        await self.send_event({
            "type": "service_registration",
            "service_name": self.service_name,
            "capabilities": list(self.event_handlers.keys())
        })
        
        self.logger.info("Connected to MTOR WebSocket bus")
        
    async def disconnect(self):
        """Gracefully disconnect from MTOR."""
        if self.websocket and self.websocket.open:
            await self.send_event({
                "type": "service_deregistration",
                "service_name": self.service_name
            })
            await self.websocket.close()
            self.running = False
            self.logger.info("Disconnected from MTOR WebSocket bus")
            
    async def send_event(self, event: Dict[str, Any]):
        """Send an event to the MTOR bus."""
        if not self.websocket:
            raise RuntimeError("Not connected to MTOR WebSocket bus")
            
        event_json = json.dumps(event)
        await self.websocket.send(event_json)
        self.logger.debug(f"Sent event: {event['type']}")
        
    def register_handler(self, event_type: str, handler: Callable):
        """Register a handler for a specific event type."""
        self.event_handlers[event_type] = handler
        self.subscribed_events.add(event_type)
        self.logger.info(f"Registered handler for event type: {event_type}")
        
    async def event_listener(self):
        """Main event loop that listens for incoming events."""
        if not self.websocket:
            raise RuntimeError("Not connected to MTOR WebSocket bus")
            
        self.logger.info("Starting event listener")
        
        while self.running:
            try:
                message = await self.websocket.recv()
                event = json.loads(message)
                
                if "type" not in event:
                    self.logger.warning(f"Received event without type: {event}")
                    continue
                    
                event_type = event["type"]
                
                if event_type in self.event_handlers:
                    # Handle event asynchronously to avoid blocking
                    asyncio.create_task(self._handle_event(event_type, event))
                else:
                    self.logger.debug(f"No handler for event type: {event_type}")
                    
            except websockets.exceptions.ConnectionClosed:
                self.logger.warning("WebSocket connection closed")
                self.running = False
                break
            except Exception as e:
                self.logger.error(f"Error in event listener: {str(e)}")
                # Don't break the loop on error, just continue
                
        self.logger.info("Event listener stopped")
        
    async def _handle_event(self, event_type: str, event: Dict[str, Any]):
        """Process an event with the registered handler."""
        try:
            handler = self.event_handlers[event_type]
            response = await handler(event)
            
            if response:
                # If handler returns a response, send it back to MTOR
                if "id" in event:
                    response["response_to"] = event["id"]
                await self.send_event(response)
                
        except Exception as e:
            self.logger.error(f"Error handling event {event_type}: {str(e)}")
            if "id" in event:
                # Send error response if the event had an ID
                await self.send_event({
                    "type": "error",
                    "response_to": event["id"],
                    "error": str(e)
                })
                
    async def run(self):
        """Main method to run the adapter."""
        await self.connect()
        
        # Subscribe to event types
        await self.send_event({
            "type": "subscribe",
            "events": list(self.subscribed_events)
        })
        
        # Start the event listener
        listener_task = asyncio.create_task(self.event_listener())
        
        try:
            # Keep running until disconnected or exception
            await listener_task
        finally:
            await self.disconnect()
```

## 3. Implementing API-Specific Handlers

Now we'll create handlers for translating between MTOR events and your API.

### REST API Example

Here's how to connect a REST API to MTOR:

```python
import httpx
from typing import Dict, Any

class RESTAPIAdapter(MTORAdapter):
    """Adapter for REST APIs."""
    
    def __init__(self, mtor_websocket_url: str, service_name: str, api_base_url: str):
        super().__init__(mtor_websocket_url, service_name, api_base_url)
        self.http_client = httpx.AsyncClient()
        
        # Register handlers for standard REST operations
        self.register_handler("api_get_request", self.handle_get_request)
        self.register_handler("api_post_request", self.handle_post_request)
        self.register_handler("api_put_request", self.handle_put_request)
        self.register_handler("api_delete_request", self.handle_delete_request)
        
    async def close(self):
        await self.http_client.aclose()
        await self.disconnect()
        
    async def handle_get_request(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle a GET request event."""
        endpoint = event.get("endpoint", "")
        params = event.get("params", {})
        
        url = f"{self.api_base_url}/{endpoint}"
        self.logger.info(f"Making GET request to {url}")
        
        try:
            response = await self.http_client.get(url, params=params)
            response.raise_for_status()
            
            return {
                "type": "api_response",
                "status_code": response.status_code,
                "data": response.json(),
                "request_type": "get",
                "endpoint": endpoint
            }
        except httpx.HTTPStatusError as e:
            return {
                "type": "api_error",
                "status_code": e.response.status_code,
                "error": str(e),
                "request_type": "get",
                "endpoint": endpoint
            }
        except Exception as e:
            return {
                "type": "api_error",
                "error": str(e),
                "request_type": "get",
                "endpoint": endpoint
            }
            
    async def handle_post_request(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle a POST request event."""
        endpoint = event.get("endpoint", "")
        data = event.get("data", {})
        
        url = f"{self.api_base_url}/{endpoint}"
        self.logger.info(f"Making POST request to {url}")
        
        try:
            response = await self.http_client.post(url, json=data)
            response.raise_for_status()
            
            return {
                "type": "api_response",
                "status_code": response.status_code,
                "data": response.json(),
                "request_type": "post",
                "endpoint": endpoint
            }
        except httpx.HTTPStatusError as e:
            return {
                "type": "api_error",
                "status_code": e.response.status_code,
                "error": str(e),
                "request_type": "post",
                "endpoint": endpoint
            }
        except Exception as e:
            return {
                "type": "api_error",
                "error": str(e),
                "request_type": "post",
                "endpoint": endpoint
            }
    
    # Similar implementations for put and delete handlers
    async def handle_put_request(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle a PUT request event."""
        # Implementation follows the same pattern as post
        pass
        
    async def handle_delete_request(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle a DELETE request event."""
        # Implementation follows the same pattern as get
        pass
```

### FastAPI Example

For FastAPI services, we can create a more integrated adapter:

```python
import httpx
from typing import Dict, Any, Optional, List
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

class FastAPIToMTOR:
    """
    Class to connect a FastAPI application directly to MTOR.
    """
    
    def __init__(self, app: FastAPI, mtor_websocket_url: str, service_name: str):
        self.app = app
        self.mtor_websocket_url = mtor_websocket_url
        self.service_name = service_name
        self.logger = logging.getLogger(f"mtor_fastapi.{service_name}")
        self.client = None  # Will hold connection to MTOR
        
        # Register the WebSocket endpoint for MTOR to connect to this service
        @app.websocket("/mtor")
        async def mtor_websocket(websocket: WebSocket):
            await self._handle_mtor_connection(websocket)
            
        # Register startup and shutdown events
        @app.on_event("startup")
        async def startup_event():
            await self.connect_to_mtor()
            
        @app.on_event("shutdown")
        async def shutdown_event():
            await self.disconnect_from_mtor()
            
    async def connect_to_mtor(self):
        """Connect to the MTOR WebSocket bus."""
        self.client = httpx.AsyncClient()
        self.logger.info(f"Connecting to MTOR at {self.mtor_websocket_url}")
        
        # Register service with MTOR
        try:
            response = await self.client.post(
                f"{self.mtor_websocket_url}/register",
                json={
                    "service_name": self.service_name,
                    "callback_url": f"{self.app.url}/mtor"
                }
            )
            response.raise_for_status()
            self.logger.info("Successfully registered with MTOR")
        except Exception as e:
            self.logger.error(f"Failed to register with MTOR: {str(e)}")
            
    async def disconnect_from_mtor(self):
        """Disconnect from the MTOR WebSocket bus."""
        if self.client:
            try:
                response = await self.client.post(
                    f"{self.mtor_websocket_url}/deregister",
                    json={"service_name": self.service_name}
                )
                response.raise_for_status()
                self.logger.info("Successfully deregistered from MTOR")
            except Exception as e:
                self.logger.error(f"Failed to deregister from MTOR: {str(e)}")
                
            await self.client.aclose()
            
    async def _handle_mtor_connection(self, websocket: WebSocket):
        """Handle an incoming WebSocket connection from MTOR."""
        await websocket.accept()
        self.logger.info("Accepted WebSocket connection from MTOR")
        
        try:
            while True:
                message = await websocket.receive_text()
                event = json.loads(message)
                
                # Process the event
                response_event = await self._process_event(event)
                
                if response_event:
                    await websocket.send_text(json.dumps(response_event))
                    
        except WebSocketDisconnect:
            self.logger.info("MTOR WebSocket connection closed")
        except Exception as e:
            self.logger.error(f"Error in MTOR WebSocket connection: {str(e)}")
            
    async def _process_event(self, event: Dict[str, Any]) -> Optional[Dict[str, Any]]:
        """Process an incoming event from MTOR."""
        if "type" not in event:
            self.logger.warning(f"Received event without type: {event}")
            return None
            
        event_type = event["type"]
        
        # Handle different event types
        if event_type == "api_request":
            return await self._handle_api_request(event)
        elif event_type == "health_check":
            return {"type": "health_response", "status": "healthy"}
        else:
            self.logger.warning(f"Unknown event type: {event_type}")
            return None
            
    async def _handle_api_request(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle an API request event from MTOR."""
        method = event.get("method", "").lower()
        endpoint = event.get("endpoint", "")
        
        if not endpoint:
            return {
                "type": "api_error",
                "error": "No endpoint specified",
                "request_id": event.get("id")
            }
            
        url = f"http://127.0.0.1:{self.app.port}/{endpoint}"
        
        try:
            if method == "get":
                params = event.get("params", {})
                response = await self.client.get(url, params=params)
            elif method == "post":
                data = event.get("data", {})
                response = await self.client.post(url, json=data)
            elif method == "put":
                data = event.get("data", {})
                response = await self.client.put(url, json=data)
            elif method == "delete":
                response = await self.client.delete(url)
            else:
                return {
                    "type": "api_error",
                    "error": f"Unsupported method: {method}",
                    "request_id": event.get("id")
                }
                
            response.raise_for_status()
            
            return {
                "type": "api_response",
                "status_code": response.status_code,
                "data": response.json(),
                "request_id": event.get("id")
            }
        except Exception as e:
            return {
                "type": "api_error",
                "error": str(e),
                "request_id": event.get("id")
            }
```

## 4. Ensuring Proper Event-Driven Patterns

When integrating with MTOR, follow these best practices:

### Statelessness

Your adapter should maintain minimal state. Any necessary state should be:
- Contained within individual event handlers
- Passed as part of events themselves
- Stored in external databases if persistence is required

```python
# Bad - storing state in the adapter
class BadAdapter(MTORAdapter):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.user_sessions = {}  # State that could grow unbounded
        
    async def handle_login(self, event):
        user_id = event.get("user_id")
        self.user_sessions[user_id] = {"logged_in": True}  # Stateful
        
# Good - keeping adapter stateless
class GoodAdapter(MTORAdapter):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.redis_client = redis.Redis()  # External state store
        
    async def handle_login(self, event):
        user_id = event.get("user_id")
        # Store state externally
        await self.redis_client.set(f"user_session:{user_id}", json.dumps({"logged_in": True}))
```

### Non-Blocking Operations

All operations should be asynchronous to avoid blocking the event loop:

```python
# Bad - blocking operation
async def handle_image_processing(self, event):
    image_data = event.get("image_data")
    # This blocks the event loop
    processed_image = process_image(image_data)  
    return {"type": "image_processed", "data": processed_image}
    
# Good - non-blocking operation
async def handle_image_processing(self, event):
    image_data = event.get("image_data")
    # Run CPU-intensive operation in a thread pool
    processed_image = await asyncio.to_thread(process_image, image_data)
    return {"type": "image_processed", "data": processed_image}
```

### Error Handling

Proper error handling is crucial in event-driven systems:

```python
async def safe_handler(self, event_type, event):
    try:
        handler = self.event_handlers[event_type]
        return await handler(event)
    except Exception as e:
        self.logger.error(f"Error in handler for {event_type}: {str(e)}")
        # Return a structured error response
        return {
            "type": "error",
            "original_type": event_type,
            "error": str(e),
            "request_id": event.get("id")
        }
```

## 5. Specialized Integration Examples

### Integrating an ML Model API

```python
class MLModelAdapter(MTORAdapter):
    """Adapter for machine learning model APIs."""
    
    def __init__(self, mtor_websocket_url: str, service_name: str, model_endpoint: str):
        super().__init__(mtor_websocket_url, service_name, model_endpoint)
        self.http_client = httpx.AsyncClient()
        
        # Register model-specific handlers
        self.register_handler("prediction_request", self.handle_prediction)
        self.register_handler("batch_prediction_request", self.handle_batch_prediction)
        
    async def handle_prediction(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle a single prediction request."""
        input_data = event.get("data", {})
        
        try:
            response = await self.http_client.post(
                self.api_base_url,
                json={"inputs": input_data}
            )
            response.raise_for_status()
            
            return {
                "type": "prediction_response",
                "prediction": response.json(),
                "request_id": event.get("id")
            }
        except Exception as e:
            return {
                "type": "prediction_error",
                "error": str(e),
                "request_id": event.get("id")
            }
            
    async def handle_batch_prediction(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle a batch prediction request."""
        batch_data = event.get("batch", [])
        
        # For large batches, process in chunks to avoid blocking
        results = []
        chunk_size = 10
        
        for i in range(0, len(batch_data), chunk_size):
            chunk = batch_data[i:i+chunk_size]
            tasks = []
            
            for item in chunk:
                task = asyncio.create_task(self._predict_single(item))
                tasks.append(task)
                
            chunk_results = await asyncio.gather(*tasks, return_exceptions=True)
            results.extend(chunk_results)
            
        return {
            "type": "batch_prediction_response",
            "predictions": results,
            "request_id": event.get("id")
        }
        
    async def _predict_single(self, input_data: Dict[str, Any]) -> Dict[str, Any]:
        """Make a prediction for a single data point."""
        try:
            response = await self.http_client.post(
                self.api_base_url,
                json={"inputs": input_data}
            )
            response.raise_for_status()
            return response.json()
        except Exception as e:
            return {"error": str(e)}
```

### Integrating Claude API

This example shows how to connect the Claude API to MTOR:

```python
import anthropic
from typing import Dict, Any, List

class ClaudeAdapter(MTORAdapter):
    """Adapter for the Claude API."""
    
    def __init__(self, mtor_websocket_url: str, service_name: str, api_key: str):
        super().__init__(mtor_websocket_url, service_name, "https://api.anthropic.com")
        self.api_key = api_key
        self.client = anthropic.AsyncAnthropic(api_key=api_key)
        
        # Register handlers
        self.register_handler("claude_completion", self.handle_completion)
        self.register_handler("claude_chat", self.handle_chat)
        
    async def handle_completion(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle a Claude completion request."""
        prompt = event.get("prompt", "")
        max_tokens = event.get("max_tokens", 1000)
        model = event.get("model", "claude-3-opus-20240229")
        
        try:
            response = await self.client.completions.create(
                prompt=prompt,
                max_tokens_to_sample=max_tokens,
                model=model
            )
            
            return {
                "type": "claude_completion_response",
                "completion": response.completion,
                "request_id": event.get("id")
            }
        except Exception as e:
            return {
                "type": "claude_error",
                "error": str(e),
                "request_id": event.get("id")
            }
            
    async def handle_chat(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle a Claude chat request."""
        messages = event.get("messages", [])
        max_tokens = event.get("max_tokens", 1000)
        model = event.get("model", "claude-3-opus-20240229")
        
        try:
            response = await self.client.messages.create(
                messages=messages,
                max_tokens=max_tokens,
                model=model
            )
            
            return {
                "type": "claude_chat_response",
                "message": response.content,
                "request_id": event.get("id")
            }
        except Exception as e:
            return {
                "type": "claude_error",
                "error": str(e),
                "request_id": event.get("id")
            }
```

## 6. Deployment and Testing

### Basic Testing Script

```python
import asyncio
import json
import logging

async def test_adapter(adapter_class, *args, **kwargs):
    """Test an MTOR adapter implementation."""
    logging.basicConfig(level=logging.INFO)
    
    adapter = adapter_class(*args, **kwargs)
    
    # Start the adapter
    adapter_task = asyncio.create_task(adapter.run())
    
    # Wait for adapter to connect
    await asyncio.sleep(2)
    
    # Send a test event
    test_event = {
        "type": "test_event",
        "id": "test-123",
        "data": {"message": "Hello, MTOR!"}
    }
    
    print(f"Sending test event: {json.dumps(test_event, indent=2)}")
    
    # Send event to MTOR (this would normally come from MTOR)
    # Here we're simulating by directly calling the handler
    if "test_event" in adapter.event_handlers:
        response = await adapter.event_handlers["test_event"](test_event)
        print(f"Received response: {json.dumps(response, indent=2)}")
    else:
        print("No handler registered for test_event")
    
    # Clean up
    await adapter.disconnect()
    adapter_task.cancel()
    
    try:
        await adapter_task
    except asyncio.CancelledError:
        pass

# Example usage
if __name__ == "__main__":
    asyncio.run(test_adapter(
        RESTAPIAdapter,
        "ws://mtor-server:8000/ws",
        "my-rest-api-service",
        "http://my-api-service:5000"
    ))
```

### Docker Deployment

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "your_adapter_service.py"]
```

Example `requirements.txt`:
```
websockets==10.3
httpx==0.22.0
redis==4.3.1
pydantic==1.9.1
anthropic==0.5.0  # If using Claude adapter
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mtor-api-adapter
  labels:
    app: mtor-api-adapter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mtor-api-adapter
  template:
    metadata:
      labels:
        app: mtor-api-adapter
    spec:
      containers:
      - name: adapter
        image: your-registry/mtor-api-adapter:latest
        env:
        - name: MTOR_WEBSOCKET_URL
          value: "ws://mtor-service:8000/ws"
        - name: SERVICE_NAME
          value: "your-api-service"
        - name: API_BASE_URL
          value: "http://your-api-service:5000"
        resources:
          limits:
            cpu: "500m"
            memory: "256Mi"
          requests:
            cpu: "100m"
            memory: "128Mi"
```

## 7. Advanced Topics

### Custom Event Types

You can define custom event types specific to your service:

```python
# Custom events for a recommendation service
class RecommendationAdapter(MTORAdapter):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        self.register_handler("get_recommendations", self.handle_recommendations)
        self.register_handler("record_interaction", self.handle_interaction)
        
    async def handle_recommendations(self, event):
        user_id = event.get("user_id")
        # Get recommendations for user
        recommendations = await self._fetch_recommendations(user_id)
        
        return {
            "type": "recommendations_response",
            "user_id": user_id,
            "recommendations": recommendations,
            "request_id": event.get("id")
        }
        
    async def handle_interaction(self, event):
        user_id = event.get("user_id")
        item_id = event.get("item_id")
        interaction_type = event.get("interaction_type")
        
        # Record the interaction
        await self._record_user_interaction(user_id, item_id, interaction_type)
        
        return {
            "type": "interaction_recorded",
            "success": True,
            "request_id": event.get("id")
        }
```

### Event Schema Validation

For robust integration, add schema validation:

```python
from pydantic import BaseModel, ValidationError
from typing import Optional, List, Dict, Any

# Define event schemas
class RecommendationRequest(BaseModel):
    user_id: str
    limit: Optional[int] = 10
    filters: Optional[Dict[str, Any]] = {}

class InteractionEvent(BaseModel):
    user_id: str
    item_id: str
    interaction_type: str
    timestamp: Optional[float] = None

# Adapter with schema validation
class ValidatedAdapter(MTORAdapter):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        self.register_handler("get_recommendations", self.handle_recommendations)
        self.register_handler("record_interaction", self.handle_interaction)
        
    async def handle_recommendations(self, event):
        try:
            # Validate incoming event
            req = RecommendationRequest(**event)
            
            # Process with validated data
            recommendations = await self._fetch_recommendations(
                req.user_id, req.limit, req.filters
            )
            
            return {
                "type": "recommendations_response",
                "user_id": req.user_id,
                "recommendations": recommendations,
                "request_id": event.get("id")
            }
        except ValidationError as e:
            return {
                "type": "validation_error",
                "errors": e.errors(),
                "request_id": event.get("id")
            }
            
    async def handle_interaction(self, event):
        try:
            # Add timestamp if not provided
            if "timestamp" not in event:
                event["timestamp"] = time.time()
                
            # Validate incoming event
            interaction = InteractionEvent(**event)
            
            # Process with validated data
            await self._record_user_interaction(
                interaction.user_id,
                interaction.item_id,
                interaction.interaction_type,
                interaction.timestamp
            )
            
            return {
                "type": "interaction_recorded",
                "success": True,
                "request_id": event.get("id")
            }
        except ValidationError as e:
            return {
                "type": "validation_error",
                "errors": e.errors(),
                "request_id": event.get("id")
            }
```

### Streaming Responses

For long-running operations or large responses, implement streaming:

```python
class StreamingAdapter(MTORAdapter):
    async def handle_large_data_request(self, event):
        request_id = event.get("id")
        user_id = event.get("user_id")
        
        # Start a background task for processing
        asyncio.create_task(self._process_and_stream(request_id, user_id))
        
        # Return immediate acknowledgment
        return {
            "type": "processing_started",
            "request_id": request_id,
            "message": "Processing started, results will be streamed"
        }
        
    async def _process_and_stream(self, request_id, user_id):
        try:
            # Simulate a long-running process
            for i in range(10):
                # Do some work
                chunk_result = await self._process_chunk(user_id, i)
                
                # Send intermediate result
                await self.send_event({
                    "type": "data_chunk",
                    "request_id": request_id,
                    "chunk_index": i,
                    "data": chunk_result,
                    "is_final": i == 9
                })
                
                # Small delay to avoid flooding
                await asyncio.sleep(0.1)
                
        except Exception as e:
            # Send error if processing fails
            await self.send_event({
                "type": "processing_error",
                "request_id": request_id,
                "error": str(e)
            })
```

## 8. Monitoring and Observability

To properly monitor your MTOR integration:

```python
import time
import prometheus_client
from prometheus_client import Counter, Histogram, Gauge

class MonitoredAdapter(MTORAdapter):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        # Set up metrics
        self.event_counter = Counter(
            'mtor_events_total',
            'Total number of events processed',
            ['event_type', 'service_name']
        )
        
        self.event_processing_time = Histogram(
            'mtor_event_processing_seconds',
            'Time spent processing events',
            ['event_type', 'service_name'],
            buckets=(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10)
        )
        
        self.connection_state = Gauge(
            'mtor_connection_state',
            'Connection state (1=connected, 0=disconnected)',
            ['service_name']
        )
        
        # Expose metrics endpoint
        prometheus_client.start_http_server(8000)
        
    async def _handle_event(self, event_type: str, event: Dict[str, Any]):
        # Track event count
        self.event_counter.labels(event_type, self.service_name).inc()
        
        # Track processing time
        start_time = time.time()
        
        try:
            result = await super()._handle_event(event_type, event)
            processing_time = time.time() - start_time
            self.event_processing_time.labels(event_type, self.service_name).observe(processing_time)
            return result
        except Exception as e:
            # Track processing time even for failed events
            processing_time = time.time() - start_time
            self.event_processing_time.labels(event_type, self.service_name).observe(processing_time)
            raise
            
    async def connect(self):
        await super().connect()
        self.connection_state.labels(self.service_name).set(1)
        
    async def disconnect(self):
        await super().disconnect()
        self.connection_state.labels(self.service_name).set(0)
```

## 9. Full Example: Ollama Integration

Here's a complete example for integrating Ollama with MTOR:

```python
import asyncio
import json
import logging
import httpx
from typing import Dict, Any, List, Optional

from mtor_adapter import MTORAdapter  # Import the base adapter

class OllamaAdapter(MTORAdapter):
    """
    Adapter for connecting Ollama API to MTOR.
    """
    
    def __init__(self, mtor_websocket_url: str, ollama_api_url: str):
        super().__init__(
            mtor_websocket_url=mtor_websocket_url,
            service_name="ollama",
            api_base_url=ollama_api_url
        )
        self.http_client = httpx.AsyncClient(timeout=60.0)  # Longer timeout for model responses
        
        # Register handlers for Ollama-specific events
        self.register_handler("ollama_chat", self.handle_chat)
        self.register_handler("ollama_generate", self.handle_generate)
        self.register_handler("ollama_embeddings", self.handle_embeddings)
        self.register_handler("ollama_list_models", self.handle_list_models)
        
    async def handle_chat(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle a chat request for Ollama."""
        model = event.get("model", "llama2")
        messages = event.get("messages", [])
        options = event.get("options", {})
        
        try:
            response = await self.http_client.post(
                f"{self.api_base_url}/api/chat",
                json={
                    "model": model,
                    "messages": messages,
                    "options": options,
                    "stream": False
                }
            )
            response.raise_for_status()
            
            return {
                "type": "ollama_chat_response",
                "response": response.json(),
                "request_id": event.get("id")
            }
        except Exception as e:
            return {
                "type": "ollama_error",
                "error": str(e),
                "request_id": event.get("id")
            }
            
    async def handle_generate(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle a text generation request for Ollama."""
        model = event.get("model", "llama2")
        prompt = event.get("prompt", "")
        options = event.get("options", {})
        
        try:
            response = await self.http_client.post(
                f"{self.api_base_url}/api/generate",
                json={
                    "model": model,
                    "prompt": prompt,
                    "options": options,
                    "stream": False
                }
            )
            response.raise_for_status()
            
            return {
                "type": "ollama_generate_response",
                "response": response.json(),
                "request_id": event.get("id")
            }
        except Exception as e:
            return {
                "type": "ollama_error",
                "error": str(e),
                "request_id": event.get("id")
            }
            
    async def handle_embeddings(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle an embeddings request for Ollama."""
        model = event.get("model", "llama2")
        prompt = event.get("prompt", "")
        
        try:
            response = await self.http_client.post(
                f"{self.api_base_url}/api/embeddings",
                json={
                    "model": model,
                    "prompt": prompt
                }
            )
            response.raise_for_status()
            
            return {
                "type": "ollama_embeddings_response",
                "embedding": response.json().get("embedding"),
                "request_id": event.get("id")
            }
        except Exception as e:
            return {
                "type": "ollama_error",
                "error": str(e),
                "request_id": event.get("id")
            }
            
    async def handle_list_models(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Handle a request to list available models."""
        try:
            response = await self.http_client.get(f"{self.api_base_url}/api/tags")
            response.raise_for_status()
            
            return {
                "type": "ollama_models_response",
                "models": response.json().get("models", []),
                "request_id": event.get("id")
            }
        except Exception as e:
            return {
                "type": "ollama_error",
                "error": str(e),
                "request_id": event.get("id")
            }
            
    async def close(self):
        """Clean up resources."""
        await self.http_client.aclose()
        await self.disconnect()


# Example usage
if __name__ == "__main__":
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    
    # Create and run the adapter
    adapter = OllamaAdapter(
        mtor_websocket_url="ws://mtor-server:8000/ws",
        ollama_api_url="http://localhost:11434"
    )
    
    # Run the adapter
    asyncio.run(adapter.run())
```

## 10. Security Considerations

When integrating with MTOR, consider these security best practices:

### Authentication

```python
class SecureAdapter(MTORAdapter):
    """Adapter with enhanced security."""
    
    def __init__(self, mtor_websocket_url: str, service_name: str, api_base_url: str, 
                 mtor_api_key: str, service_api_key: Optional[str] = None):
        super().__init__(mtor_websocket_url, service_name, api_base_url)
        self.mtor_api_key = mtor_api_key
        self.service_api_key = service_api_key
        
    async def connect(self):
        """Connect with authentication."""
        self.logger.info(f"Connecting securely to MTOR at {self.mtor_websocket_url}")
        
        # Include API key in WebSocket connection
        headers = {"Authorization": f"Bearer {self.mtor_api_key}"}
        self.websocket = await websockets.connect(self.mtor_websocket_url, extra_headers=headers)
        self.running = True
        
        # Register with enhanced security
        await self.send_event({
            "type": "secure_service_registration",
            "service_name": self.service_name,
            "capabilities": list(self.event_handlers.keys()),
            "auth": {
                "api_key_hash": self._hash_api_key(self.mtor_api_key)
            }
        })
        
        self.logger.info("Connected securely to MTOR WebSocket bus")
        
    def _hash_api_key(self, api_key: str) -> str:
        """Create a hash of the API key for verification."""
        import hashlib
        return hashlib.sha256(api_key.encode()).hexdigest()
        
    async def _handle_event(self, event_type: str, event: Dict[str, Any]):
        """Verify events before processing."""
        # Check for authentication token in event
        auth_token = event.get("auth_token")
        if not auth_token or not self._verify_token(auth_token):
            return {
                "type": "auth_error",
                "error": "Invalid or missing authentication",
                "request_id": event.get("id")
            }
            
        # Continue with normal event handling
        return await super()._handle_event(event_type, event)
        
    def _verify_token(self, token: str) -> bool:
        """Verify an authentication token."""
        # Implement token verification logic here
        # This could be JWT verification, comparing with stored tokens, etc.
        # For demonstration only:
        import hmac
        return hmac.compare_digest(token, "valid_token_here")
```

### Rate Limiting

```python
import time
import asyncio
from collections import deque

class RateLimitedAdapter(MTORAdapter):
    """Adapter with rate limiting."""
    
    def __init__(self, *args, requests_per_minute: int = 60, **kwargs):
        super().__init__(*args, **kwargs)
        self.requests_per_minute = requests_per_minute
        self.request_times = deque(maxlen=requests_per_minute)
        self.rate_limit_lock = asyncio.Lock()
        
    async def _handle_event(self, event_type: str, event: Dict[str, Any]):
        """Apply rate limiting before processing events."""
        # Check rate limit
        async with self.rate_limit_lock:
            current_time = time.time()
            
            # Clean old request times
            while self.request_times and self.request_times[0] < current_time - 60:
                self.request_times.popleft()
                
            # Check if we're at the limit
            if len(self.request_times) >= self.requests_per_minute:
                return {
                    "type": "rate_limit_error",
                    "error": "Rate limit exceeded, please try again later",
                    "request_id": event.get("id")
                }
                
            # Add current request time
            self.request_times.append(current_time)
            
        # Continue with normal event handling
        return await super()._handle_event(event_type, event)
```

## Conclusion

This guide has covered the essentials for integrating your API with MTOR's event-driven, stateless architecture. By following these patterns and examples, you can create a resilient, non-blocking adapter that participates in the MTOR ecosystem.

Remember the core principles:
- Everything is an event
- All operations should be non-blocking
- Maintain statelessness
- Handle errors gracefully and provide informative responses

Your MTOR adapter serves as a bridge between traditional API services and MTOR's real-time, event-driven environment, enabling a new level of integration and interoperability in AI-powered applications.

## Resources

- [MTOR Documentation](https://mtor-docs.example.com)
- [WebSockets Documentation](https://websockets.readthedocs.io/)
- [AsyncIO Documentation](https://docs.python.org/3/library/asyncio.html)
- [HTTPX Documentation](https://www.python-httpx.org/)
- [Pydantic Documentation](https://pydantic-docs.helpmanual.io/)