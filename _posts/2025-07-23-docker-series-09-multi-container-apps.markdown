---
layout: post
title: "[Docker 입문 #9] 멀티 컨테이너 애플리케이션"
date: 2025-07-23 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, tutorial, series, multi-container, microservices, docker-compose, beginner]
mermaid: true
---

이번 편에서는 Docker Compose를 활용해 실제 멀티 컨테이너 애플리케이션을 구축해봅니다. 프론트엔드, 백엔드, 데이터베이스가 연동되는 실전 프로젝트를 만들어보겠습니다.

## 프로젝트 개요 (2025년 업데이트)

### 최신 기술 스택 (2025년 7월)

```mermaid
graph TD
    A[프론트엔드] --> B[React 18 + TypeScript]
    C[백엔드] --> D[Node.js 22 LTS + Express]
    E[데이터베이스] --> F[PostgreSQL 16]
    G[캐시] --> H[Redis 7]
    I[프록시] --> J[Nginx Alpine]
    K[모니터링] --> L[Prometheus + Grafana]
    M[GPU 지원] --> N[AI/ML 워크로드]
    
    B --> I
    I --> D
    D --> F
    D --> H
    D --> L
    D --> N
```

### Docker Compose v2.35+ 핵심 기능

| 기능 | 설명 | 예시 |
|------|------|------|
| Watch 모드 | 파일 변경 자동 감지 | `docker compose watch` |
| Dry-Run | 실행 전 미리보기 | `docker compose --dry-run up` |
| GPU 지원 | AI/ML 컨테이너 | `gpus: all` |
| 멀티 프로필 | 선택적 서비스 실행 | `--profile dev --profile debug` |
| Lock Images | 이미지 다이제스트 고정 | `--lock-image-digests` |

할 일 관리(Todo) 애플리케이션을 만들어봅시다:
- **프론트엔드**: React
- **백엔드**: Node.js + Express
- **데이터베이스**: PostgreSQL
- **캐시**: Redis
- **리버스 프록시**: Nginx

## 프로젝트 구조

```
todo-app/
├── docker-compose.yml
├── .env
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── nginx/
│   └── nginx.conf
└── database/
    └── init.sql
```

## 1. 백엔드 구현

### backend/package.json

```json
{
  "name": "todo-backend",
  "version": "1.0.0",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js"
  },
  "dependencies": {
    "express": "^4.18.0",
    "pg": "^8.7.0",
    "redis": "^4.0.0",
    "cors": "^2.8.5",
    "dotenv": "^16.0.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.0"
  }
}
```

### backend/src/index.js

```javascript
const express = require('express');
const cors = require('cors');
const { Pool } = require('pg');
const redis = require('redis');

const app = express();
app.use(cors());
app.use(express.json());

// PostgreSQL 연결
const pool = new Pool({
  host: process.env.DB_HOST || 'database',
  user: process.env.DB_USER || 'todouser',
  password: process.env.DB_PASS || 'todopass',
  database: process.env.DB_NAME || 'tododb',
  port: 5432,
});

// Redis 연결
const redisClient = redis.createClient({
  url: `redis://${process.env.REDIS_HOST || 'cache'}:6379`
});

redisClient.on('error', err => console.log('Redis Error', err));
redisClient.connect();

// 헬스체크
app.get('/api/health', (req, res) => {
  res.json({ status: 'OK', timestamp: new Date() });
});

// 모든 할 일 조회
app.get('/api/todos', async (req, res) => {
  try {
    // Redis 캐시 확인
    const cached = await redisClient.get('todos');
    if (cached) {
      return res.json(JSON.parse(cached));
    }

    // DB에서 조회
    const result = await pool.query('SELECT * FROM todos ORDER BY id DESC');
    
    // Redis에 캐시
    await redisClient.setEx('todos', 60, JSON.stringify(result.rows));
    
    res.json(result.rows);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Server error' });
  }
});

