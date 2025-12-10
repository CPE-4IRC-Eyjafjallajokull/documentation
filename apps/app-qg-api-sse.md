# SSE Integration - app-qg-api

## Overview

This document describes the CORS setup and SSE configuration added to the API to support real-time event streaming to the frontend.

## Changes Made

### 1. CORS Middleware Addition

**File:** `src/app/main.py`

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Allow all origins for development
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Purpose:**
- Allows the frontend (running on a different port/domain) to make requests to the API
- Required for SSE EventSource connections from `http://localhost:3000` to `http://localhost:8000`

**Configuration Options:**
- `allow_origins`: List of allowed origins. Use `["*"]` for development, specific domains for production
- `allow_credentials`: Allow credential transmission
- `allow_methods`: HTTP methods allowed
- `allow_headers`: Headers allowed in requests

### 2. SSE Endpoint Configuration

**File:** `src/app/api/routes/events.py` (already existed)

The API already had the SSE endpoint implemented:

```python
@router.get("/events")
async def events_stream():
    """Server-sent events stream for one-way, real-time updates."""
    
    async def event_source():
        # Send initial connected event
        yield f"data: {json.dumps({'event': 'connected', ...})}\n\n"
        
        # Send heartbeat events
        while True:
            payload = {"event": "heartbeat", "timestamp": ...}
            yield f"data: {json.dumps(payload)}\n\n"
            await asyncio.sleep(settings.events_ping_interval_seconds)
    
    return StreamingResponse(
        event_source(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache"},
    )
```

**Key Components:**
- `StreamingResponse`: FastAPI streaming for SSE
- `media_type="text/event-stream"`: Tells browser this is SSE data
- `Cache-Control: no-cache`: Prevents caching of events
- Event format: `data: {JSON}\n\n` (double newline required)

## Architecture

### Event Flow

```
API (port 8000)
  └── GET /events
      └── StreamingResponse
          └── Browser EventSource
              └── Frontend (port 3000)
                  └── React Hook (useSSE)
                      └── Component renders
```

### Connection Lifecycle

1. **Frontend initiates:** `new EventSource(url)`
2. **Browser opens HTTP connection** to `/events`
3. **API sends initial event:** `{"event": "connected"}`
4. **API sends periodic heartbeats:** `{"event": "heartbeat"}`
5. **Browser handles events:** via `onmessage` callback
6. **Auto-reconnect on disconnect:** Built-in EventSource behavior

## Configuration

### Event Frequency

Edit `src/app/core/config.py`:

```python
events_ping_interval_seconds: int = 5  # Heartbeat interval in seconds
```

Default: 5 seconds between heartbeat events

### CORS Settings for Production

**Current (Development):**
```python
allow_origins=["*"]
```

**Production:**
```python
allow_origins=[
    "https://example.com",
    "https://www.example.com",
    "https://app.example.com"
]
```

## Testing

### Test SSE Stream with curl

```bash
# Stream events to terminal
curl -N http://localhost:8000/events

# Output:
# data: {"event": "connected", "timestamp": "2025-12-10T..."}
# 
# data: {"event": "heartbeat", "timestamp": "2025-12-10T..."}
# 
```

### Test CORS Headers

```bash
curl -i -X OPTIONS http://localhost:8000/events \
  -H "Origin: http://localhost:3000" \
  -H "Access-Control-Request-Method: GET"

# Check for CORS headers in response:
# Access-Control-Allow-Origin: *
# Access-Control-Allow-Methods: *
# Access-Control-Allow-Headers: *
```

### API Health Check

```bash
curl http://localhost:8000/health
# Output: {"status":"ok"}
```

## API Logging

The API logs SSE events:

```json
{"env": "local", "event": "events.connected", "level": "info", "timestamp": "..."}
{"event": "events.disconnected", "level": "info", "timestamp": "..."}
```

These appear in the uvicorn console output and help track client connections.

## Extending the SSE System

### Adding Custom Events

Modify `src/app/api/routes/events.py`:

