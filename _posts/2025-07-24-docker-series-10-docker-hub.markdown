---
layout: post
title: "[Docker 입문 #10] Docker Hub 활용하기 - 이미지 저장소 완벽 가이드"
date: 2025-07-24 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, tutorial, series, docker-hub, registry, image-sharing, beginner]
mermaid: true
---

Docker Hub는 Docker 이미지를 저장하고 공유하는 클라우드 기반 레지스트리 서비스입니다. GitHub이 소스 코드를 위한 것이라면, Docker Hub는 컨테이너 이미지를 위한 중앙 저장소입니다.

## Docker Hub란?

### 주요 기능 비교

| 기능 | 무료 플랜 | Pro 플랜 | Team 플랜 | Business 플랜 |
|------|-----------|----------|-----------|---------------|
| 공개 저장소 | 무제한 | 무제한 | 무제한 | 무제한 |
| 비공개 저장소 | 1개 | 무제한 | 무제한 | 무제한 |
| 병렬 빌드 | 1개 | 5개 | 무제한 | 무제한 |
| 이미지 풀 제한 | 100회/6시간 | 5,000회/일 | 15,000회/일 | 무제한 |
| 취약점 스캔 | ❌ | ✅ | ✅ | ✅ |
| 팀 협업 | ❌ | ❌ | ✅ | ✅ |
| SSO/SAML | ❌ | ❌ | ❌ | ✅ |
| 가격 | 무료 | $7/월 | $11/사용자/월 | $24/사용자/월 |

### Docker Hub 아키텍처

```mermaid
graph TB
    subgraph "Local Environment"
        A[Docker Client] --> B[Docker Daemon]
        B --> C[Local Images]
    end
    
    subgraph "Docker Hub"
        D[Registry API] --> E[Image Storage]
        D --> F[Metadata DB]
        D --> G[Auth Service]
        
        H[Web UI] --> D
        I[Webhooks] --> D
        J[Build Service] --> D
    end
    
    subgraph "Integrations"
        K[GitHub] --> J
        L[Bitbucket] --> J
        M[CI/CD Tools] --> D
    end
    
    B -->|push/pull| D
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style E fill:#bbf,stroke:#333,stroke-width:2px
    style J fill:#bfb,stroke:#333,stroke-width:2px
```

## Docker Hub 시작하기

### 1. 계정 생성 및 설정

```bash
# Docker Hub 로그인
docker login

# 특정 레지스트리 로그인
docker login registry.example.com

# 자격 증명 저장 위치
# Linux: ~/.docker/config.json
# Windows: %USERPROFILE%\.docker\config.json
# macOS: ~/.docker/config.json

# 안전한 로그인 (stdin 사용)
echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
```

### 2. 인증 토큰 관리

Docker Hub에서는 비밀번호 대신 액세스 토큰 사용을 권장합니다:

```bash
# 1. Docker Hub 웹사이트에서 토큰 생성
# Account Settings > Security > New Access Token

# 2. 토큰으로 로그인
docker login -u username
# Password: <access-token>

# 3. 환경 변수로 관리
export DOCKER_TOKEN=your-access-token
echo $DOCKER_TOKEN | docker login -u username --password-stdin
```

## 이미지 관리 워크플로우

### 이미지 라이프사이클

```mermaid
stateDiagram-v2
    [*] --> Build: docker build
    Build --> Tag: docker tag
    Tag --> Test: Local Testing
    Test --> Push: docker push
    Push --> DockerHub: Stored
    
    DockerHub --> Pull: docker pull
    Pull --> Deploy: Container Run
    Deploy --> Update: New Version
    Update --> Build
    
    DockerHub --> Scan: Security Scan
    Scan --> Alert: Vulnerabilities Found
    Alert --> Fix: Update Base Image
    Fix --> Build
```

### 1. 효율적인 태깅 전략

```bash
# 태깅 컨벤션
username/app:latest          # 최신 개발 버전
username/app:stable          # 안정 버전
username/app:1.2.3          # 특정 버전
username/app:1.2            # 마이너 버전
username/app:1              # 메이저 버전
username/app:alpine         # 베이스 이미지 표시
username/app:dev-20250124   # 개발 빌드
username/app:prod-1.2.3     # 프로덕션 버전

# 멀티 태그 적용
docker build -t username/app:latest \
             -t username/app:1.2.3 \
             -t username/app:1.2 \
             -t username/app:1 .
```

### 2. 이미지 메타데이터 추가

```dockerfile
# Dockerfile with metadata
FROM node:18-alpine

# 메타데이터 라벨
LABEL maintainer="your-email@example.com"
LABEL version="1.2.3"
LABEL description="Sample Node.js application"
LABEL org.opencontainers.image.source="https://github.com/username/repo"
LABEL org.opencontainers.image.created="2025-01-24T10:00:00Z"
LABEL org.opencontainers.image.authors="Your Name"
LABEL org.opencontainers.image.licenses="MIT"

WORKDIR /app
COPY . .
RUN npm install --production

EXPOSE 3000
CMD ["node", "server.js"]
```

## 실습: 프로덕션 레벨 이미지 배포

