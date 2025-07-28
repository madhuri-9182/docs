# Database Failures Runbook

## 🚨 Severity Levels

- **Critical**: Database completely down, no connections possible
- **High**: Major performance issues, affecting user experience
- **Medium**: Slow queries, some functionality affected
- **Low**: Minor performance degradation

## 🔍 Initial Assessment

### 1. Check Database Service Status
```bash
# Check MySQL status
sudo systemctl status mysql
sudo systemctl status mariadb

# Check PostgreSQL status
sudo systemctl status postgresql

# Check database processes
ps aux | grep mysql
ps aux | grep postgres
```

### 2. Check Database Connectivity
```bash
# Test MySQL connection
mysql -u root -p -e "SELECT 1;"
mysql -u hiringdog_user -p -e "SELECT 1;"

# Test PostgreSQL connection
sudo -u postgres psql -c "SELECT 1;"
psql -h localhost -U hiringdog_user -d hiringdog_db -c "SELECT 1;"

# Test Django database connection
python manage.py dbshell
python manage.py check --database default
```

### 3. Check Database Logs
```bash
# MySQL logs
sudo tail -f /var/log/mysql/error.log
sudo tail -f /var/log/mysql/mysql.log

# PostgreSQL logs
sudo tail -f /var/log/postgresql/postgresql-*.log

# Check for recent errors
sudo grep -i error /var/log/mysql/error.log | tail -20
```

## 🛠️ Common Failure Scenarios

### Scenario 1: Database Service Not Starting

**Symptoms:**
- `systemctl status mysql` shows failed
- Cannot connect to database
- Django shows database connection errors

**Diagnosis:**
```bash
# Check systemd service status
sudo systemctl status mysql

# Check service logs
sudo journalctl -u mysql -n 50

# Check for port conflicts
sudo netstat -tlnp | grep :3306
sudo netstat -tlnp | grep :5432

# Check disk space
df -h
```

**Resolution:**
```bash
# Restart database service
sudo systemctl restart mysql
# or
sudo systemctl restart postgresql

# If still failing, check configuration
sudo systemctl daemon-reload
sudo systemctl enable mysql
sudo systemctl restart mysql

# Check for corrupted data files
sudo mysqlcheck -u root -p --all-databases
```

### Scenario 2: Connection Limit Exceeded

**Symptoms:**
- "Too many connections" errors
- Slow response times
- Connection timeouts

**Diagnosis:**
```bash
# Check current connections
mysql -u root -p -e "SHOW PROCESSLIST;"
mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';"

# Check max connections
mysql -u root -p -e "SHOW VARIABLES LIKE 'max_connections';"
```

**Resolution:**
```bash
# Kill idle connections
mysql -u root -p -e "KILL QUERY [connection_id];"

# Increase max connections temporarily
mysql -u root -p -e "SET GLOBAL max_connections = 200;"

# Restart application to clear connection pool
sudo systemctl restart hiringdog-django
```

### Scenario 3: Database Corruption

**Symptoms:**
- Inconsistent data
- "Table is marked as crashed" errors
- Data integrity issues

**Diagnosis:**
```bash
# Check for table corruption
mysqlcheck -u root -p --all-databases --check

# Check specific database
mysqlcheck -u root -p hiringdog_db --check

# Check for errors in logs
sudo grep -i "corrupt\|crash" /var/log/mysql/error.log
```

**Resolution:**
```bash
# Repair corrupted tables
mysqlcheck -u root -p --all-databases --repair

# Repair specific database
mysqlcheck -u root -p hiringdog_db --repair

# If severe corruption, restore from backup
sudo systemctl stop hiringdog-django
# Restore from latest backup
sudo systemctl start hiringdog-django
```

### Scenario 4: Performance Issues

**Symptoms:**
- Slow query execution
- High CPU usage by database
- Timeout errors

**Diagnosis:**
```bash
# Check slow queries
mysql -u root -p -e "SHOW PROCESSLIST;"
mysql -u root -p -e "SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 10;"

# Check database performance
mysql -u root -p -e "SHOW STATUS LIKE 'Slow_queries';"
mysql -u root -p -e "SHOW STATUS LIKE 'Questions';"

# Check table sizes
mysql -u root -p -e "SELECT table_name, ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)' FROM information_schema.tables WHERE table_schema = 'hiringdog_db' ORDER BY (data_length + index_length) DESC;"
```

**Resolution:**
```bash
# Optimize tables
mysql -u root -p -e "OPTIMIZE TABLE hiringdog_db.table_name;"

# Analyze table statistics
mysql -u root -p -e "ANALYZE TABLE hiringdog_db.table_name;"

# Check and update indexes
mysql -u root -p -e "SHOW INDEX FROM hiringdog_db.table_name;"
```

