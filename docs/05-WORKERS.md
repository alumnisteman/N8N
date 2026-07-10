# Worker Configuration Documentation

Dokumentasi lengkap untuk konfigurasi dan setup worker services (AI, Render, Upload, Media, Analytics).

## 🏗️ Arsitektur Worker

```
Redis Queue
    │
    ├─→ Worker AI (Content Generation)
    ├─→ Worker Media (Video Stock Download)
    ├─→ Worker Render (Video Rendering)
    ├─→ Worker Upload (Social Media Upload)
    └─→ Worker Analytics (Performance Tracking)
```

## 📁 Struktur Direktori Workers

```
workers/
├── ai/                      # AI Content Generation Worker
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── config.py
│   ├── jobs/
│   │   ├── generate_content.py
│   │   └── generate_script.py
│   └── utils/
│       ├── gemini_client.py
│       └── qdrant_client.py
│
├── render/                  # Video Rendering Worker
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── config.py
│   ├── jobs/
│   │   ├── render_video.py
│   │   ├── add_subtitles.py
│   │   └── add_watermark.py
│   └── utils/
│       ├── ffmpeg.py
│       └── moviepy.py
│
├── upload/                  # Social Media Upload Worker
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── config.py
│   ├── jobs/
│   │   ├── upload_tiktok.py
│   │   ├── upload_instagram.py
│   │   └── upload_facebook.py
│   └── utils/
│       ├── browserless.py
│       └── social_apis.py
│
├── media/                   # Media Download Worker
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── config.py
│   ├── jobs/
│   │   ├── download_pexels.py
│   │   └── download_pixabay.py
│   └── utils/
│       ├── pexels_client.py
│       └── pixabay_client.py
│
└── analytics/               # Analytics Worker
    ├── Dockerfile
    ├── requirements.txt
    ├── main.py
    ├── config.py
    ├── jobs/
    │   ├── fetch_analytics.py
    │   └── calculate_metrics.py
    └── utils/
        ├── tiktok_analytics.py
        └── instagram_analytics.py
```

## 🤖 Worker AI (Content Generation)

### Dockerfile

```dockerfile
# workers/ai/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create logs directory
RUN mkdir -p logs

# Worker doesn't expose ports
# It processes jobs from Redis queue

CMD ["python", "main.py"]
```

### requirements.txt

```txt
# workers/ai/requirements.txt
celery==5.3.4
redis==5.0.1
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
google-generativeai==0.3.2
qdrant-client==1.7.0
pydantic==2.5.0
python-dotenv==1.0.0
loguru==0.7.2
httpx==0.25.2
```

### main.py

```python
# workers/ai/main.py
from celery import Celery
from app.config import settings
from app.utils.logger import logger
import os

# Create Celery app
celery_app = Celery(
    "worker_ai",
    broker=settings.REDIS_URL,
    backend=settings.REDIS_URL,
    include=["app.jobs.generate_content", "app.jobs.generate_script"]
)

# Configure Celery
celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="Asia/Jakarta",
    enable_utc=True,
    task_track_started=True,
    task_time_limit=300,  # 5 minutes
    task_soft_time_limit=240,  # 4 minutes
    worker_prefetch_multiplier=1,
    worker_max_tasks_per_child=50,
)

# Configure queues
celery_app.conf.task_queues = {
    "ai_queue": {
        "exchange": "ai",
        "routing_key": "ai",
    }
}

celery_app.conf.task_default_queue = "ai_queue"
celery_app.conf.task_default_routing_key = "ai"

if __name__ == "__main__":
    logger.info("Starting AI Worker...")
    celery_app.worker_main([
        "worker",
        "--loglevel=info",
        "--queues=ai_queue",
        f"--concurrency={os.getenv('MAX_CONCURRENT_JOBS', '3')}",
        "--hostname=ai-worker@%h",
    ])
```

### config.py

```python
# workers/ai/app/config.py
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    DATABASE_URL: str
    REDIS_URL: str
    GEMINI_API_KEY: str
    QDRANT_URL: str
    WORKER_TYPE: str = "ai"
    MAX_CONCURRENT_JOBS: int = 3
    
    class Config:
        env_file = ".env"


settings = Settings()
```

### generate_content.py

