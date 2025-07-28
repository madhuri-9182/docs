# Server Resources Runbook

## 🚨 Severity Levels

- **Critical**: Server unresponsive, services failing
- **High**: Severe performance degradation, affecting users
- **Medium**: Performance issues, some functionality affected
- **Low**: Minor resource constraints

## 🔍 Initial Assessment

### 1. Check System Resources
```bash
# Check CPU usage
top
htop
mpstat 1 5

# Check memory usage
free -h
cat /proc/meminfo
vmstat 1 5

# Check disk usage
df -h
df -i
iostat -x 1 5

# Check load average
uptime
cat /proc/loadavg
```

### 2. Check System Logs
```bash
# Check system messages
sudo dmesg | tail -20
sudo journalctl -f --since "10 minutes ago"

# Check for OOM killer activity
sudo grep -i "killed process" /var/log/syslog
sudo grep -i "out of memory" /var/log/syslog

# Check for disk I/O errors
sudo dmesg | grep -i "i/o error"
```

### 3. Check Process Status
```bash
# Check running processes
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10

# Check systemd services
sudo systemctl list-units --failed
sudo systemctl status hiringdog-django hiringdog-celery nginx mysql
```

## 🛠️ Common Failure Scenarios

### Scenario 1: High CPU Usage

**Symptoms:**
- System slow to respond
- High load average
- Processes consuming excessive CPU

**Diagnosis:**
```bash
# Identify CPU-intensive processes
ps aux --sort=-%cpu | head -10
top -p $(pgrep -d',' -f "python|nginx|mysql")

# Check CPU load
uptime
cat /proc/loadavg

# Check CPU usage by core
mpstat -P ALL 1 5
```

**Resolution:**
```bash
# Restart resource-intensive services
sudo systemctl restart hiringdog-django
sudo systemctl restart hiringdog-celery

# Kill runaway processes (if safe)
sudo kill -9 [process_id]

# Check for infinite loops in application code
# Review recent deployments
```

### Scenario 2: Memory Exhaustion

**Symptoms:**
- "Out of memory" errors
- Processes being killed by OOM killer
- Swap usage high

**Diagnosis:**
```bash
# Check memory usage
free -h
cat /proc/meminfo | grep -E "(MemTotal|MemFree|MemAvailable)"

# Check swap usage
swapon --show
cat /proc/swaps

# Check for memory leaks
ps aux --sort=-%mem | head -10
```

**Resolution:**
```bash
# Clear page cache (if safe)
sudo sync && sudo echo 3 > /proc/sys/vm/drop_caches

# Restart memory-intensive services
sudo systemctl restart hiringdog-django
sudo systemctl restart hiringdog-celery

# Increase swap if needed
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Scenario 3: Disk Space Issues

**Symptoms:**
- "No space left on device" errors
- Services failing to write logs
- Slow disk I/O

**Diagnosis:**
```bash
# Check disk usage
df -h
df -i

# Find large files
sudo find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | head -10

# Check for large log files
sudo find /var/log -name "*.log" -exec ls -lh {} \; | sort -k5 -hr | head -10
```

**Resolution:**
```bash
# Clean up old log files
sudo find /var/log -name "*.log.*" -mtime +7 -delete
sudo find /var/log -name "*.gz" -mtime +30 -delete

