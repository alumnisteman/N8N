# Installation Guide

Panduan lengkap instalasi sistem Affiliate Marketing Automation dari awal sampai siap digunakan.

## 📋 Prerequisites

### Hardware Requirements

- **CPU**: 4 cores minimum (8 cores recommended)
- **RAM**: 8GB minimum (16GB recommended)
- **Storage**: 100GB minimum SSD (200GB+ recommended untuk production)
- **Network**: Stable internet connection

### Software Requirements

- **OS**: Ubuntu Server 20.04+ / Debian 11+ / CentOS 8+ (atau Docker-compatible OS)
- **Docker**: Version 24.0 atau lebih baru
- **Docker Compose**: Version 2.0 atau lebih baru
- **Git**: Untuk clone repository
- **Domain**: Domain name untuk production (opsional untuk development)

### API Keys Required

Siapkan API keys berikut sebelum instalasi:

- **Gemini API Key**: Google AI untuk content generation
- **Pexels API Key**: Untuk video stock
- **Pixabay API Key**: Untuk video stock alternatif
- **Shopee API Key**: Untuk produk afiliasi
- **TikTok Shop API Key**: Untuk produk afiliasi
- **Tokopedia API Key**: Untuk produk afiliasi
- **Telegram Bot Token**: Untuk notifikasi
- **WhatsApp Business API**: Untuk notifikasi (opsional)

## 🔧 Step 1: System Preparation

### Update System

```bash
# Update package list
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y curl git wget vim ufw fail2ban
```

### Setup Firewall

```bash
# Allow SSH
sudo ufw allow 22/tcp

# Allow HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow Docker ports (adjust sesuai kebutuhan)
sudo ufw allow 3000/tcp  # Portainer
sudo ufw allow 5678/tcp  # n8n
sudo ufw allow 8000/tcp  # FastAPI
sudo ufw allow 6379/tcp  # Redis (internal only)
sudo ufw allow 5432/tcp  # PostgreSQL (internal only)

# Enable firewall
sudo ufw enable
```

### Configure Timezone

```bash
# Set timezone ke Asia/Jakarta (sesuaikan)
sudo timedatectl set-timezone Asia/Jakarta
```

## 🐳 Step 2: Docker Installation

### Install Docker

```bash
# Remove old versions
sudo apt remove -y docker docker-engine docker.io containerd runc

# Install dependencies
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Setup repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Verify installation
docker --version
docker compose version
```

### Configure Docker User (Optional)

```bash
# Add current user to docker group
sudo usermod -aG docker $USER

# Logout dan login kembali untuk apply changes
# atau jalankan:
newgrp docker
```

### Configure Docker Storage Driver (Recommended untuk SSD)

```bash
# Create Docker daemon configuration
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```

Tambahkan konfigurasi berikut:

```json
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true
}
```

Restart Docker:

```bash
sudo systemctl restart docker
```

## 📁 Step 3: Project Setup

### Clone Repository

```bash
# Clone repository
cd /opt
sudo git clone https://github.com/alumnisteman/N8N.git affiliate-automation
cd affiliate-automation

# Set permissions
sudo chown -R $USER:$USER /opt/affiliate-automation
```

### Create Directory Structure

```bash
# Create necessary directories
mkdir -p data/postgres
mkdir -p data/redis
mkdir -p data/minio
mkdir -p data/qdrant
mkdir -p data/n8n
mkdir -p data/grafana
mkdir -p data/prometheus
mkdir -p data/loki
mkdir -p logs
mkdir -p uploads
mkdir -p videos
mkdir -p thumbnails
mkdir -p subtitles
```

### Set Permissions

```bash
# Set ownership
sudo chown -R 1000:1000 data/
sudo chown -R 1000:1000 logs/
sudo chown -R 1000:1000 uploads/
sudo chown -R 1000:1000 videos/
sudo chown -R 1000:1000 thumbnails/
sudo cown -R 1000:1000 subtitles/
```

## 🔐 Step 4: Environment Configuration

### Copy Environment Template

```bash
cp .env.example .env
```

### Edit Environment Variables

```bash
nano .env
```

Isi semua required variables (lihat [Environment Setup](03-ENVIRONMENT.md) untuk detail lengkap):

```bash
# Basic Configuration
PROJECT_NAME=affiliate-automation
ENVIRONMENT=development
DOMAIN=yourdomain.com

# Database
POSTGRES_DB=affiliate_db
POSTGRES_USER=affiliate_user
POSTGRES_PASSWORD=your_secure_password_here

# Redis
REDIS_PASSWORD=your_redis_password_here

# MinIO
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=your_minio_password_here

# API Keys
GEMINI_API_KEY=your_gemini_api_key_here
PEXELS_API_KEY=your_pexels_api_key_here
PIXABAY_API_KEY=your_pixabay_api_key_here

# Affiliate APIs
SHOPEE_API_KEY=your_shopee_api_key_here
TIKTOK_SHOP_API_KEY=your_tiktok_shop_api_key_here
TOKOPEDIA_API_KEY=your_tokopedia_api_key_here

# Social Media Credentials
TIKTOK_USERNAME=your_tiktok_username
TIKTOK_PASSWORD=your_tiktok_password
INSTAGRAM_USERNAME=your_instagram_username
INSTAGRAM_PASSWORD=your_instagram_password

# Notification
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_telegram_chat_id

# n8n
N8N_ENCRYPTION_KEY=your_random_encryption_key_here
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=your_n8n_password
```