```python
# workers/ai/app/jobs/generate_content.py
from celery import current_task
from app.config import settings
from app.utils.gemini_client import GeminiClient
from app.utils.qdrant_client import QdrantClient
from app.utils.logger import logger
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
import json


# Database setup
engine = create_engine(settings.DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)


# Initialize clients
gemini_client = GeminiClient(settings.GEMINI_API_KEY)
qdrant_client = QdrantClient(settings.QDRANT_URL)


def generate_content_task(product_id: int, product_data: dict):
    """Generate AI content for product"""
    task_id = current_task.request.id
    
    try:
        logger.info(f"Task {task_id}: Generating content for product {product_id}")
        
        # Update task status
        current_task.update_state(
            state="PROGRESS",
            meta={"product_id": product_id, "status": "generating_title"}
        )
        
        # Generate title
        title = gemini_client.generate_title(product_data)
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"product_id": product_id, "status": "generating_hook"}
        )
        
        # Generate hook
        hook = gemini_client.generate_hook(product_data, title)
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"product_id": product_id, "status": "generating_script"}
        )
        
        # Generate script
        script = gemini_client.generate_script(product_data, title, hook)
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"product_id": product_id, "status": "generating_caption"}
        )
        
        # Generate caption
        caption = gemini_client.generate_caption(product_data, script)
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"product_id": product_id, "status": "generating_hashtags"}
        )
        
        # Generate hashtags
        hashtags = gemini_client.generate_hashtags(product_data)
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"product_id": product_id, "status": "generating_cta"}
        )
        
        # Generate CTA
        cta = gemini_client.generate_cta(product_data)
        
        # Store in Qdrant for RAG
        qdrant_client.store_content(
            product_id=product_id,
            content={
                "title": title,
                "hook": hook,
                "script": script,
                "caption": caption,
                "hashtags": hashtags,
                "cta": cta
            }
        )
        
        # Save to database
        db = SessionLocal()
        try:
            # Update product with generated content
            # (implementation depends on your database schema)
            db.commit()
        finally:
            db.close()
        
        logger.info(f"Task {task_id}: Content generation completed for product {product_id}")
        
        return {
            "product_id": product_id,
            "title": title,
            "hook": hook,
            "script": script,
            "caption": caption,
            "hashtags": hashtags,
            "cta": cta,
            "status": "completed"
        }
        
    except Exception as e:
        logger.error(f"Task {task_id}: Error generating content: {e}")
        current_task.update_state(
            state="FAILURE",
            meta={"product_id": product_id, "error": str(e)}
        )
        raise


# Register task
from app.main import celery_app

@celery_app.task(name="app.jobs.generate_content.generate_content_task", bind=True)
def generate_content(self, product_id: int, product_data: dict):
    return generate_content_task(product_id, product_data)
```

### gemini_client.py

