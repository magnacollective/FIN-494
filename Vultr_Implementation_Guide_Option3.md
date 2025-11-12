# Vultr Implementation Guide - Option 3: Maximum Performance
## Single GPU Instance Architecture

**Total 4-Week Cost:** ~$130-140
**Remaining Budget:** ~$110-120
**Approach:** Simple, powerful, everything on one GPU instance

---

## 🎯 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│         VULTR GPU INSTANCE (24/7)                       │
│         NVIDIA RTX A4000 16GB                           │
│         8 vCPU, 30GB RAM, 200GB NVMe SSD                │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Data Collector Service (Python/FastAPI)        │   │
│  │  - Port 8001                                    │   │
│  │  - Cron: Every 30 min                           │   │
│  └─────────────────────────────────────────────────┘   │
│                       │                                 │
│                       ▼                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Sentiment Processor (Python/FastAPI)           │   │
│  │  - Port 8002                                    │   │
│  │  - Uses GPU for FinBERT                         │   │
│  │  - CUDA-accelerated processing                  │   │
│  └─────────────────────────────────────────────────┘   │
│                       │                                 │
│                       ▼                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Cascade Detector (Node.js/TypeScript)          │   │
│  │  - Port 8003                                    │   │
│  └─────────────────────────────────────────────────┘   │
│                       │                                 │
│                       ▼                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │  API Gateway (Node.js/Express)                  │   │
│  │  - Port 3000                                    │   │
│  └─────────────────────────────────────────────────┘   │
│                       │                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │  PostgreSQL 15 (Docker container)               │   │
│  │  - Port 5432                                    │   │
│  │  - Volume: /data/postgres                       │   │
│  └─────────────────────────────────────────────────┘   │
│                       │                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Redis 7 (Docker container)                     │   │
│  │  - Port 6379                                    │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                       │
                       ▼ HTTPS
┌─────────────────────────────────────────────────────────┐
│              VERCEL FRONTEND (Next.js)                  │
│              Free hosting, unlimited bandwidth          │
└─────────────────────────────────────────────────────────┘
```

---

## 💰 Detailed Cost Breakdown

| Component | Specs | Monthly Cost | 4-Week Cost |
|-----------|-------|--------------|-------------|
| **GPU Instance** | RTX A4000, 8 vCPU, 30GB RAM | $90 | $83 |
| **Managed PostgreSQL** | 2 vCPU, 4GB RAM, 80GB SSD | $30 | $28 |
| **Managed Redis** | 1GB RAM | $10 | $9 |
| **Bandwidth** | ~150GB estimate | $3 | $3 |
| **Backups** | Database snapshots | $2 | $2 |
| **TOTAL** | | **$135/month** | **$125** |
| **Remaining** | | | **$125** |

### Alternative: Self-Host Database on GPU Instance

If you want to save money and keep it simple:

| Component | Monthly Cost | 4-Week Cost |
|-----------|--------------|-------------|
| **GPU Instance** (all services) | $90 | $83 |
| **Bandwidth** | $3 | $3 |
| **TOTAL** | **$93/month** | **$86** |
| **Remaining** | | **$164** |

**Recommendation:** Self-host for academic project. Use managed DB for production.

---

## 🚀 Step-by-Step Setup

### Step 1: Create Vultr Account & GPU Instance

```bash
# Install Vultr CLI
brew install vultr-cli  # macOS
# or
curl -sSL https://github.com/vultr/vultr-cli/releases/download/v2.17.0/vultr-cli_2.17.0_linux_amd64.tar.gz | tar xz

# Configure CLI
vultr-cli configure
# Enter your API key from https://my.vultr.com/settings/#settingsapi

# List available GPU plans
vultr-cli plans list-gpu --region ewr

# Create GPU instance
vultr-cli instance create \
  --plan vhp-8c-30gb-nvidia-rtx-a4000 \
  --region ewr \
  --os 542 \
  --label "sentiment-gpu-all" \
  --hostname "sentiment-gpu" \
  --enable-ipv6 false \
  --ddos-protection false \
  --backups disable
