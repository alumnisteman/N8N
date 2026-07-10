# Troubleshooting and Maintenance Guide

Panduan lengkap untuk troubleshooting dan maintenance sistem Affiliate Marketing Automation.

## 🔍 Common Issues and Solutions

### Container Issues

#### Container Won't Start

**Symptoms:**
- Container exits immediately after starting
- Container status shows "Exited" or "Restarting"

**Diagnosis:**
```bash
# Check container status
docker compose ps

# Check container logs
docker compose logs <service-name>

# Check detailed logs
docker compose logs --tail=100 -f <service-name>
```

**Solutions:**

1. **Check for port conflicts:**
```bash
# Check what's using the port
sudo netstat -tulpn | grep <port>

# Kill the process if needed
sudo kill -9 <pid>
```

2. **Check environment variables:**
```bash
# Verify .env file exists
cat .env

# Check for missing variables
docker compose config
```

3. **Check volume permissions:**
```bash
# Fix volume permissions
sudo chown -R 1000:1000 data/
sudo chown -R 1000:1000 logs/
```

4. **Rebuild container:**
```bash
# Rebuild specific service
docker compose up -d --build <service-name>

# Rebuild all services
docker compose up -d --build
```

#### Container High CPU/Memory Usage

**Symptoms:**
- Container using excessive CPU or memory
- System becomes slow

**Diagnosis:**
```bash
# Check container resource usage
docker stats

# Check specific container
docker stats <container-name>
```

**Solutions:**

1. **Add resource limits in docker-compose.yml:**
```yaml
service-name:
  deploy:
    resources:
      limits:
        cpus: '2.0'
        memory: 2G
      reservations:
        cpus: '1.0'
        memory: 1G
```

2. **Restart container:**
```bash
docker compose restart <service-name>
```

3. **Scale workers:**
```bash
# Reduce worker replicas
docker compose up -d --scale worker-ai=2
```

### Database Issues

#### PostgreSQL Connection Failed

**Symptoms:**
- API cannot connect to database
- "Connection refused" errors

**Diagnosis:**
```bash
# Check PostgreSQL container
docker compose ps postgres

# Check PostgreSQL logs
docker compose logs postgres

# Test connection
docker exec affiliate-postgres pg_isready -U affiliate_user
```

**Solutions:**

1. **Check PostgreSQL is running:**
```bash
docker compose start postgres
```

2. **Verify credentials:**
```bash
# Check .env file
cat .env | grep POSTGRES
```

3. **Check network connectivity:**
```bash
# Test from API container
docker exec affiliate-api ping postgres
```

4. **Restart PostgreSQL:**
```bash
docker compose restart postgres
```

#### PostgreSQL Slow Performance

**Symptoms:**
- Slow query responses
- High CPU usage

**Diagnosis:**
```sql
-- Connect to PostgreSQL
docker exec -it affiliate-postgres psql -U affiliate_user -d affiliate_db

-- Check active connections
SELECT count(*) FROM pg_stat_activity;

-- Check long-running queries
SELECT pid, now() - query_start as duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;

-- Check table sizes
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

**Solutions:**

1. **Create indexes:**
```sql
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_videos_created_at ON videos(created_at);
```

2. **Vacuum and analyze:**
```sql
VACUUM ANALYZE;
```

3. **Kill long-running queries:**
```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'active' AND pid <> pg_backend_pid();
```

4. **Increase connection pool:**
```yaml
# Update .env
DB_POOL_SIZE=30
DB_MAX_OVERFLOW=15
```

#### Redis Connection Failed

**Symptoms:**
- Workers cannot connect to Redis
- Queue operations fail

**Diagnosis:**
```bash
# Check Redis container
docker compose ps redis

# Check Redis logs
docker compose logs redis

