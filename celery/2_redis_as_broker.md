## Redis as a Broker

This section focuses on the specific interaction between Celery and Redis. Redis is the most popular broker for Celery because it is fast, easy to set up, and supports ephemeral data storage perfectly.

### **Broker Configuration**

To connect Celery to Redis, you define the `broker_url` setting in your Celery configuration.

* **The Connection String:**
The format follows the standard URI syntax:
`redis://:password@hostname:port/db_number`
* **Breakdown:**
* `redis://`: The protocol.
* `:password`: If your Redis instance is password-protected (highly recommended in production).
* `@hostname`: The IP address or domain (e.g., `localhost` or `redis-service`).
* `port`: Default is `6379`.
* `/db_number`: Redis supports multiple isolated databases (numbered 0-15). It is best practice to use different DB numbers for your **Broker** and your **Result Backend** to avoid key collisions (e.g., DB `0` for Broker, DB `1` for Results).


**Example Code:**
```python
app = Celery('myapp', broker='redis://localhost:6379/0')

```



### **Redis Data Structures (Under the Hood)**

Celery maps its concepts directly to native Redis data structures. Understanding this helps you debug by inspecting Redis directly using the `redis-cli`.

1. **The Queue -> Redis LIST**
* The default queue in Celery is named `celery`.
* In Redis, this is stored as a **List** key.
* **Producer Action:** Performs an `LPUSH` (Left Push) to add the task message JSON to the head of the list.
* **Worker Action:** Performs a `BRPOP` (Blocking Right Pop). This command waits (blocks) until an item is available in the list, then pops it from the tail. This ensures First-In-First-Out (FIFO) behavior.


2. **Scheduling -> Redis ZSET (Sorted Set)**
* If you use `countdown` (delay) or `eta` (specific time), the task is not put in the main List immediately.
* It goes into a **Sorted Set** (often keys like `unacked` or specific schedule keys). The "score" in the sorted set is the timestamp when the task *should* run.
* The worker checks this set and moves the task to the main List only when the timestamp is reached.



### **Visibility Timeout & Acknowledgments**

This is the safety mechanism that prevents task loss if a Worker crashes while processing.

* **The Scenario:** A Worker picks up a task (removing it from the Redis List) but crashes due to an `Out of Memory` error before finishing. The task is gone from the queue but wasn't done.
* **The Solution (ACKs):** Celery uses an Acknowledgement (ACK) system.
* When a worker picks up a task, it doesn't just disappear. Celery tracks it in a separate "unacknowledged" structure.
* The worker must send an `ACK` signal back to Redis upon successful completion.


* **Visibility Timeout:**
* If the Broker doesn't receive an ACK within a specific time (default is usually 1 hour), it assumes the Worker died.
* The task is then **redelivered** to the queue for another worker to pick up.
* *Warning:* If you have very long-running tasks (e.g., video processing taking 2 hours), you must increase the `visibility_timeout` setting, or the task will be sent to a second worker while the first is still processing it (causing duplication).



### **Key Eviction Policies (Critical for Production)**

Redis is an in-memory store. If it runs out of RAM, it must decide what to delete to make room for new data. This behavior is controlled by the `maxmemory-policy`.

* **The Danger:** Default Redis policies like `volatile-lru` or `allkeys-lru` might delete your task queue keys if memory gets full. **This means you lose pending tasks.**
* **The Correct Policy:** `noeviction`.
* **Setting:** Configure Redis with `maxmemory-policy noeviction`.
* **Behavior:** If Redis gets full, it will reject *new* write operations (raising an error) rather than silently deleting existing tasks. This ensures data durability.
* *Tip:* If you share your Redis instance for caching (which needs LRU eviction) and Celery (which needs `noeviction`), you should ideally use two separate Redis instances.
