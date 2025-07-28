# Monitoring Setup and Alert Response Runbook

## 📊 Monitoring Overview

This runbook covers the setup and maintenance of monitoring systems for the HiringDog Django application, including alert response procedures.

## 🔧 Monitoring Tools Setup

### 1. System Monitoring (Prometheus + Grafana)

**Installation:**
```bash
# Install Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
tar xvf prometheus-*.tar.gz
cd prometheus-*
sudo mv prometheus promtool /usr/local/bin/

# Install Grafana
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana
```

**Configuration:**
```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'django-app'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics/'
    
  - job_name: 'nginx'
    static_configs:
      - targets: ['localhost:9113']
      
  - job_name: 'mysql'
    static_configs:
      - targets: ['localhost:9104']
```

### 2. Application Monitoring (Django)

**Install Django monitoring packages:**
```bash
pip install django-prometheus
pip install django-health-check
```

**Add to Django settings:**
```python
# settings.py
INSTALLED_APPS += [
    'django_prometheus',
    'health_check',
    'health_check.db',
    'health_check.cache',
    'health_check.storage',
]

MIDDLEWARE = [
    'django_prometheus.middleware.PrometheusBeforeMiddleware',
    # ... other middleware
    'django_prometheus.middleware.PrometheusAfterMiddleware',
]

# Health check endpoints
HEALTH_CHECK = {
    'DISK_USAGE_MAX': 90,  # percentage
    'MEMORY_MIN': 100,     # in MB
}
```

**Add monitoring URLs:**
```python
# urls.py
from django.urls import path, include
from django_prometheus import exports

urlpatterns = [
    path('health/', include('health_check.urls')),
    path('metrics/', exports.ExportToDjangoView, name='prometheus-django-metrics'),
]
```

### 3. Log Monitoring (ELK Stack)

**Install Elasticsearch:**
```bash
# Add Elasticsearch repository
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update
sudo apt install elasticsearch
```

**Install Logstash:**
```bash
sudo apt install logstash
```

**Install Kibana:**
```bash
sudo apt install kibana
```

## 🚨 Alert Configuration

### 1. Prometheus Alert Rules

```yaml
# /etc/prometheus/alerts.yml
groups:
  - name: hiringdog-alerts
    rules:
      - alert: DjangoDown
        expr: up{job="django-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Django application is down"
          description: "Django application has been down for more than 1 minute"
          
      - alert: HighResponseTime
        expr: http_request_duration_seconds{job="django-app"} > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "Response time is above 2 seconds for 5 minutes"
          
      - alert: DatabaseConnectionFailed
        expr: mysql_up == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Database connection failed"
          description: "MySQL database is not accessible"
          
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage is above 80% for 5 minutes"
```

### 2. Alertmanager Configuration

```yaml
# /etc/alertmanager/alertmanager.yml
global:
  smtp_smarthost: 'localhost:587'
  smtp_from: 'alerts@hiringdog.com'
  smtp_auth_username: 'alerts@hiringdog.com'
  smtp_auth_password: 'password'

route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'team-hiringdog'

receivers:
  - name: 'team-hiringdog'
    email_configs:
      - to: 'oncall@hiringdog.com'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
```

## 📱 Alert Response Procedures

### 1. Critical Alerts (Immediate Response Required)

**Django Application Down:**
```bash
# Immediate actions
1. Check application status: sudo systemctl status hiringdog-django
2. Check logs: sudo journalctl -u hiringdog-django -n 50
3. Restart if needed: sudo systemctl restart hiringdog-django
4. Verify recovery: curl -I http://localhost:8000/health/
5. Notify team via Slack/Teams
```

**Database Connection Failed:**
```bash
# Immediate actions
1. Check database status: sudo systemctl status mysql
2. Test connection: mysql -u root -p -e "SELECT 1;"
3. Restart database: sudo systemctl restart mysql
4. Check Django connection: python manage.py dbshell
5. Escalate if unresolved in 10 minutes
```

**High CPU/Memory Usage:**
```bash
# Immediate actions
1. Check system resources: top, free -h
2. Identify resource-intensive processes: ps aux --sort=-%cpu
3. Restart services if needed: sudo systemctl restart hiringdog-django
4. Monitor for 10 minutes
5. Escalate if usage remains high
```

### 2. Warning Alerts (Monitor and Investigate)

**High Response Time:**
```bash
# Investigation steps
1. Check application logs for errors
2. Monitor database performance
3. Check for slow queries
4. Review recent deployments
5. Update incident status
```

**Disk Space Low:**
```bash
# Investigation steps
1. Check disk usage: df -h
2. Identify large files: du -sh /* | sort -hr
3. Clean up logs: sudo find /var/log -name "*.log.*" -mtime +7 -delete
4. Monitor growth rate
5. Plan capacity increase if needed
```

### 3. Alert Escalation Matrix

