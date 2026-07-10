# Production Deployment Guide

Panduan lengkap untuk deployment sistem Affiliate Marketing Automation ke environment production.

## 🎯 Pre-Production Checklist

### Security
- [ ] Change all default passwords
- [ ] Configure SSL/TLS certificates
- [ ] Setup firewall rules
- [ ] Enable fail2ban
- [ ] Configure rate limiting
- [ ] Setup IP whitelisting (if needed)
- [ ] Review and update CORS settings

### Database
- [ ] Change PostgreSQL password
- [ ] Change Redis password
- [ ] Setup database backups
- [ ] Configure connection pooling
- [ ] Enable SSL for database connections

### Application
- [ ] Update environment variables for production
- [ ] Disable debug mode
- [ ] Configure proper logging
- [ ] Setup error tracking
- [ ] Configure health checks
- [ ] Setup graceful shutdown

### Monitoring
- [ ] Configure Prometheus alerts
- [ ] Setup Grafana dashboards
- [ ] Configure log retention
- [ ] Setup uptime monitoring
- [ ] Configure notification channels
- [ ] Test alert delivery

### Performance
- [ ] Configure worker scaling
- [ ] Setup CDN (if needed)
- [ ] Configure caching
- [ ] Optimize database queries
- [ ] Configure resource limits
- [ ] Setup load balancing (if needed)

## 🔧 Production Environment Setup

### 1. Update Environment Variables

```bash
# Edit .env file
nano .env
```

Update critical variables:

```bash
# Change environment
ENVIRONMENT=production
DEBUG=false
HOT_RELOAD=false

# Production domain
DOMAIN=your-production-domain.com

# Strong passwords
POSTGRES_PASSWORD=your_strong_postgres_password_here
REDIS_PASSWORD=your_strong_redis_password_here
N8N_ENCRYPTION_KEY=your_strong_encryption_key_min_32_chars
JWT_SECRET=your_strong_jwt_secret_min_32_characters
WEBUI_SECRET_KEY=your_strong_webui_secret_key_min_32_characters

# Production URLs
N8N_HOST=n8n.your-production-domain.com
N8N_EDITOR_BASE_URL=https://n8n.your-production-domain.com
WEBHOOK_URL=https://n8n.your-production-domain.com/webhook

# Grafana admin
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=your_strong_grafana_password

# CORS (restrict to your domain)
CORS_ORIGINS=https://your-production-domain.com,https://n8n.your-production-domain.com

# Feature flags
FEATURE_AI_GENERATION=true
FEATURE_VIDEO_RENDERING=true
FEATURE_SOCIAL_UPLOAD=true
FEATURE_ANALYTICS=true
FEATURE_NOTIFICATIONS=true
```

### 2. Configure SSL/TLS with Let's Encrypt

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtain SSL certificate for main domain
sudo certbot certonly --standalone -d your-production-domain.com

# Obtain SSL certificates for subdomains
sudo certbot certonly --standalone -d n8n.your-production-domain.com
sudo certbot certonly --standalone -d api.your-production-domain.com
sudo certbot certonly --standalone -d grafana.your-production-domain.com
sudo certbot certonly --standalone -d portainer.your-production-domain.com

# Setup auto-renewal (certbot does this automatically)
sudo certbot renew --dry-run
```

### 3. Configure Traefik for SSL

Update docker-compose.yml Traefik configuration:

```yaml
traefik:
  image: traefik:v3.0
  container_name: affiliate-traefik
  restart: unless-stopped
  command:
    - "--api.insecure=false"
    - "--api.dashboard=true"
    - "--providers.docker=true"
    - "--providers.docker.exposedbydefault=false"
    - "--entrypoints.web.address=:80"
    - "--entrypoints.websecure.address=:443"
    - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
    - "--certificatesresolvers.myresolver.acme.email=${ACME_EMAIL}"
    - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
  ports:
    - "80:80"
    - "443:443"
    - "8080:8080"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - ./data/traefik/letsencrypt:/letsencrypt
    - /etc/letsencrypt:/etc/letsencrypt:ro
  networks:
    - affiliate-network
