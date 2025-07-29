
# HiringDog Backend Runbooks

This directory contains comprehensive runbooks for various failure scenarios in the HiringDog Django application.


## 📋 Available Runbooks

### Application Level
- [ Gunicorn Failures](./gunicorn-failures.md)
- [Celery Task Failures](./celery-failures.md)
- [Celery-beat Task Failures](./celery-beat-failures.md)
- [Database Connection Issues](./database-failures.md)
- [API Endpoint Failures](./api-failures.md)

### Infrastructure Level
- [Nginx Failures](./nginx-failures.md)
- [Server Resource Issues](./server-resources.md)
- [SSL Certificate Issues](./ssl-certificate-issues.md)

### Monitoring & Alerts
- [Monitoring Setup](./monitoring-setup.md)
- [Alert Response Procedures](./alert-response.md)

## 🔧 Quick Commands

```bash
# Check application status
sudo systemctl status gunicorn
sudo systemctl status celery
sudo systemctl status celery-beat
sudo systemctl status nginx

# View logs
sudo journalctl -u gunicorn -f
sudo journalctl -u celery -f
sudo tail -f /var/log/hiringdog/gunicorn_error.log
sudo tail -f /var/log/nginx/hiringdogbackend_error.log

# Database connection test
python manage.py dbshell
```

## 📊 Endpoints

- Application: `https://live.hdiplatform.in/`
- prod-app: `https://prod-api.hdiplatform.in/`
- staging-app: `https://api.hdiplatform.in/`
- landing page: `http://www.hdiplatform.in/`
- Database Status: `https://your-domain.com/api/db-status/`

## 🆘 Emergency Procedures

### Complete Service Reset
```bash
# Stop all services
sudo systemctl stop gunicorn celery celery-beat nginx

# Kill any remaining processes
sudo pkill -f gunicorn
sudo pkill -f celery

# Wait a moment
sleep 10

# Start services in order
sudo systemctl start nginx
sleep 5
sudo systemctl start gunicorn
sleep 5
sudo systemctl start celery
sudo systemctl start celery-beat

# Check status
sudo systemctl status gunicorn celery celery-beat nginx
```
### Rollback to Previous Release
```bash
# Rollback to previous release
cd /home/ubuntu/releases
PREVIOUS_RELEASE=$(ls -1t | head -2 | tail -1)
ln -sfn /home/ubuntu/releases/$PREVIOUS_RELEASE /home/ubuntu/Hiringdog-backend

# Restart services
sudo systemctl restart gunicorn celery celery-beat
```

### Zero-Downtime Restart
```bash
# Perform zero-downtime Gunicorn restart
GUNICORN_PID=$(systemctl show --property MainPID gunicorn | cut -d= -f2)

if [[ "$GUNICORN_PID" != "0" ]]; then
    # Send USR2 to start new master with new workers
    sudo kill -USR2 "$GUNICORN_PID"
    
    # Wait for new master to start
    sleep 3
    
    # Send WINCH to old master to gracefully shut down old workers
    sudo kill -WINCH "$GUNICORN_PID"
    
    # Wait a bit more for graceful shutdown
    sleep 2
    
    # Send TERM to old master to shut it down completely
    sudo kill -TERM "$GUNICORN_PID"
fi
```

1. **Immediate Response**: Follow the specific runbook for the failure type
2. **Communication**: Notify to lead via Slack/Teams
3. **Documentation**: Update incident log
4. **Post-Mortem**: Schedule within 24 hours for critical issues
---