```python
# workers/ai/app/utils/gemini_client.py
import google.generativeai as genai
from typing import Dict, List
from loguru import logger


class GeminiClient:
    def __init__(self, api_key: str):
        genai.configure(api_key=api_key)
        self.model = genai.GenerativeModel('gemini-pro')
        
    def generate_title(self, product_data: Dict) -> str:
        """Generate catchy title for product"""
        prompt = f"""
        Generate a catchy, viral-worthy title for this product:
        Product Name: {product_data.get('name')}
        Price: {product_data.get('price')}
        Description: {product_data.get('description', '')}
        
        Requirements:
        - Maximum 60 characters
        - Use emojis
        - Create curiosity
        - Include benefit
        - Make it shareable
        
        Return only the title, no explanation.
        """
        
        try:
            response = self.model.generate_content(prompt)
            title = response.text.strip()
            logger.info(f"Generated title: {title}")
            return title
        except Exception as e:
            logger.error(f"Error generating title: {e}")
            return product_data.get('name', '')[:60]
    
    def generate_hook(self, product_data: Dict, title: str) -> str:
        """Generate hook for video"""
        prompt = f"""
        Generate a viral hook for this product video:
        Title: {title}
        Product: {product_data.get('name')}
        Price: {product_data.get('price')}
        
        Requirements:
        - Maximum 20 words
        - Create immediate attention
        - Use power words
        - Address pain point
        
        Return only the hook, no explanation.
        """
        
        try:
            response = self.model.generate_content(prompt)
            hook = response.text.strip()
            logger.info(f"Generated hook: {hook}")
            return hook
        except Exception as e:
            logger.error(f"Error generating hook: {e}")
            return f"You won't believe this {product_data.get('name', 'product')}"
    
    def generate_script(self, product_data: Dict, title: str, hook: str) -> str:
        """Generate video script"""
        prompt = f"""
        Generate a 30-60 second video script:
        Title: {title}
        Hook: {hook}
        Product: {product_data.get('name')}
        Price: {product_data.get('price')}
        Description: {product_data.get('description', '')}
        
        Requirements:
        - Conversational tone
        - Include product benefits
        - Include call-to-action
        - Format: [Scene] [Visual] [Audio]
        - Maximum 150 words
        
        Return only the script, no explanation.
        """
        
        try:
            response = self.model.generate_content(prompt)
            script = response.text.strip()
            logger.info(f"Generated script: {script[:100]}...")
            return script
        except Exception as e:
            logger.error(f"Error generating script: {e}")
            return f"{hook}. Check out this amazing {product_data.get('name', 'product')} for only {product_data.get('price', '')}!"
    
    def generate_caption(self, product_data: Dict, script: str) -> str:
        """Generate social media caption"""
        prompt = f"""
        Generate an engaging caption for this video:
        Script: {script}
        Product: {product_data.get('name')}
        
        Requirements:
        - Maximum 2200 characters
        - Use emojis
        - Include hashtags placeholder
        - Include CTA
        - Engaging questions
        
        Return only the caption, no explanation.
        """
        
        try:
            response = self.model.generate_content(prompt)
            caption = response.text.strip()
            logger.info(f"Generated caption: {caption[:100]}...")
            return caption
        except Exception as e:
            logger.error(f"Error generating caption: {e}")
            return f"Check this out! {product_data.get('name', 'Amazing product')} 🔥"
    
    def generate_hashtags(self, product_data: Dict) -> List[str]:
        """Generate relevant hashtags"""
        prompt = f"""
        Generate 15 relevant hashtags for this product:
        Product: {product_data.get('name')}
        Category: {product_data.get('category', '')}
        
        Requirements:
        - Mix of popular and niche
        - Include trending tags
        - Platform-specific (TikTok/Instagram)
        
        Return only hashtags separated by spaces, no explanation.
        """
        
        try:
            response = self.model.generate_content(prompt)
            hashtags = response.text.strip().split()
            logger.info(f"Generated hashtags: {hashtags}")
            return hashtags[:15]
        except Exception as e:
            logger.error(f"Error generating hashtags: {e}")
            return ["#viral", "#fyp", "#trending", "#musthave"]
    
    def generate_cta(self, product_data: Dict) -> str:
        """Generate call-to-action"""
        prompt = f"""
        Generate a compelling call-to-action:
        Product: {product_data.get('name')}
        Affiliate Link: {product_data.get('affiliate_url')}
        
        Requirements:
        - Action-oriented
        - Create urgency
        - Clear instruction
        - Maximum 15 words
        
        Return only the CTA, no explanation.
        """
        
        try:
            response = self.model.generate_content(prompt)
            cta = response.text.strip()
            logger.info(f"Generated CTA: {cta}")
            return cta
        except Exception as e:
            logger.error(f"Error generating CTA: {e}")
            return "Link in bio! Get yours now!"
```

## 🎬 Worker Render (Video Rendering)

### Dockerfile

```dockerfile
# workers/render/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies including FFmpeg
RUN apt-get update && apt-get install -y \
    ffmpeg \
    gcc \
    g++ \
    libsm6 \
    libxext6 \
    libxrender-dev \
    libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create directories
RUN mkdir -p logs videos thumbnails subtitles

CMD ["python", "main.py"]
```

### requirements.txt

```txt
# workers/render/requirements.txt
celery==5.3.4
redis==5.0.1
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
moviepy==1.0.3
pillow==10.1.0
pydub==0.25.1
pydantic==2.5.0
python-dotenv==1.0.0
loguru==0.7.2
opencv-python==4.8.1.78
```

### main.py