// 할 일 추가
app.post('/api/todos', async (req, res) => {
  try {
    const { title } = req.body;
    const result = await pool.query(
      'INSERT INTO todos (title, completed) VALUES ($1, $2) RETURNING *',
      [title, false]
    );
    
    // 캐시 무효화
    await redisClient.del('todos');
    
    res.status(201).json(result.rows[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Server error' });
  }
});

// 할 일 완료 토글
app.patch('/api/todos/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const result = await pool.query(
      'UPDATE todos SET completed = NOT completed WHERE id = $1 RETURNING *',
      [id]
    );
    
    // 캐시 무효화
    await redisClient.del('todos');
    
    res.json(result.rows[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Server error' });
  }
});

// 할 일 삭제
app.delete('/api/todos/:id', async (req, res) => {
  try {
    const { id } = req.params;
    await pool.query('DELETE FROM todos WHERE id = $1', [id]);
    
    // 캐시 무효화
    await redisClient.del('todos');
    
    res.status(204).send();
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Server error' });
  }
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### backend/Dockerfile

```dockerfile
FROM node:22-alpine AS base

# non-root user 생성 (2025년 보안 베스트 프랙티스)
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

WORKDIR /app

# 의존성 설치
COPY --chown=nodejs:nodejs package*.json ./
RUN npm ci --only=production

# 애플리케이션 복사
COPY --chown=nodejs:nodejs . .

USER nodejs

EXPOSE 5000

CMD ["npm", "start"]
```

## 2. 프론트엔드 구현

### frontend/package.json

```json
{
  "name": "todo-frontend",
  "version": "1.0.0",
  "dependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0",
    "axios": "^0.27.0"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build"
  },
  "devDependencies": {
    "react-scripts": "5.0.0"
  },
  "proxy": "http://backend:5000"
}
```

### frontend/src/App.js

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './App.css';

function App() {
  const [todos, setTodos] = useState([]);
  const [newTodo, setNewTodo] = useState('');
  const [loading, setLoading] = useState(true);

  const API_URL = process.env.REACT_APP_API_URL || '/api';

  useEffect(() => {
    fetchTodos();
  }, []);

  const fetchTodos = async () => {
    try {
      const response = await axios.get(`${API_URL}/todos`);
      setTodos(response.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching todos:', error);
      setLoading(false);
    }
  };

  const addTodo = async (e) => {
    e.preventDefault();
    if (!newTodo.trim()) return;

    try {
      const response = await axios.post(`${API_URL}/todos`, {
        title: newTodo
      });
      setTodos([response.data, ...todos]);
      setNewTodo('');
    } catch (error) {
      console.error('Error adding todo:', error);
    }
  };

  const toggleTodo = async (id) => {
    try {
      const response = await axios.patch(`${API_URL}/todos/${id}`);
      setTodos(todos.map(todo => 
        todo.id === id ? response.data : todo
      ));
    } catch (error) {
      console.error('Error toggling todo:', error);
    }
  };

  const deleteTodo = async (id) => {
    try {
      await axios.delete(`${API_URL}/todos/${id}`);
      setTodos(todos.filter(todo => todo.id !== id));
    } catch (error) {
      console.error('Error deleting todo:', error);
    }
  };

  if (loading) return <div className="loading">Loading...</div>;

  return (
    <div className="App">
      <h1>Docker Todo App</h1>
      
      <form onSubmit={addTodo} className="add-form">
        <input
          type="text"
          value={newTodo}
          onChange={(e) => setNewTodo(e.target.value)}
          placeholder="할 일을 입력하세요..."
        />
        <button type="submit">추가</button>
      </form>

      <ul className="todo-list">
        {todos.map(todo => (
          <li key={todo.id} className={todo.completed ? 'completed' : ''}>
            <span onClick={() => toggleTodo(todo.id)}>
              {todo.title}
            </span>
            <button onClick={() => deleteTodo(todo.id)}>삭제</button>
          </li>
        ))}
      </ul>
    </div>
  );
}

export default App;
```

### frontend/Dockerfile

```dockerfile
# 빌드 스테이지
FROM node:22-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 실행 스테이지
FROM nginx:alpine

# nginx non-root user (2025년 보안)
RUN touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid && \
    chown -R nginx:nginx /var/cache/nginx

COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

USER nginx

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## 3. 데이터베이스 초기화

### database/init.sql

```sql
CREATE TABLE IF NOT EXISTS todos (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    completed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 샘플 데이터
INSERT INTO todos (title, completed) VALUES 
    ('Docker 공부하기', true),
    ('Docker Compose 마스터하기', false),
    ('멀티 컨테이너 앱 만들기', false);
```

## 4. Nginx 설정

### nginx/nginx.conf

```nginx
upstream backend {
    server backend:5000;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://frontend:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /api {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # WebSocket 지원 (실시간 기능용)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## 5. Docker Compose 설정 (2025년 최신)

### docker-compose.yml

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - frontend
      - backend
    networks:
      - todo-network

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    environment:
      - REACT_APP_API_URL=/api
    networks:
      - todo-network

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      - NODE_ENV=production
      - DB_HOST=database
      - DB_USER=todouser
      - DB_PASS=todopass
      - DB_NAME=tododb
      - REDIS_HOST=cache
    depends_on:
      database:
        condition: service_healthy
      cache:
        condition: service_healthy
    networks:
      - todo-network
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:5000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    profiles: ["app"]  # 2025년 최신: 프로필 사용

  database:
    image: postgres:13-alpine
    environment:
      - POSTGRES_USER=todouser
      - POSTGRES_PASSWORD=todopass
      - POSTGRES_DB=tododb
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - todo-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U todouser"]
      interval: 10s
      timeout: 5s
      retries: 5

  cache:
    image: redis:6-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - todo-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  todo-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
```

### .env 파일

```env
# Database
DB_USER=todouser
DB_PASS=todopass
DB_NAME=tododb

# Redis
REDIS_HOST=cache

# Node
NODE_ENV=production
```

## 6. 개발 환경 설정 (2025년 강화)

### Watch 모드 활용 (2025년 7월 최신)

```yaml
# docker-compose.yml에 watch 설정 추가
services:
  backend:
    develop:
      watch:
        - action: rebuild
          path: ./backend/package.json
        - action: sync
          path: ./backend/src
          target: /app/src
          include:
            - "**/*.js"
            - "**/*.json"
        - action: sync+restart
          path: ./backend/.env
          target: /app/.env

  frontend:
    develop:
      watch:
        - action: rebuild
          path: ./frontend/package.json
        - action: sync
          path: ./frontend/src
          target: /app/src
          include:
            - "**/*.js"
            - "**/*.jsx"
            - "**/*.css"
          ignore:
            - node_modules/
            - "**/*.test.js"
```

### Watch 모드 고급 옵션

```mermaid
graph LR
    A[Watch 시작] --> B{--no-up?}
    B -->|아니오| C[서비스 시작]
    B -->|예| D[서비스 시작 안함]
    C --> E[파일 모니터링]
    D --> E
    E --> F{변경 감지}
    F --> G[Action 실행]
    G --> H[서비스 업데이트]
```

### Watch 모드 실행 (2025년 최신 옵션)

```bash
# 파일 변경 감지 모드로 실행
docker compose watch

# Dry-run으로 미리 확인
docker compose watch --dry-run

# 서비스 시작 없이 watch만
docker compose watch --no-up

# 빌드 출력 숨기기
docker compose watch --quiet

# dangling 이미지 자동 정리
docker compose watch --prune
```

### docker-compose.dev.yml

```yaml
version: '3.8'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    volumes:
      - ./frontend/src:/app/src
      - ./frontend/public:/app/public
    environment:
      - CHOKIDAR_USEPOLLING=true
    command: npm start

  backend:
    volumes:
      - ./backend/src:/app/src
    environment:
      - NODE_ENV=development
    command: npm run dev
```

### 개발 환경 실행 (2025년 7월 고급 기능)

```bash
# 개발 환경
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# 프로덕션 환경
docker compose up -d

# 멀티 프로필 사용
docker compose --profile app --profile monitoring --profile debug up -d

# 드라이 런으로 미리 확인
docker compose --dry-run up --build

# 특정 서비스만 attach
docker compose up --attach backend --no-attach database

# 병렬 실행 제어
docker compose --parallel 1 pull

# 이미지 다이제스트 고정
docker compose config --lock-image-digests > docker-compose.locked.yml
```

## 실행 및 테스트

### 1. 애플리케이션 시작

```bash
# 이미지 빌드 및 실행
docker compose up -d --build

# 로그 확인
docker compose logs -f

# 서비스 상태 확인
docker compose ps
```

### 2. 헬스체크

```bash
# 백엔드 헬스체크
curl http://localhost/api/health

# 데이터베이스 연결 확인
docker compose exec database psql -U todouser -d tododb -c "SELECT * FROM todos;"

# Redis 확인
docker compose exec cache redis-cli ping
```

### 3. 스케일링

```bash
# 백엔드 서비스 스케일링
docker compose up -d --scale backend=3

# 로드 밸런싱 확인
for i in {1..10}; do curl http://localhost/api/health; done
```

## 모니터링 추가 (2025년 7월 확장)

### GPU 지원 AI/ML 컨테이너 (2025년 최신)

```yaml
# docker-compose.gpu.yml
version: '3.8'

services:
  ml-service:
    image: tensorflow/tensorflow:latest-gpu
    # 모든 GPU 사용
    gpus: all
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    profiles: ["ml"]
    
  inference-service:
    image: pytorch/pytorch:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 2  # 2개 GPU만 사용
              capabilities: [gpu]
    profiles: ["ml"]
    
  specific-gpu:
    image: custom-ml:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0', '3']  # 특정 GPU ID
              capabilities: [gpu]
    profiles: ["ml"]
```

### OpenTelemetry 통합 (2025년 최신)

```yaml
# docker-compose.otel.yml
version: '3.8'

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yml"]
    volumes:
      - ./otel/otel-collector-config.yml:/etc/otel-collector-config.yml
    ports:
      - "4317:4317"  # OTLP gRPC
      - "4318:4318"  # OTLP HTTP
    networks:
      - todo-network
    profiles: ["monitoring"]

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "14250:14250"  # gRPC
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    networks:
      - todo-network
    profiles: ["monitoring"]
```

### docker-compose.monitoring.yml

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - todo-network

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - todo-network

volumes:
  prometheus_data:
  grafana_data:
```

## 트러블슈팅

### 일반적인 문제 해결

```bash
# 컨테이너 재시작
docker compose restart backend

# 로그 확인
docker compose logs backend --tail=50

# 컨테이너 내부 접속
docker compose exec backend sh

# 네트워크 디버깅
docker compose exec backend ping database
docker compose exec backend nc -zv cache 6379

# 볼륨 정리
docker compose down -v
```

## 베스트 프랙티스 (2025년 업데이트)

### 보안 강화 (2025년 7월 베스트 프랙티스)

| 항목 | 기존 방식 | 2025년 권장 |
|------|-----------|-------------|
| 비밀 관리 | 환경 변수 | Docker Secrets + 암호화 |
| 이미지 스캔 | 수동 | 자동 취약점 스캔 + SBOM |
| 네트워크 | 기본 bridge | 암호화된 overlay |
| 사용자 권한 | root | non-root + read-only FS |
| 빌드 | 단일 스테이지 | 멀티 스테이지 + 캠시 |
| 런타임 | 기본 설정 | 보안 프로필 적용 |

```yaml
# 보안 강화 예시
services:
  backend:
    image: node:20-alpine
    user: "1000:1000"  # non-root 사용자
    secrets:
      - db_password
      - jwt_secret
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE

secrets:
  db_password:
    file: ./secrets/db_password.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt
```

### 성능 최적화 (2025년 최신)

```yaml
# 빌드 성능 향상
services:
  backend:
    build:
      context: .
      args:
        BUILDKIT_INLINE_CACHE: 1
      cache_from:
        - type=registry,ref=myapp/backend:buildcache
      cache_to:
        - type=registry,ref=myapp/backend:buildcache,mode=max
    x-build:
      memory: 2g  # 메모리 제한
      ssh: default  # SSH 인증
```

1. **환경 변수 관리**: `.env` 파일 + 환경별 암호화
2. **헬스체크**: 각 서비스의 상태 모니터링 + --wait 옵션
3. **로깅**: 중앙 집중식 로그 관리 + OpenTelemetry
4. **백업**: 데이터베이스 정기 백업 + 볼륨 스냅샷
5. **보안**: 네트워크 격리, 최소 권한 원칙 + 보안 스캐닝

## 마무리

2025년 7월 현재 Docker Compose v2.35+는 더욱 강력해졌습니다:

### 핵심 기능 요약

```mermaid
graph TD
    A[Docker Compose 2025] --> B[Watch Mode]
    A --> C[GPU Support]
    A --> D[Dry-Run]
    A --> E[Multi-Profile]
    A --> F[Lock Digests]
    
    B --> G[sync+restart]
    C --> H[AI/ML Workloads]
    D --> I[Safe Deployment]
    E --> J[Environment Control]
    F --> K[Reproducible Builds]
```

- **Watch 모드**: 파일 변경 자동 감지 및 재시작 (sync+restart)
- **GPU 지원**: AI/ML 워크로드를 위한 네이티브 GPU 접근
- **드라이 런**: 모든 명령어에서 실행 전 미리보기
- **헬스체크 의존성**: 서비스 시작 순서 보장 + --wait
- **이미지 다이제스트 고정**: 재현 가능한 빌드
- **플러그인 시스템**: Compose Bridge (K8s/Helm 변환)

이번 편에서는 실제 프로덕션에서 사용할 수 있는 멀티 컨테이너 애플리케이션을 구축했습니다. Docker Compose를 사용하면 복잡한 애플리케이션도 쉽게 관리하고 배포할 수 있습니다. 다음 편에서는 Docker Hub를 활용한 이미지 관리에 대해 알아보겠습니다.

## 다음 편 예고

- Docker Hub 계정 생성 및 설정
- 이미지 푸시와 풀
- 자동 빌드 설정
- 프라이빗 레지스트리 구축

이제 만든 애플리케이션을 전 세계와 공유해봅시다! 🌍