# Test connection
docker exec affiliate-redis redis-cli -a your_redis_password ping
```

**Solutions:**

1. **Check Redis is running:**
```bash
docker compose start redis
```

2. **Verify password:**
```bash
# Check .env file
cat .env | grep REDIS_PASSWORD
```

3. **Test connection:**
```bash
docker exec affiliate-redis redis-cli -a your_redis_password ping
```

4. **Flush Redis (caution - clears all data):**
```bash
docker exec affiliate-redis redis-cli -a your_redis_password FLUSHALL
```

### API Issues

#### API Returns 500 Errors

**Symptoms:**
- API endpoints return 500 status
- Internal server errors

**Diagnosis:**
```bash
# Check API logs
docker compose logs api

# Check specific endpoint
curl -v http://localhost:8000/health
```

**Solutions:**

1. **Check API logs for errors:**
```bash
docker compose logs --tail=100 api
```

2. **Verify database connection:**
```bash
docker exec affiliate-api python -c "from app.database import engine; print(engine.connect())"
```

3. **Check environment variables:**
```bash
docker exec affiliate-api env | grep DATABASE
```

4. **Restart API:**
```bash
docker compose restart api
```

#### API Slow Response Time

**Symptoms:**
- API endpoints take long to respond
- High latency

**Diagnosis:**
```bash
# Check API performance
curl -w "@curl-format.txt" -o /dev/null -s http://localhost:8000/api/v1/products
```

**Solutions:**

1. **Enable database connection pooling:**
```python
# In database.py
engine = create_engine(
    settings.DATABASE_URL,
    pool_size=20,
    max_overflow=10,
    pool_timeout=30,
    pool_pre_ping=True
)
```

2. **Add Redis caching:**
```python
# Cache frequent queries
@lru_cache(maxsize=100)
def get_product(product_id: int):
    # Implementation
    pass
```

3. **Add pagination:**
```python
# Always paginate large datasets
@app.get("/api/v1/products")
async def get_products(skip: int = 0, limit: int = 100):
    # Implementation
    pass
```

4. **Scale API:**
```yaml
# In docker-compose.yml
api:
  deploy:
    replicas: 2
```

### Worker Issues

#### Workers Not Processing Jobs

**Symptoms:**
- Jobs queue up but don't process
- Workers show no activity

**Diagnosis:**
```bash
# Check worker status
docker compose ps worker-ai worker-render worker-upload

# Check worker logs
docker compose logs worker-ai

# Check Redis queue length
docker exec affiliate-redis redis-cli -a your_redis_password LLEN ai_queue
```

**Solutions:**

1. **Check worker is connected to Redis:**
```bash
docker compose logs worker-ai | grep "Connected to Redis"
```

2. **Restart workers:**
```bash
docker compose restart worker-ai worker-render worker-upload
```

3. **Check queue configuration:**
```python
# Verify queue names match
celery_app.conf.task_default_queue = "ai_queue"
```

4. **Scale workers:**
```bash
docker compose up -d --scale worker-ai=4
```

#### Worker High Failure Rate

**Symptoms:**
- Many jobs failing
- Error logs show failures

**Diagnosis:**
```bash
# Check worker logs
docker compose logs worker-ai | grep ERROR

# Check failed jobs in database
docker exec affiliate-postgres psql -U affiliate_user -d affiliate_db -c "SELECT * FROM jobs WHERE status = 'failed' ORDER BY created_at DESC LIMIT 10;"
```

**Solutions:**

1. **Identify error pattern:**
```bash
docker compose logs worker-ai | grep ERROR | tail -20
```

2. **Retry failed jobs:**
```python
# Via API
curl -X POST http://localhost:8000/api/v1/jobs/<job_id>/retry
```

3. **Fix underlying issue:**
- Check API credentials
- Verify external service availability
- Check resource limits

4. **Increase timeout:**
```python
# In worker configuration
celery_app.conf.task_time_limit = 600  # 10 minutes
```

### n8n Issues

#### n8n Workflows Not Triggering

**Symptoms:**
- Workflows don't start automatically
- Manual trigger works but not automatic

**Diagnosis:**
```bash
# Check n8n logs
docker compose logs n8n