```python
# workers/render/main.py
from celery import Celery
from app.config import settings
from app.utils.logger import logger
import os

# Create Celery app
celery_app = Celery(
    "worker_render",
    broker=settings.REDIS_URL,
    backend=settings.REDIS_URL,
    include=["app.jobs.render_video", "app.jobs.add_subtitles"]
)

# Configure Celery
celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="Asia/Jakarta",
    enable_utc=True,
    task_track_started=True,
    task_time_limit=600,  # 10 minutes
    task_soft_time_limit=540,  # 9 minutes
    worker_prefetch_multiplier=1,
    worker_max_tasks_per_child=20,
)

# Configure queues
celery_app.conf.task_queues = {
    "render_queue": {
        "exchange": "render",
        "routing_key": "render",
    }
}

celery_app.conf.task_default_queue = "render_queue"
celery_app.conf.task_default_routing_key = "render"

if __name__ == "__main__":
    logger.info("Starting Render Worker...")
    celery_app.worker_main([
        "worker",
        "--loglevel=info",
        "--queues=render_queue",
        f"--concurrency={os.getenv('MAX_CONCURRENT_JOBS', '2')}",
        "--hostname=render-worker@%h",
    ])
```

### render_video.py

```python
# workers/render/app/jobs/render_video.py
from celery import current_task
from moviepy.editor import *
from app.config import settings
from app.utils.logger import logger
from app.utils.ffmpeg import FFmpegProcessor
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
import os


# Database setup
engine = create_engine(settings.DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)

# Initialize FFmpeg processor
ffmpeg_processor = FFmpegProcessor()


def render_video_task(video_id: int, video_data: dict):
    """Render video with subtitles, watermark, and audio"""
    task_id = current_task.request.id
    
    try:
        logger.info(f"Task {task_id}: Rendering video {video_id}")
        
        # Update task status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "loading_source"}
        )
        
        # Load source video
        source_path = video_data.get("source_path")
        if not os.path.exists(source_path):
            raise FileNotFoundError(f"Source video not found: {source_path}")
        
        video = VideoFileClip(source_path)
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "adding_subtitles"}
        )
        
        # Add subtitles
        if video_data.get("subtitles"):
            video = add_subtitles(video, video_data["subtitles"])
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "adding_watermark"}
        )
        
        # Add watermark
        if video_data.get("watermark"):
            video = add_watermark(video, video_data["watermark"])
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "adding_audio"}
        )
        
        # Add background music
        if video_data.get("audio_path"):
            video = add_audio(video, video_data["audio_path"])
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "resizing_video"}
        )
        
        # Resize for platform (9:16 for TikTok/Reels)
        video = resize_for_platform(video, video_data.get("platform", "tiktok"))
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "exporting_video"}
        )
        
        # Export video
        output_path = f"/app/videos/rendered_{video_id}.mp4"
        video.write_videofile(
            output_path,
            codec="libx264",
            audio_codec="aac",
            fps=30,
            preset="medium",
            threads=4
        )
        
        # Close video objects
        video.close()
        
        # Update database
        db = SessionLocal()
        try:
            # Update video status
            # (implementation depends on your database schema)
            db.commit()
        finally:
            db.close()
        
        logger.info(f"Task {task_id}: Video rendering completed for {video_id}")
        
        return {
            "video_id": video_id,
            "output_path": output_path,
            "status": "completed"
        }
        
    except Exception as e:
        logger.error(f"Task {task_id}: Error rendering video: {e}")
        current_task.update_state(
            state="FAILURE",
            meta={"video_id": video_id, "error": str(e)}
        )
        raise


def add_subtitles(video, subtitle_text):
    """Add subtitles to video"""
    from moviepy.video.tools.subtitles import SubtitlesClip
    
    # Create subtitle clip
    subtitle_clip = TextClip(
        subtitle_text,
        fontsize=24,
        color='white',
        font='Arial',
        stroke_color='black',
        stroke_width=2
    ).set_position(('center', 'bottom')).set_duration(video.duration)
    
    # Composite video with subtitles
    return CompositeVideoClip([video, subtitle_clip])


def add_watermark(video, watermark_text):
    """Add watermark to video"""
    watermark_clip = TextClip(
        watermark_text,
        fontsize=16,
        color='white',
        font='Arial',
        opacity=0.7
    ).set_position(('right', 'top')).set_duration(video.duration)
    
    return CompositeVideoClip([video, watermark_clip])


def add_audio(video, audio_path):
    """Add background audio to video"""
    audio = AudioFileClip(audio_path)
    
    # Loop audio if shorter than video
    if audio.duration < video.duration:
        audio = audio.loop(duration=video.duration)
    else:
        audio = audio.subclip(0, video.duration)
    
    # Mix audio (reduce volume)
    audio = audio.volumex(0.3)
    
    # Combine with original audio if exists
    if video.audio:
        final_audio = CompositeAudioClip([video.audio, audio])
    else:
        final_audio = audio
    
    return video.set_audio(final_audio)


def resize_for_platform(video, platform):
    """Resize video for specific platform"""
    if platform in ["tiktok", "instagram", "facebook"]:
        # 9:16 aspect ratio for vertical videos
        target_width = 1080
        target_height = 1920
        
        # Calculate current aspect ratio
        current_ratio = video.w / video.h
        
        if current_ratio > 9/16:
            # Crop to 9:16
            video = crop_to_aspect_ratio(video, 9/16)
        
        # Resize to target resolution
        video = video.resize((target_width, target_height))
    
    return video


def crop_to_aspect_ratio(video, target_ratio):
    """Crop video to target aspect ratio"""
    current_ratio = video.w / video.h
    
    if current_ratio > target_ratio:
        # Crop width
        new_width = int(video.h * target_ratio)
        x_center = video.w / 2
        x1 = x_center - new_width / 2
        x2 = x_center + new_width / 2
        video = video.crop(x1=x1, x2=x2)
    else:
        # Crop height
        new_height = int(video.w / target_ratio)
        y_center = video.h / 2
        y1 = y_center - new_height / 2
        y2 = y_center + new_height / 2
        video = video.crop(y1=y1, y2=y2)
    
    return video


# Register task
from app.main import celery_app

@celery_app.task(name="app.jobs.render_video.render_video_task", bind=True)
def render_video(self, video_id: int, video_data: dict):
    return render_video_task(video_id, video_data)
```

