## Scheduling & Periodic Tasks

While standard Celery tasks are triggered by user actions (e.g., "User signed up"), **Periodic Tasks** are triggered by time (e.g., "Every Monday at 9 AM"). This functionality relies on a component called **Celery Beat**.

### **Celery Beat (The Scheduler)**

* **The Role:** Celery Beat is a scheduler service. It kicks off tasks at regular intervals.
* **How it works:**
1. It reads a schedule (a list of tasks and times).
2. It checks the current time in a loop.
3. When a specific time is reached, it sends a message to the **Broker** (Redis).
4. **Crucial Distinction:** The Beat process **does not execute tasks**. It simply acts as a Producer, creating the message and putting it in the queue. The Workers pick it up and do the actual work.



### **Defining Schedules (Static Configuration)**

The simplest way to define periodic tasks is in your Celery configuration file (e.g., `celery.py` or `settings.py` in Django).

**The `beat_schedule` Dictionary:**
You define a dictionary where the key is a unique name for the schedule, and the value contains the task details.

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    # Method 1: Simple interval (every 30 seconds)
    'add-every-30-seconds': {
        'task': 'myapp.tasks.add',
        'schedule': 30.0,
        'args': (16, 16)
    },

    # Method 2: Crontab (Every Monday morning at 7:30 AM)
    'send-weekly-report': {
        'task': 'myapp.tasks.send_report',
        'schedule': crontab(hour=7, minute=30, day_of_week=1),
    },
}

```

### **Crontab Expressions**

Celery uses the `crontab()` function to mimic standard Linux cron syntax. This allows for complex scheduling logic.

* **Syntax:** `crontab(minute=..., hour=..., day_of_week=..., day_of_month=..., month_of_year=...)`
* **Wildcards (`*`):** The default value is "always" or "every".
* `crontab(minute='*')` -> Every minute.
* `crontab(hour=0, minute=0)` -> Midnight every day.


* **Specific Examples:**
* `crontab(minute=0, hour='*/4')`: Run on minute 0 of every 4th hour (00:00, 04:00, 08:00...).
* `crontab(day_of_week='mon,fri')`: Run on Mondays and Fridays.
* `crontab(day_of_month='1-7', day_of_week='fri')`: Run on the first Friday of the month (e.g., payday scripts).



### **Dynamic Scheduling (Database Schedulers)**

Hardcoding schedules in python files has a downside: to change a schedule, you must redeploy your code and restart the Beat process.

* **The Solution:** Store the schedule in a database (like PostgreSQL or MySQL).
* **Tool:** **`django-celery-beat`** (for Django projects).
* It serves as a "Database Scheduler."
* It creates database tables for `PeriodicTask`, `IntervalSchedule`, and `CrontabSchedule`.
* **Benefit:** You can log into your Django Admin panel and change a task from "Daily" to "Hourly" instantly without touching a single line of code or restarting the server. Celery Beat reads the changes from the DB periodically.



### **Production Pitfalls**

1. **The "Highlander" Rule (There Can Be Only One):**
You must run **only one** instance of Celery Beat at a time. If you spin up two Docker containers running Beat, they will both trigger the schedule, and your users will receive every email twice.
2. **Timezones:**
By default, Celery uses UTC. If you want tasks to run at "9 AM Local Time," you must configure `CELERY_TIMEZONE` (e.g., `'Asia/Kolkata'`) and ensure your server time and Django settings align.
3. **Persisting State:**
Beat needs to remember when it last ran a task so it doesn't repeat it if you restart the process. It stores this locally in a file named `celerybeat-schedule`. In Docker/Kubernetes, you must ensure this file is stored in a persistent volume, or Beat might re-run tasks on restart.