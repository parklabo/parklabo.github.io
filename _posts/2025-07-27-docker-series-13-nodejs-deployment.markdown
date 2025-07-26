---
layout: post
title: "[Docker 입문 #13] Docker로 Node.js 애플리케이션 배포"
date: 2025-07-27 10:00:00 +0900
categories: [DevOps, Docker, Node.js]
tags: [docker, container, tutorial, series, deployment, nodejs, javascript, beginner]
mermaid: true
---

Node.js는 Docker와 매우 잘 어울리는 기술 스택입니다. 이번 편에서는 실제 Node.js 애플리케이션을 Docker로 배포하는 전 과정을 다루겠습니다.

## Node.js Docker 배포 전략 비교

| 배포 전략 | 설명 | 장점 | 단점 | 적용 시나리오 |
|-----------|------|------|------|---------------|
| **단일 컨테이너** | 앱 하나에 컨테이너 하나 | 간단함 | 확장성 제한 | 소규모 프로젝트 |
| **멀티 스테이지 빌드** | 빌드와 실행 분리 | 이미지 크기 최소화 | 빌드 복잡도 | 프로덕션 환경 |
| **마이크로서비스** | 기능별 컨테이너 분리 | 독립적 확장 | 관리 복잡도 | 대규모 시스템 |
| **서버리스 컨테이너** | 요청 시 실행 | 비용 효율적 | 콜드 스타트 | 간헐적 워크로드 |
| **쿠버네티스** | 오케스트레이션 | 자동화, 확장성 | 학습 곡선 | 엔터프라이즈 |

## 프로젝트 설정

### 1. Express 애플리케이션 생성

```javascript
// app.js
const express = require('express');
const mongoose = require('mongoose');
const redis = require('redis');
const promClient = require('prom-client');

const app = express();
const PORT = process.env.PORT || 3000;

// Prometheus 메트릭 설정
const register = new promClient.Registry();
promClient.collectDefaultMetrics({ register });

// 미들웨어
app.use(express.json());
app.use(express.static('public'));

// 헬스체크
app.get('/health', (req, res) => {
  res.json({ 
    status: 'healthy',
    uptime: process.uptime(),
    timestamp: new Date().toISOString()
  });
});

// 메트릭 엔드포인트
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// MongoDB 연결
const mongoUri = process.env.MONGO_URI || 'mongodb://mongo:27017/myapp';
mongoose.connect(mongoUri, {
  useNewUrlParser: true,
  useUnifiedTopology: true
});

// Redis 연결
const redisClient = redis.createClient({
  url: process.env.REDIS_URL || 'redis://redis:6379'
});
redisClient.on('error', err => console.log('Redis Error', err));
redisClient.connect();

// 모델 정의
const ItemSchema = new mongoose.Schema({
  name: String,
  description: String,
  createdAt: { type: Date, default: Date.now }
});
const Item = mongoose.model('Item', ItemSchema);

// API 라우트
app.get('/api/items', async (req, res) => {
  try {
    // Redis 캐시 확인
    const cached = await redisClient.get('items');
    if (cached) {
      return res.json(JSON.parse(cached));
    }

    // DB에서 조회
    const items = await Item.find().sort('-createdAt');
    
    // 캐시 저장 (1분)
    await redisClient.setEx('items', 60, JSON.stringify(items));
    
    res.json(items);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/api/items', async (req, res) => {
  try {
    const item = new Item(req.body);
    await item.save();
    
    // 캐시 무효화
    await redisClient.del('items');
    
    res.status(201).json(item);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// 에러 핸들링
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong!' });
});

// 서버 시작
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully');
  await mongoose.connection.close();
  await redisClient.quit();
  process.exit(0);
});
```

### 2. package.json

```json
{
  "name": "docker-node-app",
  "version": "1.0.0",
  "description": "Node.js app with Docker",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js",
    "test": "jest",
    "lint": "eslint ."
  },
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^7.0.0",
    "redis": "^4.5.0",
    "prom-client": "^14.2.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.20",
    "jest": "^29.0.0",
    "supertest": "^6.3.0",
    "eslint": "^8.0.0"
  },
  "engines": {
    "node": ">=16.0.0"
  }
}
```

## 개발 환경 Dockerfile

### Node.js 버전별 Docker 이미지 선택 가이드

