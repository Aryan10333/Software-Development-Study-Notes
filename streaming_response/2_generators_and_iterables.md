# Generators and Iterables: Detailed Study Notes

## Overview
Generators and iterables are fundamental Python concepts that power **StreamingResponse** in frameworks like FastAPI/Starlette. They allow lazy, on-demand data production, which is essential for streaming because data can be yielded in chunks without loading everything into memory.

- **Iterables**: Objects capable of returning their elements one at a time (e.g., lists, tuples, files).
- **Iterators**: Objects with `__next__()` that track state and produce values.
- **Generators**: Special iterators created via functions with `yield`, enabling paused/resumed execution.

In streaming contexts:
- The `content` parameter of `StreamingResponse` accepts an iterable (sync generator) or async iterable (async generator).
- This yields chunks (str/bytes) progressively, sent via HTTP chunked encoding.

Key advantage: Memory-efficient for large/infinite data streams (e.g., LLM tokens, file downloads).

## Normal Generators and Iterators
### Iterator Protocol
Python's iteration follows a protocol:
1. Calling `iter(obj)` returns an iterator (calls `__iter__()`).
2. Repeated `next(it)` calls `__next__()` until `StopIteration`.

Text-based Diagram (Iterator Lifecycle):
```
Object (Iterable) --> iter() --> Iterator Object
                                   |
                                   v
                             __next__() --> Value
                                   |
                         Raises StopIteration (end)
```

### Normal (Synchronous) Generators
- Defined with `def` and `yield`.
- Execution pauses at `yield`, resumes on next `next()`.
- State preserved (local variables retained between yields).

### Code Example: Basic Generator
```python
def count_up_to(n):
    i = 0
    while i < n:
        yield i  # Pauses here, returns i
        i += 1   # Resumes here

# Usage
gen = count_up_to(5)  # Creates generator object (no execution yet)
print(next(gen))  # 0 (executes until first yield)
print(next(gen))  # 1
for num in gen:   # Consumes remaining
    print(num)    # 2, 3, 4
```

### In StreamingResponse Context
```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import time

app = FastAPI()

def slow_generator():
    for i in range(10):
        time.sleep(1)  # Simulate work
        yield f"Chunk {i}\n"

@app.get("/stream")
def stream():
    return StreamingResponse(slow_generator(), media_type="text/plain")
```
- Client receives one chunk per second.
- Note: `time.sleep()` blocks the thread—avoid in production async apps.

## Async Generators
### Motivation
Sync generators block the thread during I/O (e.g., `time.sleep()`, API calls). Async generators use `async def` and `yield` with `await`, allowing non-blocking I/O via asyncio.

### Async Iterator Protocol
Similar but asynchronous:
- `aiter(obj)` → async iterator.
- `anext(it)` → awaits `__anext__()`.

Text-based Diagram (Async Generator Flow):
```
Async Generator Function --> Call --> Async Generator Object
                                           |
                                           v
                                    anext() / for await --> Awaits until yield
                                           |
                                           v
                                       Yield Value --> Resume after await
```

### Code Example: Async Generator
```python
import asyncio

async def async_count_up_to(n):
    i = 0
    while i < n:
        await asyncio.sleep(1)  # Non-blocking delay
        yield i
        i += 1

# Usage
async def consume():
    async for num in async_count_up_to(5):
        print(num)

# In FastAPI (async endpoint)
@app.get("/async-stream")
async def async_stream():
    return StreamingResponse(async_count_up_to(10), media_type="text/plain")
```
- Preferred for FastAPI—doesn't block event loop.
- Enables concurrent handling of multiple streams.

## Yield From Delegation
### Purpose
`yield from` delegates iteration to a sub-generator/iterable, simplifying chaining. It handles `StopIteration` automatically and allows two-way communication (e.g., `send()`, `throw()`).

Without `yield from` (pre-Python 3.3):
```python
def chain(a, b):
    for item in a:
        yield item
    for item in b:
        yield item
```

With `yield from`:
```python
def chain(a, b):
    yield from a  # Delegates fully
    yield from b
```

### Flow Diagram (Yield From Delegation)
```
Main Generator --> yield from --> Sub-Generator
                         |               |
                         v               v
                   Yield values <--- Resume/Pause
                         ^
                         | (Exceptions/Values propagated)
```

### Code Example
```python
def sub_generator():
    yield "Sub 1"
    yield "Sub 2"

def main_generator():
    yield "Main start"
    yield from sub_generator()  # Delegates
    yield "Main end"

# Usage
gen = main_generator()
for item in gen:
    print(item)
# Output: Main start, Sub 1, Sub 2, Main end
```

### In Streaming
Useful for composing streams (e.g., header + data + footer).

## File-Like Objects as Iterables
Files are iterables—reading line-by-line yields chunks efficiently.

### Code Example: Streaming a File
```python
def file_stream(path):
    with open(path, "rb") as f:
        for chunk in f:  # Iterates over file in chunks (buffer size)
            yield chunk

@app.get("/download")
def download():
    return StreamingResponse(file_stream("large_file.txt"), media_type="application/octet-stream")
```
- Memory-efficient: No full file load.
- `f` is iterable (implements `__iter__` returning lines/chunks).

Text-based Diagram (File Iteration):
```
open(file) --> File Object (Iterable)
                  |
                  v
             __iter__() --> Iterator (reads buffer)
                  |
                  v
             next() --> Chunk of bytes --> Yield
```

## Differences Between Sync and Async Generators
| Aspect              | Sync Generators                          | Async Generators                              |
|---------------------|------------------------------------------|-----------------------------------------------|
| Definition          | `def func(): yield ...`                  | `async def func(): yield ...`                 |
| Pausing             | `yield` (sync)                           | `yield` + `await` for I/O                     |
| Blocking            | Blocks thread (e.g., `time.sleep()`)     | Non-blocking (`await asyncio.sleep()`)        |
| Iteration           | `for x in gen:`                          | `async for x in agen:`                        |
| Next                | `next(gen)`                              | `await anext(agen)`                           |
| Use in FastAPI      | OK for CPU-bound, but blocks event loop  | Recommended—maintains concurrency             |
| Error Handling      | `StopIteration`                          | `StopAsyncIteration`                          |
| Performance         | Simpler, but can degrade scalability     | Better for I/O-bound streaming (e.g., APIs)   |

### When to Choose
- Sync: Simple, CPU-heavy chunk generation.
- Async: Network/DB I/O, high concurrency (e.g., multiple streams).

These concepts are the backbone of efficient streaming. Mastering generators enables robust, scalable StreamingResponse implementations, especially for real-time applications like SSE or LLM token streaming. Experiment in a REPL to see pausing/resuming behavior!