### 1. 멀티스테이지 빌드 애플리케이션

```dockerfile
# Dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Runtime stage
FROM node:18-alpine
RUN apk add --no-cache tini
WORKDIR /app

# 비루트 사용자 생성
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# 빌드 스테이지에서 복사
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .

USER nodejs
EXPOSE 3000

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

### 2. 헬스체크 포함 이미지

```dockerfile
# 헬스체크 추가
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js || exit 1
```

```javascript
// healthcheck.js
const http = require('http');

const options = {
  hostname: 'localhost',
  port: 3000,
  path: '/health',
  timeout: 2000
};

const req = http.request(options, (res) => {
  process.exit(res.statusCode === 200 ? 0 : 1);
});

req.on('error', () => process.exit(1));
req.end();
```

### 3. 자동화된 빌드 및 배포

```yaml
# .github/workflows/docker-multiarch.yml
name: Docker Multi-Architecture Build

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: docker.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      
    steps:
    - name: Checkout
      uses: actions/checkout@v4
      
    - name: Set up QEMU
      uses: docker/setup-qemu-action@v3
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
      
    - name: Log in to Docker Hub
      if: github.event_name != 'pull_request'
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_TOKEN }}
        
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=semver,pattern={{major}}
          type=sha,prefix={{branch}}-
          
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        platforms: linux/amd64,linux/arm64,linux/arm/v7
        push: ${{ github.event_name != 'pull_request' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        
    - name: Run security scan
      if: github.event_name != 'pull_request'
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        
    - name: Upload scan results
      if: github.event_name != 'pull_request'
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: 'trivy-results.sarif'
```

## 고급 Docker Hub 기능

### 1. 자동 빌드 규칙 설정

| 트리거 타입 | 소스 패턴 | Docker 태그 | 빌드 컨텍스트 | 예시 |
|------------|----------|-------------|---------------|------|
| Branch | main | latest | / | main → latest |
| Branch | develop | dev | / | develop → dev |
| Tag | /^v([0-9.]+)$/ | {\1} | / | v1.2.3 → 1.2.3 |
| Tag | /^v(\d+)\.(\d+)\.(\d+)$/ | {\1}.{\2} | / | v1.2.3 → 1.2 |
| PR | /^pr-.*/ | pr-{sourceref} | / | pr-123 → pr-123 |

### 2. Webhook 통합

```javascript
// webhook-handler.js
const express = require('express');
const crypto = require('crypto');
const app = express();

app.use(express.json());

// Docker Hub webhook 처리
app.post('/webhook/dockerhub', (req, res) => {
  const { push_data, repository } = req.body;
  
  // Webhook 서명 검증
  const payload = JSON.stringify(req.body);
  const signature = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(payload)
    .digest('hex');
    
  if (signature !== req.headers['x-hub-signature']) {
    return res.status(401).send('Unauthorized');
  }
  
  console.log(`New image pushed: ${repository.repo_name}:${push_data.tag}`);
  
  // 배포 트리거
  triggerDeployment({
    image: `${repository.repo_name}:${push_data.tag}`,
    pusher: push_data.pusher
  });
  
  res.status(200).send('OK');
});

app.listen(3000);
```

### 3. Docker Scout를 이용한 취약점 관리 (2025년 기능)

```bash
# Docker Scout는 Docker Desktop에 통합됨 (CLI 설치 불필요)

# 이미지 취약점 스캔 (실시간 CVE 데이터베이스)
docker scout cves username/app:latest

# AI 기반 취약점 분석 (신기능)
docker scout analyze username/app:latest --ai-insights

# 상세 SBOM 및 라이선스 분석
docker scout sbom username/app:latest --format spdx-json

# 자동 패치 추천
docker scout recommendations username/app:latest --auto-fix

# 규정 준수 체크 (GDPR, HIPAA 등)
docker scout compliance username/app:latest --standard cis-1.5

# CI/CD 통합 및 정책 기반 게이트
docker scout policy username/app:latest --policy production-ready
```

## 프라이빗 레지스트리 구축

### 1. 프로덕션 레벨 레지스트리

```yaml
# docker-compose.yml
version: '3.8'

services:
  registry:
    image: registry:2
    container_name: docker-registry
    restart: always
    ports:
      - "5000:5000"
    environment:
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/cert.pem
      REGISTRY_HTTP_TLS_KEY: /certs/key.pem
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /var/lib/registry
    volumes:
      - registry-data:/var/lib/registry
      - ./certs:/certs:ro
      - ./auth:/auth:ro
    networks:
      - registry-net

  registry-ui:
    image: joxit/docker-registry-ui:latest
    container_name: registry-ui
    restart: always
    ports:
      - "8080:80"
    environment:
      REGISTRY_TITLE: Private Docker Registry
      REGISTRY_URL: https://registry:5000
      DELETE_IMAGES: "true"
      SINGLE_REGISTRY: "true"
    depends_on:
      - registry
    networks:
      - registry-net

volumes:
  registry-data:

networks:
  registry-net:
```

### 2. 인증 설정

```bash
# htpasswd 파일 생성
mkdir auth
docker run --rm --entrypoint htpasswd httpd:2 -Bbn username password > auth/htpasswd

# TLS 인증서 생성
mkdir certs
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout certs/key.pem -out certs/cert.pem \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=registry.local"

# 레지스트리 시작
docker-compose up -d

# 로그인
docker login registry.local:5000
```

### 3. 스토리지 백엔드 옵션

| 백엔드 | 장점 | 단점 | 사용 사례 |
|--------|------|------|-----------|
| Filesystem | 간단함, 설정 용이 | 확장성 제한 | 소규모 환경 |
| AWS S3 | 확장성, 내구성 | 비용, 지연시간 | 클라우드 환경 |
| Azure Blob | Azure 통합 | Azure 종속 | Azure 환경 |
| Google GCS | Google 통합 | GCP 종속 | GCP 환경 |
| Swift | OpenStack 통합 | 복잡성 | 프라이빗 클라우드 |

## 컨테이너 레지스트리 비교

### 주요 레지스트리 서비스 비교

| 레지스트리 | 무료 티어 | 특징 | 통합 | 가격 |
|------------|----------|------|------|------|
| Docker Hub | 1 private repo | 가장 널리 사용 | GitHub Actions | $7/월~ |
| GitHub Container Registry | 500MB/월 | GitHub 통합 우수 | GitHub 생태계 | $0.008/GB |
| AWS ECR | 500MB/월 | AWS 서비스 통합 | ECS, EKS | $0.10/GB |
| Google Container Registry | 무료 할당량 | GCP 통합 | GKE | $0.020/GB |
| Azure Container Registry | 100GB 포함 | Azure 통합 | AKS | $0.167/일~ |
| GitLab Container Registry | 10GB/프로젝트 | GitLab CI/CD | GitLab | 포함 |

## 보안 베스트 프랙티스

### 1. 이미지 서명 및 검증

```bash
# Docker Content Trust 설정
export DOCKER_CONTENT_TRUST=1
export DOCKER_CONTENT_TRUST_SERVER=https://notary.docker.io

# 키 생성 및 이미지 서명
docker trust key generate my-signer
docker trust signer add --key my-signer.pub my-signer username/app

# 서명된 이미지 푸시
docker push username/app:1.0.0

# 서명 확인
docker trust inspect --pretty username/app:1.0.0
```

### 2. 취약점 스캔 자동화

```yaml
# .gitlab-ci.yml 예시
scan-image:
  stage: test
  image: 
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  allow_failure: true
```

### 3. 런타임 보안 정책

```yaml
# Pod Security Policy 예시
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

## 모니터링 및 관리

### Docker Hub API 활용

```bash
# API 토큰 획득
TOKEN=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"'${DOCKER_USER}'","password":"'${DOCKER_PASS}'"}' \
  https://hub.docker.com/v2/users/login/ | jq -r .token)

# 저장소 목록 조회
curl -s -H "Authorization: Bearer ${TOKEN}" \
  https://hub.docker.com/v2/repositories/${DOCKER_USER}/ | jq '.results[].name'

# 태그 목록 조회
curl -s -H "Authorization: Bearer ${TOKEN}" \
  https://hub.docker.com/v2/repositories/${DOCKER_USER}/${REPO}/tags/ | jq '.results[].name'

# 이미지 삭제
curl -X DELETE -H "Authorization: Bearer ${TOKEN}" \
  https://hub.docker.com/v2/repositories/${DOCKER_USER}/${REPO}/tags/${TAG}/
```

### 이미지 크기 최적화 전략

```mermaid
graph LR
    A[Original Image<br/>1.2GB] --> B{Optimization}
    B --> C[Multi-stage Build<br/>800MB]
    B --> D[Alpine Base<br/>450MB]
    B --> E[Distroless<br/>180MB]
    B --> F[Static Binary<br/>50MB]
    
    C --> G[Remove Build Tools]
    D --> H[Minimal Dependencies]
    E --> I[No Shell/Package Manager]
    F --> J[Single Binary]
    
    style A fill:#f99,stroke:#333,stroke-width:2px
    style F fill:#9f9,stroke:#333,stroke-width:2px
```

## 마무리

Docker Hub는 컨테이너 이미지를 관리하고 배포하는 핵심 플랫폼입니다. 효율적인 태깅 전략, 자동화된 빌드 파이프라인, 그리고 보안 스캔을 통해 안전하고 확장 가능한 컨테이너 배포 환경을 구축할 수 있습니다.

### 핵심 포인트
- 📦 **체계적인 태깅**: 버전 관리와 환경 구분
- 🔒 **보안 우선**: 취약점 스캔과 이미지 서명
- 🚀 **자동화**: CI/CD 파이프라인 통합
- 📊 **모니터링**: API를 통한 이미지 관리

## 다음 편 예고

다음 편에서는 "Docker 보안 베스트 프랙티스"를 다룹니다:
- 컨테이너 보안 기본 원칙
- 이미지 보안 강화 방법
- 런타임 보안 모니터링
- 컴플라이언스 및 감사

안전한 컨테이너 환경을 만드는 방법을 배워봅시다! 🔒