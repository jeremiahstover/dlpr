# Deployment Guide

This guide covers deploying Application to production environments.

## Overview

Application is a PHP application that can be deployed to any web server supporting PHP 8.1+. The application uses SQLite for data storage, eliminating the need for a separate database server.

**Deployment Methods:**
- **Traditional Hosting**: Apache/Nginx with PHP
- **VPS/Cloud**: DigitalOcean, AWS, Linode, etc.
- **Shared Hosting**: Any host supporting PHP 8.1+
- **Docker**: Containerized deployment (manual setup)

## Prerequisites

### Server Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| PHP | 8.1 | 8.2+ |
| Memory | 256MB | 512MB+ |
| Storage | 1GB | 5GB+ |
| SQLite | 3.0+ | Latest |

### PHP Extensions

Required extensions (typically enabled by default):
- `pdo_sqlite` - SQLite database access
- `json` - JSON encoding/decoding
- `mbstring` - Multi-byte string handling
- `openssl` - Encryption/hashing

Verify with:
```bash
php -m | grep -E "pdo_sqlite|json|mbstring|openssl"
```

## Pre-Deployment Checklist

### 1. Environment Configuration

Set environment variables for production:

```bash
# In your web server config or .htaccess
SetEnv APP_ENV production
SetEnv APP_DEBUG false
```

Or in PHP-FPM pool config:
```ini
; /etc/php/8.2/fpm/pool.d/www.conf
env[APP_ENV] = production
env[APP_DEBUG] = false
```

### 2. Application Mode

Ensure config is set to production mode:

```json
{
    "mode": "production"
}
```

### 3. Encryption Key

Generate a secure encryption key:

```bash
# Generate 32-byte hex key
php -r "echo bin2hex(random_bytes(32));"
```

Update `config.json`:
```json
{
    "encryption": {
        "key": "your-generated-key-here"
    }
}
```

**Important**: Never commit the encryption key to version control.

### 4. Admin Configuration

Set admin email(s) in config:

```json
{
    "admins": "admin@example.com,backup@example.com"
}
```

## Deployment Steps

### Step 1: Upload Files

Upload the entire project to your web server:

```bash
# Using rsync
rsync -avz --exclude='.git' --exclude='Tests' \
    ./ user@server:/var/www/application/

# Or using SCP
scp -r App Data Lib Public user@server:/var/www/application/
```

**Directory Structure on Server:**
```
/var/www/application/
├── App/              # Application code
├── Data/             # Database and config
├── Lib/              # Dependencies
├── Public/           # Web root
└── ...
```

### Step 2: Set Permissions

```bash
# Web server user (www-data on Debian/Ubuntu, apache on CentOS)
WEB_USER=www-data

# Set ownership
chown -R $WEB_USER:$WEB_USER /var/www/application

# Set directory permissions
find /var/www/application -type d -exec chmod 755 {} \;

# Set file permissions
find /var/www/application -type f -exec chmod 644 {} \;

# Make Data directory writable for SQLite
chmod 775 /var/www/application/Data
chmod 664 /var/www/application/Data/*.db
chmod 664 /var/www/application/Data/*.json

# Ensure web user owns database files
chown $WEB_USER:$WEB_USER /var/www/application/Data/*.db
chown $WEB_USER:$WEB_USER /var/www/application/Data/*.json
```

### Step 3: Configure Web Server

#### Apache Configuration

Create virtual host:

```apache
# /etc/apache2/sites-available/application.conf
<VirtualHost *:80>
    ServerName memorize.live
    DocumentRoot /var/www/application/Public
    
    # Enable rewrite engine
    <Directory /var/www/application/Public>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    # PHP settings
    php_value upload_max_filesize 10M
    php_value post_max_size 10M
    php_value memory_limit 256M
    
    # Logging
    ErrorLog ${APACHE_LOG_DIR}/application-error.log
    CustomLog ${APACHE_LOG_DIR}/application-access.log combined
</VirtualHost>
```

Enable required modules:
```bash
a2enmod rewrite
a2enmod env
a2ensite application
systemctl restart apache2
```

#### Nginx Configuration

Create server block:

```nginx
# /etc/nginx/sites-available/application
server {
    listen 80;
    server_name memorize.live;
    root /var/www/application/Public;
    index index.php;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Static files
    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }
    
    # PHP handling
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        
        # Environment variables
        fastcgi_param APP_ENV production;
        fastcgi_param APP_DEBUG false;
    }
    
    # Route all requests to index.php
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    # Deny access to sensitive files
    location ~ /\. {
        deny all;
    }
    
    location ~ /(App|Data|Lib)/ {
        deny all;
    }
}
```

Enable configuration:
```bash
ln -s /etc/nginx/sites-available/application /etc/nginx/sites-enabled/
nginx -t
systemctl restart nginx
```

### Step 4: Initialize Database

The database will be created automatically on first run. To pre-initialize:

```bash
cd /var/www/application
php -r "
require_once 'App/Data/Database.php';
\$db = Application\App\Data\Database::getInstance();
echo 'Database initialized successfully\n';
"
```

### Step 5: Verify Deployment

Run health checks:

```bash
# Check PHP is working
curl -I https://memorize.live

# Check database is writable
curl https://memorize.live/health

# Verify static assets
curl -I https://memorize.live/Assets/css/default-theme.css
```

## Post-Deployment Configuration

### SSL/TLS (HTTPS)

#### Let's Encrypt with Certbot

