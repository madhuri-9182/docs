
# HiringDog Backend Runbooks

This directory contains comprehensive runbooks for various failure scenarios in the HiringDog Django application.


## 📋 Available Runbooks

### Application Level
- [ Gunicorn Failures](./gunicorn-failures.md)
- [Celery Task Failures](./celery-failures.md)
- [Celery-beat Task Failures](./celery-beat-failures.md)
- [Database Connection Issues](./database-failures.md)
- [Nginx Failures](./nginx-failures.md)
  
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

1. **Immediate Response**: Follow the specific runbook for the failure type
2. **Communication**: Notify to lead via Slack/Teams
3. **Documentation**: Update incident log
4. **Post-Mortem**: Schedule within 24 hours for critical issues
---


