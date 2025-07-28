# Celery Task Failures Runbook

## 🚨 Severity Levels

- **Critical**: All Celery workers down, no task processing
- **High**: Major task queues failing, affecting core functionality
- **Medium**: Some tasks failing, partial functionality affected
- **Low**: Minor task delays, no user impact

## 🔍 Initial Assessment

### 1. Check Celery Services Status
```bash
# Check Celery worker status
sudo systemctl status hiringdog-celery
sudo systemctl status hiringdog-celerybeat

# Check Celery processes
ps aux | grep celery
ps aux | grep -E "(celery|beat)"

# Check if Celery is connected to broker
celery -A hiringdogbackend inspect active
celery -A hiringdogbackend inspect stats
```

### 2. Check Celery Logs
```bash
# Celery worker logs
sudo journalctl -u hiringdog-celery -f --since "10 minutes ago"
tail -f /var/log/hiringdog/celery.log

# Celery beat logs
sudo journalctl -u hiringdog-celerybeat -f --since "10 minutes ago"
tail -f /var/log/hiringdog/celerybeat.log

# Check for errors
grep -i error /var/log/hiringdog/celery.log | tail -20
grep -i "task.*failed" /var/log/hiringdog/celery.log
```

### 3. Check Redis/RabbitMQ Status
```bash
# Redis status
sudo systemctl status redis
redis-cli ping
redis-cli info

# RabbitMQ status
sudo systemctl status rabbitmq-server
rabbitmqctl status
rabbitmqctl list_queues
```

## 🛠️ Common Failure Scenarios

### Scenario 1: Celery Workers Not Starting

**Symptoms:**
- `systemctl status hiringdog-celery` shows failed
- No Celery processes running
- Tasks not being processed

**Diagnosis:**
```bash
# Check systemd service status
sudo systemctl status hiringdog-celery

# Check service logs
sudo journalctl -u hiringdog-celery -n 50

# Test manual startup
cd /path/to/hiringdog-backend
source venv/bin/activate
celery -A hiringdogbackend worker --loglevel=info
```

**Resolution:**
```bash
# Restart Celery service
sudo systemctl restart hiringdog-celery

# If still failing, check configuration
sudo systemctl daemon-reload
sudo systemctl enable hiringdog-celery
sudo systemctl restart hiringdog-celery

# Check broker connection
celery -A hiringdogbackend inspect ping
```

### Scenario 2: Broker Connection Issues

**Symptoms:**
- Celery workers disconnected from broker
- Tasks stuck in queue
- Connection timeout errors

**Diagnosis:**
```bash
# Check Redis connection
redis-cli ping
redis-cli info server

# Check RabbitMQ connection
rabbitmqctl status
rabbitmqctl list_connections

# Test Celery broker connection
celery -A hiringdogbackend inspect ping
```

**Resolution:**
```bash
# Restart broker service
sudo systemctl restart redis
# or
sudo systemctl restart rabbitmq-server

# Restart Celery workers
sudo systemctl restart hiringdog-celery

# Check broker configuration in Django settings
python manage.py shell -c "from django.conf import settings; print(settings.CELERY_BROKER_URL)"
```

### Scenario 3: Task Execution Failures

**Symptoms:**
- Specific tasks failing repeatedly
- Error messages in Celery logs
- Tasks stuck in retry loop

**Diagnosis:**
```bash
# Check failed tasks
celery -A hiringdogbackend inspect failed

# Check task statistics
celery -A hiringdogbackend inspect stats

# Monitor task execution
celery -A hiringdogbackend events
```

**Resolution:**
```bash
# Purge failed tasks (if safe)
celery -A hiringdogbackend purge

# Restart workers to clear stuck tasks
sudo systemctl restart hiringdog-celery

# Check task code for errors
python manage.py shell -c "from dashboard.tasks import trigger_interview_processing; print('Task import OK')"
```

### Scenario 4: Celery Beat Scheduler Issues

**Symptoms:**
- Scheduled tasks not running
- Celery beat process not running
- Periodic tasks missing

**Diagnosis:**
```bash
# Check Celery beat status
sudo systemctl status hiringdog-celerybeat

# Check beat logs
sudo journalctl -u hiringdog-celerybeat -n 50

# Check scheduled tasks
celery -A hiringdogbackend inspect scheduled
```

