# Server-Sent Events (SSE): Detailed Study Notes

## SSE Protocol Basics
**Server-Sent Events (SSE)** is a standard HTTP-based technology that allows a server to push real-time updates to a client over a single, long-lived HTTP connection. It is built on top of HTTP and uses the `text/event-stream` media type. SSE is **unidirectional**—the server pushes data to the client, but the client cannot send data back over the same connection (use separate HTTP requests for that).

### Key Features
- **Simple Setup**: Uses standard HTTP—no need for special protocols like WebSockets.
- **Automatic Reconnection**: Browsers handle reconnections with exponential backoff.
- **Event Identification**: Supports event IDs for resuming from last known point.
- **Text-Based**: Payload is plain text (UTF-8), easy to debug.
- **Built on StreamingResponse**: In FastAPI/Starlette, SSE is implemented via `StreamingResponse` with `media_type="text/event-stream"`.

### How SSE Works
1. Client initiates a GET request with `Accept: text/event-stream`.
2. Server responds with `Content-Type: text/event-stream` and `Cache-Control: no-cache`.
3. Connection stays open; server sends formatted "events" as they occur.
4. Client receives and processes events incrementally.

Text-based Diagram (SSE Connection Architecture):
```
Client (Browser)                  Server
     |                                 |
     |-- GET /events ----------------->|
     |                                 |
     |<-- 200 OK (text/event-stream)---|
     |                                 |
     |<---------- Event 1 -------------|
     |<---------- Event 2 -------------|
     |<---------- Event 3 -------------|
     |             ...                 |
     (Connection remains open for pushes)
```

- Unlike polling (repeated requests), SSE is efficient—one connection.
- Unlike WebSockets, SSE is simpler (no bidirectional, no framing overhead).

## SSE Formatting and Event Structure
SSE messages are sent as plain text with specific prefixes. Each event is separated by a blank line (`\n\n`). Lines starting with `:` are comments (ignored by clients, used for heartbeats).

### Standard Fields
- `event:` — Custom event name (e.g., `event: message`).
- `data:` — Payload (multi-line allowed; each line prefixed with `data:`).
- `id:` — Event ID (for reconnection; client resumes with `Last-Event-ID` header).
- `retry:` — Reconnection time in ms (overrides default ~3s).
- `:` — Comment (for keep-alive/heartbeats).

### Event Format Example (Text-based Diagram)
```
: This is a comment (heartbeat)

event: update
id: 123
data: {"temperature": 25}
retry: 5000

data: First line
data: Second line
id: 124

(Blank line \n\n ends the event)
```

- Multi-line data: Last `data:` line doesn't need trailing newline in payload.
- Full event ends with double newline.

### Raw Stream Example
```
data: Starting stream...

event: message
data: Hello, world!

: heartbeat

data: {"user": "Aryan", "msg": "Hi from Mumbai!"}
id: 1

```

## SSE Heartbeats and Keep-Alive
Connections can timeout (e.g., proxies close idle connections after ~30-60s). **Heartbeats** prevent this by sending periodic comments.

### Why Needed
- Proxies/load balancers (e.g., NGINX) close idle connections.
- Ensures client detects disconnections faster.

### Implementation
Yield a comment line every 10-30 seconds:
```python
: keep-alive\n\n
```

Text-based Diagram (Heartbeat in Stream):
```
data: Event 1\n\n
: heartbeat\n\n
data: Event 2\n\n
: heartbeat\n\n
...
```

## Client-Side EventSource API
Browsers provide the native `EventSource` API for consuming SSE.

### Basic Client Code (JavaScript)
```html
<!DOCTYPE html>
<script>
const evtSource = new EventSource("/sse-endpoint");

evtSource.onmessage = (event) => {
    console.log("Data:", event.data);  // Default "message" event
};

evtSource.addEventListener("update", (event) => {
    console.log("Custom update:", event.data);
});

evtSource.onerror = (err) => {
    console.error("SSE error:", err);
};

evtSource.onopen = () => {
    console.log("Connection opened");
};
</script>
```

- **Reconnection**: Automatic; sends `Last-Event-ID` header if `id:` was set.
- **Closing**: `evtSource.close()`.

## Advantages and Limitations of SSE
### Advantages
- Simple to implement (just text over HTTP).
- Built-in reconnection and event IDs.
- Lower overhead than WebSockets (no bidirectional framing).
- Firewall/proxy friendly (plain HTTP).
- Great for real-time feeds: stock prices, notifications, live logs, LLM token streaming.

### Limitations
- Unidirectional (server → client only).
- Text-only (no binary data; base64 encode if needed).
- Limited to ~6 connections per domain (browser limit for HTTP/1.1).
- No custom status codes or headers after initial response.
- HTTP/2 has better multiplexing, but SSE works best on HTTP/1.1 for long-lived connections.

Text-based Comparison (SSE vs WebSockets):
```
Feature          | SSE                  | WebSockets
-----------------|----------------------|------------------
Direction        | Server → Client      | Bidirectional
Complexity       | Low (text/events)    | Higher (framing/handshake)
Reconnection     | Built-in             | Manual
Binary Support   | No                   | Yes
Use Cases        | Feeds, updates       | Chat, games
```

## Code Example: SSE in FastAPI
A complete example with heartbeats, custom events, and infinite streaming.

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio
import json
from datetime import datetime

app = FastAPI()

async def event_stream():
    i = 0
    while True:
        # Heartbeat every 15 seconds
        if i % 15 == 0:
            yield ": heartbeat\n\n"
        
        # Custom event
        now = datetime.now().isoformat()
        payload = json.dumps({"time": now, "message": f"Update {i}"})
        yield f"event: update\ndata: {payload}\nid: {i}\n\n"
        
        i += 1
        await asyncio.sleep(1)  # Simulate real-time updates

@app.get("/sse")
async def sse_endpoint():
    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

### Explanation
- **media_type**: Essential for SSE.
- **Heartbeats**: Comment lines prevent timeouts.
- **Events**: Use `event:`, `data:`, `id:` for structure.
- **Infinite**: Runs forever until client disconnects.
- Test: Open in browser or use `curl http://localhost:8000/sse`.

### Client Test Snippet
```javascript
const source = new EventSource("http://localhost:8000/sse");
source.onmessage = (e) => console.log(e.data);  // Fallback
source.addEventListener("update", (e) => {
    const data = JSON.parse(e.data);
    console.log("Update:", data);
});
```

SSE is ideal for push notifications and progressive responses (e.g., streaming LLM tokens as `data:` lines). Combine with client disconnect detection for production robustness!