| Node.js 버전 | 이미지 태그 | 크기 | 용도 | 보안 업데이트 |
|-------------|------------|------|------|---------------|
| **18-alpine** | node:18-alpine | ~50MB | 프로덕션 추천 | 2025년 4월까지 |
| **20-alpine** | node:20-alpine | ~52MB | 최신 LTS | 2026년 4월까지 |
| **18-slim** | node:18-slim | ~180MB | 일부 도구 필요 시 | 활발함 |
| **18** | node:18 | ~950MB | 개발 환경 | 활발함 |
| **18-bullseye** | node:18-bullseye | ~950MB | Debian 기반 | 활발함 |

### Dockerfile.dev

```dockerfile
FROM node:18-alpine

# 개발 도구 설치
RUN apk add --no-cache python3 make g++

WORKDIR /app

# package.json만 먼저 복사 (캐시 활용)
COPY package*.json ./

# 개발 의존성 포함 설치
RUN npm install

# nodemon 글로벌 설치
RUN npm install -g nodemon

# 소스 코드 복사
COPY . .

# 개발 서버 포트
EXPOSE 3000

# nodemon으로 실행
CMD ["nodemon", "app.js"]
```

## 프로덕션 Dockerfile

### Dockerfile

```dockerfile
# 빌드 스테이지
FROM node:18-alpine AS builder

WORKDIR /app

# 의존성 파일 복사
COPY package*.json ./

# 프로덕션 의존성만 설치
RUN npm ci --only=production

# 실행 스테이지
FROM node:18-alpine

# 보안을 위한 사용자 생성
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# tini 설치 (적절한 시그널 처리)
RUN apk add --no-cache tini

WORKDIR /app

# 빌드 스테이지에서 node_modules 복사
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules

# 애플리케이션 코드 복사
COPY --chown=nodejs:nodejs . .

# 사용자 전환
USER nodejs

# 포트 노출
EXPOSE 3000

# 헬스체크
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# tini를 통한 실행
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "app.js"]
```

### healthcheck.js

```javascript
const http = require('http');

const options = {
  host: 'localhost',
  port: process.env.PORT || 3000,
  path: '/health',
  timeout: 2000
};

const request = http.request(options, (res) => {
  console.log(`STATUS: ${res.statusCode}`);
  if (res.statusCode == 200) {
    process.exit(0);
  } else {
    process.exit(1);
  }
});

request.on('error', (err) => {
  console.error('ERROR:', err);
  process.exit(1);
});

request.end();
```

## Docker Compose 설정

### docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build: 
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - MONGO_URI=mongodb://mongo:27017/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      mongo:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

  mongo:
    image: mongo:6
    volumes:
      - mongo_data:/data/db
    environment:
      - MONGO_INITDB_DATABASE=myapp
    networks:
      - app-network
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  mongo_data:
  redis_data:
```

### docker-compose.dev.yml

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    command: nodemon app.js
```

## Nginx 설정

### nginx.conf

```nginx
events {
    worker_connections 1024;
}

http {
    upstream app {
        server app:3000;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://app;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
        }

        # 정적 파일 캐싱
        location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
            proxy_pass http://app;
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

## 환경별 설정

### .env.development

```env
NODE_ENV=development
PORT=3000
MONGO_URI=mongodb://localhost:27017/myapp-dev
REDIS_URL=redis://localhost:6379
LOG_LEVEL=debug
```

### .env.production

```env
NODE_ENV=production
PORT=3000
MONGO_URI=mongodb://mongo:27017/myapp
REDIS_URL=redis://redis:6379
LOG_LEVEL=info
```

## CI/CD 파이프라인

### .github/workflows/docker.yml

```yaml
name: Docker Build and Deploy

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run tests
      run: npm test
    
    - name: Run linter
      run: npm run lint

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=sha,prefix={{branch}}-
    
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

## 모니터링 설정

### docker-compose.monitoring.yml

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - app-network

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - app-network
```

### prometheus.yml

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node-app'
    static_configs:
      - targets: ['app:3000']
    metrics_path: '/metrics'
```

## 성능 최적화

### 1. 멀티 스테이지 빌드 최적화

```dockerfile
# 의존성 캐싱을 위한 별도 스테이지
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# 개발 의존성 포함 빌드
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 최종 이미지
FROM node:18-alpine
RUN apk add --no-cache tini
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
EXPOSE 3000
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/app.js"]
```

### 2. 캐싱 전략

