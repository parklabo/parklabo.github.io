---
layout: post
title: "[Docker 입문 #5] Dockerfile 작성 기초"
date: 2025-07-19 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, tutorial, series, dockerfile, docker-build, image-creation, beginner]
mermaid: true
---

지금까지는 다른 사람이 만든 이미지를 사용했습니다. 이제 Dockerfile을 작성해서 나만의 이미지를 만들어봅시다. 2025년 현재 Docker BuildKit의 발전으로 더욱 강력하고 안전한 이미지를 만들 수 있게 되었습니다.

## Dockerfile과 BuildKit (2025)

Dockerfile은 Docker 이미지를 만들기 위한 **설정 파일**입니다. Docker Engine 23.0부터 기본으로 사용되는 BuildKit은 병렬 빌드, 캐시 최적화, 보안 기능을 제공합니다.

### 2025년 Dockerfile의 주요 특징

```mermaid
graph TD
    A[Dockerfile 2025] --> B[BuildKit 기본 활성화]
    A --> C[멀티 플랫폼 빌드]
    A --> D[SBOM 자동 생성]
    A --> E[보안 강화]
    
    B --> B1[병렬 빌드]
    B --> B2[고급 캐싱]
    
    C --> C1[ARM/AMD64 동시]
    C --> C2[크로스 컴파일]
    
    D --> D1[의존성 추적]
    D --> D2[취약점 스캔]
    
    E --> E1[시크릿 마운트]
    E --> E2[Attestation]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style D fill:#9f9,stroke:#333,stroke-width:2px
    style E fill:#99f,stroke:#333,stroke-width:2px
```

### Dockerfile의 장점

| 장점 | 2020년 | 2025년 | 개선사항 |
|------|--------|--------|----------|
| **빌드 속도** | 순차적 | 병렬 처리 | 3-5배 향상 |
| **보안** | 기본적 | SBOM, Attestation | 공급망 보안 |
| **플랫폼** | 단일 | 멀티 플랫폼 | ARM/x86 동시 |
| **캐싱** | 레이어 기반 | 지능형 캐싱 | 빌드 시간 70% 감소 |

## 첫 번째 Dockerfile 작성하기

간단한 웹 서버 이미지를 만들어봅시다:

```dockerfile
# 베이스 이미지 지정
FROM nginx:alpine

# 작성자 정보
LABEL maintainer="your-email@example.com"

# 파일 복사
COPY index.html /usr/share/nginx/html/

# 포트 노출
EXPOSE 80

# 컨테이너 실행 명령
CMD ["nginx", "-g", "daemon off;"]
```

HTML 파일도 만들어줍니다:

```html
<!DOCTYPE html>
<html>
<head>
    <title>My Docker App</title>
</head>
<body>
    <h1>Hello from Dockerfile!</h1>
    <p>이 페이지는 Dockerfile로 만든 이미지에서 제공됩니다.</p>
</body>
</html>
```

## 이미지 빌드하기 (BuildKit 활용)

### 기본 빌드 명령어

```bash
# BuildKit을 사용한 이미지 빌드 (기본값)
docker build -t my-web:v1 .

# 빌드 과정 확인
docker build --no-cache -t my-web:v1 .

# 멀티 플랫폼 빌드 (ARM + x86)
docker buildx build --platform linux/amd64,linux/arm64 -t my-web:v1 .

# SBOM 생성과 함께 빌드
docker buildx build --sbom=true -t my-web:v1 --push .

# 보안 증명서(Attestation)와 함께 빌드
docker buildx build \
  --attest type=sbom \
  --attest type=provenance \
  -t my-web:v1 --push .
```

### 빌드 캐시 최적화

```bash
# 외부 캐시 사용
docker buildx build \
  --cache-from type=registry,ref=myregistry/myapp:cache \
  --cache-to type=registry,ref=myregistry/myapp:cache,mode=max \
  -t my-web:v1 .
```

## 주요 Dockerfile 명령어

### FROM - 베이스 이미지 지정

```dockerfile
# 공식 이미지 사용
FROM ubuntu:22.04

# 특정 플랫폼 지정
FROM --platform=linux/amd64 node:16

# 스크래치(빈 이미지)부터 시작
FROM scratch
```

### RUN - 명령어 실행

```dockerfile
# Shell 형식
RUN apt-get update && apt-get install -y python3

# Exec 형식 (권장)
RUN ["apt-get", "update"]
RUN ["apt-get", "install", "-y", "python3"]

# 여러 명령어 연결 (레이어 최소화)
RUN apt-get update && \
    apt-get install -y \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*
```

### COPY & ADD - 파일 복사

