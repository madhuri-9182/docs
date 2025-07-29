# Celery Task Failures Runbook

### 1. Check Celery Services Status
```bash
# Check Celery worker status
sudo systemctl status celery
sudo systemctl status celery-beat

# Check message brokers
sudo systemctl status rabbitmq-server
sudo systemctl status redis-server

# Check all processes
ps aux | grep celery
ps aux | grep rabbitmq
ps aux | grep redis
```

### 2. Check Celery Logs
```bash
# Celery logs
sudo journalctl -u celery -f
sudo tail -f /var/log/hiringdog/celery.log
sudo journalctl -u celery-beat -f
sudo tail -f /var/log/hiringdog/celery_beat.log

# RabbitMQ logs
sudo journalctl -u rabbitmq-server -f
sudo tail -f /var/log/rabbitmq/rabbit@$(hostname).log

# Redis logs
sudo journalctl -u redis-server -f
sudo tail -f /var/log/redis/redis-server.log

# Check for errors
sudo grep -i "error\|exception\|traceback" /var/log/hiringdog/celery.log | tail -20
sudo grep -i "error\|exception\|traceback" /var/log/hiringdog/celery_beat.log | tail -20
```
### 3. Test Broker Connections

- Test RabbitMQ connection
- Test Redis connection
- Test Celery broker connection

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

# Check service configuration
sudo cat /etc/systemd/system/celery.service

```

**Resolution:**
```bash
# Restart Celery service
sudo systemctl restart celery

# If still failing, check configuration
sudo systemctl daemon-reload
sudo systemctl enable celery
sudo systemctl restart celery
```

- Check broker connection
- Check environment variables

### Scenario 2: RabbitMQ Connection Issues

**Symptoms:**
- Celery workers disconnected from RabbitMQ
- Tasks stuck in queue
- Connection timeout errors

**Diagnosis:**
```bash
# Check RabbitMQ status
sudo systemctl status rabbitmq-server
rabbitmqctl status

# Check RabbitMQ queues
rabbitmqctl list_queues
rabbitmqctl list_connections

# Check RabbitMQ logs
sudo journalctl -u rabbitmq-server -n 50

# Test RabbitMQ connection
rabbitmqctl ping
```

**Resolution:**
```bash
# Restart RabbitMQ service
sudo systemctl restart rabbitmq-server

# Wait for RabbitMQ to fully start
sleep 10

# Restart Celery workers
sudo systemctl restart celery
sudo systemctl restart celery-beat

# Check RabbitMQ configuration
sudo cat /etc/rabbitmq/rabbitmq-env.conf

# Verify queues are working
rabbitmqctl list_queues
```

### Scenario 3: Memory/Resource Issues

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

```

**Resolution:**
```bash
# Restart workers to free memory
sudo systemctl restart celery

# Adjust worker concurrency
sudo nano /etc/systemd/system/celery.service
# Add to ExecStart: --concurrency=2

# Increase memory limits
sudo nano /etc/systemd/system/celery.service
# Add: LimitMEMLOCK=infinity

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart celery
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


