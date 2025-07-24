---
layout: post
title: "[Docker 입문 #11] Docker 보안 베스트 프랙티스"
date: 2025-07-25 10:00:00 +0900
categories: [DevOps, Docker, Security]
tags: [docker, container, tutorial, series, security, best-practices, vulnerabilities, beginner]
mermaid: true
---

Docker는 편리하지만 잘못 사용하면 보안 위험이 될 수 있습니다. 이번 편에서는 Docker를 안전하게 사용하기 위한 보안 베스트 프랙티스를 알아보겠습니다.

## Docker 보안의 중요성

컨테이너는 프로세스 격리를 제공하지만, 완벽한 보안 경계는 아닙니다. 적절한 보안 조치 없이는 다음과 같은 위험이 있습니다:

### 주요 보안 위협

| 위협 유형 | 설명 | 영향도 | 대응 방안 |
|-----------|------|--------|-----------|
| **컨테이너 탈출** | 컨테이너에서 호스트로 접근 | 🔴 매우 높음 | 권한 제한, 보안 프로필 |
| **권한 상승** | 일반 사용자가 root 권한 획득 | 🔴 매우 높음 | 비특권 사용자 실행 |
| **민감정보 노출** | 시크릿, 비밀번호 유출 | 🔴 매우 높음 | 시크릿 관리 도구 사용 |
| **악성 이미지** | 악의적 코드 포함 이미지 | 🟡 높음 | 이미지 스캔, 서명 확인 |
| **네트워크 공격** | 컨테이너 간 침투 | 🟡 높음 | 네트워크 격리 |
| **리소스 고갈** | DoS 공격 | 🟢 중간 | 리소스 제한 설정 |

## 1. 이미지 보안

### 공식 이미지 사용

```bash
# 좋은 예: 공식 이미지
docker pull nginx:latest
docker pull node:16-alpine

# 확인이 필요한 예: 비공식 이미지
docker pull unknown-user/nginx

# 이미지 정보 확인
docker inspect nginx:latest | grep -i "Author\|Created"
```

### 이미지 스캔

```bash
# Docker Scout로 취약점 스캔
docker scout cves nginx:latest

# Trivy로 상세 스캔
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image nginx:latest

# Grype로 스캔
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  anchore/grype:latest nginx:latest
```

### 최소 베이스 이미지 사용

```dockerfile
# 나쁜 예: 큰 이미지
FROM ubuntu:latest
RUN apt-get update && apt-get install -y nodejs

# 좋은 예: 최소 이미지
FROM node:16-alpine

# 더 좋은 예: distroless
FROM gcr.io/distroless/nodejs:16
```

## 2. Dockerfile 보안

### 비특권 사용자 실행

```dockerfile
# 나쁜 예: root로 실행
FROM node:16-alpine
COPY . /app
CMD ["node", "app.js"]

# 좋은 예: 일반 사용자로 실행
FROM node:16-alpine

# 사용자 생성
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# 작업 디렉토리 설정 및 권한
WORKDIR /app
COPY --chown=nodejs:nodejs . .

# 사용자 전환
USER nodejs

CMD ["node", "app.js"]
```

### 민감한 정보 제외

**.dockerignore 파일:**
```
.env
.git
*.pem
*.key
secrets/
credentials/
.aws/
.ssh/
```

### 멀티 스테이지 빌드

```dockerfile
# 빌드 스테이지
FROM node:16-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# 실행 스테이지 (빌드 도구 제외)
FROM node:16-alpine
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001
WORKDIR /app
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .
USER nodejs
CMD ["node", "app.js"]
```

### 헬스체크 추가

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD node healthcheck.js || exit 1
```

## 3. 컨테이너 실행 보안

### 읽기 전용 루트 파일시스템

```bash
# 읽기 전용으로 실행
docker run --read-only --tmpfs /tmp \
  --tmpfs /var/run nginx:alpine
```

### 권한 제한

```bash
# 권한 제거
docker run --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  nginx:alpine