```dockerfile
# COPY: 로컬 파일을 이미지로 복사
COPY app.py /app/
COPY . /app/

# ADD: COPY + 추가 기능 (압축 해제, URL 다운로드)
ADD app.tar.gz /app/
ADD https://example.com/file.txt /app/
```

**참고**: 단순 파일 복사는 COPY 사용을 권장합니다.

### WORKDIR - 작업 디렉토리 설정

```dockerfile
# 작업 디렉토리 설정
WORKDIR /app

# 이후 명령어는 /app에서 실행됨
COPY . .
RUN npm install
```

### ENV - 환경 변수 설정

```dockerfile
# 환경 변수 설정
ENV NODE_ENV=production
ENV PORT=3000

# 여러 개 한번에 설정
ENV NODE_ENV=production \
    PORT=3000 \
    DB_HOST=localhost
```

### EXPOSE - 포트 노출

```dockerfile
# 포트 문서화 (실제 바인딩은 docker run -p로)
EXPOSE 80
EXPOSE 443
EXPOSE 3000/tcp
EXPOSE 3000/udp
```

### CMD vs ENTRYPOINT

```dockerfile
# CMD: 기본 실행 명령 (docker run으로 덮어쓸 수 있음)
CMD ["python", "app.py"]

# ENTRYPOINT: 고정 실행 명령
ENTRYPOINT ["python"]
CMD ["app.py"]  # ENTRYPOINT의 인자로 전달됨
```

## 실습: Node.js 애플리케이션 이미지 만들기

### 1. Node.js 앱 생성

```javascript
// app.js
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => {
    res.json({ 
        message: 'Hello from Docker!',
        timestamp: new Date().toISOString()
    });
});

app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
```

```json
// package.json
{
  "name": "docker-node-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

### 2. Dockerfile 작성

```dockerfile
# 베이스 이미지
FROM node:16-alpine

# 작업 디렉토리 생성
WORKDIR /usr/src/app

# 의존성 파일 복사 (캐시 활용)
COPY package*.json ./

# 의존성 설치
RUN npm ci --only=production

# 애플리케이션 소스 복사
COPY . .

# 포트 노출
EXPOSE 3000

# 앱 실행
CMD ["node", "app.js"]
```

### 3. .dockerignore 파일

```
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.DS_Store
```

### 4. 빌드 및 실행

```bash
# 이미지 빌드
docker build -t node-app:v1 .

# 컨테이너 실행
docker run -d -p 3000:3000 --name my-node-app node-app:v1

# 확인
curl http://localhost:3000
```

## 멀티 스테이지 빌드 (2025 고급 패턴)

### 기본 멀티 스테이지 패턴

```dockerfile
# 빌드 스테이지
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 실행 스테이지
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
EXPOSE 3000
CMD ["node", "dist/app.js"]
```

### 고급 멀티 스테이지 패턴 (Java Spring Boot)

```dockerfile
# syntax=docker/dockerfile:1

# 의존성 캐싱 스테이지
FROM eclipse-temurin:17-jdk AS deps
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline

# 빌드 스테이지
FROM deps AS builder
COPY src ./src
RUN mvn package -DskipTests

# 실행 스테이지 (Distroless)
FROM gcr.io/distroless/java17-debian12
COPY --from=builder /build/target/app.jar /app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### 멀티 플랫폼 최적화 예시

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.21 AS builder
ARG TARGETOS TARGETARCH
WORKDIR /app
COPY go.* .
RUN go mod download
COPY . .
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o app .

FROM alpine:3.19
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/app .
CMD ["./app"]
```

## Dockerfile 작성 베스트 프랙티스

### 1. 베이스 이미지 선택

```dockerfile
# 나쁜 예: 큰 이미지
FROM ubuntu:latest

# 좋은 예: 작고 특정 버전
FROM alpine:3.16
```

### 2. 레이어 캐시 활용

```dockerfile
# 나쁜 예: 소스 변경 시 의존성도 재설치
COPY . .
RUN npm install

# 좋은 예: 의존성 파일만 먼저 복사
COPY package*.json ./
RUN npm install
COPY . .
```

### 3. 레이어 최소화

```dockerfile
# 나쁜 예: 많은 레이어
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git

# 좋은 예: 하나의 레이어로
RUN apt-get update && apt-get install -y \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*
```

### 4. 불필요한 파일 제외

```dockerfile
# 나쁜 예: 빌드 파일 포함
COPY . .

# 좋은 예: .dockerignore 활용
COPY src/ ./src/
COPY package*.json ./
```

### 5. 보안 고려 (2025 보안 강화)

```dockerfile
# 비특권 사용자 생성
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

