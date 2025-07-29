# Nginx Failures Runbook

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
sudo tail -f /var/log/nginx/hiringdogbackend_access.log

# Check for recent errors
sudo grep -i error /var/log/nginx/error.log | tail -20
sudo grep -i "emerg\|alert\|crit" /var/log/nginx/error.log
sudo grep -i error /var/log/nginx/hiringdogbackend_error.log | tail -20
```

### 3. Test Nginx Configuration
```bash
# Test configuration syntax
sudo nginx -t

# Check enabled sites
sudo nginx -T | grep -E "(server_name|listen)"
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
sudo systemctl status nginx --no-pager

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
echo | openssl s_client -servername api.hdiplatform.in -connect api.hdiplatform.in:443 2>/dev/null | openssl x509 -noout -dates

# Check certificate expiration
echo | openssl s_client -servername prod-api.hdiplatform.in -connect prod-api.hdiplatform.in:443 2>/dev/null | openssl x509 -noout -dates

# Test SSL configuration
sudo nginx -t

# Check certificate files
sudo ls -la /etc/ssl/certs/
sudo ls -la /etc/ssl/private/
```

**Resolution:**
```bash
# Renew SSL certificate (Let's Encrypt)
sudo certbot renew --nginx

# Update certificate manually
sudo cp new-certificate.crt /etc/ssl/certs/
sudo cp new-private-key.key /etc/ssl/private/
sudo systemctl reload nginx
sudo systemctl reload nginx

# Check SSL configuration
sudo nginx -t
sudo systemctl reload nginx

# Test SSL connection
openssl s_client -connect api.hdiplatform.in:443 -servername api.hdiplatform.in
```

### Scenario 3: Upstream Connection Issues              

**Symptoms:**
- 502 Bad Gateway errors
- 504 Gateway Timeout errors
- Django application not reachable

**Diagnosis:**
```bash
# Check if Django is running
sudo systemctl status gunicorn
curl -I http://localhost:8000/

# Check Nginx upstream configuration
sudo nginx -T | grep -A 10 "upstream"

# Test upstream connection
curl -I http://127.0.0.1:8000/
```

**Resolution:**
```bash
# Restart Django application
sudo systemctl restart gunicorn

# Check upstream configuration in Nginx
sudo nano /etc/nginx/sites-available/hiringdogbackend

# Reload Nginx configuration
sudo nginx -t
sudo systemctl reload nginx

# Test the fix
curl -I https://api.hdiplatform.in/
```

### Scenario 4: File Permission Issues

**Symptoms:**
- 403 Forbidden errors
- Static files not serving
- Log files not writable
**Diagnosis:**
```bash
# Check file permissions
sudo ls -la /var/log/nginx/
sudo ls -la /etc/nginx/
sudo ls -la /var/www/

# Check Nginx user
ps aux | grep nginx | head -1

# Check log file permissions
sudo ls -la /var/log/nginx/hiringdogbackend_*
```

**Resolution:**
```bash
# Fix log file permissions
sudo chown -R www-data:www-data /var/log/nginx/
sudo chmod 755 /var/log/nginx/

# Fix configuration file permissions
sudo chown root:root /etc/nginx/nginx.conf
sudo chmod 644 /etc/nginx/nginx.conf

# Fix static files permissions
sudo chown -R www-data:www-data /var/www/
sudo chmod -R 755 /var/www/

# Restart Nginx
sudo systemctl restart nginx
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

### Nginx Performance
```bash
# Real-time logs
sudo tail -f /var/log/nginx/hiringdogbackend_access.log

# Check response times
sudo tail -f /var/log/nginx/hiringdogbackend_access.log | awk '{print $10}' | sort -n

# Monitor error rates
sudo tail -f /var/log/nginx/hiringdogbackend_error.log | grep -c "error"
```
### SSL/TLS Troubleshooting
```bash
# Test SSL configuration
sudo nginx -t
sudo openssl s_client -connect localhost:443 -servername api.hdiplatform.in

# Check SSL protocols
sudo nginx -V 2>&1 | grep -o with-http_ssl_module

# Test certificate chain
echo | openssl s_client -servername api.hdiplatform.in -connect api.hdiplatform.in:443 2>/dev/null | openssl x509 -noout -text

# Check SSL configuration
sudo nginx -T | grep -A 10 -B 5 "ssl_"
```

## 🚨 Emergency Procedures

### Quick Restart
```bash
# Emergency restart
sudo systemctl stop nginx
sudo systemctl start nginx

# If still failing, check logs immediately
sudo journalctl -u nginx -n 20
sudo tail -n 20 /var/log/nginx/error.log
```

### Fallback Configuration
```bash
# Use minimal configuration
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
sudo cp /etc/nginx/nginx.conf.bck /etc/nginx/nginx.conf
sudo nginx -t
sudo systemctl restart nginx
```