```

**Instance Details:**
- Plan: `vhp-8c-30gb-nvidia-rtx-a4000`
- OS: Ubuntu 22.04 LTS (os_id: 542)
- Region: New Jersey (ewr) - closest to financial data sources
- Monthly: $90

### Step 2: Initial Server Setup

SSH into your instance:

```bash
# Get instance IP
vultr-cli instance list

# SSH as root
ssh root@<your-instance-ip>
```

Run initial setup:

```bash
#!/bin/bash
# Save as setup.sh on your instance

# Update system
apt update && apt upgrade -y

# Install essential tools
apt install -y \
  curl \
  wget \
  git \
  vim \
  htop \
  tmux \
  build-essential \
  software-properties-common

# Install NVIDIA drivers and CUDA
apt install -y nvidia-driver-535 nvidia-utils-535
apt install -y nvidia-cuda-toolkit

# Verify GPU
nvidia-smi

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Install NVIDIA Container Toolkit (for GPU access in containers)
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | tee /etc/apt/sources.list.d/nvidia-docker.list
apt update
apt install -y nvidia-container-toolkit
systemctl restart docker

# Install Python 3.11
add-apt-repository ppa:deadsnakes/ppa -y
apt install -y python3.11 python3.11-venv python3.11-dev python3-pip

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# Create project directory
mkdir -p /opt/sentiment-cascade
cd /opt/sentiment-cascade

echo "✅ Setup complete! GPU detected:"
nvidia-smi --query-gpu=name,driver_version,memory.total --format=csv
```

### Step 3: Set Up Docker Compose

Create `/opt/sentiment-cascade/docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: sentiment-postgres
    restart: always
    environment:
      POSTGRES_DB: sentiment_db
      POSTGRES_USER: sentiment_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sentiment_user -d sentiment_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: sentiment-redis
    restart: always
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 512mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  data-collector:
    build:
      context: ./services/data-collector
      dockerfile: Dockerfile
    container_name: sentiment-data-collector
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://sentiment_user:${POSTGRES_PASSWORD}@postgres:5432/sentiment_db
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      REDDIT_CLIENT_ID: ${REDDIT_CLIENT_ID}
      REDDIT_CLIENT_SECRET: ${REDDIT_CLIENT_SECRET}
      REDDIT_USER_AGENT: ${REDDIT_USER_AGENT}
      STOCKTWITS_API_KEY: ${STOCKTWITS_API_KEY}
      NEWS_API_KEY: ${NEWS_API_KEY}
    ports:
      - "8001:8001"
    volumes:
      - ./services/data-collector:/app
      - collector_logs:/app/logs

  sentiment-processor:
    build:
      context: ./services/sentiment-processor
      dockerfile: Dockerfile
    container_name: sentiment-sentiment-processor
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://sentiment_user:${POSTGRES_PASSWORD}@postgres:5432/sentiment_db
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      CUDA_VISIBLE_DEVICES: "0"
    ports:
      - "8002:8002"
    volumes:
      - ./services/sentiment-processor:/app
      - processor_logs:/app/logs
      - huggingface_cache:/root/.cache/huggingface
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  cascade-detector:
    build:
      context: ./services/cascade-detector
      dockerfile: Dockerfile
    container_name: sentiment-cascade-detector
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://sentiment_user:${POSTGRES_PASSWORD}@postgres:5432/sentiment_db
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
    ports:
      - "8003:8003"
    volumes:
      - ./services/cascade-detector:/app
      - detector_logs:/app/logs

  api-gateway:
    build:
      context: ./services/api-gateway
      dockerfile: Dockerfile
    container_name: sentiment-api-gateway
    restart: always
    depends_on:
      - postgres
      - redis
    environment:
      DATABASE_URL: postgresql://sentiment_user:${POSTGRES_PASSWORD}@postgres:5432/sentiment_db
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      DATA_COLLECTOR_URL: http://data-collector:8001
      SENTIMENT_PROCESSOR_URL: http://sentiment-processor:8002
      CASCADE_DETECTOR_URL: http://cascade-detector:8003
      JWT_SECRET: ${JWT_SECRET}
      CORS_ORIGIN: ${CORS_ORIGIN}
    ports:
      - "3000:3000"
    volumes:
      - ./services/api-gateway:/app
      - gateway_logs:/app/logs