# 시크릿 마운트 사용 (빌드 시에만)
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) \
    npm install --production

# 헬스체크 추가
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js || exit 1

USER nodejs

# 읽기 전용 루트 파일시스템
# 런타임에 --read-only 플래그 사용
```

### 6. 2025년 새로운 기능들

#### SBOM (Software Bill of Materials) 생성

```bash
# SBOM과 함께 이미지 빌드
docker buildx build \
  --sbom=true \
  --output type=image,name=myapp:latest,push=true .

# 커스텀 SBOM 스캐너 사용
docker buildx build \
  --sbom=generator=trivy \
  --output type=image,name=myapp:latest,push=true .
```

#### 보안 스캔 통합

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.19
RUN apk add --no-cache python3

# 빌드 시 취약점 스캔
# docker build --check .
```

#### 재현 가능한 빌드

```dockerfile
# SOURCE_DATE_EPOCH 설정으로 타임스탬프 고정
ARG SOURCE_DATE_EPOCH
RUN echo "Built at: $(date -d @${SOURCE_DATE_EPOCH})"
```

## 빌드 인자(ARG) 사용

```dockerfile
# 빌드 시점에 값 전달
ARG NODE_VERSION=16
FROM node:${NODE_VERSION}

ARG APP_VERSION=1.0.0
LABEL version=${APP_VERSION}
```

빌드 시:
```bash
docker build --build-arg NODE_VERSION=18 -t my-app .
```

## 실습: Python Flask 앱 이미지 만들기

```python
# app.py
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
        'message': 'Hello from Docker!',
        'environment': os.environ.get('ENV', 'development')
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

```dockerfile
# Dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

ENV ENV=production

EXPOSE 5000

CMD ["python", "app.py"]
```

## 실전 예제: React 앱 멀티 플랫폼 이미지

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM node:20-alpine AS builder
WORKDIR /app

# 의존성 캐싱
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# 소스 복사 및 빌드
COPY . .
RUN npm run build

# Nginx 실행 스테이지
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

# 보안 헤더 추가
RUN echo 'add_header X-Frame-Options "SAMEORIGIN" always;' >> /etc/nginx/conf.d/security.conf
RUN echo 'add_header X-Content-Type-Options "nosniff" always;' >> /etc/nginx/conf.d/security.conf

EXPOSE 80
HEALTHCHECK CMD wget --quiet --tries=1 --spider http://localhost || exit 1
```

## 이미지 크기 비교표

| 접근 방식 | 베이스 이미지 | 최종 크기 | 보안 | 권장 사용 |
|----------|--------------|-----------|------|-----------|
| **일반** | ubuntu:latest | 500MB+ | 낮음 | 개발 환경 |
| **Alpine** | alpine:3.19 | 50-100MB | 중간 | 일반 프로덕션 |
| **Distroless** | gcr.io/distroless | 10-50MB | 높음 | 보안 중요 |
| **Scratch** | scratch | 5-20MB | 매우 높음 | 정적 바이너리 |

## 보안 체크리스트

```mermaid
graph LR
    A[Dockerfile 보안] --> B[베이스 이미지]
    A --> C[사용자 권한]
    A --> D[시크릿 관리]
    A --> E[취약점 스캔]
    
    B --> B1[공식 이미지 사용]
    B --> B2[특정 버전 태그]
    
    C --> C1[비특권 사용자]
    C --> C2[최소 권한]
    
    D --> D1[빌드 시크릿]
    D --> D2[런타임 시크릿]
    
    E --> E1[SBOM 생성]
    E --> E2[정기 스캔]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style E fill:#9f9,stroke:#333,stroke-width:2px
```

## 마무리

2025년의 Dockerfile은 단순한 이미지 빌드를 넘어 보안, 성능, 멀티 플랫폼을 모두 고려한 현대적인 애플리케이션 패키징 도구로 진화했습니다. BuildKit의 고급 기능들을 활용하면 더 안전하고 효율적인 컨테이너 이미지를 만들 수 있습니다.

### 핵심 포인트
- ✅ BuildKit 기본 활성화로 빌드 성능 향상
- ✅ SBOM 생성으로 공급망 보안 강화
- ✅ 멀티 플랫폼 빌드로 다양한 환경 지원
- ✅ Distroless 이미지로 공격 표면 최소화

## 다음 편 예고

- Docker 네트워크 타입과 최신 네트워킹 기능
- 컨테이너 간 안전한 통신
- Service Mesh 통합
- Zero Trust 네트워크 보안

이제 2025년 최신 기술로 안전하고 효율적인 Docker 이미지를 만들 수 있습니다! 🐳