## 📤 Worker Upload (Social Media Upload)

### Dockerfile

```dockerfile
# workers/upload/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    chromium \
    chromium-driver \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create logs directory
RUN mkdir -p logs

CMD ["python", "main.py"]
```

### requirements.txt

```txt
# workers/upload/requirements.txt
celery==5.3.4
redis==5.0.1
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
selenium==4.15.2
webdriver-manager==4.0.1
pydantic==2.5.0
python-dotenv==1.0.0
loguru==0.7.2
httpx==0.25.2
```

### main.py

```python
# workers/upload/main.py
from celery import Celery
from app.config import settings
from app.utils.logger import logger
import os

# Create Celery app
celery_app = Celery(
    "worker_upload",
    broker=settings.REDIS_URL,
    backend=settings.REDIS_URL,
    include=["app.jobs.upload_tiktok", "app.jobs.upload_instagram"]
)

# Configure Celery
celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="Asia/Jakarta",
    enable_utc=True,
    task_track_started=True,
    task_time_limit=300,  # 5 minutes
    task_soft_time_limit=270,  # 4.5 minutes
    worker_prefetch_multiplier=1,
    worker_max_tasks_per_child=30,
)

# Configure queues
celery_app.conf.task_queues = {
    "upload_queue": {
        "exchange": "upload",
        "routing_key": "upload",
    }
}

celery_app.conf.task_default_queue = "upload_queue"
celery_app.conf.task_default_routing_key = "upload"

if __name__ == "__main__":
    logger.info("Starting Upload Worker...")
    celery_app.worker_main([
        "worker",
        "--loglevel=info",
        "--queues=upload_queue",
        f"--concurrency={os.getenv('MAX_CONCURRENT_JOBS', '3')}",
        "--hostname=upload-worker@%h",
    ])
```

### upload_tiktok.py

