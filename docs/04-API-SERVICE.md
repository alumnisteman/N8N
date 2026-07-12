# API Service Documentation

Dokumentasi lengkap untuk FastAPI service yang berfungsi sebagai API layer utama sistem.

## 🏗️ Arsitektur API

```
n8n → FastAPI → Database/Redis
         ↓
    Business Logic
         ↓
    Workers (via Redis Queue)
```

## 📁 Struktur Direktori

```
api/
├── Dockerfile
├── requirements.txt
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI application entry point
│   ├── config.py            # Configuration management
│   ├── database.py          # Database connection
│   ├── redis.py             # Redis connection
│   ├── models/              # SQLAlchemy models
│   │   ├── __init__.py
│   │   ├── product.py
│   │   ├── video.py
│   │   ├── job.py
│   │   └── analytics.py
│   ├── schemas/             # Pydantic schemas
│   │   ├── __init__.py
│   │   ├── product.py
│   │   ├── video.py
│   │   ├── job.py
│   │   └── analytics.py
│   ├── api/                 # API routes
│   │   ├── __init__.py
│   │   ├── products.py
│   │   ├── videos.py
│   │   ├── jobs.py
│   │   ├── analytics.py
│   │   └── health.py
│   ├── services/            # Business logic
│   │   ├── __init__.py
│   │   ├── product_service.py
│   │   ├── video_service.py
│   │   ├── job_service.py
│   │   └── analytics_service.py
│   ├── workers/             # Queue job definitions
│   │   ├── __init__.py
│   │   ├── ai_worker.py
│   │   ├── render_worker.py
│   │   └── upload_worker.py
│   └── utils/               # Utility functions
│       ├── __init__.py
│       ├── logger.py
│       ├── security.py
│       └── helpers.py
└── tests/                   # Unit tests
    ├── __init__.py
    ├── test_products.py
    ├── test_videos.py
    └── test_jobs.py
```

## 🚀 Setup API Service

### 1. Buat Dockerfile

```dockerfile
# api/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create directories
RUN mkdir -p logs uploads

# Expose port
EXPOSE 8000

# Run application
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

### 2. Buat requirements.txt

```txt
# api/requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
pydantic-settings==2.1.0
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
alembic==1.13.0
redis==5.0.1
celery==5.3.4
python-multipart==0.0.6
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-dotenv==1.0.0
httpx==0.25.2
aiofiles==23.2.1
loguru==0.7.2
prometheus-client==0.19.0
opentelemetry-api==1.21.0
opentelemetry-sdk==1.21.0
opentelemetry-instrumentation-fastapi==0.42b0
```

### 3. Buat main.py

```python
# api/app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from contextlib import asynccontextmanager
from app.config import settings
from app.database import engine, Base
from app.redis import redis_client
from app.api import products, videos, jobs, analytics, health
from app.utils.logger import logger
import prometheus_client as prom


# Prometheus metrics
REQUEST_COUNT = prom.Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint'])
REQUEST_LATENCY = prom.Histogram('http_request_latency_seconds', 'HTTP request latency')

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    logger.info("Starting API service...")
    # Create database tables
    Base.metadata.create_all(bind=engine)
    # Test Redis connection
    try:
        await redis_client.ping()
        logger.info("Redis connection successful")
    except Exception as e:
        logger.error(f"Redis connection failed: {e}")
    yield
    # Shutdown
    logger.info("Shutting down API service...")
    await redis_client.close()

