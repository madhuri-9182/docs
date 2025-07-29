# Django Application Failures Runbook

## 🔍 Initial Assessment

### 1. Check Application Status
```bash
# Check if Django process is running
sudo systemctl status gunicorn
ps aux | grep manage.py

# Check port availability
netstat -tlnp | grep :8000
lsof -i :8000

# Test application response
curl -I http://localhost:8000/
curl -I http://localhost:8000/admin
curl -I https://prod-api.hdiplatform.in/
curl -I https://api.hdiplatform.in/
```

### 2. Check Logs
```bash
# Gunicorn application logs
sudo journalctl -u gunicorn -f
sudo tail -f /var/log/hiringdog/gunicorn_error.log
sudo tail -f /var/log/hiringdog/gunicorn.log

# Error logs
sudo tail -f /var/log/hiringdog/app_error.log
sudo tail -f /var/log/hiringdog/errors.log
```

## 🛠️ Common Failure Scenarios

### Scenario 1: Gunicorn Process Not Starting

**Symptoms:**
- `sudo systemctl status gunicorn` shows failed
- Port 8000 not listening
- Application not responding

**Diagnosis:**
```bash
# Check systemd service status
sudo systemctl status gunicorn --no-pager

# Check service logs
sudo journalctl -u gunicorn --no-pager -n 50

# Test manual startup
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
gunicorn --bind 0.0.0.0:8000 hiringdogbackend.wsgi:application
```

**Resolution:**
```bash
# Restart the service
sudo systemctl restart gunicorn

# If still failing, check configuration
sudo systemctl daemon-reload
sudo systemctl enable gunicorn
sudo systemctl restart gunicorn

# Check for missing dependencies
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
pip install -r requirements.txt
```

### Scenario 2: Database Connection Issues

**Symptoms:**
- 500 errors on database-dependent endpoints
- Application logs show database connection errors
- `OperationalError: (2006, 'MySQL server has gone away')`

**Diagnosis:**
```bash
# Test database connection
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
python manage.py dbshell
python manage.py check --database default

# Check database server status (GCP Cloud SQL)
sudo systemctl status mysql
```

**Resolution:**
```bash
# Restart database service
sudo systemctl restart mysql

# Check Django database settings
python manage.py check --deploy

# Verify database migrations
python manage.py migrate --plan
python manage.py migrate --noinput
```

### Scenario 3: Memory/Resource Exhaustion

**Symptoms:**
- Slow response times
- Out of memory errors in logs
- Process killed by OOM killer

**Diagnosis:**
```bash
# Check system resources
free -h
df -h
top
htop

# Check Django process memory usage
ps aux | grep gunicorn
ps aux --sort=-%mem | head -10
```

**Resolution:**
```bash
# Restart application to free memory
sudo systemctl restart gunicorn

# If persistent, increase memory limits
# Edit systemd service file
sudo nano /etc/systemd/system/gunicorn.service
# Add: LimitMEMLOCK=infinity

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
```

### Scenario 4: Import/Module Errors

**Symptoms:**
- ImportError in logs
- ModuleNotFoundError
- Application fails to start

**Diagnosis:**
```bash
# Check Python environment
cd /home/ubuntu/Hiringdog-backend
which python
python --version
pip list

# Test imports
python manage.py check
```

**Resolution:**
```bash
# Reinstall dependencies
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
pip install -r requirements.txt --force-reinstall

# Check virtual environment
source venv/bin/activate
pip install -r requirements.txt

# Update Django settings
python manage.py check
```

## 🔧 Advanced Troubleshooting

### Debug Mode Investigation
```bash
# Enable debug logging temporarily
# Edit settings.py: DEBUG = True
# Add to LOGGING configuration

# Check for specific errors
grep -r "Exception" /var/log/hiringdog/
grep -r "Traceback" /var/log/hiringdog/
```

### Performance Issues
```bash
# Check slow queries
python manage.py dbshell
# In MySQL: SHOW PROCESSLIST;
# In PostgreSQL: SELECT * FROM pg_stat_activity;

# Check Django debug toolbar (if enabled)
# Look for slow queries in browser
```

### Static Files Issues
```bash
# Collect static files
python manage.py collectstatic --noinput

# Check static files permissions
ls -la /path/to/static/files/
chmod -R 755 /path/to/static/files/
```

## 📊 Monitoring Commands

```bash
# Health check script
#!/bin/bash
response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/)
if [ $response -eq 200 ]; then
    echo "gunicorn is healthy"
else
    echo "gunicorn is unhealthy: $response"
    # Send alert
fi
```

## 🚨 Escalation

### When to Escalate:
- Application down for >15 minutes
- Database corruption suspected
- Security breach indicators
- Multiple services affected

### Escalation Steps:
1. Notify on-slack to engineer
2. Update incident status
3. Contact system administrator if unresolved in 60 minutes

## 📝 Post-Incident Actions

1. **Document the incident** in incident log
2. **Update runbook** with new findings
3. **Schedule post-mortem** for critical issues
4. **Implement preventive measures**

## 🔗 Related Runbooks

- [Database Failures](./database-failures.md)
- [Celery Failures](./celery-failures.md)
- [Server Resources](./server-resources.md)

---

*Last Updated: [Date]*
*Maintained by: [Team Name]*