```python
# workers/upload/app/jobs/upload_tiktok.py
from celery import current_task
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from app.config import settings
from app.utils.logger import logger
from app.utils.browserless import BrowserlessClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
import time
import os


# Database setup
engine = create_engine(settings.DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)

# Initialize Browserless client
browserless_client = BrowserlessClient(
    endpoints=[
        "http://browserless-1:3000",
        "http://browserless-2:3000",
        "http://browserless-3:3000"
    ]
)


def upload_tiktok_task(video_id: int, video_data: dict):
    """Upload video to TikTok using Browserless"""
    task_id = current_task.request.id
    
    browser = None
    try:
        logger.info(f"Task {task_id}: Uploading video {video_id} to TikTok")
        
        # Update task status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "initializing_browser"}
        )
        
        # Get available Browserless instance
        browserless_url = browserless_client.get_available_instance()
        
        # Configure Chrome options
        chrome_options = Options()
        chrome_options.add_argument("--headless")
        chrome_options.add_argument("--no-sandbox")
        chrome_options.add_argument("--disable-dev-shm-usage")
        chrome_options.add_argument("--window-size=1920,1080")
        
        # Connect to Browserless
        browser = webdriver.Remote(
            command_executor=f"{browserless_url}/webdriver",
            options=chrome_options
        )
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "logging_in"}
        )
        
        # Login to TikTok
        browser.get("https://www.tiktok.com/login")
        
        # Wait for login page
        WebDriverWait(browser, 30).until(
            EC.presence_of_element_located((By.CSS_SELECTOR, "[data-e2e='login-button']"))
        )
        
        # Enter credentials
        username_field = browser.find_element(By.NAME, "username")
        password_field = browser.find_element(By.NAME, "password")
        
        username_field.send_keys(settings.TIKTOK_USERNAME)
        password_field.send_keys(settings.TIKTOK_PASSWORD)
        
        # Click login
        login_button = browser.find_element(By.CSS_SELECTOR, "[data-e2e='login-button']")
        login_button.click()
        
        # Wait for login to complete
        time.sleep(5)
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "navigating_to_upload"}
        )
        
        # Navigate to upload page
        browser.get("https://www.tiktok.com/upload")
        
        # Wait for upload page
        WebDriverWait(browser, 30).until(
            EC.presence_of_element_located((By.CSS_SELECTOR, "[data-e2e='upload-input']"))
        )
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "uploading_video"}
        )
        
        # Upload video file
        video_path = video_data.get("video_path")
        file_input = browser.find_element(By.CSS_SELECTOR, "[data-e2e='upload-input']")
        file_input.send_keys(os.path.abspath(video_path))
        
        # Wait for upload to complete
        time.sleep(10)
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "adding_caption"}
        )
        
        # Add caption
        caption_field = browser.find_element(By.CSS_SELECTOR, "[data-e2e='caption-input']")
        caption_field.send_keys(video_data.get("caption", ""))
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "adding_hashtags"}
        )
        
        # Add hashtags
        hashtags = " ".join(video_data.get("hashtags", []))
        caption_field.send_keys(f" {hashtags}")
        
        # Update status
        current_task.update_state(
            state="PROGRESS",
            meta={"video_id": video_id, "status": "publishing"}
        )
        
        # Publish video
        publish_button = browser.find_element(By.CSS_SELECTOR, "[data-e2e='post-button']")
        publish_button.click()
        
        # Wait for publish to complete
        time.sleep(5)
        
        # Get video URL
        video_url = browser.current_url
        
        # Update database
        db = SessionLocal()
        try:
            # Update video status and URL
            # (implementation depends on your database schema)
            db.commit()
        finally:
            db.close()
        
        logger.info(f"Task {task_id}: Video {video_id} uploaded to TikTok successfully")
        
        return {
            "video_id": video_id,
            "platform": "tiktok",
            "video_url": video_url,
            "status": "completed"
        }
        
    except Exception as e:
        logger.error(f"Task {task_id}: Error uploading to TikTok: {e}")
        current_task.update_state(
            state="FAILURE",
            meta={"video_id": video_id, "error": str(e)}
        )
        raise
    finally:
        if browser:
            browser.quit()


# Register task
from app.main import celery_app

@celery_app.task(name="app.jobs.upload_tiktok.upload_tiktok_task", bind=True)
def upload_tiktok(self, video_id: int, video_data: dict):
    return upload_tiktok_task(video_id, video_data)
```

### browserless.py

```python
# workers/upload/app/utils/browserless.py
import random
import requests
from loguru import logger


class BrowserlessClient:
    def __init__(self, endpoints: list):
        self.endpoints = endpoints
        self.current_index = 0
        
    def get_available_instance(self) -> str:
        """Get next available Browserless instance (round-robin)"""
        endpoint = self.endpoints[self.current_index]
        self.current_index = (self.current_index + 1) % len(self.endpoints)
        
        # Check if instance is healthy
        if self._check_health(endpoint):
            return endpoint
        
        # Try other instances
        for i, endpoint in enumerate(self.endpoints):
            if i != self.current_index and self._check_health(endpoint):
                return endpoint
        
        # If all fail, return first one anyway
        logger.warning("All Browserless instances unhealthy, using first one")
        return self.endpoints[0]
    
    def _check_health(self, endpoint: str) -> bool:
        """Check if Browserless instance is healthy"""
        try:
            response = requests.get(f"{endpoint}/health", timeout=5)
            return response.status_code == 200
        except Exception as e:
            logger.error(f"Health check failed for {endpoint}: {e}")
            return False
```