# Clean up temporary files
sudo rm -rf /tmp/*
sudo rm -rf /var/tmp/*

# Rotate logs
sudo logrotate -f /etc/logrotate.conf
```

### Scenario 4: Disk I/O Issues

**Symptoms:**
- Slow system response
- High iowait in CPU stats
- Database performance issues

**Diagnosis:**
```bash
# Check I/O statistics
iostat -x 1 5
iotop

# Check for I/O errors
sudo dmesg | grep -i "i/o error"
sudo smartctl -a /dev/sda
```

**Resolution:**
```bash
# Check disk health
sudo smartctl -H /dev/sda
sudo badblocks -v /dev/sda

# Optimize I/O scheduling
echo 'deadline' | sudo tee /sys/block/sda/queue/scheduler

# If hardware issue, prepare for disk replacement
```

### Scenario 5: Network Issues

**Symptoms:**
- Slow network response
- Connection timeouts
- High network latency

**Diagnosis:**
```bash
# Check network interfaces
ip addr show
ip route show

# Check network statistics
ss -tuln
netstat -i

# Test network connectivity
ping -c 5 google.com
traceroute google.com
```

**Resolution:**
```bash
# Restart network services
sudo systemctl restart networking
sudo systemctl restart systemd-networkd

# Check network configuration
sudo nano /etc/network/interfaces
sudo nano /etc/systemd/network/*.network
```

## 🔧 Advanced Troubleshooting

### Performance Monitoring
```bash
# Real-time monitoring script
#!/bin/bash
while true; do
    echo "=== $(date) ==="
    echo "Load: $(uptime | awk '{print $10 $11 $12}')"
    echo "CPU: $(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)%"
    echo "Memory: $(free | grep Mem | awk '{printf "%.2f%%", $3/$2 * 100.0}')"
    echo "Disk: $(df / | tail -1 | awk '{print $5}')"
    echo "---"
    sleep 30
done
```

### Resource Optimization
```bash
# Optimize system parameters
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
echo 'vm.dirty_ratio=15' | sudo tee -a /etc/sysctl.conf
echo 'vm.dirty_background_ratio=5' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Process Management
```bash
# Monitor specific processes
watch -n 1 'ps aux | grep -E "(python|nginx|mysql)"'

# Check process tree
pstree -p $(pgrep -f "hiringdog")

# Monitor file descriptors
lsof -p $(pgrep -f "hiringdog-django") | wc -l
```

## 📊 Monitoring Commands

```bash
# Health check script
#!/bin/bash
# Check CPU usage
cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
if (( $(echo "$cpu_usage > 80" | bc -l) )); then
    echo "High CPU usage: ${cpu_usage}%"
    # Send alert
fi

# Check memory usage
mem_usage=$(free | grep Mem | awk '{printf "%.2f", $3/$2 * 100.0}')
if (( $(echo "$mem_usage > 90" | bc -l) )); then
    echo "High memory usage: ${mem_usage}%"
    # Send alert
fi

# Check disk usage
disk_usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$disk_usage" -gt 90 ]; then
    echo "High disk usage: ${disk_usage}%"
    # Send alert
fi
```

## 🚨 Emergency Procedures

### Quick Recovery
```bash
# Emergency resource cleanup
sudo sync
sudo echo 3 > /proc/sys/vm/drop_caches

# Restart critical services
sudo systemctl restart hiringdog-django
sudo systemctl restart hiringdog-celery
sudo systemctl restart nginx

# Check system stability
uptime
free -h
df -h
```

### Resource Limits
```bash
# Set resource limits for services
sudo nano /etc/systemd/system/hiringdog-django.service
# Add:
# LimitCPU=200%
# LimitMEMLOCK=infinity
# LimitNOFILE=65536

sudo systemctl daemon-reload
sudo systemctl restart hiringdog-django
```

## 📝 Configuration Best Practices

### System Tuning
```bash
# Optimize for web server
echo 'net.core.somaxconn = 65535' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv4.tcp_max_syn_backlog = 65535' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv4.tcp_fin_timeout = 30' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Service Limits
```bash
# Set limits in /etc/security/limits.conf
hiringdog soft nofile 65536
hiringdog hard nofile 65536
hiringdog soft nproc 32768
hiringdog hard nproc 32768
```

## 🚨 Escalation

### When to Escalate:
- Server unresponsive for >10 minutes
- Multiple services failing due to resources
- Hardware failure suspected
- Performance issues affecting all users

### Escalation Steps:
1. Notify on-call engineer
2. Update incident status
3. Contact senior engineer if unresolved in 20 minutes
4. Contact system administrator if unresolved in 40 minutes

## 📝 Post-Incident Actions

1. **Document the incident** in incident log
2. **Update runbook** with new findings
3. **Review resource allocation** for improvements
4. **Implement monitoring** for detected issues

## 🔗 Related Runbooks

- [Django Failures](./django-failures.md)
- [Database Failures](./database-failures.md)
- [Celery Failures](./celery-failures.md)

---

*Last Updated: [Date]*
*Maintained by: [Team Name]*