| Alert Type | Initial Response | Escalation Time | Escalation Level |
|------------|------------------|-----------------|------------------|
| Critical | 5 minutes | 15 minutes | Senior Engineer |
| Warning | 15 minutes | 30 minutes | On-call Engineer |
| Info | 30 minutes | 60 minutes | Team Lead |

## 📊 Dashboard Configuration

### 1. Grafana Dashboard Setup

**System Overview Dashboard:**
```json
{
  "dashboard": {
    "title": "HiringDog System Overview",
    "panels": [
      {
        "title": "Application Status",
        "type": "stat",
        "targets": [
          {
            "expr": "up{job=\"django-app\"}",
            "legendFormat": "Django Status"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "http_request_duration_seconds{job=\"django-app\"}",
            "legendFormat": "Response Time"
          }
        ]
      },
      {
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "100 - (avg by(instance) (irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "CPU %"
          }
        ]
      },
      {
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100",
            "legendFormat": "Memory %"
          }
        ]
      }
    ]
  }
}
```

### 2. Custom Metrics Collection

**Django Custom Metrics:**
```python
# monitoring.py
from prometheus_client import Counter, Histogram, Gauge
from django.http import HttpResponse

# Custom metrics
request_count = Counter('django_http_requests_total', 'Total HTTP requests', ['method', 'endpoint'])
request_latency = Histogram('django_http_request_duration_seconds', 'HTTP request latency')
active_users = Gauge('django_active_users', 'Number of active users')

# Middleware to collect metrics
class MetricsMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start_time = time.time()
        
        response = self.get_response(request)
        
        duration = time.time() - start_time
        request_latency.observe(duration)
        request_count.labels(method=request.method, endpoint=request.path).inc()
        
        return response
```

## 🔍 Health Check Endpoints

### 1. Application Health Checks

```python
# health_checks.py
from health_check.backends import BaseHealthCheckBackend
from django.db import connection

class DatabaseHealthCheck(BaseHealthCheckBackend):
    critical_service = True

    def check_status(self):
        try:
            cursor = connection.cursor()
            cursor.execute("SELECT 1")
            cursor.fetchone()
        except Exception as e:
            self.add_error(ServiceUnavailable("Database connection failed"))

class CeleryHealthCheck(BaseHealthCheckBackend):
    critical_service = True

    def check_status(self):
        try:
            from hiringdogbackend.celery import app
            inspect = app.control.inspect()
            stats = inspect.stats()
            if not stats:
                self.add_error(ServiceUnavailable("No Celery workers available"))
        except Exception as e:
            self.add_error(ServiceUnavailable("Celery health check failed"))
```

### 2. External Service Health Checks

```python
# external_health_checks.py
import requests
from health_check.backends import BaseHealthCheckBackend

class PaymentGatewayHealthCheck(BaseHealthCheckBackend):
    def check_status(self):
        try:
            response = requests.get('https://api.payment-gateway.com/health', timeout=5)
            if response.status_code != 200:
                self.add_error(ServiceUnavailable("Payment gateway unhealthy"))
        except Exception as e:
            self.add_error(ServiceUnavailable("Payment gateway unreachable"))

class EmailServiceHealthCheck(BaseHealthCheckBackend):
    def check_status(self):
        try:
            response = requests.get('https://api.email-service.com/health', timeout=5)
            if response.status_code != 200:
                self.add_error(ServiceUnavailable("Email service unhealthy"))
        except Exception as e:
            self.add_error(ServiceUnavailable("Email service unreachable"))
```

## 📝 Incident Documentation

### 1. Incident Log Template

```markdown
## Incident Report

**Date/Time:** [Timestamp]
**Severity:** [Critical/High/Medium/Low]
**Alert Type:** [Alert Name]
**Affected Services:** [List services]

### Initial Assessment
- [ ] Alert received at [time]
- [ ] Initial investigation started
- [ ] Root cause identified
- [ ] Resolution implemented

### Actions Taken
1. [Action 1]
2. [Action 2]
3. [Action 3]

### Resolution
- **Time to Detection:** [Duration]
- **Time to Resolution:** [Duration]
- **Root Cause:** [Description]

### Lessons Learned
- [Improvement 1]
- [Improvement 2]

### Follow-up Actions
- [ ] Update monitoring rules
- [ ] Review runbook
- [ ] Schedule post-mortem
```

## 📊 Performance Baselines

| Metric | Normal Range | Warning Threshold | Critical Threshold |
|--------|--------------|-------------------|-------------------|
| Response Time | < 500ms | 1-2s | > 2s |
| CPU Usage | < 70% | 70-80% | > 80% |
| Memory Usage | < 80% | 80-90% | > 90% |
| Disk Usage | < 80% | 80-90% | > 90% |
| Database Connections | < 150 | 150-180 | > 180 |

---

*Last Updated: [Date]*
*Maintained by: [Team Name]*