```javascript
// 캐싱 미들웨어
const cache = (duration) => {
  return (req, res, next) => {
    const key = '__express__' + req.originalUrl || req.url;
    redisClient.get(key).then(reply => {
      if (reply) {
        res.setHeader('X-Cache', 'HIT');
        res.json(JSON.parse(reply));
      } else {
        res.setHeader('X-Cache', 'MISS');
        res.sendResponse = res.json;
        res.json = (body) => {
          redisClient.setEx(key, duration, JSON.stringify(body));
          res.sendResponse(body);
        };
        next();
      }
    });
  };
};

// 사용 예
app.get('/api/items', cache(300), async (req, res) => {
  // ...
});
```

## 배포 전략

### 배포 아키텍처 다이어그램

```mermaid
graph TB
    subgraph "Development"
        A[로컬 개발] --> B[Docker Build]
        B --> C[로컬 테스트]
    end
    
    subgraph "CI/CD Pipeline"
        C --> D[GitHub Actions]
        D --> E[자동 테스트]
        E --> F[이미지 빌드]
        F --> G[Container Registry]
    end
    
    subgraph "Production"
        G --> H{배포 전략}
        H -->|Rolling| I[순차 업데이트]
        H -->|Blue-Green| J[전환 배포]
        H -->|Canary| K[점진적 배포]
        
        I --> L[Load Balancer]
        J --> L
        K --> L
        L --> M[사용자]
    end
    
    subgraph "Monitoring"
        I --> N[Logs]
        J --> N
        K --> N
        N --> O[Prometheus/Grafana]
    end
```

### 1. 롤링 업데이트

```bash
# 새 버전 빌드
docker compose build app

# 무중단 배포
docker compose up -d --no-deps --scale app=2 app
sleep 10
docker compose up -d --no-deps app
```

### 2. 블루-그린 배포

```yaml
# docker-compose.blue-green.yml
version: '3.8'

services:
  app-blue:
    build: .
    networks:
      - app-network
    environment:
      - VERSION=blue

  app-green:
    build: .
    networks:
      - app-network
    environment:
      - VERSION=green

  nginx:
    image: nginx
    volumes:
      - ./nginx-blue-green.conf:/etc/nginx/nginx.conf
    ports:
      - "80:80"
    networks:
      - app-network
```

## 트러블슈팅

### 일반적인 문제 해결

```bash
# 로그 확인
docker compose logs -f app

# 컨테이너 내부 디버깅
docker compose exec app sh
> node --inspect app.js

# 메모리 사용량 확인
docker stats

# 네트워크 문제 디버깅
docker compose exec app ping mongo
docker compose exec app nc -zv redis 6379
```

## 2025년 Node.js Docker 최적화 팁

### 이미지 크기 최적화 비교

| 최적화 기법 | 적용 전 | 적용 후 | 감소율 | 구현 난이도 |
|------------|---------|---------|--------|------------|
| **Alpine 사용** | 950MB | 180MB | 81% | 쉬움 |
| **멀티스테이지** | 950MB | 50MB | 95% | 중간 |
| **Dependencies 정리** | 180MB | 120MB | 33% | 쉬움 |
| **.dockerignore** | 120MB | 100MB | 17% | 쉬움 |
| **레이어 최적화** | 100MB | 85MB | 15% | 어려움 |

### 성능 최적화 체크리스트

| 항목 | 설명 | 효과 | 우선순위 |
|------|------|------|----------|
| **npm ci 사용** | npm install 대신 사용 | 빌드 시간 30% 단축 | 높음 |
| **레이어 캐싱** | package.json 먼저 복사 | 재빌드 시간 단축 | 높음 |
| **Production 모드** | NODE_ENV=production | 메모리 사용량 감소 | 높음 |
| **Health Check** | 컨테이너 상태 모니터링 | 안정성 향상 | 중간 |
| **Graceful Shutdown** | 안전한 종료 처리 | 데이터 무결성 | 중간 |

## 마무리

Node.js 애플리케이션을 Docker로 배포하면 일관된 환경에서 안정적으로 운영할 수 있습니다. 개발부터 프로덕션까지 같은 환경을 유지하고, 확장과 배포가 쉬워집니다. 다음 편에서는 Python 애플리케이션의 Docker 배포를 다루겠습니다.

## 다음 편 예고

- Python 애플리케이션 컨테이너화
- Django/Flask 배포
- 가상 환경과 의존성 관리
- Python 특화 최적화

Node.js 다음은 Python입니다! 🐍