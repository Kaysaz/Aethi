# 🚀 DataStream — Real-Time Data Pipeline Dashboard

A production-ready, full-stack streaming dashboard that demonstrates a complete
data pipeline: **ingestion → processing → WebSocket streaming → React visualisation**.

Built with Python (asyncio), WebSockets, React 18, Recharts, Docker, and Terraform.

---

## 📐 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                       DATA PIPELINE                         │
│                                                             │
│  [Price APIs]──▶[Python Ingestion]──▶[asyncio Processing]  │
│                          │                                  │
│                  [WebSocket Server :8765]                   │
│                  [REST API Server  :8080]                   │
│                          │                                  │
│               [React Dashboard :3000]                       │
│          (live prices, charts, pipeline status)             │
└─────────────────────────────────────────────────────────────┘
```

**In production**, replace the mock price simulation in `backend/server.py` with:
- **Binance WebSocket** (`wss://stream.binance.com:9443/ws`)
- **CoinGecko REST API** (`https://api.coingecko.com/api/v3/simple/price`)
- **Apache Kafka / AWS Kinesis** for high-throughput ingestion

---

## 📁 Project Structure

```
pipeline-dashboard/
├── backend/
│   ├── server.py          # Python asyncio WebSocket + REST server
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── components/
│   │   │   ├── Header.jsx
│   │   │   ├── TickerTape.jsx
│   │   │   ├── CoinCard.jsx
│   │   │   ├── PriceChart.jsx
│   │   │   └── PipelineSidebar.jsx
│   │   ├── hooks/
│   │   │   ├── useLivePrices.js   # WebSocket hook
│   │   │   └── useHistory.js      # REST history hook
│   │   └── lib/format.js
│   ├── .env                       # Local dev env vars
│   ├── .env.production            # Production env vars
│   ├── Dockerfile
│   └── nginx.conf
├── infra/
│   └── main.tf                    # Terraform for AWS ECS
├── .github/workflows/deploy.yml   # CI/CD pipeline
├── docker-compose.yml
└── README.md
```

---

## ▶️ Option 1 — Run Locally (Development)

### Prerequisites
- Python 3.11+
- Node.js 20+

### 1. Start the backend

```bash
cd backend
pip install -r requirements.txt
python server.py
```

The backend starts:
- WebSocket on `ws://localhost:8765`
- REST API on `http://localhost:8080`

### 2. Start the frontend

```bash
cd frontend
npm install
npm run dev
```

Open **http://localhost:3000** — live prices stream immediately.

---

## 🐳 Option 2 — Docker Compose (Recommended for staging)

```bash
# From project root
docker compose up --build
```

| Service   | URL                        |
|-----------|----------------------------|
| Frontend  | http://localhost:3000      |
| REST API  | http://localhost:8080/api/coins |
| WebSocket | ws://localhost:8765        |
| Health    | http://localhost:8080/health |

To stop:
```bash
docker compose down
```

---

## ☁️ Option 3 — Deploy to a VPS / Cloud VM (easiest cloud path)

### Step 1 — Provision a server
- **DigitalOcean Droplet** ($6/mo, Ubuntu 24.04)
- **Hetzner Cloud CX11** (~€4/mo)
- **AWS EC2 t3.micro** (free tier eligible)
- **Google Cloud e2-micro** (always free tier)

### Step 2 — Install Docker on the server

```bash
ssh user@YOUR_SERVER_IP

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Install Docker Compose
sudo apt-get install -y docker-compose-plugin
```

### Step 3 — Copy project to server

```bash
# From your local machine:
scp -r pipeline-dashboard/ user@YOUR_SERVER_IP:/opt/pipeline-dashboard
```

Or use git:
```bash
# On the server:
git clone https://github.com/YOUR_USERNAME/pipeline-dashboard.git /opt/pipeline-dashboard
```

### Step 4 — Set production environment variables

```bash
cd /opt/pipeline-dashboard/frontend
# Edit .env.production with your server's domain/IP:
nano .env.production
# VITE_WS_URL=ws://YOUR_SERVER_IP:8765
# VITE_API_URL=http://YOUR_SERVER_IP:8080
```

### Step 5 — Launch

```bash
cd /opt/pipeline-dashboard
docker compose up -d --build
```

### Step 6 — (Optional) Add a domain + HTTPS with Caddy

```bash
sudo apt install -y caddy

# /etc/caddy/Caddyfile:
yourdomain.com {
    reverse_proxy localhost:3000
}

api.yourdomain.com {
    reverse_proxy localhost:8080
}
```

For WSS (secure WebSocket), update `.env.production`:
```
VITE_WS_URL=wss://api.yourdomain.com/ws
```

---

## 🏗️ Option 4 — AWS ECS Fargate (Terraform)

### Prerequisites
- AWS CLI configured (`aws configure`)
- Terraform installed (`brew install terraform`)
- Docker images pushed to ECR or GHCR

### Step 1 — Build and push images

