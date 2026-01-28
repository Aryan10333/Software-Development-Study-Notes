## Advanced Worker Optimization

This section focuses on fine-tuning how workers consume tasks. Default settings work for general cases, but high-performance or specialized workloads require tweaking.

### **Prefetch Multiplier (`worker_prefetch_multiplier`)**

* **The Concept:**
To improve performance, a worker doesn't ask Redis for tasks one by one. It grabs a batch of tasks at once to reduce network latency.
* **Default:** `4` (The worker reserves 4 tasks per concurrent process).
* **Math:** If you have concurrency `-c 4` and prefetch `4`, the worker grabs 16 tasks immediately.


* **The Problem (The "Long Task" starvation):**
Imagine you have 1 heavy task (takes 1 hour) and 100 light tasks (take 1 second).
If Worker A grabs the heavy task + 3 light tasks (prefetch), those 3 light tasks are **stuck** in Worker A's buffer. Even if Worker B is idle, it cannot touch them because they are "Reserved" by Worker A.
* **The Solution:**
* **For Long Tasks:** Set `worker_prefetch_multiplier = 1` (often called "Fair Dispatch"). This forces the worker to only reserve one task at a time.
* **For Short Tasks:** Keep the default (4) or increase it to reduce network overhead.



### **Rate Limits**

You can restrict how often a task runs, which is critical when interacting with third-party APIs that charge by the request.

* **Mechanism:** Celery uses a "Token Bucket" algorithm.
* **Usage:**
```python
# Limit this task to 10 executions per minute across ALL workers
@app.task(rate_limit='10/m')
def scrape_twitter(user_id):
    pass

```


* **Note:** This does not pause the worker. If the limit is hit, the task goes back to the queue (or sleeps) until a "token" is available, freeing the worker to do other things in the meantime.

---

## Specific Canvas Primitives (Map & Starmap)

While `group()` runs independent tasks, `map` and `starmap` are syntactical sugar designed to process lists of data, mirroring Python's native functional tools.

### **`task.map()`**

* **Use Case:** You have a list of single arguments, and you want to run the *same* task for each item.
* **Syntax:**
```python
# Equivalent to: group(add.s(x) for x in [1, 2, 3])
# Note: 'add' task here must be partially applied or expect 1 arg

# Example: Verify a list of emails
verify_email.map(['a@x.com', 'b@x.com', 'c@x.com']).delay()

```



### **`task.starmap()`**

* **Use Case:** Your task takes multiple arguments, and you have a list of tuples (sets of arguments).
* **Syntax:**
```python
# Equivalent to: group(add.s(x, y) for x, y in [(2, 2), (4, 4)])

# Example: Add pairs of numbers
add.starmap([(2, 2), (4, 4), (8, 8)]).delay()

```



---

## Signals (The Event Hooks)

Signals allow you to "hook" into the task lifecycle to execute code automatically. This is cleaner than putting `try/finally` blocks inside every single task.

* **How it works:** You define a function and connect it to a specific signal (event).

### **Common Signals**

1. **`task_prerun`:** Runs before the task executes.
* *Usage:* Setting up logging context, measuring start time.


2. **`task_postrun`:** Runs after the task finishes (success or failure).
* *Usage:* Closing database connections to prevent leaks.


3. **`task_failure`:** Runs only on exception.
* *Usage:* Sending alerts to Slack/Sentry.



### **Example: Closing DB Connections**

```python
from celery.signals import task_postrun

@task_postrun.connect
def close_session(sender=None, headers=None, body=None, **kwargs):
    # This runs after EVERY task in the system
    db_session.remove()
    print(f"Cleaned up DB session for task {sender.name}")

```

---

## Integration Patterns

### **Django Integration**

Django is the most common partner for Celery.

* **Setup:** You create a `celery.py` file next to `wsgi.py`.
* **Config:** You use `app.config_from_object('django.conf:settings', namespace='CELERY')`. This allows you to put settings in `settings.py` with a `CELERY_` prefix (e.g., `CELERY_BROKER_URL`).
* **Auto-discovery:** The line `app.autodiscover_tasks()` automatically looks for a `tasks.py` file inside every installed Django app, so you don't have to manually register them.

### **Flask Integration**

Flask is trickier because it requires an "Application Context" (e.g., to access `current_app` or database extensions).

* **The Pattern:** You must subclass `celery.Task` to wrap the execution within the Flask app context.
```python
class ContextTask(celery.Task):
    def __call__(self, *args, **kwargs):
        with flask_app.app_context():
            return self.run(*args, **kwargs)

celery.Task = ContextTask

```



### **Standalone Scripts**

For simple scripts (no framework):

* Just instantiate `app = Celery(...)` in a file (e.g., `tasks.py`).
* Start the worker pointing to that module: `celery -A tasks worker`.