# 보안 옵션
docker run --security-opt=no-new-privileges:true \
  --security-opt=apparmor:docker-default \
  nginx:alpine
```

### 리소스 제한

```bash
# CPU와 메모리 제한
docker run -d \
  --cpus="0.5" \
  --memory="512m" \
  --memory-swap="512m" \
  --pids-limit=100 \
  nginx:alpine
```

## 4. 네트워크 보안

### 네트워크 격리

```bash
# 격리된 네트워크 생성
docker network create --driver bridge isolated_network

# 내부 전용 네트워크
docker network create --internal internal_network

# 컨테이너 연결
docker run -d --network isolated_network --name app myapp
```

### 포트 바인딩 제한

```bash
# 나쁜 예: 모든 인터페이스에 바인딩
docker run -p 3306:3306 mysql

# 좋은 예: localhost만 바인딩
docker run -p 127.0.0.1:3306:3306 mysql
```

## 5. 시크릿 관리

### Docker Secrets (Swarm mode)

```bash
# 시크릿 생성
echo "my-secret-password" | docker secret create db_password -

# 서비스에서 사용
docker service create \
  --name myapp \
  --secret db_password \
  myapp:latest
```

### BuildKit 시크릿

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine

# 빌드 시 시크릿 사용
RUN --mount=type=secret,id=mysecret \
  cat /run/secrets/mysecret
```

빌드:
```bash
docker build --secret id=mysecret,src=secret.txt .
```

### 환경 변수 대신 파일 사용

```dockerfile
# 환경 변수 대신 파일에서 읽기
CMD DB_PASSWORD=$(cat /run/secrets/db_password) node app.js
```

## 6. Docker Compose 보안

### docker-compose.yml 보안 설정

```yaml
version: '3.8'

services:
  app:
    image: myapp:latest
    # 보안 옵션
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
    # 사용자
    user: "1001:1001"
    # 리소스 제한
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    # 헬스체크
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## 7. 로깅과 모니터링

### 보안 이벤트 로깅

```bash
# Docker 이벤트 모니터링
docker events --filter type=container

# 로그 드라이버 설정
docker run -d \
  --log-driver=syslog \
  --log-opt syslog-address=tcp://192.168.1.100:514 \
  nginx:alpine
```

### 감사 로깅

```json
// /etc/docker/daemon.json
{
  "log-level": "info",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "experimental": true,
  "metrics-addr": "127.0.0.1:9323"
}
```

## 8. 실습: 보안 강화된 웹 애플리케이션

### Dockerfile

```dockerfile
FROM node:16-alpine AS builder
WORKDIR /build
COPY package*.json ./
RUN npm ci --only=production

