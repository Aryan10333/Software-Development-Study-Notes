# Error Handling & Reliability

In a distributed system, failure is inevitable. The network might blink, an external API might go down, or the database might lock. Celery provides robust tools to handle these failures gracefully.

### **Automatic Retries**

Instead of failing immediately, tasks can be configured to try again later.

#### **`autoretry_for` (The Modern Way)**

This is the cleanest way to handle expected exceptions. You define it directly in the decorator.

```python
import requests
from celery import shared_task

@shared_task(autoretry_for=(requests.exceptions.RequestException,), retry_kwargs={'max_retries': 5})
def fetch_url(url):
    # If this raises a RequestException, Celery auto-retries
    return requests.get(url)

```

#### **Exponential Backoff (`retry_backoff`)**

If a service is down, hammering it with retries every second will only make things worse. "Exponential Backoff" increases the wait time between each retry (e.g., 1s, 2s, 4s, 8s, 16s).

* **Configuration:** `retry_backoff=True` (or a number like `retry_backoff=60` to set the starting factor).
* **Jitter:** Celery also adds "jitter" (randomness) to the wait time so that if 100 tasks fail at once, they don't all retry at the exact same millisecond.

#### **Manual Retries (`self.retry`)**

For complex logic (e.g., "retry only if the error message contains 'timeout'"), you can trigger retries manually inside the function.

```python
@shared_task(bind=True) # bind=True gives access to 'self' (the task instance)
def process_payment(self, user_id):
    try:
        # payment logic
        pass
    except PaymentGatewayError as exc:
        # Retry in 10 seconds
        raise self.retry(exc=exc, countdown=10)

```

### **Timeouts (Preventing Zombies)**

Sometimes a task doesn't fail; it just hangs forever (e.g., a socket connection that never closes). This blocks the worker process indefinitely.

1. **`time_limit` (Hard Limit):**
* The OS sends a `SIGKILL` signal to the worker process if the time is exceeded.
* **Result:** The process is terminated immediately. No cleanup is possible.
* **Usage:** Safety net for "stuck" processes.


2. **`soft_time_limit` (Soft Limit):**
* The worker raises a `SoftTimeLimitExceeded` exception inside the Python code *before* the hard limit is reached.
* **Result:** You can catch this exception to clean up resources (close files, log errors) before the hard kill happens.


```python
@shared_task(time_limit=60, soft_time_limit=50)
def export_heavy_file():
    try:
        # heavy work
        pass
    except SoftTimeLimitExceeded:
        cleanup_temp_files()

```



### **Task Acknowledgement (Data Safety)**

This controls *when* the message is removed from Redis.

* **Late Acknowledgement (`acks_late=True`):**
* **Behavior:** The task is acknowledged (removed from Redis) only **after** execution is finished.
* **Pros:** "At-least-once" delivery. If the worker crashes mid-task, the task remains in Redis and will be picked up by another worker.
* **Cons:** Tasks must be **Idempotent** (safe to run twice). If the worker crashes *after* charging a credit card but *before* sending the ACK, the next worker might charge the card again.


* **Early Acknowledgement (Default):**
* **Behavior:** The task is acknowledged as soon as the worker receives it.
* **Pros:** No risk of executing twice.
* **Cons:** If the worker crashes, the task is lost forever.



### **Dead Letter Queues (The Graveyard)**

What happens if a task fails 5 times, and you've exhausted your `max_retries`?

* **The Problem:** By default, Celery discards the task. It's just gone.
* **The Solution:** Route failed tasks to a separate queue (a "Dead Letter Exchange") so you can inspect them later.
* **Implementation:** You configure `task_acks_on_failure_or_timeout=False` and set up routing rules such that when a task permanently fails, it is pushed to a `failed_tasks` queue. You can then write scripts to inspect this queue and decide whether to fix the data and re-run them or delete them.