# Check n8n status
curl http://localhost:5678/healthz
```

**Solutions:**

1. **Check workflow is active:**
- Open n8n UI
- Check workflow toggle is ON

2. **Verify trigger configuration:**
- Check cron expression
- Verify webhook URL
- Check database trigger settings

3. **Check n8n queue mode:**
```bash
# Verify queue mode is enabled
docker compose logs n8n | grep "EXECUTIONS_MODE"
```

4. **Restart n8n:**
```bash
docker compose restart n8n
```

#### n8n Database Connection Issues

**Symptoms:**
- n8n cannot connect to PostgreSQL
- Workflows fail to save

**Diagnosis:**
```bash
# Check n8n logs
docker compose logs n8n | grep "database"

# Test connection from n8n container
docker exec affiliate-n8n ping postgres
```

**Solutions:**

1. **Verify database credentials in .env:**
```bash
cat .env | grep DB_POSTGRESDB
```

2. **Check PostgreSQL is accessible:**
```bash
docker exec affiliate-n8n nc -zv postgres 5432
```

3. **Restart n8n:**
```bash
docker compose restart n8n
```

### Storage Issues

#### Disk Space Full

**Symptoms:**
- Container cannot write files
- "No space left on device" errors

**Diagnosis:**
```bash
# Check disk usage
df -h

# Check Docker disk usage
docker system df