### Scenario 5: Disk Space Issues

**Symptoms:**
- "Disk full" errors
- Database cannot write data
- Slow performance due to disk I/O

**Diagnosis:**
```bash
# Check disk space
df -h
df -i

# Check database size
mysql -u root -p -e "SELECT table_schema AS 'Database', ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)' FROM information_schema.tables GROUP BY table_schema;"

# Check log file sizes
ls -lh /var/log/mysql/
```

**Resolution:**
```bash
# Clean up old log files
sudo find /var/log/mysql -name "*.log.*" -mtime +7 -delete

# Rotate logs
sudo logrotate -f /etc/logrotate.d/mysql

# If critical, free up space immediately
sudo systemctl stop hiringdog-django
# Clean up temporary files
sudo systemctl start hiringdog-django
```

## 🔧 Advanced Troubleshooting

### Debug Database Queries
```bash
# Enable query logging
mysql -u root -p -e "SET GLOBAL general_log = 'ON';"
mysql -u root -p -e "SET GLOBAL log_output = 'TABLE';"

# Monitor queries in real-time
mysql -u root -p -e "SELECT * FROM mysql.general_log ORDER BY event_time DESC LIMIT 20;"
```

### Check Django Database Settings
```bash
# Test Django database configuration
python manage.py check --deploy

# Check database migrations
python manage.py showmigrations
python manage.py migrate --plan

# Test database connection from Django
python manage.py shell -c "
from django.db import connection
cursor = connection.cursor()
cursor.execute('SELECT 1')
print('Database connection OK')
"
```

### Monitor Database Performance
```bash
# Real-time monitoring script
#!/bin/bash
while true; do
    connections=$(mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';" | tail -1 | awk '{print $2}')
    slow_queries=$(mysql -u root -p -e "SHOW STATUS LIKE 'Slow_queries';" | tail -1 | awk '{print $2}')
    echo "$(date): Connections: $connections, Slow queries: $slow_queries"
    sleep 30
done
```

## 📊 Monitoring Commands

```bash
# Health check script
#!/bin/bash
# Test database connection
if mysql -u hiringdog_user -p -e "SELECT 1;" >/dev/null 2>&1; then
    echo "Database is healthy"
else
    echo "Database connection failed"
    # Send alert
fi

# Check database size
db_size=$(mysql -u root -p -e "SELECT ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)' FROM information_schema.tables WHERE table_schema = 'hiringdog_db';" | tail -1)
echo "Database size: ${db_size}MB"
```

## 🚨 Emergency Procedures

### Quick Recovery
```bash
# Emergency restart database
sudo systemctl stop mysql
sudo systemctl start mysql

# Check if database is accessible
mysql -u root -p -e "SELECT 1;"

# Restart Django application
sudo systemctl restart hiringdog-django
```

### Backup and Restore
```bash
# Create emergency backup
mysqldump -u root -p --all-databases > /tmp/emergency_backup_$(date +%Y%m%d_%H%M%S).sql

# Restore from backup
mysql -u root -p < /path/to/backup.sql
```

## 📝 Configuration Best Practices

### MySQL Configuration
```ini
# In /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
max_connections = 200
innodb_buffer_pool_size = 1G
innodb_log_file_size = 256M
slow_query_log = 1
long_query_time = 2
```

### Django Database Settings
```python
# In settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'hiringdog_db',
        'USER': 'hiringdog_user',
        'PASSWORD': 'secure_password',
        'HOST': 'localhost',
        'PORT': '3306',
        'OPTIONS': {
            'init_command': "SET sql_mode='STRICT_TRANS_TABLES'",
            'charset': 'utf8mb4',
        },
        'CONN_MAX_AGE': 60,
    }
}
```

## 🚨 Escalation

### When to Escalate:
- Database down for >15 minutes
- Data corruption suspected
- Performance issues affecting users
- Backup/restore required

### Escalation Steps:
1. Notify on-call engineer
2. Update incident status
3. Contact senior engineer if unresolved in 30 minutes
4. Contact database administrator if unresolved in 60 minutes

## 📝 Post-Incident Actions

1. **Document the incident** in incident log
2. **Update runbook** with new findings
3. **Review database configuration** for improvements
4. **Implement monitoring** for detected issues

## 🔗 Related Runbooks

- [Django Failures](./django-failures.md)
- [Celery Failures](./celery-failures.md)
- [Server Resources](./server-resources.md)

---

*Last Updated: [Date]*
*Maintained by: [Team Name]*
