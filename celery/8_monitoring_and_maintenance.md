## Monitoring & Maintenance

Once your Celery workers are running in the background, they become "invisible." You cannot see them in your web server logs. You need specific tools to see what is happening inside the queues.

### **Flower (The Real-Time Web Monitor)**

Flower is the de facto monitoring tool for Celery. It connects to your Broker (Redis) and provides a web-based dashboard.

* **What it shows:**
* **Task Progress:** Real-time graphs of processed vs. failed tasks.
* **Worker Status:** Which workers are online/offline and their CPU usage.
* **Task Details:** Arguments, start time, runtime, and exceptions for individual tasks.
* **Control:** You can remotely restart workers or revoke tasks (if configured).


* **Installation & Running:**
```bash
pip install flower
celery -A myapp flower --port=5555

```


You can now access the dashboard at `http://localhost:5555`.
* **Production Note:** In production, Flower should be protected by Basic Auth (`--basic_auth=user:pass`) and run behind a reverse proxy (Nginx), as it exposes sensitive data about your infrastructure.

### **Command Line Inspection (`inspect`)**

If you don't want to run a web server like Flower, Celery provides powerful CLI tools to "ping" the workers.

* **`celery status`:**
Simple ping. Returns "OK" if workers are reachable.
* **`celery inspect active`:**
Lists tasks currently being executed by workers *right now*. Essential for debugging "stuck" queues.
* **`celery inspect reserved`:**
Lists tasks that a worker has prefetched (taken from Redis) but hasn't started executing yet.
* **`celery inspect stats`:**
Dumps a massive JSON object with internal statistics (memory usage, PID, pool size, total tasks processed).

### **Logging Best Practices**

Debugging asynchronous code is hard. Standard `print()` statements often get lost in standard output or mixed up between multiple worker processes.

#### **1. Use the Task Logger**

Celery provides a special logger that automatically adds the `task_id` and `task_name` to every log entry. This makes it possible to filter logs for a specific task execution in tools like Datadog or ELK.

```python
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

@shared_task
def processing_task(user_id):
    logger.info(f"Starting processing for user {user_id}")
    try:
        # logic
        pass
    except Exception:
        logger.error("Processing failed!", exc_info=True)

```

#### **2. Log Levels**

* **INFO:** Task start/end (Celery does this automatically by default).
* **WARNING:** Retries (Soft failures).
* **ERROR:** Exceptions (Hard failures).

### **Purging (The "Nuclear Option")**

Sometimes, you deploy bad code that queues 100,000 broken tasks. You fix the code, but the queue is still full of the "bad" tasks that will just fail again. You need to empty the queue.

* **The Command:**
```bash
celery -A myapp purge

```


* **What it does:** It deletes **ALL** waiting tasks from the configured Redis Broker.
* **Warning:** This is irreversible. You cannot choose to delete "only the email tasks." It deletes everything pending in the queue.
* **Redis Alternative:** If you are comfortable with Redis CLI, you can delete specific list keys (e.g., `DEL celery`) to target specific queues without wiping everything.