```python
@router.get("/events")
async def events_stream():
    async def event_source():
        yield f"data: {json.dumps({'event': 'connected'})}\n\n"
        
        while True:
            # Add custom events here
            custom_event = await get_custom_event()  # Your logic
            yield f"data: {json.dumps(custom_event)}\n\n"
            
            await asyncio.sleep(settings.events_ping_interval_seconds)
    
    return StreamingResponse(...)
```

### Broadcasting from Other Endpoints

Create a global event queue (if needed):

```python
# In a services module
import asyncio

class EventBroadcaster:
    def __init__(self):
        self.queue = asyncio.Queue()
    
    async def broadcast(self, event: dict):
        await self.queue.put(event)
    
    async def subscribe(self):
        while True:
            event = await self.queue.get()
            yield event

broadcaster = EventBroadcaster()
```

## Troubleshooting

### CORS Errors in Browser

**Error:** "Access to XMLHttpRequest blocked by CORS policy"

**Solutions:**
1. Ensure CORS middleware is added to FastAPI app
2. Check `allow_origins` includes frontend URL
3. Verify headers are properly configured
4. Test with curl to isolate issue

### Events Not Streaming

**Error:** Connection opens but no data received

**Solutions:**
1. Check API is running: `curl http://localhost:8000/health`
2. Test endpoint directly: `curl -N http://localhost:8000/events`
3. Check API logs for errors
4. Verify `StreamingResponse` is correct

### Client Disconnects Frequently

**Error:** Events stop after a few seconds

**Potential Causes:**
- Network timeout: Adjust heartbeat interval
- Exception in `event_source()`: Check API logs
- Resource leaks: Ensure proper cleanup
- Proxy/firewall: May close long-lived connections

**Solutions:**
1. Review API logs for exceptions
2. Reduce `events_ping_interval_seconds` (keep clients active)
3. Check network configuration
4. Ensure sufficient server resources

## Performance

### Memory Usage
- Each connected client has one StreamingResponse instance
- Minimal memory per connection (< 1MB typically)
- No buffering by default (events sent directly)

### Scalability
- Current implementation: Single-threaded, one event stream per client
- For many clients: Consider message queue (RabbitMQ integration exists)
- For high-frequency events: Consider WebSockets instead

### Optimization Tips
- Use shorter heartbeat interval to detect stale connections: `events_ping_interval_seconds = 3`
- Filter events on backend to reduce payload
- Use client-side event filtering if needed
- Monitor connection count and resource usage

## Security Considerations

### CORS in Production
```python
# ❌ NOT for production
allow_origins=["*"]

# ✅ Recommended for production
allow_origins=[
    "https://yourdomain.com",
    "https://app.yourdomain.com"
]
```

### Data Validation
- Ensure events contain only serializable data
- Validate timestamp format
- Sanitize any user-provided data in events

### Rate Limiting
- Consider adding rate limiting to `/events` endpoint
- Set maximum concurrent connections if needed

### HTTPS Requirement
- Use HTTPS in production
- SSE works with HTTPS, but ensure proper SSL certificates

## Related Files

- `src/app/main.py` - CORS setup
- `src/app/api/routes/events.py` - SSE implementation
- `src/app/core/config.py` - Configuration (heartbeat interval)
- `app-qg-front/hooks/useSSE.ts` - Frontend hook
- `app-qg-front/app/events/page.tsx` - Demo page

## References

- [FastAPI CORS](https://fastapi.tiangolo.com/tutorial/cors/)
- [FastAPI StreamingResponse](https://fastapi.tiangolo.com/advanced/streaming-response/)
- [Server-Sent Events Specification](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

## Changelog

### v0.1.0 - Initial CORS & SSE Setup

**Added:**
- CORSMiddleware configuration in `main.py`
- Support for SSE connections from frontend
- Logging for connection events

**Verified:**
- `/events` endpoint working correctly
- Heartbeat events streaming properly
- CORS headers properly set

---

**Last Updated:** December 10, 2025
**Status:** Working ✅