# Create FastAPI app
app = FastAPI(
    title="Affiliate Automation API",
    description="API for affiliate marketing automation system",
    version="1.0.0",
    lifespan=lifespan
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routers
app.include_router(health.router, prefix="/health", tags=["Health"])
app.include_router(products.router, prefix="/api/v1/products", tags=["Products"])
app.include_router(videos.router, prefix="/api/v1/videos", tags=["Videos"])
app.include_router(jobs.router, prefix="/api/v1/jobs", tags=["Jobs"])
app.include_router(analytics.router, prefix="/api/v1/analytics", tags=["Analytics"])

# Metrics endpoint
@app.get("/metrics")
async def metrics():
    return prom.generate_latest()

# Global exception handler
@app.exception_handler(Exception)
async def global_exception_handler(request, exc):
    logger.error(f"Global exception: {exc}")
    return JSONResponse(
        status_code=500,
        content={"detail": "Internal server error"}
    )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 4. Buat config.py

```python
# api/app/config.py
from pydantic_settings import BaseSettings
from typing import List
import os


class Settings(BaseSettings):
    # Application
    APP_NAME: str = "Affiliate Automation API"
    DEBUG: bool = True
    ENVIRONMENT: str = "development"
    
    # Database
    DATABASE_URL: str
    
    # Redis
    REDIS_URL: str
    
    # API Keys
    GEMINI_API_KEY: str
    PEXELS_API_KEY: str
    PIXABAY_API_KEY: str
    SHOPEE_API_KEY: str
    TIKTOK_SHOP_API_KEY: str
    TOKOPEDIA_API_KEY: str
    
    # JWT
    JWT_SECRET: str
    JWT_ALGORITHM: str = "HS256"
    JWT_EXPIRE_MINUTES: int = 60
    
    # CORS
    CORS_ORIGINS: List[str] = ["*"]
    
    # Workers
    WORKER_AI_MAX_CONCURRENT: int = 3
    WORKER_RENDER_MAX_CONCURRENT: int = 2
    WORKER_UPLOAD_MAX_CONCURRENT: int = 3
    
    class Config:
        env_file = ".env"
        case_sensitive = True


settings = Settings()
```

### 5. Buat database.py

```python
# api/app/database.py
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from app.config import settings

# Create database engine
engine = create_engine(settings.DATABASE_URL, pool_pre_ping=True)

# Create session factory
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Create base class for models
Base = declarative_base()


# Dependency to get database session
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 6. Buat redis.py

```python
# api/app/redis.py
import redis.asyncio as redis
from app.config import settings

# Create Redis client
redis_client = redis.from_url(
    settings.REDIS_URL,
    encoding="utf-8",
    decode_responses=True
)


async def get_redis():
    return redis_client
```

## 📊 API Endpoints

### Health Check

```python
# api/app/api/health.py
from fastapi import APIRouter
from app.redis import redis_client
from app.database import engine

router = APIRouter()


@router.get("/")
async def health_check():
    """Basic health check"""
    return {"status": "healthy", "service": "affiliate-api"}


@router.get("/detailed")
async def detailed_health():
    """Detailed health check with dependencies"""
    health_status = {
        "status": "healthy",
        "service": "affiliate-api",
        "dependencies": {}
    }
    
    # Check database
    try:
        with engine.connect() as conn:
            conn.execute("SELECT 1")
        health_status["dependencies"]["database"] = "healthy"
    except Exception as e:
        health_status["dependencies"]["database"] = f"unhealthy: {str(e)}"
        health_status["status"] = "degraded"
    
    # Check Redis
    try:
        await redis_client.ping()
        health_status["dependencies"]["redis"] = "healthy"
    except Exception as e:
        health_status["dependencies"]["redis"] = f"unhealthy: {str(e)}"
        health_status["status"] = "degraded"
    
    return health_status
```

### Products API

```python
# api/app/api/products.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List
from app.database import get_db
from app.schemas.product import ProductCreate, ProductResponse, ProductUpdate
from app.services.product_service import ProductService

router = APIRouter()
product_service = ProductService()


@router.post("/", response_model=ProductResponse)
async def create_product(product: ProductCreate, db: Session = Depends(get_db)):
    """Create new product"""
    return await product_service.create_product(db, product)


@router.get("/", response_model=List[ProductResponse])
async def get_products(
    skip: int = 0,
    limit: int = 100,
    db: Session = Depends(get_db)
):
    """Get all products with pagination"""
    return await product_service.get_products(db, skip, limit)


@router.get("/{product_id}", response_model=ProductResponse)
async def get_product(product_id: int, db: Session = Depends(get_db)):
    """Get product by ID"""
    product = await product_service.get_product(db, product_id)
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    return product


@router.put("/{product_id}", response_model=ProductResponse)
async def update_product(
    product_id: int,
    product: ProductUpdate,
    db: Session = Depends(get_db)
):
    """Update product"""
    return await product_service.update_product(db, product_id, product)


@router.delete("/{product_id}")
async def delete_product(product_id: int, db: Session = Depends(get_db)):
    """Delete product"""
    await product_service.delete_product(db, product_id)
    return {"message": "Product deleted successfully"}


@router.post("/{product_id}/generate-content")
async def generate_content(product_id: int, db: Session = Depends(get_db)):
    """Trigger AI content generation for product"""
    job = await product_service.generate_content(db, product_id)
    return {"job_id": job.id, "status": "queued"}
```

### Videos API

```python
# api/app/api/videos.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List
from app.database import get_db
from app.schemas.video import VideoCreate, VideoResponse, VideoUpdate
from app.services.video_service import VideoService

router = APIRouter()
video_service = VideoService()


@router.post("/", response_model=VideoResponse)
async def create_video(video: VideoCreate, db: Session = Depends(get_db)):
    """Create new video"""
    return await video_service.create_video(db, video)


@router.get("/", response_model=List[VideoResponse])
async def get_videos(
    skip: int = 0,
    limit: int = 100,
    status: str = None,
    db: Session = Depends(get_db)
):
    """Get all videos with optional status filter"""
    return await video_service.get_videos(db, skip, limit, status)


@router.get("/{video_id}", response_model=VideoResponse)
async def get_video(video_id: int, db: Session = Depends(get_db)):
    """Get video by ID"""
    video = await video_service.get_video(db, video_id)
    if not video:
        raise HTTPException(status_code=404, detail="Video not found")
    return video


@router.post("/{video_id}/render")
async def render_video(video_id: int, db: Session = Depends(get_db)):
    """Trigger video rendering"""
    job = await video_service.render_video(db, video_id)
    return {"job_id": job.id, "status": "queued"}


@router.post("/{video_id}/upload")
async def upload_video(
    video_id: int,
    platform: str,
    db: Session = Depends(get_db)
):
    """Trigger video upload to platform"""
    job = await video_service.upload_video(db, video_id, platform)
    return {"job_id": job.id, "status": "queued"}
```

### Jobs API

```python
# api/app/api/jobs.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List
from app.database import get_db
from app.schemas.job import JobResponse
from app.services.job_service import JobService

router = APIRouter()
job_service = JobService()


@router.get("/", response_model=List[JobResponse])
async def get_jobs(
    skip: int = 0,
    limit: int = 100,
    status: str = None,
    worker_type: str = None,
    db: Session = Depends(get_db)
):
    """Get all jobs with filters"""
    return await job_service.get_jobs(db, skip, limit, status, worker_type)


@router.get("/{job_id}", response_model=JobResponse)
async def get_job(job_id: str, db: Session = Depends(get_db)):
    """Get job by ID"""
    job = await job_service.get_job(db, job_id)
    if not job:
        raise HTTPException(status_code=404, detail="Job not found")
    return job


@router.post("/{job_id}/retry")
async def retry_job(job_id: str, db: Session = Depends(get_db)):
    """Retry failed job"""
    job = await job_service.retry_job(db, job_id)
    return {"job_id": job.id, "status": "queued"}


@router.delete("/{job_id}")
async def cancel_job(job_id: str, db: Session = Depends(get_db)):
    """Cancel job"""
    await job_service.cancel_job(db, job_id)
    return {"message": "Job cancelled successfully"}
```

### Analytics API

```python
# api/app/api/analytics.py
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from datetime import datetime, timedelta
from app.database import get_db
from app.services.analytics_service import AnalyticsService

router = APIRouter()
analytics_service = AnalyticsService()


@router.get("/summary")
async def get_analytics_summary(
    start_date: datetime = None,
    end_date: datetime = None,
    db: Session = Depends(get_db)
):
    """Get analytics summary for date range"""
    if not start_date:
        start_date = datetime.now() - timedelta(days=7)
    if not end_date:
        end_date = datetime.now()
    
    return await analytics_service.get_summary(db, start_date, end_date)


@router.get("/top-products")
async def get_top_products(
    limit: int = 10,
    db: Session = Depends(get_db)
):
    """Get top performing products"""
    return await analytics_service.get_top_products(db, limit)


@router.get("/platform-performance")
async def get_platform_performance(db: Session = Depends(get_db)):
    """Get performance by platform"""
    return await analytics_service.get_platform_performance(db)


@router.get("/viral-videos")
async def get_viral_videos(
    limit: int = 10,
    db: Session = Depends(get_db)
):
    """Get most viral videos"""
    return await analytics_service.get_viral_videos(db, limit)
```

## 🔐 Security

### JWT Authentication

```python
# api/app/utils/security.py
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from app.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify password against hash"""
    return pwd_context.verify(plain_password, hashed_password)


def get_password_hash(password: str) -> str:
    """Hash password"""
    return pwd_context.hash(password)


def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """Create JWT access token"""
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=settings.JWT_EXPIRE_MINUTES)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.JWT_ALGORITHM)
    return encoded_jwt


def decode_access_token(token: str) -> Optional[dict]:
    """Decode JWT access token"""
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.JWT_ALGORITHM])
        return payload
    except JWTError:
        return None
```

## 🧪 Testing

### Run Tests

```bash
# Install test dependencies
pip install pytest pytest-asyncio httpx

# Run tests
pytest tests/

# Run with coverage
pytest tests/ --cov=app --cov-report=html
```

### Example Test

```python
# api/tests/test_products.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)


def test_health_check():
    """Test health check endpoint"""
    response = client.get("/health/")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"


def test_create_product():
    """Test product creation"""
    product_data = {
        "name": "Test Product",
        "price": 99.99,
        "affiliate_url": "https://example.com/product",
        "platform": "shopee"
    }
    response = client.post("/api/v1/products/", json=product_data)
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Test Product"
    assert "id" in data
```

## 📈 Monitoring

### Prometheus Metrics

API service exposes metrics at `/metrics` endpoint:

- `http_requests_total`: Total HTTP requests by method and endpoint
- `http_request_latency_seconds`: HTTP request latency histogram
- Custom metrics for business logic

### Logging

Structured JSON logging using Loguru:

```python
# api/app/utils/logger.py
from loguru import logger
import sys

logger.remove()
logger.add(
    sys.stdout,
    format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - <level>{message}</level>",
    level="INFO"
)
logger.add(
    "logs/api.log",
    rotation="10 MB",
    retention="10 days",
    format="{time:YYYY-MM-DD HH:mm:ss} | {level: <8} | {name}:{function}:{line} - {message}",
    level="INFO"
)
```

## 🚀 Deployment

### Build and Run

```bash
# Build Docker image
docker build -t affiliate-api ./api

# Run container
docker run -d \
  --name affiliate-api \
  -p 8000:8000 \
  --env-file .env \
  affiliate-api

# Or use docker-compose
docker compose up -d api
```

### Health Check

```bash
# Check API health
curl http://localhost:8000/health/

# Check metrics
curl http://localhost:8000/metrics
```

## 📚 Additional Resources

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [SQLAlchemy Documentation](https://docs.sqlalchemy.org/)
- [Redis Documentation](https://redis.io/docs/)