volumes:
  postgres_data:
  redis_data:
  collector_logs:
  processor_logs:
  detector_logs:
  gateway_logs:
  huggingface_cache:
```

Create `.env` file:

```bash
# /opt/sentiment-cascade/.env

# Database
POSTGRES_PASSWORD=your_secure_password_here

# Redis
REDIS_PASSWORD=your_redis_password_here

# Reddit API
REDDIT_CLIENT_ID=your_client_id
REDDIT_CLIENT_SECRET=your_client_secret
REDDIT_USER_AGENT=sentiment-cascade:v1.0 (by /u/yourusername)

# StockTwits
STOCKTWITS_API_KEY=your_stocktwits_key

# News API
NEWS_API_KEY=your_newsapi_key

# JWT
JWT_SECRET=your_jwt_secret_here

# CORS (Vercel frontend URL)
CORS_ORIGIN=https://your-app.vercel.app
```

### Step 4: Create Service Structure

```bash
cd /opt/sentiment-cascade

# Create directory structure
mkdir -p services/{data-collector,sentiment-processor,cascade-detector,api-gateway}
mkdir -p services/data-collector/{src,tests,logs}
mkdir -p services/sentiment-processor/{src,tests,logs}
mkdir -p services/cascade-detector/{src,tests,logs}
mkdir -p services/api-gateway/{src,tests,logs}

# Create initial files
touch services/data-collector/Dockerfile
touch services/sentiment-processor/Dockerfile
touch services/cascade-detector/Dockerfile
touch services/api-gateway/Dockerfile
```

### Step 5: Data Collector Service

Create `/opt/sentiment-cascade/services/data-collector/Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Expose port
EXPOSE 8001

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8001/health')"

# Run application
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8001", "--reload"]
```

Create `/opt/sentiment-cascade/services/data-collector/requirements.txt`:

```
fastapi==0.104.1
uvicorn[standard]==0.24.0
praw==7.7.1
requests==2.31.0
psycopg2-binary==2.9.9
redis==5.0.1
python-dotenv==1.0.0
pydantic==2.5.0
pydantic-settings==2.1.0
aiohttp==3.9.1
asyncpg==0.29.0
sqlalchemy==2.0.23
yfinance==0.2.32
```

Create `/opt/sentiment-cascade/services/data-collector/src/main.py`:

```python
from fastapi import FastAPI, HTTPException
from contextlib import asynccontextmanager
import asyncio
from .collectors.reddit_collector import RedditCollector
from .collectors.stocktwits_collector import StockTwitsCollector
from .collectors.news_collector import NewsCollector
from .collectors.market_data_collector import MarketDataCollector
from .database import Database
from .config import settings
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Initialize collectors
reddit_collector = None
stocktwits_collector = None
news_collector = None
market_collector = None
db = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    global reddit_collector, stocktwits_collector, news_collector, market_collector, db

    logger.info("🚀 Starting Data Collector Service...")

    # Initialize database
    db = Database()
    await db.connect()

    # Initialize collectors
    reddit_collector = RedditCollector(db)
    stocktwits_collector = StockTwitsCollector(db)
    news_collector = NewsCollector(db)
    market_collector = MarketDataCollector(db)

    # Start background collection tasks
    asyncio.create_task(collection_scheduler())

    logger.info("✅ Data Collector Service ready!")

    yield

    # Shutdown
    logger.info("🛑 Shutting down Data Collector Service...")
    await db.disconnect()

app = FastAPI(title="Sentiment Cascade Data Collector", lifespan=lifespan)