```

### 4. Configure Resource Limits

Update docker-compose.yml with resource limits:

```yaml
# API Service
api:
  # ... existing config
  deploy:
    resources:
      limits:
        cpus: '2.0'
        memory: 2G
      reservations:
        cpus: '1.0'
        memory: 1G

# Worker AI
worker-ai:
  # ... existing config
  deploy:
    resources:
      limits:
        cpus: '2.0'
        memory: 4G
      reservations:
        cpus: '1.0'
        memory: 2G

# Worker Render (CPU intensive)
worker-render:
  # ... existing config
  deploy:
    resources:
      limits:
        cpus: '4.0'
        memory: 8G
      reservations:
        cpus: '2.0'
        memory: 4G

# PostgreSQL
postgres:
  # ... existing config
  deploy:
    resources:
      limits:
        cpus: '2.0'
        memory: 4G
      reservations:
        cpus: '1.0'
        memory: 2G

# Redis
redis:
  # ... existing config
  deploy:
    resources:
      limits:
        cpus: '1.0'
        memory: 2G
      reservations:
        cpus: '0.5'
        memory: 1G
```

## 💾 Database Backup Setup

### 1. PostgreSQL Backup Script

```bash
# Create backup script
nano scripts/backup-postgres.sh
```

```bash
#!/bin/bash

# PostgreSQL Backup Script
# Usage: ./backup-postgres.sh

BACKUP_DIR="/opt/backups/postgres"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="affiliate_db_${TIMESTAMP}.sql.gz"
RETENTION_DAYS=7

# Create backup directory if not exists
mkdir -p ${BACKUP_DIR}

# Get database credentials from environment
DB_HOST="postgres"
DB_NAME="${POSTGRES_DB}"
DB_USER="${POSTGRES_USER}"
DB_PASSWORD="${POSTGRES_PASSWORD}"

# Perform backup
docker exec affiliate-postgres pg_dump -U ${DB_USER} ${DB_NAME} | gzip > ${BACKUP_DIR}/${BACKUP_FILE}

# Check if backup was successful
if [ $? -eq 0 ]; then
    echo "Backup successful: ${BACKUP_FILE}"
    
    # Remove old backups (older than RETENTION_DAYS)
    find ${BACKUP_DIR} -name "affiliate_db_*.sql.gz" -mtime +${RETENTION_DAYS} -delete
    echo "Old backups removed (older than ${RETENTION_DAYS} days)"
else
    echo "Backup failed!"
    exit 1
fi
```

```bash
# Make script executable
chmod +x scripts/backup-postgres.sh
```


### 3. Setup Cron Jobs

```bash
# Edit crontab
crontab -e
```

Add cron jobs:

```bash
# PostgreSQL backup every day at 2 AM
0 2 * * * /opt/affiliate-automation/scripts/backup-postgres.sh >> /var/log/postgres-backup.log 2>&1

# Database cleanup every Sunday at 4 AM
0 4 * * 0 docker exec affiliate-postgres psql -U affiliate_user -d affiliate_db -c "VACUUM ANALYZE;" >> /var/log/postgres-vacuum.log 2>&1

# Log rotation every day at 5 AM
0 5 * * * find /opt/affiliate-automation/logs -name "*.log" -mtime +7 -delete >> /var/log/log-cleanup.log 2>&1
```

### 4. S3 Backup (Optional)

For cloud backup to AWS S3:

```bash
# Install AWS CLI
sudo apt install -y awscli

# Configure AWS credentials
aws configure

# Create S3 backup script
nano scripts/backup-s3.sh
```

```bash
#!/bin/bash

# S3 Backup Script
# Usage: ./backup-s3.sh

S3_BUCKET="your-backup-bucket-name"
BACKUP_DIR="/opt/backups"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")

