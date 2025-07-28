# Nginx Failures Runbook

## 🚨 Severity Levels

- **Critical**: Nginx completely down, no web traffic
- **High**: Major routing issues, SSL problems
- **Medium**: Performance issues, some endpoints affected
- **Low**: Minor configuration issues

## 🔍 Initial Assessment

### 1. Check Nginx Status
```bash
# Check if Nginx is running
sudo systemctl status nginx
ps aux | grep nginx

# Check Nginx processes
ps aux | grep nginx | grep -v grep

# Check if Nginx is listening on ports
netstat -tlnp | grep nginx
lsof -i :80
lsof -i :443
```

### 2. Check Nginx Logs
```bash
# Access logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# Check for recent errors
sudo grep -i error /var/log/nginx/error.log | tail -20
sudo grep -i "emerg\|alert\|crit" /var/log/nginx/error.log
```

### 3. Test Nginx Configuration
```bash
# Test configuration syntax
sudo nginx -t

# Check configuration file
sudo nginx -T | head -20
```

## 🛠️ Common Failure Scenarios

### Scenario 1: Nginx Service Not Starting

**Symptoms:**
- `systemctl status nginx` shows failed
- Ports 80/443 not listening
- Website completely inaccessible

**Diagnosis:**
```bash
# Check systemd service status
sudo systemctl status nginx

# Check service logs
sudo journalctl -u nginx -n 50

# Test configuration
sudo nginx -t

# Check for port conflicts
sudo netstat -tlnp | grep :80
sudo netstat -tlnp | grep :443
```

**Resolution:**
```bash
# Restart Nginx
sudo systemctl restart nginx

# If still failing, check configuration
sudo nginx -t
sudo systemctl daemon-reload
sudo systemctl enable nginx
sudo systemctl restart nginx

# Check for missing files
sudo find /etc/nginx -name "*.conf" -exec nginx -t {} \;
```

### Scenario 2: SSL Certificate Issues

**Symptoms:**
- SSL certificate errors in browser
- Mixed content warnings
- Certificate expired errors

**Diagnosis:**
```bash
# Check SSL certificate
openssl s_client -connect your-domain.com:443 -servername your-domain.com

# Check certificate expiration
openssl x509 -in /path/to/certificate.crt -text -noout | grep -i "not after"

# Test SSL configuration
sudo nginx -t
```

**Resolution:**
```bash
# Renew SSL certificate (Let's Encrypt)
sudo certbot renew --nginx

# Update certificate manually
sudo cp new-certificate.crt /etc/ssl/certs/
sudo cp new-private-key.key /etc/ssl/private/
sudo systemctl reload nginx

# Check SSL configuration
sudo nginx -t
sudo systemctl reload nginx
```

### Scenario 3: Upstream Connection Issues

**Symptoms:**
- 502 Bad Gateway errors
- 504 Gateway Timeout errors
- Django application not reachable

**Diagnosis:**
```bash
# Check if Django is running
sudo systemctl status hiringdog-django
curl -I http://localhost:8000/health/

# Check Nginx upstream configuration
sudo nginx -T | grep -A 10 "upstream"

# Test upstream connection
curl -I http://127.0.0.1:8000/
```

**Resolution:**
```bash
# Restart Django application
sudo systemctl restart hiringdog-django

# Check upstream configuration in Nginx
sudo nano /etc/nginx/sites-available/hiringdog

# Reload Nginx configuration
sudo nginx -t
sudo systemctl reload nginx
```

### Scenario 4: Performance Issues

**Symptoms:**
- Slow response times
- High CPU usage by Nginx
- Connection timeouts

**Diagnosis:**
```bash
# Check Nginx performance
sudo nginx -V 2>&1 | grep -o with-http_stub_status_module

# Check worker processes
ps aux | grep nginx | grep worker

# Monitor real-time performance
sudo nginx -s status  # If status module enabled
```