---

## Advanced Topics & Internals

### **Custom Task Classes**

Instead of using the default `@app.task`, you can create a base class to share functionality across many tasks.

* **Scenario:** You want every task to log its own execution time and handle a specific error type silently.
* **Implementation:**
```python
import celery

class BaseTask(celery.Task):
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        print(f"Task {task_id} failed: {exc}")
        # Custom alerting logic here

@app.task(base=BaseTask)  # Bind the custom behavior
def divide(x, y):
    return x / y

```



### **Chord Unlock Issues (The "Redis Trap")**

* **The Architecture:**
When you run a **Chord** (Group + Callback), Celery needs to know when the Group is finished.
Since Redis doesn't trigger events naturally, Celery starts a hidden task called `celery.chord_unlock`.
* **How it works:**
This task wakes up every second, checks if the group is done, and if not, retries itself.
* **The Issue:**
If your Redis is overloaded, or if the `chord_unlock` task keeps failing (e.g., due to strict visibility timeouts), the Callback will **never** run. The group finishes, but the "unlocker" dies, and the callback stays in limbo.
* **Fix:** Ensure your `visibility_timeout` is long enough and that your result backend (Redis) is not running out of memory.

### **Canvas Immortals**

* **Concept:** Standard Celery workflows exist only in memory (or loosely in Redis). If you have a chain `A -> B -> C` and the system crashes during `B`, the link to `C` might be lost if not carefully persisted.
* **Solution:** For truly "Immortal" workflows that can survive total system destruction and resume days later, standard Celery Canvas is sometimes insufficient. You might need to use **persistent state machines** stored in a database (e.g., tracking the workflow state in a Postgres table and launching the next step only upon DB confirmation). This is often a manual implementation pattern rather than a built-in Celery feature.


Yes, most of these were covered in **Section 6 (Error Handling)** and **Section 10 (Advanced Worker Optimization)**, but **Idempotent Design**, **Per-User Rate Limiting**, and **Backpressure** were only briefly touched upon or need more specific detail to be fully understood in a production context.

Here is the breakdown of what was covered and the **detailed study notes for the missing nuances**.

---

### **Status Check**

| Topic | Status | Where it was covered |
| --- | --- | --- |
| **soft_time_limit vs time_limit** | ✅ **Covered** | Section 6 (Timeouts). |
| **Retries with backoff** | ✅ **Covered** | Section 6 (Automatic Retries). |
| **Dead-letter queues (DLQ)** | ✅ **Covered** | Section 6 (Routing failed tasks). |
| **Idempotent task design** | ⚠️ **Partial** | Mentioned in "Late Acks", but needs design patterns. |
| **Rate limiting** | ⚠️ **Partial** | Global task limits covered in Section 10. **Per-user/org** limits were not. |
| **Backpressure** | ❌ **Missing** | Not explicitly covered. |

---


### **Rate Limiting (Per User / Per Org)**

The built-in `@app.task(rate_limit='10/m')` is **global**. It limits the *total* throughput of that task across all workers. It does **not** stop "User A" from hogging all the slots while "User B" waits.

* **How to implement Per-User Limits:**
You must implement this manually inside the task (usually at the start) or using a custom router.
* **Manual Token Bucket (Inside Task):**
```python
@shared_task(bind=True)
def scrape_for_user(self, user_id):
    # Check Redis for user's usage count in the last minute
    key = f"rate_limit:{user_id}"
    if redis.get(key) > 10:
        # Re-queue the task for later (delay 60s)
        raise self.retry(countdown=60)

    redis.incr(key)
    redis.expire(key, 60)

    # ... proceed with logic ...
    ```


```


### **Backpressure (Protecting the System)**

Backpressure is the mechanism of saying "I am too busy, stop sending more work." By default, Celery + Redis will happily accept 10 million tasks until Redis runs out of RAM (OOM).

* **Strategy 1: Queue Limits (Redis-side)**
* Redis Lists don't have hard limits by default. You have to monitor queue length.
* **Application-side check:** Before calling `.delay()`, check the queue size.
```python
queue_len = app.control.inspect().active() # Expensive!
# Better: use direct Redis client
queue_len = redis_client.llen("celery")

if queue_len > 10000:
    raise ServiceUnavailable("System overloaded. Try again later.")
else:
    task.delay()

```


* **Strategy 2: Circuit Breakers**
If tasks are failing rapidly (e.g., API is down), stop enqueueing new ones. Use a library like `pybreaker` to wrap your `.delay()` calls.
* **Strategy 3: Consumer-side Backpressure (Prefetch)**
As discussed in Section 10, setting `worker_prefetch_multiplier=1` prevents a slow worker from hoarding tasks, keeping them in Redis where other (faster) workers can pick them up. This is a form of load balancing backpressure.