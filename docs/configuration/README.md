# Configuration Guide

This guide covers all configuration aspects of Hospital360 system.

## 📋 Table of Contents

- [Environment Configuration](#environment-configuration)
- [Database Configuration](#database-configuration)
- [Security Settings](#security-settings)
- [Email and Notifications](#email-and-notifications)
- [File Storage Configuration](#file-storage-configuration)
- [API Configuration](#api-configuration)
- [Performance Tuning](#performance-tuning)
- [Logging Configuration](#logging-configuration)
- [Backup Configuration](#backup-configuration)

## 🌍 Environment Configuration

### Environment Variables

Create and configure your `.env` file with the following variables:

```env
# Application Settings
NODE_ENV=production
APP_NAME=Hospital360
APP_URL=https://your-hospital360-domain.com
APP_PORT=3000
APP_KEY=your-unique-32-character-secret-key

# Database Configuration
DB_CONNECTION=postgresql
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=hospital360_db
DB_USERNAME=hospital360
DB_PASSWORD=your_secure_database_password

# Redis Configuration (for caching and sessions)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your_redis_password
REDIS_DB=0

# JWT Authentication
JWT_SECRET=your-jwt-secret-key-minimum-32-characters
JWT_EXPIRE=7d
REFRESH_TOKEN_EXPIRE=30d

# Email Configuration
MAIL_DRIVER=smtp
MAIL_HOST=your.smtp.server.com
MAIL_PORT=587
MAIL_USERNAME=noreply@yourhospital.com
MAIL_PASSWORD=your_smtp_password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@yourhospital.com
MAIL_FROM_NAME=Hospital360

# File Storage
STORAGE_DRIVER=local
STORAGE_PATH=/var/lib/hospital360/storage
MAX_FILE_SIZE=10485760  # 10MB in bytes
ALLOWED_FILE_TYPES=jpg,jpeg,png,pdf,doc,docx

# External Services
PAYMENT_GATEWAY_URL=https://api.paymentprovider.com
PAYMENT_GATEWAY_KEY=your_payment_gateway_key
INSURANCE_API_URL=https://api.insuranceprovider.com
INSURANCE_API_KEY=your_insurance_api_key

# Security Settings
CORS_ORIGIN=https://your-frontend-domain.com
RATE_LIMIT_MAX=100
RATE_LIMIT_WINDOW=900000  # 15 minutes in milliseconds
SESSION_LIFETIME=1800     # 30 minutes in seconds
PASSWORD_RESET_EXPIRE=3600 # 1 hour in seconds

# SSL/TLS Configuration
SSL_CERT_PATH=/etc/ssl/certs/hospital360.crt
SSL_KEY_PATH=/etc/ssl/private/hospital360.key

# Logging
LOG_LEVEL=info
LOG_PATH=/var/log/hospital360
LOG_MAX_SIZE=10m
LOG_MAX_FILES=5

# Backup Configuration
BACKUP_ENABLED=true
BACKUP_SCHEDULE=0 2 * * *  # Daily at 2 AM
BACKUP_RETENTION_DAYS=30
BACKUP_STORAGE_PATH=/var/backups/hospital360

# Monitoring
MONITORING_ENABLED=true
HEALTH_CHECK_INTERVAL=300  # 5 minutes
PERFORMANCE_MONITORING=true
ERROR_REPORTING_URL=https://your-error-tracking-service.com
```

### Configuration Validation

Validate your configuration with:

```bash
npm run config:validate
```

## 🗄️ Database Configuration

### PostgreSQL Configuration

#### Main Database Configuration

```javascript
// config/database.js
module.exports = {
  development: {
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_DATABASE,
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    dialect: 'postgres',
    logging: console.log,
    pool: {
      max: 5,
      min: 0,
      acquire: 30000,
      idle: 10000
    }
  },
  production: {
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_DATABASE,
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    dialect: 'postgres',
    logging: false,
    pool: {
      max: 20,
      min: 5,
      acquire: 30000,
      idle: 10000
    },
    ssl: {
      rejectUnauthorized: false
    }
  }
};
```

#### PostgreSQL Server Configuration

Edit `/etc/postgresql/14/main/postgresql.conf`:

```conf
# Connection Settings
listen_addresses = '*'
port = 5432
max_connections = 200
shared_buffers = 256MB

# Memory Settings
effective_cache_size = 1GB
work_mem = 4MB
maintenance_work_mem = 64MB

# WAL Settings
wal_buffers = 16MB
checkpoint_completion_target = 0.9
checkpoint_timeout = 10min

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_statement = 'mod'
log_min_duration_statement = 1000

# Performance
random_page_cost = 1.1
effective_io_concurrency = 200
```

#### Database Security Configuration

Edit `/etc/postgresql/14/main/pg_hba.conf`:

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                peer
local   hospital360_db  hospital360                             md5
host    hospital360_db  hospital360     127.0.0.1/32           md5
host    hospital360_db  hospital360     ::1/128                md5
```

### Database Connection Pooling

```javascript
// config/connectionPool.js
const { Pool } = require('pg');

const pool = new Pool({
  user: process.env.DB_USERNAME,
  host: process.env.DB_HOST,
  database: process.env.DB_DATABASE,
  password: process.env.DB_PASSWORD,
  port: process.env.DB_PORT,
  max: 20,                    // Maximum number of clients
  min: 5,                     // Minimum number of clients
  idleTimeoutMillis: 30000,   // How long a client is allowed to remain idle
  connectionTimeoutMillis: 2000, // How long to wait when connecting
  maxUses: 7500,              // Close connections after this many uses
});

pool.on('error', (err, client) => {
  console.error('Unexpected error on idle client', err);
  process.exit(-1);
});

module.exports = pool;
```

## 🔐 Security Settings

### Authentication Configuration

```javascript
// config/auth.js
module.exports = {
  // JWT Configuration
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRE || '7d',
    refreshExpiresIn: process.env.REFRESH_TOKEN_EXPIRE || '30d',
    issuer: 'hospital360',
    audience: 'hospital360-users'
  },
  
  // Password Policy
  password: {
    minLength: 8,
    requireUppercase: true,
    requireLowercase: true,
    requireNumbers: true,
    requireSpecialChars: true,
    maxAge: 90, // days
    preventReuse: 5 // number of previous passwords to check
  },
  
  // Account Lockout Policy
  lockout: {
    maxAttempts: 5,
    lockoutTime: 15 * 60 * 1000, // 15 minutes in milliseconds
    resetTime: 24 * 60 * 60 * 1000 // 24 hours
  },
  
  // Session Configuration
  session: {
    lifetime: parseInt(process.env.SESSION_LIFETIME) || 1800, // 30 minutes
    extendOnActivity: true,
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    sameSite: 'strict'
  },
  
  // Two-Factor Authentication
  twoFactor: {
    enabled: true,
    issuer: 'Hospital360',
    window: 2, // Allow codes from 2 time periods before and after
    codeLength: 6,
    secretLength: 32
  }
};
```

### HTTPS Configuration

#### Nginx SSL Configuration

```nginx
# /etc/nginx/sites-available/hospital360
server {
    listen 80;
    server_name your-hospital360-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-hospital360-domain.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/your-hospital360-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-hospital360-domain.com/privkey.pem;
    
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE+AESGCM:ECDHE+AES256:ECDHE+AES128:!RSA:!aNULL:!MD5:!DSS;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # File Upload Limits
    client_max_body_size 50M;
    client_body_timeout 60s;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 300s;
        proxy_connect_timeout 75s;
    }
}
```

## 📧 Email and Notifications

### SMTP Configuration

```javascript
// config/mail.js
const nodemailer = require('nodemailer');

const mailConfig = {
  smtp: {
    host: process.env.MAIL_HOST,
    port: parseInt(process.env.MAIL_PORT),
    secure: process.env.MAIL_PORT === '465', // true for 465, false for other ports
    auth: {
      user: process.env.MAIL_USERNAME,
      pass: process.env.MAIL_PASSWORD
    },
    tls: {
      rejectUnauthorized: false
    }
  },
  from: {
    address: process.env.MAIL_FROM_ADDRESS,
    name: process.env.MAIL_FROM_NAME
  },
  templates: {
    path: './email-templates',
    options: {
      extension: 'hbs',
      layoutsDir: './email-templates/layouts',
      defaultLayout: 'main'
    }
  }
};

const transporter = nodemailer.createTransporter(mailConfig.smtp);

module.exports = { mailConfig, transporter };
```

### Notification Settings

```javascript
// config/notifications.js
module.exports = {
  // Email Notifications
  email: {
    enabled: true,
    appointmentReminder: {
      enabled: true,
      schedule: '24h', // Send 24 hours before appointment
      template: 'appointment-reminder'
    },
    labResults: {
      enabled: true,
      template: 'lab-results-available'
    },
    passwordReset: {
      enabled: true,
      template: 'password-reset',
      expiry: 3600 // 1 hour
    },
    welcomeEmail: {
      enabled: true,
      template: 'welcome-user'
    }
  },
  
  // SMS Notifications (if configured)
  sms: {
    enabled: false,
    provider: 'twilio', // or 'aws-sns'
    appointmentReminder: {
      enabled: false,
      schedule: '2h' // Send 2 hours before appointment
    }
  },
  
  // In-App Notifications
  inApp: {
    enabled: true,
    retention: 30, // days
    categories: [
      'appointment',
      'lab_results',
      'billing',
      'system'
    ]
  }
};
```

## 📁 File Storage Configuration

### Local Storage Configuration

```javascript
// config/storage.js
const multer = require('multer');
const path = require('path');

const storageConfig = {
  local: {
    destination: process.env.STORAGE_PATH || './storage/uploads',
    maxFileSize: parseInt(process.env.MAX_FILE_SIZE) || 10485760, // 10MB
    allowedMimeTypes: [
      'image/jpeg',
      'image/png',
      'image/gif',
      'application/pdf',
      'application/msword',
      'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
    ],
    filename: (req, file, cb) => {
      const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
      cb(null, file.fieldname + '-' + uniqueSuffix + path.extname(file.originalname));
    }
  }
};

const upload = multer({
  storage: multer.diskStorage(storageConfig.local),
  limits: {
    fileSize: storageConfig.local.maxFileSize
  },
  fileFilter: (req, file, cb) => {
    if (storageConfig.local.allowedMimeTypes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('File type not allowed'), false);
    }
  }
});

module.exports = { storageConfig, upload };
```

### AWS S3 Configuration (Optional)

```javascript
// config/s3.js
const AWS = require('aws-sdk');

const s3Config = {
  accessKeyId: process.env.AWS_ACCESS_KEY_ID,
  secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  region: process.env.AWS_REGION || 'us-east-1',
  bucket: process.env.AWS_S3_BUCKET,
  signedUrlExpiry: 3600 // 1 hour
};

const s3 = new AWS.S3(s3Config);

module.exports = { s3Config, s3 };
```

## 🔌 API Configuration

### Rate Limiting

```javascript
// config/rateLimiting.js
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const redis = require('redis');

const redisClient = redis.createClient({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT,
  password: process.env.REDIS_PASSWORD
});

const rateLimitConfig = {
  // General API rate limiting
  general: rateLimit({
    store: new RedisStore({
      client: redisClient,
      prefix: 'rl:general:'
    }),
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per windowMs
    message: {
      error: {
        code: 'RATE_LIMIT_EXCEEDED',
        message: 'Too many requests, please try again later.'
      }
    },
    standardHeaders: true,
    legacyHeaders: false
  }),
  
  // Authentication rate limiting
  auth: rateLimit({
    store: new RedisStore({
      client: redisClient,
      prefix: 'rl:auth:'
    }),
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 5, // limit each IP to 5 requests per windowMs
    message: {
      error: {
        code: 'AUTH_RATE_LIMIT_EXCEEDED',
        message: 'Too many authentication attempts, please try again later.'
      }
    }
  })
};

module.exports = rateLimitConfig;
```

### CORS Configuration

```javascript
// config/cors.js
const corsOptions = {
  origin: function (origin, callback) {
    const allowedOrigins = process.env.CORS_ORIGIN?.split(',') || ['http://localhost:3000'];
    
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  optionsSuccessStatus: 200,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With']
};

module.exports = corsOptions;
```

## ⚡ Performance Tuning

### Application Performance

```javascript
// config/performance.js
module.exports = {
  // Enable gzip compression
  compression: {
    enabled: true,
    level: 6,
    threshold: 1024
  },
  
  // Cache configuration
  cache: {
    enabled: true,
    defaultTTL: 300, // 5 minutes
    maxKeys: 10000,
    updateAgeOnGet: true
  },
  
  // Connection pooling
  database: {
    poolMin: 5,
    poolMax: 20,
    acquireTimeout: 30000,
    idleTimeout: 10000
  },
  
  // Request timeout
  timeout: {
    server: 30000, // 30 seconds
    database: 10000, // 10 seconds
    external: 15000 // 15 seconds for external API calls
  }
};
```

### Redis Caching

```javascript
// config/redis.js
const redis = require('redis');

const redisConfig = {
  host: process.env.REDIS_HOST || 'localhost',
  port: process.env.REDIS_PORT || 6379,
  password: process.env.REDIS_PASSWORD,
  db: process.env.REDIS_DB || 0,
  retryDelayOnFailover: 100,
  maxRetriesPerRequest: 3,
  lazyConnect: true,
  keepAlive: 30000,
  commandTimeout: 5000
};

const client = redis.createClient(redisConfig);

client.on('error', (err) => {
  console.error('Redis Client Error', err);
});

client.on('connect', () => {
  console.log('Connected to Redis');
});

module.exports = { redisConfig, client };
```

## 📊 Logging Configuration

### Winston Logger Configuration

```javascript
// config/logger.js
const winston = require('winston');
const path = require('path');

const loggerConfig = {
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'hospital360' },
  transports: [
    // Write all logs with level 'error' and below to error.log
    new winston.transports.File({
      filename: path.join(process.env.LOG_PATH || './logs', 'error.log'),
      level: 'error',
      maxsize: 10485760, // 10MB
      maxFiles: 5,
      tailable: true
    }),
    
    // Write all logs to combined.log
    new winston.transports.File({
      filename: path.join(process.env.LOG_PATH || './logs', 'combined.log'),
      maxsize: 10485760, // 10MB
      maxFiles: 5,
      tailable: true
    })
  ]
};

// Add console transport in development
if (process.env.NODE_ENV !== 'production') {
  loggerConfig.transports.push(new winston.transports.Console({
    format: winston.format.simple()
  }));
}

const logger = winston.createLogger(loggerConfig);

module.exports = logger;
```

## 💾 Backup Configuration

### Automated Backup Script

```bash
#!/bin/bash
# /usr/local/bin/hospital360-backup.sh

# Configuration
DB_NAME="hospital360_db"
DB_USER="hospital360"
BACKUP_DIR="/var/backups/hospital360"
STORAGE_DIR="/var/lib/hospital360/storage"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Database backup
echo "Starting database backup..."
pg_dump -U "$DB_USER" -h localhost "$DB_NAME" | gzip > "$BACKUP_DIR/db_backup_$DATE.sql.gz"

# File storage backup
echo "Starting file storage backup..."
tar -czf "$BACKUP_DIR/storage_backup_$DATE.tar.gz" -C "$STORAGE_DIR" .

# Application configuration backup
echo "Starting configuration backup..."
tar -czf "$BACKUP_DIR/config_backup_$DATE.tar.gz" /etc/hospital360 /etc/nginx/sites-available/hospital360

# Clean old backups
echo "Cleaning old backups..."
find "$BACKUP_DIR" -name "*.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed successfully!"

# Log backup completion
logger "Hospital360 backup completed successfully - $DATE"
```

### Backup Cron Job

```bash
# Add to crontab: crontab -e
# Daily backup at 2 AM
0 2 * * * /usr/local/bin/hospital360-backup.sh >> /var/log/hospital360-backup.log 2>&1
```

---

For advanced configuration options, contact your system administrator or refer to the specific component documentation.