# Check large files
du -sh /opt/affiliate-automation/data/* | sort -rh
```

**Solutions:**

1. **Clean Docker system:**
```bash
# Remove unused containers
docker container prune

# Remove unused images
docker image prune -a

# Remove unused volumes
docker volume prune

# Remove build cache
docker builder prune
```

2. **Clean old logs:**
```bash
# Remove logs older than 7 days
find /opt/affiliate-automation/logs -name "*.log" -mtime +7 -delete
```

3. **Clean old backups:**
```bash
# Remove backups older than retention period
find /opt/backups -name "*.sql.gz" -mtime +30 -delete
```

### Monitoring Issues

#### Prometheus Not Scraping Metrics

**Symptoms:**
- Grafana shows no data
- Prometheus targets show "down"

**Diagnosis:**
```bash
# Check Prometheus targets
curl http://localhost:9090/api/v1/targets

# Check Prometheus logs
docker compose logs prometheus
```

**Solutions:**

1. **Check Prometheus configuration:**
```bash
# Verify prometheus.yml
cat config/prometheus/prometheus.yml
```

2. **Check service exposes metrics:**
```bash
# Test metrics endpoint
curl http://localhost:8000/metrics
```

3. **Check network connectivity:**
```bash
docker exec affiliate-prometheus ping api
```

4. **Reload Prometheus:**
```bash
docker compose restart prometheus
```

#### Grafana Not Showing Data

**Symptoms:**
- Dashboards show "No data"
- Panels are empty

**Diagnosis:**
```bash
# Check Grafana logs
docker compose logs grafana

# Test datasource connection
# Via Grafana UI → Configuration → Datasources
```

**Solutions:**

1. **Verify datasource configuration:**
- Open Grafana UI
- Go to Configuration → Datasources
- Test Prometheus connection

2. **Check time range:**
- Ensure dashboard time range includes data
- Try "Last 1 hour" instead of custom range

3. **Verify query syntax:**
- Check panel queries
- Test query in Prometheus UI first

4. **Restart Grafana:**
```bash
docker compose restart grafana
```

## 🔧 Maintenance Tasks

### Daily Maintenance

```bash
#!/bin/bash
# daily-maintenance.sh

# Check container status
docker compose ps

# Check disk usage
df -h

# Check Docker resource usage
docker stats --no-stream

# Check error logs
docker compose logs --since=24h | grep ERROR

# Check failed jobs
docker exec affiliate-postgres psql -U affiliate_user -d affiliate_db -c "SELECT COUNT(*) FROM jobs WHERE status = 'failed' AND created_at > NOW() - INTERVAL '24 hours';"
```

### Weekly Maintenance

```bash
#!/bin/bash
# weekly-maintenance.sh

# Database vacuum
docker exec affiliate-postgres psql -U affiliate_user -d affiliate_db -c "VACUUM ANALYZE;"

# Clean old logs
find /opt/affiliate-automation/logs -name "*.log" -mtime +7 -delete

# Clean Docker system
docker system prune -f

# Check SSL certificate expiry
echo | openssl s_client -servername your-domain.com -connect your-domain.com:443 2>/dev/null | openssl x509 -noout -dates

# Backup verification
ls -lh /opt/backups/postgres/ | tail -5
```

### Monthly Maintenance

```bash
#!/bin/bash
# monthly-maintenance.sh

# Full database backup
./scripts/backup-postgres.sh

# Update system packages
sudo apt update && sudo apt upgrade -y

# Update Docker
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Review and update API keys
# Check if any API keys need rotation

# Review monitoring alerts
# Check if alert thresholds need adjustment

# Capacity planning
# Review resource usage trends
# Plan for scaling if needed
```

## 📊 Health Check Script

```bash
#!/bin/bash
# health-check.sh

echo "=== Affiliate Automation Health Check ==="
echo ""

# Color codes
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Function to check service
check_service() {
    local service=$1
    local container=$(docker compose ps -q $service)
    
    if [ -n "$container" ]; then
        local status=$(docker inspect --format='{{.State.Status}}' $container)
        if [ "$status" == "running" ]; then
            echo -e "${GREEN}✓${NC} $service is running"
            return 0
        else
            echo -e "${RED}✗${NC} $service is not running (status: $status)"
            return 1
        fi
    else
        echo -e "${RED}✗${NC} $service container not found"
        return 1
    fi
}

# Check all services
echo "Checking services..."
check_service "traefik"
check_service "portainer"
check_service "n8n"
check_service "postgres"
check_service "redis"
check_service "api"
check_service "worker-ai"
check_service "worker-render"
check_service "worker-upload"
check_service "prometheus"
check_service "grafana"
check_service "loki"
check_service "uptime-kuma"

echo ""

# Check disk usage
echo "Checking disk usage..."
DISK_USAGE=$(df / | awk 'NR==2 {print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 80 ]; then
    echo -e "${YELLOW}⚠${NC} Disk usage: ${DISK_USAGE}% (high)"
else
    echo -e "${GREEN}✓${NC} Disk usage: ${DISK_USAGE}%"
fi

echo ""

# Check memory usage
echo "Checking memory usage..."
MEM_USAGE=$(free | awk 'NR==2 {printf "%.0f", $3/$2*100}')
if [ $MEM_USAGE -gt 80 ]; then
    echo -e "${YELLOW}⚠${NC} Memory usage: ${MEM_USAGE}% (high)"
else
    echo -e "${GREEN}✓${NC} Memory usage: ${MEM_USAGE}%"
fi

echo ""

# Check database connection
echo "Checking database connection..."
if docker exec affiliate-postgres pg_isready -U affiliate_user > /dev/null 2>&1; then
    echo -e "${GREEN}✓${NC} PostgreSQL is ready"
else
    echo -e "${RED}✗${NC} PostgreSQL is not ready"
fi

echo ""

# Check Redis connection
echo "Checking Redis connection..."
if docker exec affiliate-redis redis-cli -a ${REDIS_PASSWORD} ping > /dev/null 2>&1; then
    echo -e "${GREEN}✓${NC} Redis is ready"
else
    echo -e "${RED}✗${NC} Redis is not ready"
fi

echo ""

# Check API health
echo "Checking API health..."
if curl -s http://localhost:8000/health > /dev/null 2>&1; then
    echo -e "${GREEN}✓${NC} API is healthy"
else
    echo -e "${RED}✗${NC} API is not healthy"
fi

echo ""

# Check recent errors
echo "Checking recent errors..."
ERROR_COUNT=$(docker compose logs --since=1h | grep -i error | wc -l)
if [ $ERROR_COUNT -gt 0 ]; then
    echo -e "${YELLOW}⚠${NC} Found $ERROR_COUNT errors in the last hour"
else
    echo -e "${GREEN}✓${NC} No errors in the last hour"
fi

echo ""
echo "=== Health Check Complete ==="
```

## 🚨 Emergency Procedures

### System Down - All Services Stopped

```bash
# 1. Check Docker daemon
sudo systemctl status docker

# 2. Restart Docker if needed
sudo systemctl restart docker

# 3. Start all services
docker compose up -d

# 4. Check status
docker compose ps

# 5. Check logs
docker compose logs -f
```

### Database Corruption

```bash
# 1. Stop all services
docker compose down

# 2. Restore from backup
gunzip < /opt/backups/postgres/affiliate_db_YYYYMMDD_HHMMSS.sql.gz | \
docker run --rm -i \
    -v affiliate-automation_postgres:/var/lib/postgresql/data \
    postgres:15-alpine \
    psql -U affiliate_user -d affiliate_db

# 3. Start services
docker compose up -d

# 4. Verify data
docker exec affiliate-postgres psql -U affiliate_user -d affiliate_db -c "SELECT COUNT(*) FROM products;"
```

### Redis Data Loss

```bash
# 1. Redis is ephemeral - jobs will be requeued
# 2. Restart Redis
docker compose restart redis

# 3. Restart workers to requeue jobs
docker compose restart worker-ai worker-render worker-upload

# 4. Monitor queue
docker exec affiliate-redis redis-cli -a your_redis_password LLEN ai_queue
```

### Security Incident

```bash
# 1. Immediately change all passwords
# Update .env file with new passwords

# 2. Restart all services
docker compose down
docker compose up -d

# 3. Review access logs
docker compose logs traefik | grep <suspicious-ip>

# 4. Block suspicious IPs
sudo ufw deny from <suspicious-ip>

# 5. Enable additional security
# - Rate limiting
# - IP whitelisting
# - 2FA for admin access
```

## 📞 Getting Help

### Collect Diagnostic Information

```bash
#!/bin/bash
# collect-diagnostics.sh

DIAG_DIR="/tmp/diagnostics-$(date +%Y%m%d-%H%M%S)"
mkdir -p $DIAG_DIR

# Container status
docker compose ps > $DIAG_DIR/container-status.txt

# Container logs
docker compose logs --tail=500 > $DIAG_DIR/all-logs.txt

# Resource usage
docker stats --no-stream > $DIAG_DIR/resource-usage.txt

# Disk usage
df -h > $DIAG_DIR/disk-usage.txt

# Network info
docker network inspect affiliate-network > $DIAG_DIR/network-info.json

# Environment variables (sanitized)
cat .env | sed 's/PASSWORD=.*/PASSWORD=***/' > $DIAG_DIR/env.txt