### Generate Secure Passwords

```bash
# Generate random passwords
openssl rand -base64 32
```

## 🚀 Step 5: Start Services

### Start All Services

```bash
# Start all containers
docker compose up -d

# Check container status
docker compose ps

# Check logs
docker compose logs -f
```

### Verify Services

```bash
# Check if all services are running
docker compose ps

# Expected output: semua container status "Up"
```

### Access Services

- **Portainer**: http://your-server-ip:3000
- **n8n**: http://your-server-ip:5678
- **Grafana**: http://your-server-ip:3001
- **FastAPI**: http://your-server-ip:8000
- **Open WebUI**: http://your-server-ip:3002

## 🔍 Step 6: Initial Configuration

### Configure PostgreSQL

```bash
# Connect to PostgreSQL
docker exec -it affiliate-automation-postgres psql -U affiliate_user -d affiliate_db

# Create tables (jika belum otomatis)
# Exit dengan \q
```

### Configure Redis

```bash
# Test Redis connection
docker exec -it affiliate-automation-redis redis-cli -a your_redis_password ping
# Expected response: PONG
```

### Configure MinIO

```bash
# Access MinIO Console
# URL: http://your-server-ip:9001
# Username: minioadmin
# Password: your_minio_password_here

# Create buckets:
# - raw-videos
# - rendered-videos
# - subtitles
# - thumbnails
# - audio
```

### Configure n8n

1. Buka http://your-server-ip:5678
2. Login dengan credentials dari .env
3. Configure credentials untuk external APIs
4. Import workflow templates

### Configure Grafana

1. Buka http://your-server-ip:3001
2. Default credentials: admin/admin
3. Change password pada first login
4. Add Prometheus datasource
5. Import dashboard templates

## 🧪 Step 7: Testing

### Test API Health

```bash
# Test FastAPI health endpoint
curl http://localhost:8000/health

# Expected response: {"status": "healthy"}
```

### Test Database Connection

```bash
# Test PostgreSQL connection
docker exec -it affiliate-automation-postgres pg_isready -U affiliate_user
```

### Test Redis Connection

```bash
# Test Redis connection
docker exec -it affiliate-automation-redis redis-cli -a your_redis_password ping
```

### Test Worker Services

```bash
# Check worker logs
docker compose logs -f worker-ai
docker compose logs -f worker-render
docker compose logs -f worker-upload
```

## 📊 Step 8: Monitoring Setup

### Verify Monitoring Stack

```bash
# Check Prometheus
curl http://localhost:9090/-/healthy

# Check Grafana
curl http://localhost:3001/api/health

# Check Uptime Kuma
curl http://localhost:3001/api/status
```

### Configure Alerts

Setup alerts di Grafana untuk:
- Container down
- High CPU usage
- High memory usage
- Disk space low
- Job queue backlog

## 🔒 Step 9: Security Hardening

### SSL/TLS Configuration (Production)

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtain SSL certificate
sudo certbot --nginx -d yourdomain.com

# Auto-renewal sudah terkonfigurasi otomatis
```

### Configure Fail2Ban

```bash
# Edit jail configuration
sudo nano /etc/fail2ban/jail.local
```

Tambahkan konfigurasi untuk Docker services:

```ini
[nginx-noscript]
enabled = true
port = http,https
logpath = /var/log/nginx/access.log
maxretry = 6
```

Restart Fail2Ban:

```bash
sudo systemctl restart fail2ban
```

### Backup Configuration

Setup automated backup (lihat [Production Deployment](08-PRODUCTION.md)).

## 🎉 Step 10: Ready to Use

Sistem sudah siap digunakan!

### Next Steps

1. Configure n8n workflows (lihat [n8n Workflows](06-N8N-WORKFLOWS.md))
2. Setup affiliate API credentials
3. Test dengan produk sample
4. Monitor dashboard Grafana
5. Setup notification system

## 🆘 Troubleshooting

Jika mengalami masalah, lihat [Troubleshooting Guide](09-TROUBLESHOOTING.md).

### Common Issues

**Container tidak start**
```bash
# Check logs
docker compose logs <service-name>

# Restart service
docker compose restart <service-name>
```

**Permission denied**
```bash
# Fix permissions
sudo chown -R 1000:1000 data/
```

**Port already in use**
```bash
# Check port usage
sudo netstat -tulpn | grep <port>

# Kill process jika perlu
sudo kill -9 <pid>
```

## 📚 Additional Resources

- [Docker Documentation](https://docs.docker.com/)
- [n8n Documentation](https://docs.n8n.io/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Grafana Documentation](https://grafana.com/docs/)
