# Database Failures Runbook

## 📌 Overview

This runbook covers monitoring alerts and incident resolution procedures for Google Cloud SQL, including:
- High CPU utilization
- High memory usage
- Instance unavailability or downtime

## 🔍 Symptoms

- Alert received via Stackdriver / Monitoring:
  - `Cloud SQL CPU utilization > 85%`
  - `Cloud SQL memory usage > 90%`
  - `Cloud SQL instance not responding`
- Application logs showing database connection failures
- 5xx errors in API/Frontend logs due to database issues

## 🧪 Diagnosis

### Check Database Service Status

Steps:

1. Go to GCP Console → SQL
2. Click on your instance name
Review:
  - Instance status (RUNNING / STOPPED / ERROR)
  - Connection count
  - CPU/Memory usage

### **Check Monitoring Dashboards**
1. Go to **GCP Console → Monitoring → Metrics Explorer**
2. In the resource dropdown, select: `Cloud SQL Database`
3. Set filters:
   - **Instance ID**
4. View metrics:
   - CPU utilization
   - Memory usage
   - Disk usage
   - Active connections

### **Check Cloud SQL Logs**
- GCP → Logging → Filter by `resource.type="cloudsql_database"`
- Look for:
  - `Out of memory (OOM) errors` `Connection failures` `too many connections` `Restart` events
 
### **Connectivity Check**
- Run the following from a GCE instance:
  ```bash
  nc -zv [INSTANCE_IP] 3306
  ```

## 🛠️ Common Failure Scenarios

### Scenario 1: Database Service Not Starting

**Symptoms:**
- `cloud mysql` shows failed
- Cannot connect to database
- Application shows database connection errors

**Resolution:**
Resolution Steps:
- Check instance logs via Logging
- Restart instance
- If misconfiguration is suspected, clone from backup or failover to replica


### Scenario 2: Connection Limit Exceeded

**Symptoms:**
- "Too many connections" errors
- Slow response times
- Connection timeouts

**Resolution:**
- Go to SQL → Instance → Connections
    - Increase connection limit in DB flags: For MySQL: `max_connections`
    - Kill idle or long-running connections:

### Scenario 3: Database Corruption

**Symptoms:**
- Inconsistent data
- "Table is marked as crashed" errors
- Data integrity issues

**Resolution:**
- Restore from last known good backup
- GCP Console → SQL → Backups → Restore
- Check for auto-backup status and retention settings

### Scenario 4: Performance Issues

**Symptoms:**
- Slow query execution
- High CPU usage by database
- Timeout errors

**Resolution:**
GCP Console → SQL → Insights (if enabled)
- Check slow queries
- Optimize queries, add indexes
- Consider upgrading machine type


### Scenario 5: Disk Space Issues

**Symptoms:**
- "Disk full" errors
- Database cannot write data
- Slow performance due to disk I/O

**Resolution:**
GCP Console → SQL → Edit → Increase storage size (manual or enable auto storage increase)
- Enable auto-growth in Settings → Storage
- Cleanup unused data/logs from DB

## 🔧 Advanced Troubleshooting

### Debug Database Queries
Enable and use Cloud SQL Insights for slow query analysis.

### Check Django application Database Settings
Ensure DATABASES config is optimized for pooling and retries.


# Check database size
``` sql
SELECT table_schema "DB Name", SUM(data_length + index_length)/1024/1024 "Size (MB)"
FROM information_schema.tables GROUP BY table_schema;
```

## 🚨 Emergency Procedures

### Quick Recovery
GCP Console → SQL → Instance → Backups → "Create Backup"

### Backup and Restore
SQL → Backups → Select a backup → Click Restore

## 🚨 Escalation

### When to Escalate:
- Database down for >15 minutes
- Data corruption suspected
- Performance issues affecting users
- Backup/restore required

## 📝 Post-Incident Actions

1. **Document the incident** in incident log
2. **Update runbook** with new findings
3. **Review database configuration** for improvements
4. **Implement monitoring** for detected issues

## 🔗 Related Runbooks

- [Gunicorn Failures](./gunicorn-failures.md)
- [Celery Failures](./celery-failures.md)
- [Server Resources](./server-resources.md)

