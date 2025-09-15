# Troubleshooting Guide

This guide helps diagnose and resolve common issues with Hospital360.

## 📋 Table of Contents

- [General Troubleshooting](#general-troubleshooting)
- [Login and Authentication Issues](#login-and-authentication-issues)
- [Database Problems](#database-problems)
- [Performance Issues](#performance-issues)
- [Network and Connectivity](#network-and-connectivity)
- [File Upload Problems](#file-upload-problems)
- [API Integration Issues](#api-integration-issues)
- [Browser and Client Issues](#browser-and-client-issues)
- [System Monitoring](#system-monitoring)
- [Emergency Procedures](#emergency-procedures)

## 🔍 General Troubleshooting

### Basic Diagnostic Steps

1. **Check System Status**
   ```bash
   # Check if services are running
   sudo systemctl status hospital360
   sudo systemctl status postgresql
   sudo systemctl status nginx
   
   # Check service logs
   sudo journalctl -u hospital360 -f
   ```

2. **Verify Network Connectivity**
   ```bash
   # Test database connection
   pg_isready -h localhost -p 5432
   
   # Test API endpoint
   curl -I https://your-hospital360-domain.com/api/health
   
   # Check DNS resolution
   nslookup your-hospital360-domain.com
   ```

3. **Review Application Logs**
   ```bash
   # Application logs
   tail -f /var/log/hospital360/app.log
   
   # Error logs
   tail -f /var/log/hospital360/error.log
   
   # Access logs
   tail -f /var/log/nginx/access.log
   ```

### Log Analysis

#### Common Log Patterns

**Successful Request**
```
2024-01-15 10:30:15 INFO [RequestHandler] GET /api/v1/patients - 200 - 45ms
```

**Authentication Error**
```
2024-01-15 10:30:16 ERROR [AuthMiddleware] Authentication failed for user@hospital.com - Invalid token
```

**Database Error**
```
2024-01-15 10:30:17 ERROR [DatabaseService] Connection timeout - host: localhost, database: hospital360
```

**Performance Warning**
```
2024-01-15 10:30:18 WARN [PerformanceMonitor] Slow query detected - 2.5s - SELECT * FROM patients
```

## 🔐 Login and Authentication Issues

### Problem: Users Cannot Log In

#### Symptoms
- Login page displays error message
- Valid credentials rejected
- Redirect loops after login

#### Troubleshooting Steps

1. **Check User Account Status**
   ```sql
   SELECT id, email, status, failed_login_attempts, locked_at 
   FROM users 
   WHERE email = 'user@hospital.com';
   ```

2. **Verify Password Policy**
   ```bash
   # Check password requirements in logs
   grep "password validation" /var/log/hospital360/app.log
   ```

3. **Test Authentication Service**
   ```bash
   curl -X POST https://your-domain.com/api/auth/login \
     -H "Content-Type: application/json" \
     -d '{"email":"user@hospital.com","password":"password123"}'
   ```

#### Solutions

**Account Locked**
```sql
-- Unlock user account
UPDATE users 
SET status = 'active', failed_login_attempts = 0, locked_at = NULL 
WHERE email = 'user@hospital.com';
```

**Password Reset**
```bash
# Generate password reset token
npm run generate-reset-token --email=user@hospital.com
```

**Clear Session Data**
```bash
# Clear Redis sessions
redis-cli FLUSHDB
```

### Problem: Two-Factor Authentication Not Working

#### Symptoms
- 2FA codes not received
- Invalid 2FA code errors
- SMS/Email delivery failures

#### Troubleshooting Steps

1. **Check 2FA Configuration**
   ```bash
   # Verify SMTP settings
   grep "SMTP" /var/log/hospital360/app.log
   
   # Test email delivery
   npm run test-email --to=user@hospital.com
   ```

2. **Validate Time Synchronization**
   ```bash
   # Check server time
   date
   
   # Sync with NTP server
   sudo ntpdate -s time.nist.gov
   ```

#### Solutions

**Disable 2FA Temporarily**
```sql
UPDATE users 
SET two_factor_enabled = false 
WHERE email = 'user@hospital.com';
```

**Regenerate 2FA Secret**
```bash
npm run reset-2fa --email=user@hospital.com
```

## 🗄️ Database Problems

### Problem: Database Connection Failures

#### Symptoms
- "Cannot connect to database" errors
- Timeout errors
- Connection pool exhausted

#### Troubleshooting Steps

1. **Check Database Service**
   ```bash
   # Check PostgreSQL status
   sudo systemctl status postgresql
   
   # Check database connectivity
   psql -h localhost -U hospital360 -d hospital360_db -c "SELECT 1;"
   ```

2. **Review Connection Configuration**
   ```bash
   # Check connection string
   grep "DATABASE_URL" .env
   
   # Verify connection pool settings
   grep "pool" config/database.js
   ```

3. **Monitor Active Connections**
   ```sql
   SELECT count(*) as active_connections 
   FROM pg_stat_activity 
   WHERE state = 'active';
   
   SELECT pid, usename, state, query 
   FROM pg_stat_activity 
   WHERE state != 'idle';
   ```

#### Solutions

**Restart Database Service**
```bash
sudo systemctl restart postgresql
```

**Increase Connection Pool Size**
```javascript
// config/database.js
module.exports = {
  pool: {
    max: 20,        // Increase from default
    min: 2,
    acquire: 30000,
    idle: 10000
  }
};
```

**Kill Blocking Queries**
```sql
-- Find and terminate long-running queries
SELECT pid, query_start, query 
FROM pg_stat_activity 
WHERE state = 'active' AND query_start < now() - interval '5 minutes';

-- Terminate specific query
SELECT pg_terminate_backend(12345);
```

### Problem: Slow Database Performance

#### Symptoms
- Page load times > 5 seconds
- Query timeout errors
- High CPU/memory usage

#### Diagnosis

```sql
-- Find slow queries
SELECT query, mean_time, calls, total_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Check table statistics
SELECT schemaname, tablename, n_tup_ins, n_tup_upd, n_tup_del, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;

-- Analyze index usage
SELECT schemaname, tablename, indexname, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_tup_read DESC;
```

#### Solutions

**Add Database Indexes**
```sql
-- Create indexes for common queries
CREATE INDEX idx_patients_email ON patients(email);
CREATE INDEX idx_appointments_date ON appointments(appointment_date);
CREATE INDEX idx_medical_records_patient ON medical_records(patient_id);
```

**Update Table Statistics**
```sql
-- Update statistics
ANALYZE;

-- Reindex tables
REINDEX DATABASE hospital360_db;
```

## 🚀 Performance Issues

### Problem: Slow Application Response

#### Symptoms
- High response times (> 2 seconds)
- Timeouts on API requests
- UI becomes unresponsive

#### Diagnosis Tools

```bash
# Monitor system resources
top -p $(pgrep -f hospital360)
htop

# Check memory usage
free -h

# Monitor disk I/O
iotop

# Network monitoring
nethogs

# Application performance monitoring
npm run monitor:performance
```

#### Performance Optimization

**Enable Application Caching**
```javascript
// config/cache.js
const redis = require('redis');

const client = redis.createClient({
  host: 'localhost',
  port: 6379,
  password: process.env.REDIS_PASSWORD
});

// Cache frequently accessed data
const cachePatientData = async (patientId, data) => {
  await client.setex(`patient:${patientId}`, 300, JSON.stringify(data));
};
```

**Optimize Database Queries**
```javascript
// Use pagination for large datasets
const getPatients = async (page = 1, limit = 20) => {
  const offset = (page - 1) * limit;
  
  return await Patient.findAndCountAll({
    limit,
    offset,
    include: [{
      model: Insurance,
      attributes: ['provider', 'policy_number']
    }]
  });
};
```

**Enable Compression**
```javascript
// app.js
const compression = require('compression');
app.use(compression());
```

## 🌐 Network and Connectivity

### Problem: API Requests Failing

#### Symptoms
- 502 Bad Gateway errors
- Connection refused
- DNS resolution failures

#### Troubleshooting Steps

1. **Check Network Services**
   ```bash
   # Test API endpoint
   curl -v https://api.hospital360.com/health
   
   # Check SSL certificate
   openssl s_client -connect api.hospital360.com:443
   
   # Verify DNS resolution
   dig api.hospital360.com
   ```

2. **Review Proxy Configuration**
   ```bash
   # Check nginx configuration
   sudo nginx -t
   
   # Review proxy settings
   cat /etc/nginx/sites-available/hospital360
   ```

#### Solutions

**Restart Web Server**
```bash
sudo systemctl restart nginx
sudo systemctl restart hospital360
```

**Update SSL Certificate**
```bash
# Using Let's Encrypt
sudo certbot renew --nginx
```

**Fix DNS Issues**
```bash
# Update DNS records
# Contact your DNS provider to update A/CNAME records
```

## 📁 File Upload Problems

### Problem: File Uploads Failing

#### Symptoms
- Upload progress stuck at 0%
- "File too large" errors
- Upload timeouts

#### Troubleshooting Steps

1. **Check File Size Limits**
   ```bash
   # Check application limits
   grep "MAX_FILE_SIZE" .env
   
   # Check nginx limits
   grep "client_max_body_size" /etc/nginx/nginx.conf
   ```

2. **Verify Storage Space**
   ```bash
   # Check disk space
   df -h
   
   # Check upload directory permissions
   ls -la /var/lib/hospital360/uploads/
   ```

#### Solutions

**Increase Upload Limits**
```nginx
# /etc/nginx/sites-available/hospital360
server {
    client_max_body_size 50M;
    client_body_timeout 60s;
}
```

**Fix Permissions**
```bash
sudo chown -R hospital360:hospital360 /var/lib/hospital360/uploads/
sudo chmod -R 755 /var/lib/hospital360/uploads/
```

## 🔌 API Integration Issues

### Problem: Third-party API Integration Failing

#### Symptoms
- External API calls timing out
- Authentication errors with external services
- Data synchronization failures

#### Troubleshooting Steps

1. **Test External API Connectivity**
   ```bash
   # Test connection to external API
   curl -v https://external-api.com/health
   
   # Check API credentials
   curl -H "Authorization: Bearer API_KEY" https://external-api.com/test
   ```

2. **Review Integration Logs**
   ```bash
   grep "external_api" /var/log/hospital360/app.log
   ```

#### Solutions

**Update API Credentials**
```bash
# Update environment variables
export EXTERNAL_API_KEY=new_api_key
export EXTERNAL_API_SECRET=new_secret
```

**Implement Retry Logic**
```javascript
const retry = async (fn, retries = 3) => {
  try {
    return await fn();
  } catch (error) {
    if (retries > 0) {
      await new Promise(resolve => setTimeout(resolve, 1000));
      return retry(fn, retries - 1);
    }
    throw error;
  }
};
```

## 🌐 Browser and Client Issues

### Problem: Browser Compatibility Issues

#### Symptoms
- Features not working in specific browsers
- JavaScript errors in console
- UI elements not displaying correctly

#### Troubleshooting Steps

1. **Check Browser Console**
   - Open Developer Tools (F12)
   - Look for JavaScript errors in Console tab
   - Check Network tab for failed requests

2. **Verify Browser Support**
   ```html
   <!-- Check if modern browser features are available -->
   <script>
   if (!window.Promise || !window.fetch) {
     alert('Your browser is not supported. Please update to a modern browser.');
   }
   </script>
   ```

#### Solutions

**Update Browser**
- Recommend users update to latest browser version
- Provide browser compatibility matrix

**Add Polyfills**
```javascript
// Add polyfills for older browsers
import 'core-js/stable';
import 'regenerator-runtime/runtime';
```

## 📊 System Monitoring

### Health Check Endpoints

```bash
# Application health
curl https://your-domain.com/health

# Database health  
curl https://your-domain.com/health/db

# External services health
curl https://your-domain.com/health/external
```

### Monitoring Commands

```bash
# System resource usage
htop
iostat -x 1
vmstat 1

# Application logs
tail -f /var/log/hospital360/*.log

# Database performance
sudo -u postgres psql -c "SELECT * FROM pg_stat_activity;"
```

## 🚨 Emergency Procedures

### System Outage Response

1. **Immediate Assessment (0-15 minutes)**
   ```bash
   # Quick health check
   systemctl status hospital360 postgresql nginx
   
   # Check critical errors
   tail -50 /var/log/hospital360/error.log
   ```

2. **Emergency Recovery (15-60 minutes)**
   ```bash
   # Restart all services
   sudo systemctl restart hospital360 postgresql nginx redis
   
   # Switch to backup database if needed
   # (Follow your disaster recovery plan)
   ```

3. **Communication**
   - Notify system administrators
   - Update status page
   - Communicate with users

### Emergency Contacts

- **System Administrator**: ext. 5555
- **Database Administrator**: ext. 5556  
- **Network Administrator**: ext. 5557
- **Vendor Support**: 1-800-SUPPORT

### Escalation Procedures

1. **Level 1**: Local IT team (0-30 minutes)
2. **Level 2**: Senior administrators (30-60 minutes)
3. **Level 3**: Vendor support (60+ minutes)

---

If you cannot resolve an issue using this guide, please contact technical support with:
- Error messages and logs
- Steps to reproduce the issue
- System configuration details
- Recent changes made to the system