```bash
# Authenticate to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Build and push backend
docker build -t pipeline-backend ./backend
docker tag pipeline-backend:latest \
  YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/pipeline-backend:latest
docker push YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/pipeline-backend:latest

# Build and push frontend
docker build -t pipeline-frontend ./frontend
docker tag pipeline-frontend:latest \
  YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/pipeline-frontend:latest
docker push YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/pipeline-frontend:latest
```

### Step 2 — Deploy with Terraform

```bash
cd infra
terraform init
terraform plan \
  -var="backend_image=YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/pipeline-backend:latest" \
  -var="frontend_image=YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/pipeline-frontend:latest"
terraform apply
```

Terraform provisions:
- ECS Fargate cluster
- Task definition (backend + frontend containers)
- Security groups (ports 3000, 8080, 8765)
- IAM roles
- CloudWatch log groups

---

## 🔄 Option 5 — CI/CD with GitHub Actions

### Setup secrets in GitHub repo settings:

| Secret            | Value                               |
|-------------------|-------------------------------------|
| `DEPLOY_HOST`     | Your server IP or hostname          |
| `DEPLOY_USER`     | SSH username (e.g., `ubuntu`)       |
| `DEPLOY_KEY`      | Private SSH key (contents of id_rsa)|
| `VITE_WS_URL`     | `wss://YOUR_DOMAIN/ws`              |
| `VITE_API_URL`    | `https://YOUR_DOMAIN`               |

Every push to `main` will:
1. Build Docker images
2. Push to GitHub Container Registry
3. SSH into your server and run `docker compose pull && docker compose up -d`

---

## 🔌 Connecting Real Data Sources

### Option A — Binance WebSocket (replace mock)

```python
# In backend/server.py, replace tick_prices() with:
import websockets, json

async def binance_feed():
    url = "wss://stream.binance.com:9443/ws/btcusdt@ticker/ethusdt@ticker"
    async with websockets.connect(url) as ws:
        async for msg in ws:
            data = json.loads(msg)
            symbol = data["s"].replace("USDT", "")
            if symbol in price_state:
                price_state[symbol]["price"] = float(data["c"])
                price_state[symbol]["change_24h"] = float(data["P"])
                price_state[symbol]["volume_24h"] = float(data["v"]) * float(data["c"])
```

### Option B — CoinGecko REST polling

```python
import aiohttp, asyncio

async def coingecko_poll():
    ids = "bitcoin,ethereum,solana,binancecoin,ripple,cardano"
    url = f"https://api.coingecko.com/api/v3/simple/price?ids={ids}&vs_currencies=usd&include_24hr_change=true"
    async with aiohttp.ClientSession() as session:
        while True:
            async with session.get(url) as r:
                data = await r.json()
                # map to price_state...
            await asyncio.sleep(10)  # free tier: max 30 req/min
```

### Option C — Apache Kafka (production scale)

```python
from aiokafka import AIOKafkaConsumer

async def kafka_consumer():
    consumer = AIOKafkaConsumer("crypto-prices", bootstrap_servers="kafka:9092")
    await consumer.start()
    async for msg in consumer:
        data = json.loads(msg.value)
        price_state[data["symbol"]]["price"] = data["price"]
```

---

## 📊 Adding a Data Warehouse

For persisting OHLCV data:

### TimescaleDB (PostgreSQL for time-series)

```bash
# docker-compose.yml — add:
  timescaledb:
    image: timescale/timescaledb:latest-pg15
    environment:
      POSTGRES_PASSWORD: secret
    ports: ["5432:5432"]
```

```python
# In server.py — save each tick:
import asyncpg

async def save_tick(symbol, price, ts):
    conn = await asyncpg.connect("postgresql://postgres:secret@timescaledb/prices")
    await conn.execute("""
        INSERT INTO ohlcv (time, symbol, price)
        VALUES ($1, $2, $3)
    """, ts, symbol, price)
```

---

## 📡 REST API Reference

| Method | Path                    | Description              |
|--------|-------------------------|--------------------------|
| GET    | `/health`               | Server health check      |
| GET    | `/api/coins`            | All coins snapshot       |
| GET    | `/api/coins/{SYMBOL}`   | OHLCV history for coin   |

### WebSocket Protocol

Connect to `ws://localhost:8765`. Receives JSON every second:

```json
{
  "type": "snapshot",
  "ts": "2024-01-15T12:34:56.789",
  "data": {
    "BTC": {
      "symbol": "BTC",
      "name": "Bitcoin",
      "price": 67234.50,
      "change_24h": 2.34,
      "volume_24h": 28500000000,
      "market_cap": 1320000000000
    }
  }
}
```

---

## 🛠️ Tech Stack Summary

| Layer         | Technology                        |
|---------------|-----------------------------------|
| Ingestion     | Python asyncio, aiohttp, websockets |
| Processing    | In-process (swap for Spark/Flink) |
| Streaming     | WebSocket (swap for Kafka)        |
| Storage       | In-memory (swap for TimescaleDB)  |
| Cloud         | Docker → ECS Fargate / Cloud Run  |
| Frontend      | React 18, Recharts, Vite          |
| CI/CD         | GitHub Actions                    |
| IaC           | Terraform (AWS)                   |
