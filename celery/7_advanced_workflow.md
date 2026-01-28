## Advanced Workflows (The Canvas)

Simple tasks are easy: "Send one email." But real-world applications often require complex logic: "Scan 50 websites in parallel, aggregate the results, and *then* email the summary."

Celery handles this using **Canvas primitives** (workflows).

### **Signatures (The Building Block)**

Before you can link tasks together, you need to understand **Signatures** (often called `s()`).

* **Definition:** A signature wraps the arguments, keyword arguments, and execution options of a single task invocation into a passible object. It’s like "freezing" a function call to be executed later.
* **Why use it?** You can pass a signature around as a variable, store it, or send it to another process.
* **Syntax:**
```python
from celery import signature
# Standard call (executes immediately)
# add.delay(2, 2)

# Signature (defines the call, doesn't execute yet)
task_sig = add.s(2, 2) 

# Execute later
task_sig.delay()

```



### **Chains (Sequential Execution)**

* **Concept:** Run Task A, then pass its output as the input to Task B, then Task C.
* **Use Case:** ETL Pipelines (Extract -> Transform -> Load).
* **Behavior:** The return value of the previous task is automatically passed as the *first argument* to the next task.
* **Syntax:**
```python
from celery import chain

# (2 + 2) -> Result is 4 -> (4 + 8) -> Result is 12
workflow = chain(add.s(2, 2), add.s(8))
result = workflow.delay()

```


*Note:* Notice `add.s(8)` only has one argument? That's because the result of the first task (4) becomes the first argument here.

### **Groups (Parallel Execution)**

* **Concept:** Run a list of tasks at the same time (in parallel workers) and return a list of results.
* **Use Case:** Sending 100 emails at once, or scraping 10 different URLs.
* **Behavior:** The tasks are independent. The group result is ready only when *all* tasks are finished.
* **Syntax:**
```python
from celery import group

# Run ten 'add' tasks in parallel
job = group(add.s(i, i) for i in range(10))
result = job.delay()
# result.get() returns [0, 2, 4, 6, 8, ...]

```



### **Chords (Group + Callback)**

* **Concept:** This is the "MapReduce" of Celery. It allows you to execute a **Group** (Map) and, once *all* of them are finished, execute a single **Callback** task (Reduce).
* **Use Case:** "Resize 50 images (Group), then zip them all into a single archive (Callback)."
* **The Challenge:** Standard groups don't trigger a "finished" event easily. Chords solve this synchronization problem.
* **Requirement:** You **must** have a Result Backend configured (Redis/Memcached) for chords to work, as the system needs to track the status of every task in the group.
* **Syntax:**
```python
from celery import chord

# Header: The parallel tasks
header = [add.s(i, i) for i in range(10)]

# Body: The callback task (receives list of results from header)
callback = sum_all_numbers.s()

# Execute
chord(header)(callback)

```



### **Immutable Signatures (`.si`)**

* **The Problem:** By default, Chains and Chords pass the result of the previous task to the next one. Sometimes you don't want that (e.g., Task B doesn't care about Task A's return value).
* **The Solution:** Use `.si()` (Signature Immutable) instead of `.s()`.
* **Behavior:** It ignores any arguments passed from previous tasks in the chain.
```python
# Task 2 (log_success) will NOT receive the result of Task 1 (add)
chain(add.s(2, 2), log_success.si()).delay()

```
