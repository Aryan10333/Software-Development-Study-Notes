## Production & Deployment

Running Celery locally (`celery -A app worker`) is easy. Running it in production requires ensuring the process stays alive, scales correctly, and is secure.

### **Process Supervision (Keeping Workers Alive)**

Celery workers are long-running processes, but they can crash (due to memory leaks, unhandled exceptions, or server reboots). You need a **Process Supervisor** to monitor them and automatically restart them if they die.

#### **1. Systemd (The Linux Standard)**

Most modern Linux servers (Ubuntu 16.04+, CentOS 7+) use `systemd`. You create service files (e.g., `/etc/systemd/system/celery.service`).

* **Key Directive:** `Restart=always`
This tells Linux: "If this process stops for any reason, start it again immediately."
* **User Management:** You should define `User=www-data` (or your app user). **Never** run Celery as root (Celery will actually warn you or refuse to run if you try).

#### **2. Supervisor (The Classic Alternative)**

`Supervisor` is a Python-based process control system often used if `systemd` is not available or if you prefer python-based configuration.

* **Config File:** `supervisord.conf`
* **Behavior:** It creates a socket to control processes. You can stop/start workers via a command-line interface (`supervisorctl status`).

### **Dockerization (Container Strategy)**

In modern DevOps, you likely use Docker. The golden rule for Celery in Docker is: **One Process Per Container.**

Do **not** try to run Django, the Worker, and the Beat scheduler all in one "mega-container." If the worker crashes, it might take down the web server, or vice versa.

#### **Docker Compose Structure**

You usually define three distinct services sharing the same image but running different commands.

```yaml
version: '3.8'

services:
  # 1. The Broker
  redis:
    image: redis:6-alpine

  # 2. The Web App (Producer)
  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    depends_on:
      - redis

  # 3. The Worker (Consumer)
  celery_worker:
    build: .
    # Note: Use -l info for logs. In prod, maybe -l error.
    command: celery -A myapp worker -l info 
    depends_on:
      - redis

  # 4. The Scheduler (Beat) - ONLY ONE INSTANCE
  celery_beat:
    build: .
    command: celery -A myapp beat -l info
    depends_on:
      - redis

```

### **Security**

#### **1. Redis Security**

* **Password Protection:** By default, Redis has no password. In production, edit `redis.conf` to set `requirepass yourpassword`. Update your Celery `BROKER_URL` to include it: `redis://:yourpassword@...`
* **Network Isolation:** Redis should **never** be exposed to the public internet. Bind it to `127.0.0.1` or use a private VPC network (if using AWS/DigitalOcean).

#### **2. Input Sanitation (Pickle Warning)**

* As mentioned in Section 3, ensure your serializer is set to `json`.
```python
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT = ['json']

```


* **Why?** If you use `pickle` and an attacker can send a message to your Redis queue, they can execute arbitrary code on your server (Remote Code Execution) when the worker deserializes the message.

### **Production Checklist**

1. **Broker Connection Pool:** Ensure your Redis connection limit isn't hit.
2. **Log Level:** Change `-l info` to `-l warning` or `-l error` to save disk space, unless you have centralized logging (ELK/Datadog).
3. **Result Backend Cleanup:** If using Redis for results, ensure you set `CELERY_RESULT_EXPIRES` (e.g., 3600 seconds). Otherwise, your Redis memory will fill up with the results of old tasks forever.
4. **Health Checks:** Implement a `/health` endpoint in your web app that checks if it can ping Celery/Redis.

