# Resource and Large Data Streaming: Detailed Study Notes

## Overview
Resource and large data streaming focuses on efficiently handling high-volume or large payloads (e.g., files, database results) without loading everything into memory. This prevents OutOfMemory errors, reduces latency, and supports use cases like downloads, exports, or real-time data feeds.

Key goals:
- **Memory Efficiency**: Yield data in chunks.
- **Scalability**: Handle large/infinite datasets.
- **Responsiveness**: Start sending data early.
- **Robustness**: Support resumes, backpressure, and interruptions.

In FastAPI/Starlette, `StreamingResponse` excels here by accepting generators that read/produce data incrementally.

Common challenges:
- Blocking I/O (use async where possible).
- Client speed variability (backpressure).
- Partial requests (range headers).

## File Streaming
### Definition
Stream file contents directly from disk in chunks, avoiding full load into memory. Ideal for large files (GBs+).

### Benefits
- Low memory footprint.
- Supports downloads/resumes.
- Efficient for static assets.

### Implementation
- Open file in binary mode.
- Yield fixed-size chunks (e.g., 8KB-1MB).
- Set appropriate `media_type` and headers (e.g., `Content-Disposition` for downloads).

### Text-based Diagram (File Streaming Flow)
```
Client Request --> FastAPI Endpoint --> StreamingResponse
                                             |
                                             v
                                      Generator opens file
                                             |
                                             v
                                 Read chunk (e.g., 8192 bytes)
                                             |
                                             v
                                        Yield chunk --> HTTP chunked
                                             |
                                       Loop until EOF
```

### Code Example: Basic File Streaming
```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import os

app = FastAPI()

def file_generator(path: str, chunk_size: int = 8192):
    with open(path, "rb") as f:
        while chunk := f.read(chunk_size):  # Read until empty
            yield chunk

@app.get("/download/{filename}")
async def stream_file(filename: str):
    file_path = f"./files/{filename}"  # Secure path handling in prod!
    if not os.path.exists(file_path):
        return {"error": "File not found"}
    
    headers = {"Content-Disposition": f"attachment; filename={filename}"}
    return StreamingResponse(
        file_generator(file_path),
        media_type="application/octet-stream",
        headers=headers
    )
```
- Test: Large file downloads progressively (e.g., via curl or browser).
- `chunk_size`: Balance CPU vs network (too small = overhead; too large = memory).

## File Proxies and Range Requests
### Definition
**File Proxies**: Stream files from remote URLs or storage (e.g., S3) without downloading fully server-side.
**Range Requests**: HTTP feature (RFC 7233) allowing partial content via `Range` header (e.g., for resumes, video seeking).

### Key Headers
- Client: `Range: bytes=0-1023` (request first 1KB).
- Server: 
  - `206 Partial Content` status.
  - `Accept-Ranges: bytes`.
  - `Content-Range: bytes 0-1023/5242880`.

### Benefits
- Resumable downloads.
- Video/audio seeking.
- Bandwidth savings.

### Text-based Diagram (Range Request Flow)
```
Client (with Range header) --> Server checks Range
                                     |
                     Valid? -- No --> 416 Range Not Satisfiable
                                     |
                                    Yes
                                     |
                                     v
                              Seek file to start offset
                                     |
                                     v
                          Yield bytes from start to end
                                     |
                                     v
                           Respond 206 + Content-Range
```