# Upload PostgreSQL backup
aws s3 cp ${BACKUP_DIR}/postgres/affiliate_db_${TIMESTAMP}.sql.gz s3://${S3_BUCKET}/postgres/

# Set lifecycle policy for 30-day retention
aws s3api put-bucket-lifecycle-configuration \
    --bucket ${S3_BUCKET} \
    --lifecycle-configuration file://config/s3-lifecycle.json
```

## 🔒 Security Hardening

### 1. Firewall Configuration

```bash
# Configure UFW for production
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (restrict to your IP if possible)
sudo ufw allow from YOUR_IP_ADDRESS to any port 22

# Allow HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow Docker internal communication
sudo ufw allow from 172.20.0.0/16

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status verbose
```

### 2. Fail2Ban Configuration

```bash
# Create custom jail
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
maxretry = 3

[nginx-noscript]
enabled = true
port = http,https
logpath = /var/log/nginx/access.log
maxretry = 6

[nginx-badbots]
enabled = true
port = http,https
logpath = /var/log/nginx/access.log
maxretry = 2

[nginx-noproxy]
enabled = true
port = http,https
logpath = /var/log/nginx/access.log
maxretry = 2
```

```bash
# Restart Fail2Ban
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
```

### 3. Docker Security

```bash
# Update Docker daemon configuration
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true,
  "icc": false,
  "default-address-pools": [
    {
      "base": "172.20.0.0/16",
      "size": 24
    }
  ]
}
```

```bash
# Restart Docker
sudo systemctl restart docker
```

### 4. Application Security

Update docker-compose.yml with security options:

```yaml
# Add to all services
security_opt:
  - no-new-privileges:true
read_only: true
tmpfs:
  - /tmp
```

## 🚀 Deployment Steps

### 1. Prepare Production Server

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install dependencies
sudo apt install -y curl git wget vim ufw fail2ban docker docker-compose

# Clone repository
cd /opt
sudo git clone https://github.com/alumnisteman/N8N.git affiliate-automation
cd affiliate-automation

# Set permissions
sudo chown -R $USER:$USER /opt/affiliate-automation

# Create directories
mkdir -p data/{postgres,redis,qdrant,n8n,grafana,prometheus,loki,ollama,open-webui,uptime-kuma,portainer,traefik}
mkdir -p logs uploads videos thumbnails subtitles
mkdir -p scripts config/{prometheus,grafana,loki,promtail,alertmanager}
mkdir -p backups/postgres

# Set ownership
sudo chown -R 1000:1000 data/
sudo chown -R 1000:1000 logs/
sudo chown -R 1000:1000 uploads/
```

### 2. Configure Environment

```bash
# Copy environment template
cp .env.example .env

# Edit environment variables
nano .env

# Generate secure passwords
openssl rand -base64 32
```

### 3. Setup SSL Certificates

```bash
# Obtain SSL certificates
sudo certbot certonly --standalone -d your-domain.com
sudo certbot certonly --standalone -d n8n.your-domain.com
sudo certbot certonly --standalone -d api.your-domain.com
sudo certbot certonly --standalone -d grafana.your-domain.com
```

### 4. Start Services

```bash
# Start all services
docker compose up -d

# Check status
docker compose ps

# Check logs
docker compose logs -f
```

### 5. Verify Deployment

```bash
# Check all services are running
docker compose ps

# Test API health
curl https://api.your-domain.com/health

# Test n8n
curl https://n8n.your-domain.com/healthz

# Test Grafana
curl https://grafana.your-domain.com/api/health

# Check database connection
docker exec affiliate-postgres pg_isready -U affiliate_user

# Check Redis connection
docker exec affiliate-redis redis-cli -a your_redis_password ping
```

### 6. Setup Monitoring

```bash
# Verify Prometheus targets
curl http://localhost:9090/api/v1/targets

# Check Grafana dashboards
# Access: https://grafana.your-domain.com

# Configure Uptime Kuma monitors
# Access: https://uptime.your-domain.com
```