# System info
uname -a > $DIAG_DIR/system-info.txt
docker version > $DIAG_DIR/docker-version.txt
docker compose version > $DIAG_DIR/compose-version.txt

# Create archive
tar -czf $DIAG_DIR.tar.gz $DIAG_DIR

echo "Diagnostics collected: $DIAG_DIR.tar.gz"
```

### Log Locations

- **Application logs**: `/opt/affiliate-automation/logs/`
- **Docker logs**: `docker compose logs <service>`
- **System logs**: `/var/log/`
- **Nginx/Traefik logs**: Check reverse proxy logs
- **PostgreSQL logs**: Docker container logs

### Useful Commands

```bash
# View real-time logs
docker compose logs -f

# View logs for specific service
docker compose logs -f api

# View last 100 lines
docker compose logs --tail=100

# View logs since specific time
docker compose logs --since=1h

# Export logs to file
docker compose logs > logs.txt

# Execute command in container
docker exec -it <container-name> bash

# Check container resource usage
docker stats

# Inspect container
docker inspect <container-name>

# View container processes
docker top <container-name>

# Access database
docker exec -it affiliate-postgres psql -U affiliate_user -d affiliate_db

# Access Redis
docker exec -it affiliate-redis redis-cli -a your_redis_password
```

## 📚 Additional Resources

- [Docker Troubleshooting](https://docs.docker.com/config/troubleshooting/)
- [PostgreSQL Troubleshooting](https://www.postgresql.org/docs/current/troubleshooting.html)
- [Redis Troubleshooting](https://redis.io/topics/troubleshooting)
- [n8n Troubleshooting](https://docs.n8n.io/troubleshooting/)
- [Linux System Administration](https://www.linux.org/docs/)
