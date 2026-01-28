## Task Definition & Execution

This section covers how you write code to define work and how you instruct Celery to execute that work.

### **Defining Tasks (The Decorators)**

To turn a standard Python function into a Celery task, you wrap it with a decorator.

#### **`@app.task`**

* **Usage:** Used when you have a standalone script or a simple module where the Celery `app` instance is globally available.
* **Example:**
```python
@app.task
def add(x, y):
    return x + y

```



#### **`@shared_task` (Crucial for Django)**

* **Usage:** Used in reusable libraries or Django apps.
* **Why:** It allows you to define tasks without importing the specific Celery app instance, preventing circular import errors. It attaches the task to whatever Celery app is currently running.
* **Example:**
```python
from celery import shared_task

@shared_task
def send_welcome_email(user_id):
    # Logic here
    pass

```



### **Calling Tasks**

You do not call a task function directly (e.g., `add(2, 2)`), as that would run it synchronously in the local process. Instead, you invoke methods that send a message to the Broker.

#### **1. `.delay()` (The Shortcut)**

* **Description:** The most common way to call a task. It is a shortcut for `.apply_async` that automatically passes arguments.
* **Syntax:** `task_name.delay(arg1, arg2, kwarg1=value)`
* **Example:** `send_welcome_email.delay(user_id=42)`

#### **2. `.apply_async()` (The Power Tool)**

* **Description:** Allows you to control execution options (routing, timing, expiration).
* **Syntax:** You must pass args as a list and kwargs as a dictionary.
* **Key Parameters:**
* `args` / `kwargs`: The inputs for the function.
* `countdown`: Execute after X seconds (e.g., `countdown=10`).
* `eta` (Estimated Time of Arrival): Execute at a specific `datetime`.
* `expires`: If the task hasn't started by this time, revoke it (prevents stale tasks from jamming the queue).
* `retry`: Whether to retry on failure (boolean).
* `queue`: Route this specific task to a specific queue (e.g., `queue='high_priority'`).


**Example:**
```python
# Send email 10 minutes from now, expire if not sent within 20 mins
send_welcome_email.apply_async(
    args=[42],
    countdown=600,
    expires=1200
)

```



### **Task Arguments (Serialization)**

When you call `.delay(x)`, Celery must convert `x` into a format that can be stored in Redis (JSON). This leads to a critical best practice.

#### **The Golden Rule: Pass IDs, Not Objects**

* **Bad Practice:** Passing a complex database object (like a Django ORM User object).
```python
# DON'T DO THIS
user = User.objects.get(id=1)
process_user.delay(user)

```


* **Why it fails:**
1. **Serialization:** JSON cannot encode Python objects easily.
2. **Stale Data:** By the time the worker processes the task (e.g., 5 mins later), the data in the `user` object might be outdated.




* **Best Practice:** Pass the primary key (ID) and re-fetch the data in the worker.
```python
# DO THIS
process_user.delay(user_id=1)

# Inside the task:
@shared_task
def process_user(user_id):
    user = User.objects.get(id=user_id) # Fetch fresh data
    ...

```



#### **JSON vs. Pickle**

* **JSON:** The modern default. Safe, language-agnostic, but creates limitations (can't pass complex Python types like `datetime` or `sets` directly without custom encoders).
* **Pickle:** Supports almost any Python object but is a **security risk**. Never use Pickle if your broker is exposed or accepts data from untrusted sources.

### **Task States**

Celery tracks the progress of tasks using a standard state machine.

* **PENDING:** The task is waiting in the queue. (Note: To Celery, any unknown task ID is considered "PENDING" by default).
* **STARTED:** The worker has received the task and is currently executing it.
* *Note:* This state is not enabled by default because it requires extra write operations to Redis. You must enable `task_track_started=True` if you need it.


* **SUCCESS:** The task finished without raising an exception.
* **FAILURE:** The task raised an exception. The stack trace is stored in the result backend.
* **RETRY:** The task failed but is scheduled to run again.
* **REVOKED:** The task was cancelled by the user before the worker could start it.