async def collection_scheduler():
    """Background task that runs data collection every 30 minutes"""
    while True:
        try:
            logger.info("📡 Starting data collection cycle...")

            # Run all collectors in parallel
            await asyncio.gather(
                reddit_collector.collect(),
                stocktwits_collector.collect(),
                news_collector.collect(),
                market_collector.collect(),
                return_exceptions=True
            )

            logger.info("✅ Data collection cycle complete!")

        except Exception as e:
            logger.error(f"❌ Error in collection cycle: {e}")

        # Wait 30 minutes
        await asyncio.sleep(30 * 60)

@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "service": "data-collector",
        "version": "1.0.0"
    }

@app.post("/collect/reddit")
async def trigger_reddit_collection():
    """Manual trigger for Reddit collection"""
    try:
        result = await reddit_collector.collect()
        return {"status": "success", "data": result}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/collect/stocktwits")
async def trigger_stocktwits_collection():
    """Manual trigger for StockTwits collection"""
    try:
        result = await stocktwits_collector.collect()
        return {"status": "success", "data": result}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/collect/news")
async def trigger_news_collection():
    """Manual trigger for News collection"""
    try:
        result = await news_collector.collect()
        return {"status": "success", "data": result}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/collect/market")
async def trigger_market_collection():
    """Manual trigger for market data collection"""
    try:
        result = await market_collector.collect()
        return {"status": "success", "data": result}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/stats")
async def get_collection_stats():
    """Get collection statistics"""
    stats = await db.get_collection_stats()
    return stats
```

### Step 6: Sentiment Processor Service (GPU-Accelerated)

Create `/opt/sentiment-cascade/services/sentiment-processor/Dockerfile`:

```dockerfile
FROM nvidia/cuda:12.1.0-cudnn8-runtime-ubuntu22.04

WORKDIR /app

