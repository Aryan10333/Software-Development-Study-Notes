# Streaming Data Formats: Detailed Study Notes

## Overview
Streaming data formats define how chunked data is structured and interpreted in a `StreamingResponse`. The format influences client parsing, efficiency, and use cases. Proper formatting ensures clients (browsers, curl, or custom apps) can process partial data meaningfully.

Key considerations:
- **Text vs Binary**: Text is human-readable and default for most streaming (e.g., SSE, tokens); binary for files/media.
- **Line-Based vs Raw**: Many formats are newline-delimited for easy parsing.
- **Progressive Parsing**: Formats like token streams or NDJSON allow clients to handle data incrementally without waiting for the end.
- **Common in LLMs**: Token-by-token streaming is popular for real-time AI responses.

Choice of format depends on:
- Client capabilities (e.g., browser EventSource for SSE).
- Data structure (JSON objects vs raw text).
- Need for metadata (e.g., events in SSE).

## Token Streams (Character-by-Character or Word-by-Word)
### Definition
Token streams send LLM (or generated) output incrementally—either as individual characters, words, or tokens. This creates a "typing" effect, improving perceived responsiveness.

### Types
- **Character-by-Character**: Yield single chars or small strings—fine-grained but chatty.
- **Word/Token-by-Word**: Yield complete words or model tokens—balances smoothness and efficiency.
- **Common in LLMs**: Providers like OpenAI/Anthropic stream `delta` content (partial and open source proxies mimic this).

### Advantages
- Real-time display (e.g., ChatGPT-style streaming).
- Early user feedback.

### Text-based Diagram (Token Stream Flow)
```
Server Generator                  Client
      |                                 |
      |-- yield "The " ---------------->|
      |-- yield "quick " --------------->|
      |-- yield "brown " --------------->|
      |-- yield "fox" ----------------->|
      |                                 |
      (Client appends progressively: "The quick brown fox")
```

### Code Example: Simple Token Stream
```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

async def token_stream():
    sentence = "The quick brown fox jumps over the lazy dog.".split()
    for word in sentence:
        yield word + " "  # Word-by-word
        await asyncio.sleep(0.2)  # Simulate token delay
    
    # Finer-grained: character-by-character alternative
    # for char in "Streaming chars...":
    #     yield char
    #     await asyncio.sleep(0.05)

@app.get("/token-stream")
async def stream_tokens():
    return StreamingResponse(token_stream(), media_type="text/plain")
```
- Client sees words appear progressively (test with curl or browser).
- For LLMs: Proxy OpenAI's stream and yield `choice.delta.content`.

## NDJSON (Newline Delimited JSON)
### Definition
**NDJSON** (Newline Delimited JSON, aka JSON Lines or JSON Streaming) is a format where each line is a complete JSON object, separated by newlines (`\n`). No commas or arrays—ideal for streaming append-only logs or results.

### Structure
- Each yielded chunk ends with `\n`.
- Client parses line-by-line (no need to wait for full response).

### Text-based Diagram (NDJSON Stream)
```
{"id": 1, "name": "Alice"}\n
{"id": 2, "name": "Bob"}\n
{"id": 3, "name": "Charlie"}\n
(Each line = valid JSON; no outer array)
```

### Advantages
- Robust: Partial streams are still valid (crash-tolerant).
- Efficient parsing (line-based).
- Standard for logs (e.g., Docker, ELK).

### Code Example: NDJSON Stream
```python
import json
import asyncio

async def ndjson_stream():
    data = [
        {"id": 1, "value": "First"},
        {"id": 2, "value": "Second"},
        {"id": 3, "value": "Third"}
    ]
    for item in data:
        yield json.dumps(item) + "\n"
        await asyncio.sleep(1)

@app.get("/ndjson")
async def stream_ndjson():
    return StreamingResponse(ndjson_stream(), media_type="application/x-ndjson")
```
- Client can split on `\n` and `json.loads()` each line.
- Useful for streaming database results or API batches.

## Streaming Dummy Tokens (yield "token\n")
### Purpose
For testing/debugging streaming endpoints, yield simple delimited tokens (e.g., "token\n"). Mimics real token streams without complex logic.

### Pattern
- Yield fixed strings with newline delimiters.
- Clients parse by splitting on `\n`.

### Code Example: Dummy Token Stream
```python
async def dummy_tokens():
    tokens = ["Hello", ", ", "world", "!", " This", " is", " a", " test", "."]
    for token in tokens:
        yield token + "\n"  # Explicit delimiter
        await asyncio.sleep(0.5)

@app.get("/dummy-tokens")
async def stream_dummy():
    return StreamingResponse(dummy_tokens(), media_type="text/plain")
```
- Output: One token per line, appearing over time.
- Common in tutorials to demonstrate progressive rendering.

## Binary vs Text Streaming
### Text Streaming
- Yield `str` objects (Python auto-encodes to bytes with UTF-8).
- Suitable for human-readable data (SSE, tokens, JSON).
- media_type: "text/plain", "text/event-stream", etc.

### Binary Streaming
- Yield `bytes` objects.
- For files, images, video, or compressed data.
- No encoding overhead; direct byte transfer.

### Comparison Table
| Aspect             | Text Streaming                  | Binary Streaming                  |
|--------------------|---------------------------------|-----------------------------------|
| Yield Type         | str                             | bytes                             |
| Encoding           | UTF-8 (automatic)               | Manual (none needed)              |
| Use Cases          | Tokens, SSE, logs               | Files, media, archives            |
| Parsing            | Easy (split lines)              | Application-specific              |
| media_type         | text/*                          | application/octet-stream, etc.    |

### Code Example: Binary File Stream
```python
def binary_file_stream(path="large_video.mp4"):
    with open(path, "rb") as f:
        while chunk := f.read(8192):  # 8KB chunks
            yield chunk  # bytes

@app.get("/binary-stream")
def stream_binary():
    return StreamingResponse(binary_file_stream(), media_type="application/octet-stream")
```
- Efficient for large files—supports range requests with custom logic.

## Compression in Streaming Responses
### Overview
Compress chunks on-the-fly (e.g., gzip) to reduce bandwidth. Clients must support decompression (most browsers do via `Accept-Encoding: gzip`).

### Approaches
- **Manual Chunk Compression**: Compress each yield (advanced).
- **Server-Level**: Use middleware (e.g., GzipMiddleware in Starlette) for automatic compression.
- **Trade-offs**: Adds CPU overhead; not always beneficial for small/text data.

### Text-based Diagram (Gzip Compression Flow)
```
Generator --> Compress Chunk (gzip) --> Yield bytes --> Client decompresses
```

### Code Example: Manual Gzip (Advanced)
```python
import gzip
from io import BytesIO

async def compressed_stream():
    text_chunks = ["Large text chunk 1\n", "Large text chunk 2\n"]
    for chunk in text_chunks:
        buffer = BytesIO()
        with gzip.GzipFile(fileobj=buffer, mode="wb") as gz:
            gz.write(chunk.encode())
        yield buffer.getvalue()  # Compressed bytes
        await asyncio.sleep(1)

@app.get("/gzip-stream")
async def stream_gzip():
    headers = {"Content-Encoding": "gzip"}
    return StreamingResponse(compressed_stream(), media_type="text/plain", headers=headers)
```
- Client auto-decompresses if header set.
- Prefer middleware for production.

These formats enable flexible, efficient streaming. Token streams and NDJSON are especially powerful for real-time AI/LLM applications. Experiment with different media_types to see client behavior!