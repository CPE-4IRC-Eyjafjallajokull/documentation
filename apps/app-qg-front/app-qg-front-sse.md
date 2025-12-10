# SSE Integration - app-qg-front

## Overview

Server-Sent Events (SSE) integration for real-time, one-way communication from the API to the frontend. The frontend establishes a persistent connection to the API's `/events` endpoint and displays received events in real-time.

## Architecture

### What is SSE?

SSE (Server-Sent Events) is a standard for server-to-client push notifications using HTTP. Unlike WebSockets, it's unidirectional (server → client) and uses standard HTTP, making it simpler for one-way communication patterns.

**Benefits:**
- Simple HTTP-based (no special WebSocket protocol needed)
- Automatic reconnection on disconnection
- Built-in EventSource API in browsers
- Perfect for notifications, live updates, and dashboards

## Implementation

### 1. Hook: `hooks/useSSE.ts`

Custom React hook that encapsulates EventSource logic and state management.

```typescript
export function useSSE(url: string) {
  const [data, setData] = useState<SSEEvent | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // Returns: { data, isConnected, error }
}
```

**Features:**
- Handles EventSource lifecycle (open, message, error)
- Parses JSON data from events
- Auto-cleanup on unmount
- Error state management

**Usage:**
```typescript
const { data, isConnected, error } = useSSE('http://localhost:8000/events');
```

### 2. Page: `app/events/page.tsx`

Demo page showcasing real-time SSE event streaming.

**Features:**
- Live status indicator (Connected/Disconnected)
- Events log with timestamps
- Error display
- Configurable API URL via environment variables

**Key Points:**
- Uses `"use client"` directive (Client Component)
- Updates event list as new events arrive
- Timestamps formatted to local time
- Responsive UI with Tailwind CSS

### 3. Configuration: `.env.local`

```env
NEXT_PUBLIC_API_URL=http://localhost:8000
```

**Note:** The `NEXT_PUBLIC_` prefix makes this variable accessible in the browser. Change the URL if deploying to production.

## API Integration

### Backend: `app-qg-api`

The API provides the SSE stream at the `/events` endpoint:

```python
@router.get("/events")
async def events_stream():
    """Server-sent events stream for one-way, real-time updates."""
    async def event_source():
        yield f"data: {json.dumps({'event': 'connected', 'timestamp': ...})}\n\n"
        while True:
            payload = {"event": "heartbeat", "timestamp": ...}
            yield f"data: {json.dumps(payload)}\n\n"
            await asyncio.sleep(settings.events_ping_interval_seconds)
```

**Event Format:**
- `data: {JSON payload}\n\n`
- Events include `event` type and `timestamp`
- Heartbeats sent every 5 seconds (configurable)

### CORS Setup

The API includes CORS middleware to allow frontend requests:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

⚠️ **Security Note:** For production, restrict `allow_origins` to specific domains instead of `["*"]`.

## Setup Instructions

### 1. Prerequisites
- Node.js 18+
- Python 3.12+ (for API)
- npm or yarn

### 2. Install Frontend Dependencies
```bash
cd app-qg-front
npm install
```

### 3. Install API Dependencies
```bash
cd app-qg-api
pip install -e .
```

### 4. Start the API
```bash
cd app-qg-api
python -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

### 5. Start the Frontend
```bash
cd app-qg-front
npm run dev
```

### 6. Access the Demo
Open http://localhost:3000/events in your browser.

## File Structure

```
app-qg-front/
├── hooks/
│   └── useSSE.ts           # Custom React hook for SSE
├── app/
│   ├── events/
│   │   └── page.tsx        # Demo page
│   ├── layout.tsx
│   └── page.tsx
├── .env.local              # API URL configuration
└── package.json
```

## Usage Examples

### Basic Usage in a Component

```typescript
"use client";

import { useSSE } from "@/hooks/useSSE";

export default function Dashboard() {
  const apiUrl = process.env.NEXT_PUBLIC_API_URL || "http://localhost:8000";
  const { data, isConnected, error } = useSSE(`${apiUrl}/events`);

  return (
    <div>
      <p>Status: {isConnected ? "Connected" : "Disconnected"}</p>
      {error && <p>Error: {error}</p>}
      {data && <p>Latest event: {data.event}</p>}
    </div>
  );
}
```

### Tracking All Events

```typescript
const [events, setEvents] = useState<SSEEvent[]>([]);
const { data } = useSSE(url);

if (data && !events.find(e => e.timestamp === data.timestamp)) {
  setEvents(prev => [...prev, data]);
}
```

## Testing

### Manual Testing

1. **Start both servers** (API on 8000, Frontend on 3000)
2. **Open** http://localhost:3000/events
3. **Expected behavior:**
   - Status should show "✓ Connected"
   - Events log should fill with "connected" event
   - Heartbeats should appear every 5 seconds

### Browser DevTools Testing

1. Open **DevTools** (F12)
2. Go to **Network** tab
3. Filter for type "fetch" or "xhr"
4. Look for the `/events` request
5. Observe the **Streaming** tab showing incoming data

### API Testing with curl

```bash
curl -N http://localhost:8000/events
```

The `-N` flag disables buffering to see streaming data in real-time.

## Troubleshooting

### No events appearing in the frontend

**Problem:** Connection shows "Connected" but no events appear.

**Solutions:**
1. Check browser DevTools Console for JavaScript errors
2. Verify API is running: `curl http://localhost:8000/health`
3. Check Network tab: ensure `/events` request shows status 200
4. Verify `.env.local` has correct `NEXT_PUBLIC_API_URL`

### Connection fails immediately

**Problem:** Status shows "Disconnected" or error message appears.

**Solutions:**
1. Verify CORS is enabled in API (`main.py`)
2. Check API logs for errors
3. Ensure API URL in `.env.local` is correct and accessible
4. Try direct request: `curl http://localhost:8000/events`

### Events disconnect after a few seconds

**Problem:** Connection drops after brief period.

**Solutions:**
1. Check API logs for exceptions
2. Verify RabbitMQ/MongoDB connections if used
3. Check browser console for network errors
4. Ensure no firewall/proxy is blocking streaming

## Production Considerations

### Environment Variables
```env
NEXT_PUBLIC_API_URL=https://api.example.com
```

### Security
- Change CORS `allow_origins` from `["*"]` to specific domains
- Use HTTPS in production
- Validate/sanitize data received from API

### Performance
- Consider filtering events on backend to reduce volume
- Implement client-side event buffering for large volumes
- Add event type filtering in the hook

### Deployment
- Ensure API and frontend are on same domain or properly configured for CORS
- Use environment-specific configuration
- Consider API rate limiting for event stream

## API Configuration

Edit `app-qg-api/src/app/core/config.py` to adjust:

```python
events_ping_interval_seconds: int = 5  # Heartbeat interval
```

## References

- [MDN: Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [EventSource API](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)
- [FastAPI Streaming Response](https://fastapi.tiangolo.com/advanced/streaming-response/)
- [Next.js Environment Variables](https://nextjs.org/docs/basic-features/environment-variables)

## Changelog

### v0.1.0 - Initial SSE Implementation

**Added:**
- `useSSE` hook for EventSource abstraction
- `/events` demo page with real-time updates
- CORS configuration in API
- `.env.local` for API URL configuration
- Complete documentation

**Features:**
- Real-time event streaming
- Connection status indicator
- Error handling and display
- Timestamp tracking

---

**Last Updated:** December 10, 2025
**Status:** Working ✅