FROM gcr.io/distroless/nodejs:16
WORKDIR /app
COPY --from=builder /build/node_modules ./node_modules
COPY . .
USER nonroot
EXPOSE 3000
CMD ["app.js"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      - NODE_ENV=production
    secrets:
      - db_password
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    read_only: true
    tmpfs:
      - /tmp
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
    healthcheck:
      test: ["CMD", "node", "healthcheck.js"]
      interval: 30s
      timeout: 3s
      retries: 3

  db:
    image: postgres:13-alpine
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db_data:/var/lib/postgresql/data
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - FOWNER
      - SETGID
      - SETUID

secrets:
  db_password:
    file: ./secrets/db_password.txt

volumes:
  db_data:
```

## 9. 컨테이너 런타임 보안

### Seccomp 프로필

```json
// seccomp-profile.json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["accept", "bind", "connect"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

사용:
```bash
docker run --security-opt seccomp=seccomp-profile.json myapp
```

### AppArmor 프로필

```bash
# AppArmor 프로필 로드
sudo apparmor_parser -r -W /etc/apparmor.d/docker-nginx

# 프로필 적용
docker run --security-opt apparmor=docker-nginx nginx
```

## 10. 보안 체크리스트

### 이미지 보안
- [ ] 공식/검증된 이미지 사용
- [ ] 최소 베이스 이미지 선택
- [ ] 정기적인 이미지 스캔
- [ ] 이미지 서명 확인

### Dockerfile 보안
- [ ] 비특권 사용자로 실행
- [ ] 민감한 정보 제외
- [ ] 불필요한 패키지 제거
- [ ] 헬스체크 구현

### 런타임 보안
- [ ] 읽기 전용 파일시스템
- [ ] 권한 최소화 (cap_drop)
- [ ] 리소스 제한 설정
- [ ] 네트워크 격리

### 시크릿 관리
- [ ] 하드코딩된 시크릿 제거
- [ ] Docker Secrets 또는 외부 시크릿 관리 도구 사용
- [ ] 환경 변수 노출 최소화

### 모니터링
- [ ] 로그 수집 및 분석
- [ ] 보안 이벤트 모니터링
- [ ] 정기적인 보안 감사

## 보안 도구

### 2025년 최신 보안 도구 비교

| 도구 | 유형 | 주요 기능 | 장점 | 단점 | 가격 |
|------|------|-----------|------|------|------|
| **Docker Scout** | 취약점 스캐너 | 이미지 취약점 분석 | Docker 통합 | 기본 기능만 | 무료/Pro |
| **Trivy** | 종합 스캐너 | 이미지, IaC, 설정 검사 | 빠른 속도 | CLI 중심 | 무료 |
| **Snyk** | 취약점 스캐너 | 실시간 모니터링 | 개발자 친화적 | 비용 | 무료/유료 |
| **Falco** | 런타임 보안 | 실시간 위협 감지 | CNCF 프로젝트 | 복잡한 설정 | 무료 |
| **Aqua Security** | 엔터프라이즈 | 전체 라이프사이클 | 종합 솔루션 | 높은 비용 | 유료 |

### 보안 도구 통합 워크플로우

```mermaid
graph LR
    A[개발] --> B[빌드]
    B --> C{이미지 스캔}
    C -->|통과| D[레지스트리]
    C -->|실패| E[수정]
    E --> B
    D --> F[배포]
    F --> G[런타임 모니터링]
    G --> H[로그 분석]
    H --> I[보안 대응]
```

## 보안 강화 실전 예제

### 완전히 보안 강화된 웹 애플리케이션 구성

```yaml
# docker-compose-secure.yml
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.secure
    image: myapp:secure
    ports:
      - "127.0.0.1:8080:8080"
    security_opt:
      - no-new-privileges:true
      - apparmor=docker-default
      - seccomp=/path/to/seccomp/profile.json
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100M
      - /var/run:noexec,nosuid,size=50M
    user: "10001:10001"
    environment:
      - NODE_ENV=production
    secrets:
      - db_password
      - api_key
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    networks:
      - frontend
      - backend

  db:
    image: postgres:15-alpine
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - FOWNER
      - SETGID
      - SETUID
    user: postgres
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: secure_app
    secrets:
      - db_password
    volumes:
      - db_data:/var/lib/postgresql/data:Z
    networks:
      - backend
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    file: ./secrets/api_key.txt

volumes:
  db_data:
    driver: local
```

## 마무리

Docker 보안은 단일 조치가 아닌 여러 계층의 방어가 필요합니다. 이미지부터 런타임까지 각 단계에서 보안을 고려해야 합니다. 

### 핵심 보안 원칙
1. **최소 권한 원칙**: 필요한 최소한의 권한만 부여
2. **Defense in Depth**: 다층 보안 구현
3. **Zero Trust**: 모든 것을 검증
4. **지속적 모니터링**: 실시간 위협 감지
5. **정기적 업데이트**: 취약점 패치 적용

다음 편에서는 Docker 로그와 모니터링에 대해 자세히 알아보겠습니다.

## 다음 편 예고

- Docker 로깅 전략
- 중앙 집중식 로그 관리
- 컨테이너 메트릭 수집
- 실시간 모니터링 구축

안전하고 관찰 가능한 컨테이너 환경을 만들어봅시다! 🔍