### 7. Configure Backups

```bash
# Setup backup scripts
chmod +x scripts/*.sh

# Test backup scripts
./scripts/backup-postgres.sh

# Setup cron jobs
crontab -e
```

## 🔄 Update and Maintenance

### 1. Update Application

```bash
# Pull latest changes
cd /opt/affiliate-automation
git pull origin main

# Backup current version
docker compose backup

# Rebuild and restart
docker compose up -d --build

# Check status
docker compose ps
```

### 2. Rolling Updates

For zero-downtime updates:

```bash
# Update specific service
docker compose up -d --no-deps --build api

# Wait for health check
sleep 30

# Update workers one by one
docker compose up -d --no-deps --build worker-ai
sleep 30

docker compose up -d --no-deps --build worker-render
sleep 30

docker compose up -d --no-deps --build worker-upload
```

### 3. Database Migrations

```bash
# Run migrations
docker exec affiliate-api alembic upgrade head

# Verify migration
docker exec affiliate-api alembic current
```

## 📊 Performance Optimization

### 1. Database Optimization

```sql
-- Connect to PostgreSQL
docker exec -it affiliate-postgres psql -U affiliate_user -d affiliate_db

-- Create indexes
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_videos_status ON videos(status);
CREATE INDEX idx_jobs_created_at ON jobs(created_at);
CREATE INDEX idx_analytics_date ON analytics(date);

-- Analyze tables
ANALYZE products;
ANALYZE videos;
ANALYZE jobs;
ANALYZE analytics;

-- Vacuum
VACUUM ANALYZE;
```

### 2. Redis Optimization

```bash
# Configure Redis for production
# Edit docker-compose.yml redis service
redis:
  # ... existing config
  command: >
    redis-server
    --requirepass ${REDIS_PASSWORD}
    --appendonly yes
    --maxmemory 2gb
    --maxmemory-policy allkeys-lru
    --save 900 1
    --save 300 10
    --save 60 10000
```

### 3. Worker Scaling

Adjust worker replicas based on load:

```yaml
# Update docker-compose.yml
worker-ai:
  deploy:
    replicas: 4  # Increase for more AI processing capacity

worker-render:
  deploy:
    replicas: 3  # Rendering is CPU intensive

worker-upload:
  deploy:
    replicas: 5  # More upload workers for parallel uploads
```

## 🆘 Disaster Recovery

### 1. Restore PostgreSQL from Backup

```bash
# Stop application
docker compose stop api workers

# Restore database
gunzip < /opt/backups/postgres/affiliate_db_YYYYMMDD_HHMMSS.sql.gz | \
docker exec -i affiliate-postgres psql -U affiliate_user -d affiliate_db

# Restart application
docker compose start api workers
```

### 3. Full System Restore

```bash
# Stop all services
docker compose down

# Restore data directories
rsync -av /backups/data/ /opt/affiliate-automation/data/

# Restore environment
cp /backups/.env /opt/affiliate-automation/.env

# Start services
docker compose up -d

# Verify restoration
docker compose ps
```

## 📝 Post-Deployment Checklist

- [ ] All services are running
- [ ] SSL certificates are valid
- [ ] Database connections are working
- [ ] Redis is accessible
- [ ] API health checks pass
- [ ] n8n workflows are active
- [ ] Workers are processing jobs
- [ ] Monitoring dashboards show data
- [ ] Alerts are configured
- [ ] Backup scripts are working
- [ ] Cron jobs are scheduled
- [ ] Firewall rules are active
- [ ] SSL certificates auto-renew
- [ ] Log rotation is configured
- [ ] Resource limits are set
- [ ] Security headers are configured
- [ ] CORS is properly configured
- [ ] Rate limiting is enabled
- [ ] Error tracking is configured

## 📚 Additional Resources

- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [PostgreSQL Performance Tuning](https://wiki.postgresql.org/wiki/Performance_Optimization)
- [Redis Configuration](https://redis.io/topics/config)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