```bash
# Install Certbot
apt install certbot python3-certbot-apache  # For Apache
# or
apt install certbot python3-certbot-nginx   # For Nginx

# Obtain certificate
certbot --apache -d memorize.live -d www.memorize.live
# or
certbot --nginx -d memorize.live -d www.memorize.live

# Auto-renewal is configured automatically
certbot renew --dry-run
```

### Cron Jobs

Set up automated tasks:

```bash
# Edit crontab
crontab -e

# Add notification processing every minute
* * * * * cd /var/www/application && /usr/bin/php Public/index.php /cron/process-notifications >> /var/log/application-cron.log 2>&1

# Add daily cleanup (optional)
0 2 * * * cd /var/www/application && /usr/bin/php -r "require 'App/Data/Database.php'; \$db = Application\App\Data\Database::getInstance(); \$db->exec('DELETE FROM notifications WHERE sent = 1 AND created < datetime(\"now\", \"-30 days\")');" >> /var/log/application-cleanup.log 2>&1
```

### Log Rotation

Configure log rotation:

```bash
# /etc/logrotate.d/application
/var/log/application-*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 644 www-data www-data
}
```

## Environment-Specific Configurations

### Hostname-Based Config

Application supports hostname-specific config files:

```
Data/
├── config.json              # Default config
├── config.memorize.live.json    # Production
├── config.staging.memorize.live.json  # Staging
└── config.localhost.json    # Development
```

The application automatically loads the config matching the current hostname.

### Staging Environment

1. Create staging config:
```bash
cp Data/config.json Data/config.staging.memorize.live.json
```

2. Update staging-specific values:
```json
{
    "app": {
        "base_url": "https://staging.memorize.live"
    },
    "mode": "development"
}
```

3. Set environment:
```bash
SetEnv APP_ENV staging
```

## Security Hardening

### File Permissions

```bash
# Remove write permissions from code files
chmod -R 555 /var/www/application/App
chmod -R 555 /var/www/application/Lib

# Keep Data writable
chmod 775 /var/www/application/Data
```

### Disable Dangerous Functions

In `php.ini`:
```ini
disable_functions = exec,passthru,shell_exec,system,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,show_source
```

### Database Security

```bash
# Store database outside web root (optional)
mkdir /var/lib/application
mv /var/www/application/Data/*.db /var/lib/application/
chown -R www-data:www-data /var/lib/application
chmod 700 /var/lib/application

# Update Database.php to use new path (requires code change)
```

### Hide Server Information

In `php.ini`:
```ini
expose_php = Off
```

In Apache/Nginx config, remove or modify `Server` headers.

## Monitoring

### Health Check Endpoint

Create a simple health check:

```bash
# Add to cron for monitoring
curl -f https://memorize.live/health || echo "Site down" | mail -s "Application Alert" admin@example.com
```

### Log Monitoring

```bash
# Watch for errors
tail -f /var/log/application-error.log | grep -i error

# Monitor cron job output
tail -f /var/log/application-cron.log
```

## Backup Strategy

### Database Backup

```bash
#!/bin/bash
# /usr/local/bin/backup-application.sh

BACKUP_DIR="/backups/application"
DATE=$(date +%Y%m%d_%H%M%S)
DB_FILE="/var/www/application/Data/memorize.db"

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup SQLite database
sqlite3 $DB_FILE ".backup '$BACKUP_DIR/memorize_$DATE.db'"

# Compress backup
gzip $BACKUP_DIR/memorize_$DATE.db

# Keep only last 30 days
find $BACKUP_DIR -name "memorize_*.db.gz" -mtime +30 -delete

# Sync to remote (optional)
# rsync -avz $BACKUP_DIR/ backup-server:/backups/application/
```

Add to cron:
```bash
0 3 * * * /usr/local/bin/backup-application.sh
```

### Configuration Backup

```bash
# Backup config files
tar czf /backups/application/config_$(date +%Y%m%d).tar.gz /var/www/application/Data/config*.json
```

## Troubleshooting

### Permission Denied Errors

```bash
# Check file ownership
ls -la /var/www/application/Data/

# Fix ownership
chown -R www-data:www-data /var/www/application/Data
chmod 775 /var/www/application/Data
```

### 500 Internal Server Error

1. Check PHP error log:
```bash
tail -f /var/log/apache2/error.log
# or
tail -f /var/log/nginx/error.log
```

2. Enable debug mode temporarily:
```bash
# In index.php or via environment
putenv('APP_DEBUG=true');
```

### Database Locked

```bash
# Check for locks
lsof /var/www/application/Data/memorize.db

# Restart web server if necessary
systemctl restart apache2
# or
systemctl restart php8.2-fpm
```

### Slow Performance

1. Enable OPcache in `php.ini`:
```ini
opcache.enable=1
opcache.memory_consumption=128
opcache.max_accelerated_files=4000
```

2. Verify SQLite is using WAL mode (enabled by default)

3. Check disk I/O:
```bash
iostat -x 1
```

## Rollback Procedure

If deployment fails:

```bash
# 1. Restore previous version from backup
cd /var/www
tar xzf backups/application_$(date -d yesterday +%Y%m%d).tar.gz

# 2. Restore database
sqlite3 /var/www/application/Data/memorize.db ".restore '/backups/application/memorize_latest.db'"

# 3. Restart web server
systemctl restart apache2
```

## Related Documentation

- [Config System](../architecture/CONFIG_SYSTEM.md) - Configuration management
- [Testing Guide](./TESTING.md) - Pre-deployment testing
- [Security Considerations](../architecture/SECURITY.md) - Security best practices