**Resolution:**
```bash
# Optimize Nginx configuration
sudo nano /etc/nginx/nginx.conf

# Adjust worker processes
# worker_processes auto;
# worker_connections 1024;

# Enable gzip compression
# gzip on;
# gzip_types text/plain text/css application/json application/javascript;

# Reload configuration
sudo nginx -t
sudo systemctl reload nginx
```

### Scenario 5: File Permission Issues

**Symptoms:**
- 403 Forbidden errors
- Static files not serving
- Log files not writable

**Diagnosis:**
```bash
# Check file permissions
ls -la /var/log/nginx/
ls -la /etc/nginx/
ls -la /var/www/

# Check Nginx user
ps aux | grep nginx | head -1
```

**Resolution:**
```bash
# Fix log file permissions
sudo chown -R nginx:nginx /var/log/nginx/
sudo chmod 755 /var/log/nginx/

# Fix configuration file permissions
sudo chown root:root /etc/nginx/nginx.conf
sudo chmod 644 /etc/nginx/nginx.conf

# Fix static files permissions
sudo chown -R www-data:www-data /var/www/
sudo chmod -R 755 /var/www/
```

## 🔧 Advanced Troubleshooting

### Debug Nginx Configuration
```bash
# Show full configuration
sudo nginx -T

# Test specific configuration file
sudo nginx -t -c /etc/nginx/nginx.conf

# Check for syntax errors
sudo nginx -t 2>&1 | grep -i error
```

### Monitor Nginx Performance
```bash
# Real-time monitoring
sudo tail -f /var/log/nginx/access.log | grep -v health

# Check response times
sudo tail -f /var/log/nginx/access.log | awk '{print $10}' | sort -n

# Monitor error rates
sudo tail -f /var/log/nginx/error.log | grep -c "error"
```

### SSL/TLS Troubleshooting
```bash
# Test SSL configuration
sudo nginx -t
sudo openssl s_client -connect localhost:443 -servername your-domain.com

# Check SSL protocols
sudo nginx -V 2>&1 | grep -o with-http_ssl_module

# Test certificate chain
openssl x509 -in /path/to/certificate.crt -text -noout
```

## 📊 Monitoring Commands

```bash
# Health check script
#!/bin/bash
response=$(curl -s -o /dev/null -w "%{http_code}" https://your-domain.com/)
if [ $response -eq 200 ]; then
    echo "Nginx is healthy"
else
    echo "Nginx is unhealthy: $response"
    # Send alert
fi
```

## 🚨 Emergency Procedures

### Quick Restart
```bash
# Emergency restart
sudo systemctl stop nginx
sudo systemctl start nginx

# If still failing, check logs immediately
sudo journalctl -u nginx -n 20
```

### Fallback Configuration
```bash
# Use minimal configuration
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
sudo cp /etc/nginx/nginx.conf.minimal /etc/nginx/nginx.conf
sudo systemctl restart nginx
```

## 📝 Configuration Best Practices

### Security Headers
```nginx
# Add security headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

### Rate Limiting
```nginx
# Rate limiting configuration
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
limit_req zone=api burst=20 nodelay;
```

## 🚨 Escalation

### When to Escalate:
- Nginx down for >10 minutes
- SSL certificate expired
- Security breach indicators
- Multiple upstream services affected

### Escalation Steps:
1. Notify on-call engineer
2. Update incident status
3. Contact senior engineer if unresolved in 20 minutes
4. Contact system administrator if unresolved in 40 minutes

## 📝 Post-Incident Actions

1. **Document the incident** in incident log
2. **Update runbook** with new findings
3. **Review Nginx configuration** for improvements
4. **Implement monitoring** for detected issues

## 🔗 Related Runbooks

- [Django Failures](./django-failures.md)
- [SSL Certificate Issues](./ssl-certificate-issues.md)
- [Server Resources](./server-resources.md)

---

*Last Updated: [Date]*
*Maintained by: [Team Name]*
