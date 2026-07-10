# Affiliate Marketing Automation System

Sistem otomatisasi pemasaran afiliasi berbasis AI untuk pembuatan dan distribusi konten video ke berbagai platform sosial media (TikTok, Instagram, Facebook).

## 🏗️ Arsitektur Sistem

```
                                   INTERNET
                                        │
 ┌──────────────────────────────────────┼──────────────────────────────────────┐
 │                                      │                                      │
 │  Affiliate API                       AI API                            Media API
 │ Shopee / TikTok Shop / Tokopedia     Gemini                          Pexels / Pixabay
 │                                      │                                      │
 └──────────────────────────────────────┼──────────────────────────────────────┘
                                        │
──────────────────────────────── Docker Network ────────────────────────────────

                               Traefik / Nginx
                                        │
       ┌────────────────────────────────┼───────────────────────────────┐
       │                                │                               │
       ▼                                ▼                               ▼
      n8n                         FastAPI / Laravel API            Open WebUI
       │                                │
       │                                │
       ▼                                ▼
                  Redis Queue (Job Processing)
                               │
                               ▼
                       Worker Container
                  (Python / Node.js Worker)
                               │
     ┌─────────────────────────┼──────────────────────────┐
     ▼                         ▼                          ▼
Affiliate Fetch         Gemini Generate          Media Downloader
     │                         │                          │
     └──────────────┬──────────┴──────────────┬───────────┘
                    ▼                         ▼
              PostgreSQL              Local Temp Storage
                    │                         │
                    └──────────┬──────────────┘
                               ▼
             Video Rendering Worker
         (FFmpeg / Remotion / MoviePy)
                    │
                    ▼
             Browserless Chrome
                    │
          TikTok / Instagram / Facebook
                    │
                    ▼
           Analytics & Status Update
                    │
                    ▼
        Telegram / WhatsApp Notification
```

## 🚀 Fitur Utama

- **Otomatisasi Produk Afiliasi**: Mengambil produk dari Shopee, TikTok Shop, Tokopedia
- **AI Content Generation**: Generate judul, hook, naskah, caption, hashtag, dan CTA menggunakan Gemini
- **Video Rendering**: Pembuatan video otomatis dengan subtitle, voice, watermark, dan musik
- **Multi-Platform Upload**: Upload ke TikTok, Instagram, Facebook secara paralel
- **Job Queue System**: Redis Queue untuk pemrosesan job yang stabil dan skalabel
- **Analytics Tracking**: Monitoring view, like, share, CTR, dan komisi
- **Notification System**: Notifikasi Telegram/WhatsApp untuk status upload
- **Monitoring Lengkap**: Prometheus, Grafana, Loki, Uptime Kuma

## 📦 Komponen Sistem

### Core Services
- **Traefik/Nginx**: Reverse proxy dan load balancer
- **Portainer**: Manajemen Docker container
- **Watchtower**: Auto-update container
- **n8n**: Workflow orchestration
- **PostgreSQL**: Database utama
- **Redis**: Job queue dan cache

### Application Services
- **FastAPI**: API layer untuk logika bisnis
- **Worker AI**: Pemrosesan AI (Gemini)
- **Worker Render**: Video rendering (FFmpeg/Remotion)
- **Worker Upload**: Upload ke platform sosial
- **Browserless-1/2/3**: Headless Chrome untuk upload paralel

### AI & ML Services
- **Qdrant**: Vector database untuk RAG
- **Ollama**: Local LLM inference
- **Open WebUI**: UI untuk Ollama

### Monitoring Services
- **Prometheus**: Metrics collection
- **Grafana**: Dashboard monitoring
- **Loki**: Log aggregation
- **Uptime Kuma**: Uptime monitoring

## 🔄 Tahapan Otomatisasi

1. **Cron n8n** mengambil produk afiliasi
2. **API** melakukan validasi dan menyimpan ke PostgreSQL
3. **AI** menghasilkan judul, hook, naskah, caption, hashtag, dan CTA
4. **Worker media** mencari video stok dari Pexels/Pixabay
5. **Worker render** membuat video lengkap (subtitle, voice, watermark, musik)
6. Video disimpan di **local temp storage**
7. **Upload Worker** memakai Browserless untuk mengunggah langsung ke TikTok/Instagram
8. Status upload dicatat ke PostgreSQL
9. **Analytics** mengambil data performa video
10. **Telegram/WhatsApp** mengirim notifikasi jika berhasil atau gagal

## 📊 Dashboard Analytics

Dashboard menampilkan:
- Upload hari ini
- Upload gagal
- Retry status
- Komisi hari ini
- Produk terbaik
- Platform terbaik
- CTR (Click-Through Rate)
- Video paling viral

## 🎯 Keunggulan Arsitektur

- **Skalabel**: Mudah menambah worker saat beban meningkat
- **Mudah dipelihara**: Setiap layanan memiliki tanggung jawab jelas
- **Tahan gagal**: Job diproses melalui antrean Redis
- **Siap 24/7**: Monitoring dan logging lengkap
- **Ekstensibel**: Mudah menambah platform baru (YouTube Shorts, Facebook Reels, Pinterest)

## 📖 Dokumentasi

- [Installation Guide](docs/01-INSTALLATION.md) - Panduan instalasi dan setup awal
- [API Service](docs/04-API-SERVICE.md) - Setup dan dokumentasi API FastAPI
- [Worker Configuration](docs/05-WORKERS.md) - Konfigurasi worker (AI, Render, Upload)
- [n8n Workflows](docs/06-N8N-WORKFLOWS.md) - Konfigurasi workflow n8n
- [Monitoring Setup](docs/07-MONITORING.md) - Setup monitoring stack
- [Production Deployment](docs/08-PRODUCTION.md) - Panduan deployment ke production
- [Troubleshooting](docs/09-TROUBLESHOOTING.md) - Panduan troubleshooting dan maintenance

## 🔧 Prerequisites

- Ubuntu Server 20.04+ atau Docker-compatible OS
- Docker 24.0+
- Docker Compose 2.0+
- Minimum 8GB RAM (16GB recommended)
- 100GB+ storage space
- Domain name (untuk production)

## 🚀 Quick Start

```bash
# Clone repository
git clone https://github.com/alumnisteman/N8N.git affiliate-automation
cd affiliate-automation

# Copy environment template
cp .env.example .env

# Edit environment variables
nano .env

# Start all services
docker-compose up -d

# Check status
docker-compose ps
```

## 📝 License

MIT License

## 🤝 Contributing

Contributions are welcome!

## 📞 Support

Untuk pertanyaan dan support, silakan buka issue di repository.
