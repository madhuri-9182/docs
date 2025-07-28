
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
sudo tail -f /var/log/gunicorn/error.log
sudo tail -f /var/log/celery/error.log
sudo tail -f /var/log/nginx/error.log

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

1. **Immediate Response**: Follow the specific runbook for the failure type
2. **Communication**: Notify stakeholders via Slack/Teams
3. **Documentation**: Update incident log
4. **Post-Mortem**: Schedule within 24 hours for critical issues

---

*Last Updated: [Date]*
*Maintained by: [Team Name]*


