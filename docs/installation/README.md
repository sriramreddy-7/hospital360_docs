# Installation Guide

This guide provides step-by-step instructions for installing and deploying Hospital360.

## 📋 Table of Contents

- [System Requirements](#system-requirements)
- [Pre-installation Checklist](#pre-installation-checklist)
- [Installation Methods](#installation-methods)
- [Database Setup](#database-setup)
- [Application Configuration](#application-configuration)
- [Initial Setup](#initial-setup)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## 💻 System Requirements

### Minimum Requirements

- **Operating System**: 
  - Linux (Ubuntu 20.04 LTS or CentOS 8+)
  - Windows Server 2019+
  - macOS 10.15+ (for development only)
- **CPU**: 2 cores, 2.4 GHz
- **Memory**: 4 GB RAM
- **Storage**: 20 GB available disk space
- **Network**: Broadband internet connection

### Recommended Requirements

- **Operating System**: Ubuntu 22.04 LTS
- **CPU**: 4+ cores, 3.0+ GHz
- **Memory**: 8+ GB RAM
- **Storage**: 100+ GB SSD
- **Network**: Dedicated network connection
- **Backup**: Additional storage for backups

### Software Dependencies

- **Database**: PostgreSQL 12+ or MySQL 8.0+
- **Runtime**: Node.js 16+ LTS
- **Web Server**: Nginx 1.18+ or Apache 2.4+
- **SSL Certificate**: Required for production
- **Email Server**: SMTP server for notifications

## ✅ Pre-installation Checklist

Before starting the installation:

- [ ] Verify system meets minimum requirements
- [ ] Obtain necessary licenses and certificates
- [ ] Plan network configuration and security
- [ ] Prepare database server
- [ ] Configure backup solution
- [ ] Review security policies
- [ ] Prepare system administrator credentials

## 🚀 Installation Methods

### Method 1: Docker Installation (Recommended)

#### Prerequisites
```bash
# Install Docker and Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

#### Installation Steps
```bash
# Clone the repository
git clone https://github.com/hospital360/hospital360.git
cd hospital360

# Copy configuration template
cp .env.example .env

# Edit configuration (see Configuration section)
nano .env

# Start services
docker-compose up -d

# Run database migrations
docker-compose exec app npm run migrate

# Create initial admin user
docker-compose exec app npm run seed:admin
```

### Method 2: Manual Installation

#### Step 1: Install Node.js
```bash
# Ubuntu/Debian
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# CentOS/RHEL
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo yum install -y nodejs
```

#### Step 2: Install Database
```bash
# PostgreSQL (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install postgresql postgresql-contrib

# Start PostgreSQL service
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

#### Step 3: Clone and Setup Application
```bash
# Clone repository
git clone https://github.com/hospital360/hospital360.git
cd hospital360

# Install dependencies
npm install

# Build application
npm run build

# Copy configuration
cp config/config.example.json config/config.json
```

## 🗄️ Database Setup

### PostgreSQL Setup

1. **Create Database User**
```sql
sudo -u postgres psql
CREATE USER hospital360 WITH ENCRYPTED PASSWORD 'your_secure_password';
CREATE DATABASE hospital360_db OWNER hospital360;
GRANT ALL PRIVILEGES ON DATABASE hospital360_db TO hospital360;
\q
```

2. **Configure Database Access**
```bash
# Edit PostgreSQL configuration
sudo nano /etc/postgresql/12/main/pg_hba.conf

# Add line for application access
local   hospital360_db    hospital360                     md5

# Restart PostgreSQL
sudo systemctl restart postgresql
```

### MySQL Setup

1. **Create Database and User**
```sql
mysql -u root -p
CREATE DATABASE hospital360_db;
CREATE USER 'hospital360'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON hospital360_db.* TO 'hospital360'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## ⚙️ Application Configuration

### Environment Configuration

Edit the `.env` file with your specific settings:

```env
# Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=hospital360_db
DB_USER=hospital360
DB_PASSWORD=your_secure_password

# Application Settings
APP_PORT=3000
APP_ENV=production
SECRET_KEY=your_secret_key_here

# Email Configuration
SMTP_HOST=your.smtp.server
SMTP_PORT=587
SMTP_USER=noreply@yourhospital.com
SMTP_PASSWORD=smtp_password

# File Upload Settings
MAX_FILE_SIZE=10MB
UPLOAD_PATH=/var/lib/hospital360/uploads

# Security Settings
SESSION_TIMEOUT=30
ENABLE_2FA=true
PASSWORD_POLICY=strong
```

### SSL Configuration

For production environments, configure SSL:

```nginx
# /etc/nginx/sites-available/hospital360
server {
    listen 443 ssl http2;
    server_name yourdomain.com;
    
    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/private.key;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 🏁 Initial Setup

### Database Migration

```bash
# Run database migrations
npm run migrate

# Seed initial data
npm run seed
```

### Create Admin User

```bash
# Interactive admin user creation
npm run create-admin

# Or use command line
npm run create-admin -- --email admin@hospital.com --password SecurePass123 --name "System Admin"
```

### Start Application

```bash
# Development mode
npm run dev

# Production mode
npm run start

# With PM2 process manager (recommended for production)
npm install -g pm2
pm2 start ecosystem.config.js
pm2 save
pm2 startup
```

## ✅ Verification

### Health Check

Visit the following URLs to verify installation:

- **Application**: http://your-server:3000
- **Health Check**: http://your-server:3000/health
- **API Status**: http://your-server:3000/api/status

### Login Test

1. Navigate to the login page
2. Use the admin credentials created during setup
3. Verify you can access the dashboard
4. Check that all menu items are accessible

### Database Connection Test

```bash
# Test database connectivity
npm run test:db

# Verify all tables are created
npm run db:status
```

## 🔧 Troubleshooting

### Common Issues

#### Port Already in Use
```bash
# Check what's using port 3000
sudo lsof -i :3000

# Kill the process or change port in configuration
```

#### Database Connection Failed
```bash
# Check database service status
sudo systemctl status postgresql

# Test database connection
psql -h localhost -U hospital360 -d hospital360_db
```

#### Permission Errors
```bash
# Fix file permissions
sudo chown -R hospital360:hospital360 /opt/hospital360
sudo chmod -R 755 /opt/hospital360
```

#### Memory Issues
```bash
# Check available memory
free -h

# Monitor application memory usage
top -p $(pgrep -f hospital360)
```

### Log Files

Important log locations:

- **Application Logs**: `/var/log/hospital360/app.log`
- **Error Logs**: `/var/log/hospital360/error.log`
- **Database Logs**: `/var/log/postgresql/postgresql-12-main.log`
- **Web Server Logs**: `/var/log/nginx/access.log`

### Getting Help

If you encounter issues:

1. Check the [Troubleshooting Guide](../troubleshooting/README.md)
2. Review log files for error messages
3. Search the documentation for similar issues
4. Contact technical support with log details

---

**Next Steps**: After successful installation, proceed to the [Configuration Guide](../configuration/README.md) to customize your Hospital360 instance.