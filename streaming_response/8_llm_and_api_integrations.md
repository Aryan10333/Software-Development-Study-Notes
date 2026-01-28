# LLM and API Integrations: Detailed Study Notes

## Overview
Integrating Large Language Models (LLMs) with `StreamingResponse` enables real-time, interactive applications like chatbots, assistants, and content generators. The core idea is to **proxy** requests to third-party LLM APIs (e.g., OpenAI, Anthropic, Grok) and stream their token-by-token responses back to the client.

### Why Streaming for LLMs?
- **Improved UX**: Tokens appear progressively (typing effect), reducing perceived latency.
- **Efficiency**: No need to wait for full completion; clients process partial output.
- **Cost/Performance**: Early disconnect detection stops expensive API calls.
- **Common Providers**:
  - OpenAI (GPT series): Native streaming support.
  - Anthropic (Claude): Streaming via `stream=True`.
  - Grok/xAI: API streaming (check docs for specifics).
  - Open-source (e.g., via Ollama, Hugging Face): Custom streaming setups.

Challenges:
- Handling variable token rates and errors.
- Formatting (raw text, SSE, NDJSON).
- Security (auth, input validation).
- Rate limiting to prevent abuse/costs.

## OpenAI/ChatGPT Streaming Integration
### Mechanism
OpenAI's API supports streaming with `stream=True`. Responses arrive as Server-Sent Events (SSE) chunks containing `delta` (new tokens).

### Key Concepts
- Use `AsyncOpenAI` for non-blocking integration in FastAPI.
- Iterate over the stream asynchronously.
- Yield `delta.content` (or empty string if none).
- Handle finish reasons and errors.

### Text-based Diagram (OpenAI Streaming Flow)
```
Client POST /chat --> FastAPI Endpoint (with prompt)
                          |
                          v
                AsyncOpenAI.chat.completions.create(stream=True)
                          |
                          v
               Async for chunk in response stream
                          |
                 Extract delta.content
                          |
                 Yield delta --> StreamingResponse
                          |
               Check disconnect --> Cancel if needed
```

### Code Example: Basic OpenAI Streaming Proxy
```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import StreamingResponse
from openai import AsyncOpenAI
import asyncio

app = FastAPI()

client = AsyncOpenAI(api_key="your-openai-api-key")  # Use env var in prod

async def openai_stream(request: Request, prompt: str = "Tell me a story"):
    try:
        stream = await client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": prompt}],
            stream=True,
            temperature=0.7,
            max_tokens=500
        )
        
        async for chunk in stream:
            if await request.is_disconnected():
                print("Client disconnected - stopping OpenAI stream")
                break  # Stops further API consumption
            
            delta = chunk.choices[0].delta.content or ""
            if delta:
                yield delta  # Raw token stream
        
        # Optional: Yield end marker
        if not await request.is_disconnected():
            yield "\n[Done]"
    
    except Exception as e:
        yield f"\n[Error: {str(e)}]"

@app.get("/openai-stream")  # Or POST with body for real apps
async def openai_chat(request: Request, prompt: str = "Hello"):
    return StreamingResponse(
        openai_stream(request, prompt),
        media_type="text/plain"  # Or "text/event-stream" for SSE
    )
```
- **Test**: `curl "http://localhost:8000/openai-stream?prompt=Explain%20quantum%20computing"`
- **Enhancements**: Use POST with JSON body for messages/history.

## Proxying Third-Party Streaming APIs
### Definition
Act as a middleman: Client → Your FastAPI → LLM Provider → Stream back.
- Hides API keys.
- Adds logging, filtering, or custom formatting.
- Supports multiple providers.

### Benefits
- Unified endpoint for clients.
- Censor/filter content.
- Analytics and monitoring.

### General Pattern
- Forward user input.
- Stream provider response.
- Transform if needed (e.g., add prefixes).

### Code Example: Generic Proxy with SSE Formatting
```python
async def proxy_llm_stream(request: Request, provider="openai"):
    prompt = "Why is the sky blue?"  # From request body in prod
    
    if provider == "openai":
        stream = await client.chat.completions.create(...)  # As above
    
    # Transform to SSE (common for clients)
    async for chunk in stream:
        if await request.is_disconnected():
            break
        delta = chunk.choices[0].delta.content or ""
        if delta:
            yield f"data: {delta}\n\n"  # SSE format
    
    yield "event: done\ndata: [END]\n\n"

@app.post("/llm-proxy")
async def llm_proxy(request: Request):
    body = await request.json()  # {"prompt": "...", "provider": "openai"}
    return StreamingResponse(
        proxy_llm_stream(request, body.get("provider")),
        media_type="text/event-stream"
    )
```

## Handling Token Streams from LLMs
### Strategies
- **Raw Deltas**: Yield directly (fastest, simplest).
- **Buffered** SSE/NDJSON: Structured for client parsing.
- **Buffering**: Accumulate small deltas to reduce chunks.
- **Tool Calls/Functions**: Parse structured output incrementally.

### Text-based Diagram (Token Handling)
```
LLM API Chunk --> delta.content (" The")
                  |
                  v
            Buffer or yield immediately
                  |
                  v
         Format (raw / SSE / JSON)
                  |
                  v
            Yield to client
```

### Code Example: Buffered Token Stream (Smoother Output)
```python
async def buffered_token_stream(request: Request):
    buffer = ""
    async for chunk in openai_stream:  # From API
        if await request.is_disconnected():
            break
        buffer += chunk.choices[0].delta.content or ""
        
        # Yield on word boundaries or fixed size
        if " " in buffer or "\n" in buffer or len(buffer) > 20:
            yield buffer
            buffer = ""
    
    if buffer:  # Flush remaining
        yield buffer
```

## Combining with Authentication and Rate Limiting
### Authentication
- Use FastAPI dependencies (e.g., API keys, OAuth).
- Validate before proxying.

### Rate Limiting
- Use `slowapi` or Redis-based limiters to prevent abuse.
- Per-user/IP limits to control costs.

### Code Example: With Simple Auth and Rate Limiting
```python
from fastapi import Depends, Header
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

async def verify_api_key(x_api_key: str = Header(...)):
    if x_api_key != "secret-key":  # Validate properly
        raise HTTPException(401, "Invalid API key")

@app.post("/secure-llm-stream")
@limiter.limit("5/minute")  # Example limit
async def secure_stream(request: Request, _: str = Depends(verify_api_key)):
    body = await request.json()
    return StreamingResponse(openai_stream(request, body["prompt"]), media_type="text/plain")
```
- **slowapi**: Install via pip; integrates seamlessly.
- **Production**: Use Redis for distributed limiting.

## Best Practices
- **Error Propagation**: Yield provider errors as final chunks.
- **Timeouts**: Set API timeouts; cancel on disconnect.
- **Content Filtering**: Scan for policy violations before yielding.
- **Monitoring**: Log token usage, costs.
- **Fallbacks**: Switch models on errors.

These integrations power modern AI apps. Start with OpenAI proxy, add features iteratively. Always secure keys and monitor usage!