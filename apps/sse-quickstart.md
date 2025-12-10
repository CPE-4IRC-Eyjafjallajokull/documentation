# SSE Quick Start Guide

## 5-Minute Setup

### Prerequisites
- Node.js 18+
- Python 3.12+
- npm or yarn

### Step 1: API Setup
```bash
cd app-qg-api
pip install -e .
python -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

### Step 2: Frontend Setup (new terminal)
```bash
cd app-qg-front
npm install
npm run dev
```

### Step 3: Test
Open http://localhost:3000/events in your browser and watch the events stream in real-time!

## What You Should See

✅ **Status Badge:** Shows "✓ Connected" in green  
✅ **Events Log:** Displays incoming events with timestamps  
✅ **Heartbeats:** New events appear every 5 seconds  

## Architecture Overview

```
┌──────────────────┐
│  Browser         │
│  (localhost:3000)│
│  ┌────────────┐  │
│  │ useSSE     │  │ Hook manages EventSource
│  │ Hook       │  │
│  └──────┬─────┘  │
│         │        │
│    HTTP │ Stream │ (text/event-stream)
│    GET  │        │
└────────┼────────┘
         │
    ┌────▼─────────────────────┐
    │  API (localhost:8000)     │
    │  ┌──────────────────────┐ │
    │  │ GET /events          │ │
    │  │ ├─ connected event   │ │
    │  │ ├─ heartbeat every 5s│ │
    │  │ └─ ...               │ │
    │  └──────────────────────┘ │
    │  + CORS middleware         │
    └────────────────────────────┘
```

## File Locations

```
app-qg-front/
├── hooks/useSSE.ts          ← Hook implementation
├── app/events/page.tsx      ← Demo page
└── .env.local               ← API URL config

app-qg-api/
└── src/app/main.py          ← CORS setup
```

## Common Tasks

### Change API URL
Edit `.env.local`:
```env
NEXT_PUBLIC_API_URL=https://api.example.com
```

### Adjust Heartbeat Interval
Edit `app-qg-api/src/app/core/config.py`:
```python
events_ping_interval_seconds = 3  # 3 seconds instead of 5
```

### Use in Your Component
```typescript
"use client";

import { useSSE } from "@/hooks/useSSE";

export default function MyComponent() {
  const { data, isConnected, error } = useSSE(
    `${process.env.NEXT_PUBLIC_API_URL}/events`
  );

  return <div>Status: {isConnected ? "🟢" : "🔴"}</div>;
}
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| No events | Check API running: `curl http://localhost:8000/health` |
| Connection fails | Verify `.env.local` has correct API URL |
| CORS error | CORS middleware must be in `main.py` |
| Events drop | Check API logs for exceptions |

## Next Steps

1. **Test with real data** - Modify `/events` endpoint to emit actual events
2. **Create custom events** - Add event types for your use case
3. **Add event filtering** - Filter events on frontend or backend
4. **Deploy to production** - Update CORS origins and API URL

## Full Documentation

See:
- `documentation/apps/app-qg-front-sse.md` - Frontend details
- `documentation/apps/app-qg-api-sse.md` - API details

---

**Last Updated:** December 10, 2025