## 📊 Worker Analytics

### Dockerfile

```dockerfile
# workers/analytics/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create logs directory
RUN mkdir -p logs

CMD ["python", "main.py"]
```

### requirements.txt

```txt
# workers/analytics/requirements.txt
celery==5.3.4
redis==5.0.1
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
pydantic==2.5.0
python-dotenv==1.0.0
loguru==0.7.2
httpx==0.25.2
pandas==2.1.3
```

### main.py

```python
# workers/analytics/main.py
from celery import Celery
from app.config import settings
from app.utils.logger import logger
import os

# Create Celery app
celery_app = Celery(
    "worker_analytics",
    broker=settings.REDIS_URL,
    backend=settings.REDIS_URL,
    include=["app.jobs.fetch_analytics"]
)

# Configure Celery
celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="Asia/Jakarta",
    enable_utc=True,
    task_track_started=True,
    task_time_limit=120,  # 2 minutes
    task_soft_time_limit=100,  # 100 seconds
    worker_prefetch_multiplier=1,
    worker_max_tasks_per_child=100,
)

# Configure queues
celery_app.conf.task_queues = {
    "analytics_queue": {
        "exchange": "analytics",
        "routing_key": "analytics",
    }
}

celery_app.conf.task_default_queue = "analytics_queue"
celery_app.conf.task_default_routing_key = "analytics"

if __name__ == "__main__":
    logger.info("Starting Analytics Worker...")
    celery_app.worker_main([
        "worker",
        "--loglevel=info",
        "--queues=analytics_queue",
        f"--concurrency={os.getenv('MAX_CONCURRENT_JOBS', '5')}",
        "--hostname=analytics-worker@%h",
    ])
```

## 🔧 Konfigurasi Worker

### Scaling Workers

Untuk scaling worker, edit `docker-compose.yml`:

```yaml
worker-ai:
  # ... existing config
  deploy:
    replicas: 4  # Increase for more capacity

worker-render:
  # ... existing config
  deploy:
    replicas: 3  # Rendering is CPU intensive

worker-upload:
  # ... existing config
  deploy:
    replicas: 5  # More upload workers for parallel uploads
```

### Monitoring Worker Performance

Gunakan Celery Flower untuk monitoring:

```yaml
# Add to docker-compose.yml
flower:
  image: mher/flower:latest
  container_name: affiliate-flower
  restart: unless-stopped
  ports:
    - "5555:5555"
  environment:
    - CELERY_BROKER_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
    - CELERY_RESULT_BACKEND=redis://:${REDIS_PASSWORD}@redis:6379/0
  command: celery flower --broker=redis://:${REDIS_PASSWORD}@redis:6379/0
  depends_on:
    - redis
  networks:
    - affiliate-network
```

### Restart Workers

```bash
# Restart specific worker
docker compose restart worker-ai

# Restart all workers
docker compose restart worker-ai worker-render worker-upload worker-media worker-analytics

# View worker logs
docker compose logs -f worker-ai
```

## 🧪 Testing Workers

### Test AI Worker

```python
# Test AI content generation
from workers.ai.app.jobs.generate_content import generate_content_task

result = generate_content_task(
    product_id=1,
    product_data={
        "name": "Test Product",
        "price": "Rp 99.000",
        "description": "Amazing product description"
    }
)
print(result)
```

### Test Render Worker

```python
# Test video rendering
from workers.render.app.jobs.render_video import render_video_task

result = render_video_task(
    video_id=1,
    video_data={
        "source_path": "/app/videos/source.mp4",
        "subtitles": "Test subtitle",
        "watermark": "@affiliate",
        "platform": "tiktok"
    }
)
print(result)
```

## 📚 Additional Resources

- [Celery Documentation](https://docs.celeryq.dev/)
- [MoviePy Documentation](https://zulko.github.io/moviepy/)
- [Selenium Documentation](https://www.selenium.dev/documentation/)