# Install Python 3.11
RUN apt-get update && apt-get install -y \
    python3.11 \
    python3.11-venv \
    python3-pip \
    git \
    && rm -rf /var/lib/apt/lists/*

# Set Python 3.11 as default
RUN update-alternatives --install /usr/bin/python python /usr/bin/python3.11 1
RUN update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.11 1

# Copy requirements
COPY requirements.txt .

# Install PyTorch with CUDA support
RUN pip install --no-cache-dir torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

# Install other requirements
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Expose port
EXPOSE 8002

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8002/health')"

# Run application
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8002"]
```

Create `/opt/sentiment-cascade/services/sentiment-processor/requirements.txt`:

```
fastapi==0.104.1
uvicorn[standard]==0.24.0
transformers==4.35.2
vaderSentiment==3.3.2
torch>=2.1.0
psycopg2-binary==2.9.9
redis==5.0.1
python-dotenv==1.0.0
pydantic==2.5.0
sqlalchemy==2.0.23
asyncpg==0.29.0
numpy==1.26.2
```

Create `/opt/sentiment-cascade/services/sentiment-processor/src/main.py`:

```python
from fastapi import FastAPI, HTTPException
from contextlib import asynccontextmanager
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
from .database import Database
from .config import settings
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Global models
finbert_model = None
finbert_tokenizer = None
vader_analyzer = None
device = None
db = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    global finbert_model, finbert_tokenizer, vader_analyzer, device, db

    logger.info("🚀 Starting Sentiment Processor Service...")

    # Check GPU availability
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    logger.info(f"🎮 Using device: {device}")

    if torch.cuda.is_available():
        logger.info(f"   GPU: {torch.cuda.get_device_name(0)}")
        logger.info(f"   Memory: {torch.cuda.get_device_properties(0).total_memory / 1e9:.2f} GB")

    # Load FinBERT model
    logger.info("📦 Loading FinBERT model...")
    finbert_tokenizer = AutoTokenizer.from_pretrained("ProsusAI/finbert")
    finbert_model = AutoModelForSequenceClassification.from_pretrained("ProsusAI/finbert")
    finbert_model.to(device)
    finbert_model.eval()
    logger.info("✅ FinBERT loaded!")

    # Load VADER
    logger.info("📦 Loading VADER...")
    vader_analyzer = SentimentIntensityAnalyzer()
    logger.info("✅ VADER loaded!")

    # Connect to database
    db = Database()
    await db.connect()

    logger.info("✅ Sentiment Processor Service ready!")

    yield

    # Shutdown
    logger.info("🛑 Shutting down Sentiment Processor Service...")
    await db.disconnect()

app = FastAPI(title="Sentiment Cascade Processor", lifespan=lifespan)

def analyze_with_finbert(text: str) -> float:
    """Analyze sentiment using FinBERT (GPU-accelerated)"""
    try:
        inputs = finbert_tokenizer(text, return_tensors="pt", truncation=True, max_length=512)
        inputs = {k: v.to(device) for k, v in inputs.items()}

        with torch.no_grad():
            outputs = finbert_model(**inputs)
            predictions = torch.nn.functional.softmax(outputs.logits, dim=-1)

        # FinBERT outputs: [positive, negative, neutral]
        # Convert to -1 to 1 scale
        positive = predictions[0][0].item()
        negative = predictions[0][1].item()
        neutral = predictions[0][2].item()

        score = positive - negative  # -1 to 1
        return score

    except Exception as e:
        logger.error(f"FinBERT error: {e}")
        return 0.0

def analyze_with_vader(text: str) -> float:
    """Analyze sentiment using VADER"""
    try:
        scores = vader_analyzer.polarity_scores(text)
        return scores['compound']  # -1 to 1
    except Exception as e:
        logger.error(f"VADER error: {e}")
        return 0.0

def hybrid_sentiment(text: str, source: str, engagement: int) -> dict:
    """Hybrid sentiment analysis strategy"""
    vader_score = analyze_with_vader(text)

    # Use FinBERT for:
    # 1. News articles (always)
    # 2. High-engagement posts (>100)
    # 3. Complex financial text
    use_finbert = (
        source == "news" or
        engagement > 100 or
        any(term in text.lower() for term in ["earnings", "revenue", "eps", "guidance", "forecast"])
    )

    if use_finbert:
        finbert_score = analyze_with_finbert(text)
        final_score = finbert_score  # Trust FinBERT more
    else:
        finbert_score = None
        final_score = vader_score

    return {
        "vader_score": vader_score,
        "finbert_score": finbert_score,
        "final_score": final_score,
        "used_finbert": use_finbert
    }

@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "service": "sentiment-processor",
        "device": str(device),
        "gpu_available": torch.cuda.is_available(),
        "models_loaded": finbert_model is not None and vader_analyzer is not None
    }

@app.post("/analyze")
async def analyze_sentiment(text: str, source: str = "reddit", engagement: int = 0):
    """Analyze sentiment for a single text"""
    try:
        result = hybrid_sentiment(text, source, engagement)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/process/batch")
async def process_batch():
    """Process all unprocessed posts in database"""
    try:
        # Get unprocessed posts
        posts = await db.get_unprocessed_posts()

        logger.info(f"Processing {len(posts)} posts...")

        processed_count = 0
        for post in posts:
            sentiment = hybrid_sentiment(
                post['text'],
                post['source'],
                post['engagement_score']
            )

            # Store sentiment
            await db.store_sentiment(post['id'], sentiment)
            processed_count += 1

            if processed_count % 100 == 0:
                logger.info(f"Processed {processed_count}/{len(posts)} posts")

        # Aggregate by ticker
        await db.aggregate_sentiments()

        return {
            "status": "success",
            "processed_count": processed_count
        }

    except Exception as e:
        logger.error(f"Batch processing error: {e}")
        raise HTTPException(status_code=500, detail=str(e))
```

---

## 🔄 Automated Deployment Script

Create `/opt/sentiment-cascade/deploy.sh`:

```bash
#!/bin/bash

set -e  # Exit on error

echo "🚀 Deploying Sentiment Cascade Detection Platform..."

# Pull latest code
cd /opt/sentiment-cascade
git pull origin main

# Load environment variables
source .env

# Build and start services
echo "🐳 Starting Docker Compose..."
docker-compose down
docker-compose build --no-cache
docker-compose up -d

# Wait for services to be healthy
echo "⏳ Waiting for services to be healthy..."
sleep 30

# Check service health
echo "🏥 Checking service health..."
curl -f http://localhost:8001/health || echo "❌ Data Collector unhealthy"
curl -f http://localhost:8002/health || echo "❌ Sentiment Processor unhealthy"
curl -f http://localhost:8003/health || echo "❌ Cascade Detector unhealthy"
curl -f http://localhost:3000/health || echo "❌ API Gateway unhealthy"

# Show logs
echo "📋 Service status:"
docker-compose ps

echo "✅ Deployment complete!"
echo "🌐 API Gateway: http://$(hostname -I | awk '{print $1}'):3000"
echo "📊 View logs: docker-compose logs -f"
```

Make it executable:

```bash
chmod +x /opt/sentiment-cascade/deploy.sh
```

---

## 📊 Monitoring & Maintenance

### System Monitoring Script

Create `/opt/sentiment-cascade/monitor.sh`:

```bash
#!/bin/bash

echo "=== Sentiment Cascade System Status ==="
echo ""

# GPU Status
echo "🎮 GPU Status:"
nvidia-smi --query-gpu=name,temperature.gpu,utilization.gpu,utilization.memory,memory.used,memory.total --format=csv,noheader,nounits

echo ""

# Docker Status
echo "🐳 Docker Services:"
docker-compose ps

echo ""

# Database Size
echo "💾 Database Size:"
docker exec sentiment-postgres psql -U sentiment_user -d sentiment_db -c "
SELECT
    pg_size_pretty(pg_database_size('sentiment_db')) as db_size,
    (SELECT count(*) FROM raw_posts) as raw_posts_count,
    (SELECT count(*) FROM sentiment_scores) as sentiment_count,
    (SELECT count(*) FROM cascades) as cascade_count;
"

echo ""

# Recent Logs
echo "📋 Recent Errors (last 10):"
docker-compose logs --tail=10 | grep -i error || echo "No recent errors"

echo ""

# Disk Usage
echo "💽 Disk Usage:"
df -h /opt/sentiment-cascade
```

Make it executable:

```bash
chmod +x /opt/sentiment-cascade/monitor.sh
```

Run monitoring every hour:

```bash
# Add to crontab
crontab -e

# Add this line:
0 * * * * /opt/sentiment-cascade/monitor.sh >> /var/log/sentiment-monitor.log 2>&1
```

---

## 🔒 Security Checklist

- [ ] Change default passwords in `.env`
- [ ] Set up firewall (UFW)
- [ ] Enable fail2ban for SSH protection
- [ ] Set up SSL/TLS for API Gateway (Let's Encrypt)
- [ ] Restrict PostgreSQL access to localhost
- [ ] Enable Redis password authentication
- [ ] Set up automated backups
- [ ] Configure log rotation
- [ ] Set up monitoring alerts

### Quick Security Setup

```bash
#!/bin/bash

# Firewall
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 3000  # API Gateway
ufw enable

# Fail2ban
apt install -y fail2ban
systemctl enable fail2ban
systemctl start fail2ban

# Automated backups
mkdir -p /backups/postgres
echo "0 2 * * * docker exec sentiment-postgres pg_dump -U sentiment_user sentiment_db > /backups/postgres/backup-\$(date +\%Y\%m\%d).sql" | crontab -

echo "✅ Basic security configured!"
```

---

## 📈 Performance Optimization

### GPU Memory Management

Monitor GPU memory:

```python
# In sentiment processor service
import torch

def get_gpu_memory():
    if torch.cuda.is_available():
        return {
            "allocated": torch.cuda.memory_allocated() / 1e9,
            "reserved": torch.cuda.memory_reserved() / 1e9,
            "total": torch.cuda.get_device_properties(0).total_memory / 1e9
        }
    return None

# Clear cache periodically
torch.cuda.empty_cache()
```

### Database Indexing

Ensure proper indexes exist:

```sql
-- Add to init.sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_sentiment_ticker_time
ON sentiment_scores(ticker_id, timestamp DESC);

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_cascades_ticker_detected
ON cascades(ticker_id, detected_at DESC);

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_raw_posts_created
ON raw_posts(created_at DESC);

-- Analyze tables for query optimization
ANALYZE sentiment_scores;
ANALYZE cascades;
ANALYZE raw_posts;
```

---

## 🎓 Cost Tracking

Create a daily cost tracking script:

```bash
#!/bin/bash
# /opt/sentiment-cascade/track-costs.sh

LOGFILE="/var/log/sentiment-costs.log"
BUDGET=250
START_DATE="2024-11-12"

# Calculate days elapsed
DAYS_ELAPSED=$(( ($(date +%s) - $(date -d "$START_DATE" +%s)) / 86400 ))

# Estimate daily cost
DAILY_COST=3.10  # $93/month ÷ 30 days

# Calculate total spent
TOTAL_SPENT=$(echo "$DAYS_ELAPSED * $DAILY_COST" | bc)

# Calculate remaining
REMAINING=$(echo "$BUDGET - $TOTAL_SPENT" | bc)

# Calculate days remaining at current rate
DAYS_REMAINING=$(echo "$REMAINING / $DAILY_COST" | bc)

echo "$(date): Days: $DAYS_ELAPSED | Spent: \$$TOTAL_SPENT | Remaining: \$$REMAINING | Days Left: $DAYS_REMAINING" >> $LOGFILE

# Alert if low
if (( $(echo "$REMAINING < 50" | bc -l) )); then
    echo "⚠️  WARNING: Less than \$50 remaining!" | mail -s "Vultr Budget Alert" your-email@example.com
fi
```

Run daily:

```bash
# Add to crontab
0 9 * * * /opt/sentiment-cascade/track-costs.sh
```

---

## 📞 Troubleshooting

### GPU Not Detected

```bash
# Check NVIDIA drivers
nvidia-smi

# If not working, reinstall
apt purge nvidia-*
apt install nvidia-driver-535
reboot
```

### Docker Container Not Using GPU

```bash
# Test GPU in container
docker run --rm --gpus all nvidia/cuda:12.1.0-base-ubuntu22.04 nvidia-smi

# If error, restart Docker
systemctl restart docker
```

### High Memory Usage

```bash
# Check memory
free -h

# Check top processes
htop

# Clear caches
sync && echo 3 > /proc/sys/vm/drop_caches
```

### Database Connection Issues

```bash
# Check PostgreSQL logs
docker logs sentiment-postgres

# Restart database
docker-compose restart postgres

# Connect manually
docker exec -it sentiment-postgres psql -U sentiment_user -d sentiment_db
```

---

## ✅ Final Checklist

Before going live:

- [ ] All services running (`docker-compose ps`)
- [ ] GPU detected (`nvidia-smi`)
- [ ] Database initialized with schema
- [ ] API keys configured in `.env`
- [ ] Cron jobs set up for data collection
- [ ] Monitoring scripts running
- [ ] Cost tracking enabled
- [ ] Backups configured
- [ ] Firewall enabled
- [ ] Frontend deployed on Vercel
- [ ] API Gateway accessible from Vercel
- [ ] Test data collection manually
- [ ] Test sentiment processing
- [ ] Test cascade detection
- [ ] Verify GPU is being used for FinBERT

---

**Implementation Guide Version:** 1.0
**Last Updated:** November 12, 2024
**Estimated Setup Time:** 4-6 hours
**Estimated Monthly Cost:** $93 (self-hosted DB) or $135 (managed DB)
**Remaining Budget:** $157 (self-hosted) or $115 (managed)

Ready to deploy! 🚀