**Resolution:**
```bash
# Restart Celery beat
sudo systemctl restart hiringdog-celerybeat

# Check beat configuration
python manage.py shell -c "from hiringdogbackend.celery import app; print(app.conf.beat_schedule)"

# Clear beat schedule and restart
sudo systemctl stop hiringdog-celerybeat
rm -f /var/lib/celery/beat-schedule
sudo systemctl start hiringdog-celerybeat
```

### Scenario 5: Memory/Resource Issues

**Symptoms:**
- Workers running out of memory
- High CPU usage
- Workers being killed by OOM

**Diagnosis:**
```bash
# Check system resources
free -h
top
htop

# Check Celery worker memory usage
ps aux | grep celery | grep -v grep

# Monitor Celery performance
celery -A hiringdogbackend inspect stats
```

**Resolution:**
```bash
# Restart workers to free memory
sudo systemctl restart hiringdog-celery

# Adjust worker concurrency
# Edit systemd service file to add: --concurrency=2

# Increase memory limits
sudo nano /etc/systemd/system/hiringdog-celery.service
# Add: LimitMEMLOCK=infinity

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart hiringdog-celery
```

## 🔧 Advanced Troubleshooting

### Debug Task Execution
```bash
# Run worker in debug mode
celery -A hiringdogbackend worker --loglevel=debug

# Test specific task
python manage.py shell -c "
from dashboard.tasks import trigger_interview_processing
result = trigger_interview_processing.delay()
print(f'Task ID: {result.id}')
"
```

### Monitor Task Queues
```bash
# Check queue status
celery -A hiringdogbackend inspect active_queues

# Monitor queue lengths
celery -A hiringdogbackend inspect stats | grep -A 5 "queues"

# Check task routing
celery -A hiringdogbackend inspect active
```

### Database Connection Issues in Tasks
```bash
# Test database connection in task context
python manage.py shell -c "
from django.db import connection
cursor = connection.cursor()
cursor.execute('SELECT 1')
print('Database connection OK')
"
```

## 📊 Monitoring Commands

```bash
# Health check script
#!/bin/bash
# Check Celery workers
worker_status=$(celery -A hiringdogbackend inspect ping 2>/dev/null | grep -c "pong")
if [ $worker_status -gt 0 ]; then
    echo "Celery workers are healthy"
else
    echo "Celery workers are down"
    # Send alert
fi

# Check task queue
queue_length=$(celery -A hiringdogbackend inspect stats 2>/dev/null | grep -o '"length": [0-9]*' | head -1 | grep -o '[0-9]*')
if [ "$queue_length" -gt 100 ]; then
    echo "Task queue is backing up: $queue_length tasks"
    # Send alert
fi
```

## 🚨 Emergency Procedures

### Quick Recovery
```bash
# Emergency restart all Celery services
sudo systemctl restart hiringdog-celery
sudo systemctl restart hiringdog-celerybeat
sudo systemctl restart redis

# Check if services are running
sudo systemctl status hiringdog-celery hiringdog-celerybeat redis
```

### Fallback Task Processing
```bash
# Run tasks synchronously if needed
python manage.py shell -c "
from dashboard.tasks import trigger_interview_processing
trigger_interview_processing()  # Run synchronously
"
```

## 📝 Configuration Best Practices

### Worker Configuration
```python
# In celery.py
app.conf.update(
    worker_prefetch_multiplier=1,
    task_acks_late=True,
    worker_max_tasks_per_child=1000,
    task_time_limit=3600,
    task_soft_time_limit=3000,
)
```

### Task Retry Configuration
```python
# In task definitions
@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def my_task(self):
    try:
        # Task logic
        pass
    except Exception as exc:
        self.retry(exc=exc)
```

## 🚨 Escalation

### When to Escalate:
- All Celery workers down for >10 minutes
- Critical tasks not processing for >30 minutes
- Database connection issues in tasks
- Multiple task types failing

### Escalation Steps:
1. Notify on-call engineer
2. Update incident status
3. Contact senior engineer if unresolved in 20 minutes
4. Contact system administrator if unresolved in 40 minutes

## 📝 Post-Incident Actions

1. **Document the incident** in incident log
2. **Update runbook** with new findings
3. **Review task code** for improvements
4. **Implement monitoring** for detected issues

## 🔗 Related Runbooks

- [Django Failures](./django-failures.md)
- [Database Failures](./database-failures.md)
- [Server Resources](./server-resources.md)

---

*Last Updated: [Date]*
*Maintained by: [Team Name]*