### Code Example: Range-Aware File Streaming
```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import StreamingResponse
import os

app = FastAPI()

def ranged_file_generator(file_path: str, start: int, end: int | None, chunk_size=8192):
    with open(file_path, "rb") as f:
        f.seek(start)
        remaining = (end - start + 1) if end else None
        while True:
            if remaining is not None and remaining <= 0:
                break
            read_size = min(chunk_size, remaining or chunk_size)
            chunk = f.read(read_size)
            if not chunk:
                break
            yield chunk
            if remaining:
                remaining -= len(chunk)

@app.get("/range-download/{filename}")
async def range_stream_file(request: Request, filename: str):
    file_path = f"./files/{filename}"
    if not os.path.exists(file_path):
        raise HTTPException(404)
    
    file_size = os.path.getsize(file_path)
    range_header = request.headers.get("Range")
    
    if not range_header:
        # Full file
        headers = {
            "Accept-Ranges": "bytes",
            "Content-Length": str(file_size)
        }
        return StreamingResponse(file_generator(file_path), media_type="video/mp4", headers=headers)
    
    # Parse Range: bytes=start-end
    try:
        range_str = range_header.split("=")[1]
        start, end = range_str.split("-")
        start = int(start)
        end = int(end) if end else file_size - 1
        if start >= file_size or end >= file_size or start > end:
            raise ValueError
    except:
        raise HTTPException(416, headers={"Content-Range": f"bytes */{file_size}"})
    
    headers = {
        "Content-Range": f"bytes {start}-{end}/{file_size}",
        "Accept-Ranges": "bytes",
        "Content-Length": str(end - start + 1)
    }
    return StreamingResponse(
        ranged_file_generator(file_path, start, end),
        status_code=206,
        media_type="video/mp4",
        headers=headers
    )
```
- Supports video players (e.g., <video> tag seeking).

## Database Cursors for Streaming Query Results
### Definition
Use DB cursors to fetch rows incrementally (server-side cursors in PostgreSQL/asyncpg). Avoid loading full result set.

### Benefits
- Stream large query results (e.g., exports).
- Low memory for millions of rows.

### Code Example: Streaming PostgreSQL Results (asyncpg)
```python
import asyncpg
from fastapi import FastAPI
import json

app = FastAPI()

async def db_stream_generator(conn):
    async with conn.transaction():
        cursor = conn.cursor("SELECT id, name FROM large_table", prefetch=1000)  # Server-side cursor
        async for row in cursor:
            yield json.dumps({"id": row[0], "name": row[1]}) + "\n"
            # await asyncio.sleep(0)  # Yield control if needed

@app.get("/db-stream")
async def stream_db():
    conn = await asyncpg.connect("postgresql://user:pass@localhost/db")
    try:
        return StreamingResponse(db_stream_generator(conn), media_type="application/x-ndjson")
    finally:
        await conn.close()
```
- Uses NDJSON format for line-by-line JSON.

## Queues and Buffered Streaming
### Definition
Use queues (e.g., `asyncio.Queue`) for producer-consumer patterns: Producer generates data asynchronously, consumer yields from queue.

### Use Cases
- Decouple slow production (e.g., API calls) from streaming.
- Buffering for bursty data.

### Code Example: Queue-Based Streaming
```python
import asyncio
from fastapi import FastAPI

app = FastAPI()

async def producer(queue: asyncio.Queue):
    for i in range(20):
        await asyncio.sleep(0.5)  # Simulate work
        await queue.put(f"Data {i}\n")
    await queue.put(None)  # Sentinel to end

async def queue_stream():
    queue = asyncio.Queue()
    asyncio.create_task(producer(queue))  # Start producer
    while True:
        item = await queue.get()
        if item is None:
            break
        yield item

@app.get("/queue-stream")
async def stream_queue():
    return StreamingResponse(queue_stream(), media_type="text/plain")
```

## Backpressure Handling
### Definition
Backpressure occurs when producer outpaces consumer (e.g., slow client). Queues naturally apply backpressure via bounded size or flow control.

### Strategies
- Bounded queues (raise if full).
- Client-side buffering.
- In FastAPI: Starlette handles some via event loop.

### Enhancement to Queue Example
```python
queue = asyncio.Queue(maxsize=10)  # Bounded: Producer blocks if full
```
- Prevents unbounded memory growth.

These techniques enable robust large-scale streaming. Combine with async for production (e.g., async file reads with aiofiles). Test with large files/DBs to observe memory stability!