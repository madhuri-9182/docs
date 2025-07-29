# Celery Task Failures Runbook

## 🔍 Initial Assessment

### 1. Check Celery Services Status
```bash
# Check Celery worker status
sudo systemctl status celery
sudo systemctl status celery-beat

# Check Celery processes
ps aux | grep celery
ps aux | grep -E "(celery|beat)"

# Check if Celery is connected to broker
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
celery -A hiringdogbackend inspect ping
celery -A hiringdogbackend inspect active
```

### 2. Check Celery Logs
```bash
# Celery worker logs
sudo journalctl -u celery -f
sudo tail -f /var/log/hiringdog/celery.log

# Celery beat logs
sudo journalctl -u celery-beat -f
sudo tail -f /var/log/hiringdog/celery_beat.log

# Check for errors
sudo grep -i "error\|exception\|traceback" /var/log/hiringdog/celery.log | tail -20
sudo grep -i "error\|exception\|traceback" /var/log/hiringdog/celery_beat.log | tail -20
```

### 3. Check Redis/RabbitMQ Status
```bash
# Redis status
sudo systemctl status redis
redis-cli ping
redis-cli info server
redis-cli info memory
redis-cli keys "*celery*"

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
sudo systemctl status celery

# Check service logs
sudo journalctl -u celery -n 50

# Test manual startup
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
celery -A hiringdogbackend worker --loglevel=info
```

**Resolution:**
```bash
# Restart Celery service
sudo systemctl restart celery

# If still failing, check configuration
sudo systemctl daemon-reload
sudo systemctl enable celery
sudo systemctl restart celery

# Check broker connection
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
celery -A hiringdogbackend inspect ping
```

### Scenario 2: Redis Connection Issues

**Symptoms:**
- Celery workers disconnected from Redis
- Tasks stuck in queue
- Connection timeout errors

**Diagnosis:**
```bash
# Check Redis connection
redis-cli ping
redis-cli info server

# Check Redis memory
redis-cli info memory

# Test Celery broker connection
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
celery -A hiringdogbackend inspect ping
```

**Resolution:**
```bash
# Restart Redis service
sudo systemctl restart redis

# Restart Celery workers
sudo systemctl restart celery

sudo systemctl restart rabbitmq-server

# Check Redis configuration
sudo cat /etc/redis/redis.conf | grep -E "(maxmemory|timeout)"

# Check broker configuration in Django settings
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
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
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
celery -A hiringdogbackend inspect failed

# Check task statistics
celery -A hiringdogbackend inspect stats

# Monitor task execution
celery -A hiringdogbackend events

# Check recent task failures
sudo grep -i "failed\|error" /var/log/hiringdog/celery.log | tail -20
```

**Resolution:**
```bash
# Purge failed tasks (if safe)
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
celery -A hiringdogbackend purge

# Restart workers to clear stuck tasks
sudo systemctl restart celery

# Check task code for errors
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
python manage.py shell -c "
from django_celery_results.models import TaskResult
failed_tasks = TaskResult.objects.filter(status='FAILURE')
print(f'Failed tasks: {failed_tasks.count()}')
for task in failed_tasks[:5]:
    print(f'{task.task_id}: {task.result}')
"
```

### Scenario 4: Celery Beat Scheduler Issues

**Symptoms:**
- Scheduled tasks not running
- Celery beat process not running
- Periodic tasks missing

**Diagnosis:**
```bash
# Check Celery beat status
sudo systemctl status celery-beat

# Check beat logs
sudo journalctl -u celery-beat -n 50

# Check scheduled tasks
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
celery -A hiringdogbackend inspect scheduled
```

**Resolution:**
```bash
# Restart Celery beat
sudo systemctl restart celery-beat

# Check beat configuration
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
python manage.py shell -c "
from hiringdogbackend.celery import app
print('Beat schedule:', app.conf.beat_schedule)
"

# Clear beat schedule and restart
sudo systemctl stop celery-beat
sudo rm -f /var/lib/celery/beat-schedule
sudo systemctl start celery-beat
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
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
celery -A hiringdogbackend inspect stats
```

**Resolution:**
```bash
# Restart workers to free memory
sudo systemctl restart celery

# Adjust worker concurrency
# Edit systemd service file to add: --concurrency=2

# Increase memory limits
sudo nano /etc/systemd/system/hiringdog-celery.service
# Add: LimitMEMLOCK=infinity

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart celery
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
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
celery -A hiringdogbackend inspect active_queues

# Monitor queue lengths
celery -A hiringdogbackend inspect stats | grep -A 5 "queues"

# Check task routing
celery -A hiringdogbackend inspect active
```

## 🚨 Emergency Procedures

### Quick Recovery
```bash
# Emergency restart all Celery services
sudo systemctl restart celery
sudo systemctl restart celery-beat
sudo systemctl restart redis

# Check if services are running
sudo systemctl status celery celery-beat redis
```
### Complete Service Reset
```bash
# Stop all Celery services
sudo systemctl stop celery celery-beat
sudo pkill -f celery

# Wait a moment
sleep 10

# Start services in order
sudo systemctl start redis
sleep 5
sudo systemctl start celery
sleep 5
sudo systemctl start celery-beat

# Check status
sudo systemctl status celery celery-beat redis
```

### Fallback Task Processing
```bash
# Run tasks synchronously if needed
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
python manage.py shell -c "
from dashboard.tasks import trigger_interview_processing
result = trigger_interview_processing()  # Run synchronously
print('Task completed synchronously')
"
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
3. Contact administrator if unresolved in 40 minutes

## 📝 Post-Incident Actions

1. **Document the incident** in incident log
2. **Update runbook** with new findings
3. **Review task code** for improvements
4. **Implement monitoring** for detected issues

## 🔗 Related Runbooks

- [Gunicorn Failures](./gunicorn-failures.md)
- [Database Failures](./database-failures.md)
- [Server Resources](./server-resources.md)

