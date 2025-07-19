---
layout: post
title: "[Docker 입문 #4] Docker 이미지 이해하기"
date: 2025-07-18 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, tutorial, series, docker-images, layers, registry, beginner]
mermaid: true
---

컨테이너를 실행해봤으니 이제 Docker의 핵심 개념인 이미지(Image)에 대해 자세히 알아봅시다. Docker 이미지는 컨테이너를 만들기 위한 템플릿으로, 애플리케이션 실행에 필요한 모든 것을 포함합니다.

## Docker 이미지란?

Docker 이미지는 컨테이너 실행에 필요한 파일들과 설정값들을 포함하는 **읽기 전용(read-only) 템플릿**입니다. 

### 이미지의 특징

| 특징 | 설명 | 예시 |
|------|------|------|
| **불변성(Immutable)** | 한 번 생성된 이미지는 변경할 수 없음 | `ubuntu:22.04` SHA256: `1234...` |
| **레이어 구조** | 여러 레이어가 쌓여서 하나의 이미지 구성 | Base OS + Runtime + App |
| **공유 가능** | Docker Hub 등을 통해 다른 사람과 공유 | `docker.io/library/nginx` |
| **버전 관리** | 태그를 통해 버전 관리 | `node:18-alpine`, `node:20-alpine` |
| **콘텐츠 주소 기반** | 내용에 따른 고유 식별자 | SHA256 해시로 실제 내용 검증 |

## 이미지 레이어 구조

Docker 이미지의 가장 중요한 특징은 레이어(Layer) 구조입니다:

```
┌─────────────────┐
│   애플리케이션   │  ← 최상위 레이어
├─────────────────┤
│   라이브러리     │
├─────────────────┤
│   런타임        │
├─────────────────┤
│   OS 파일       │  ← 베이스 레이어
└─────────────────┘
```

각 레이어는 **이전 레이어의 변경사항만을 저장**하므로 저장 공간을 효율적으로 사용합니다.

### 레이어 공유 예시

| 이미지 | 베이스 레이어 | 공유 레이어 | 고유 레이어 |
|------|-------------|-------------|-------------|
| `nginx:1.21` | `ubuntu:20.04` | nginx 런타임 | 설정 파일 |
| `my-app:1.0` | `ubuntu:20.04` | nginx 런타임 | 애플리케이션 |
| `node:18-alpine` | `alpine:3.18` | Node.js 런타임 | npm 패키지 |

### 레이어 확인하기

```bash
# 이미지의 레이어 정보 확인
docker image inspect ubuntu:latest

# 이미지 히스토리 확인
docker history ubuntu:latest

# 레이어 SHA256 ID 확인
docker image inspect --format "{{json .RootFS.Layers}}" ubuntu:latest
```

**예시 출력:**
```json
[
  "sha256:72e830a4dff5f0d5225cdc0a320e85ab1ce06ea5673acfe8d83a7645cbd0e9cf",
  "sha256:07b4a9068b6af337e8b8f1f1dae3dd14185b2c0003a9a1f0a6fd2587495b204a",
  "sha256:cc644054967e516db4689b5282ee98e4bc4b11ea2255c9630309f559ab96562e"
]
```

## 이미지 이름과 태그

Docker 이미지는 다음과 같은 형식으로 이름이 지정됩니다:

```
[레지스트리/]이름[:태그]
```

예시:
- `ubuntu:latest` - Docker Hub의 공식 Ubuntu 이미지
- `nginx:1.21` - 특정 버전의 Nginx
- `myregistry.com/myapp:v2.0` - 사설 레지스트리의 이미지

### 태그의 중요성

```bash
# latest 태그 (기본값)
docker pull ubuntu
docker pull ubuntu:latest  # 위와 동일

# 특정 버전 태그
docker pull ubuntu:20.04
docker pull ubuntu:22.04

# 여러 태그 확인
docker images ubuntu
```

**주의**: `latest` 태그가 항상 최신 버전을 의미하는 것은 아닙니다. 프로덕션에서는 명시적인 버전 태그 사용을 권장합니다.

## 이미지 관리 명령어

### 이미지 검색

```bash
# Docker Hub에서 이미지 검색
docker search nginx

# 공식 이미지만 검색
docker search --filter is-official=true nginx

# 별점이 높은 이미지 검색
docker search --filter stars=100 nginx
```

### 이미지 다운로드

```bash
# 이미지 다운로드
docker pull nginx

# 특정 버전 다운로드
docker pull nginx:1.21-alpine

# 모든 태그 다운로드
docker pull -a nginx
```

### 이미지 목록 확인

```bash
# 로컬 이미지 목록
docker images

# 특정 이미지만 표시
docker images nginx

# 이미지 ID만 표시
docker images -q

# 상세 정보 표시
docker images --no-trunc
```

### 이미지 삭제

```bash
# 이미지 삭제
docker rmi nginx:latest

# 여러 이미지 한번에 삭제
docker rmi nginx:latest ubuntu:latest

# 사용하지 않는 이미지 모두 삭제
docker image prune

# 모든 이미지 삭제 (주의!)
docker rmi $(docker images -q)
```

## 이미지 태그 작업

기존 이미지에 새로운 태그를 추가할 수 있습니다:

```bash
# 새 태그 추가
docker tag ubuntu:latest my-ubuntu:v1.0

# 확인
docker images | grep ubuntu
```

## 이미지 크기 최적화

### Alpine Linux 사용

Alpine Linux는 보안과 크기에 최적화된 리눅스 배포판입니다:

```bash
# 일반 Ubuntu vs Alpine 비교
docker pull ubuntu:latest
docker pull alpine:latest
docker images
```

### 이미지 크기 비교

| 베이스 이미지 | 크기 | 특징 | 사용 예시 |
|-------------|------|------|----------|
| `ubuntu:22.04` | ~70MB | 일반적인 리눅스 | 개발/테스트 환경 |
| `alpine:latest` | ~5MB | 최소한 크기 | 프로덕션 마이크로서비스 |
| `node:18-alpine` | ~110MB | Node.js + Alpine | Node.js 애플리케이션 |
| `node:18` | ~900MB | Node.js + 전체 Debian | 개발 도구 필요시 |
| `distroless` | ~2MB | 런타임만 포함 | 최대 보안 필요시 |

**성능 비교 예시:**
```bash
# 시간 측정
time docker pull ubuntu:latest
time docker pull alpine:latest

# 실제 사용량 확인
docker system df
```

### 멀티 스테이지 빌드 예시

```dockerfile
# 빌드 스테이지
FROM golang:1.19 AS builder
WORKDIR /app
COPY . .
RUN go build -o main .

# 실행 스테이지
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .
CMD ["./main"]
```

## 이미지 저장과 불러오기

이미지를 파일로 저장하고 다시 불러올 수 있습니다:

```bash
# 이미지를 tar 파일로 저장
docker save -o ubuntu.tar ubuntu:latest

# tar 파일에서 이미지 불러오기
docker load -i ubuntu.tar

# 압축해서 저장
docker save ubuntu:latest | gzip > ubuntu.tar.gz

# 압축 해제하며 불러오기
gunzip -c ubuntu.tar.gz | docker load
```

## 이미지와 컨테이너의 관계

```bash
# 이미지에서 컨테이너 생성
docker create --name my-container ubuntu:latest

# 컨테이너에서 이미지 생성
docker commit my-container my-image:v1.0
```

### 컨테이너 변경사항을 이미지로 저장

```bash
# 컨테이너 실행 및 변경
docker run -it --name test-container ubuntu:latest /bin/bash
# 컨테이너 내부에서
apt-get update
apt-get install -y curl
exit

# 변경사항을 새 이미지로 저장
docker commit test-container ubuntu-with-curl:v1.0

# 확인
docker images | grep ubuntu-with-curl
```

## 이미지 정보 확인

### 상세 정보 보기

```bash
# JSON 형식으로 상세 정보 확인
docker inspect ubuntu:latest

# 특정 정보만 추출
docker inspect -f '{{.Architecture}}' ubuntu:latest
docker inspect -f '{{.Os}}' ubuntu:latest
docker inspect -f '{{.Config.Env}}' ubuntu:latest
```

### 이미지 레이어 분석

```bash
# 이미지 생성 과정 확인
docker history nginx:latest

# 각 레이어의 크기 확인
docker history --no-trunc nginx:latest
```

## 실습: 이미지 다루기

### 1. 이미지 비교하기

```bash
# 다양한 이미지 다운로드
docker pull nginx:latest
docker pull nginx:alpine
docker pull httpd:latest

# 크기 비교
docker images | grep -E "nginx|httpd"
```

### 2. 태그 활용하기

```bash
# 개발/운영 환경 구분
docker tag nginx:latest myapp:development
docker tag nginx:latest myapp:production
docker tag nginx:latest myapp:v1.0.0

# 확인
docker images myapp
```

### 3. 이미지 정리하기

```bash
# 사용하지 않는 이미지 확인
docker images -f "dangling=true"

# 정리
docker image prune

# 더 적극적인 정리 (주의!)
docker image prune -a
```

## 베스트 프랙티스

### 이미지 최적화 전략

| 전략 | 설명 | 예시 | 효과 |
|------|------|------|------|
| **구체적인 태그** | `latest` 대신 명시적 버전 | `nginx:1.21.6` | 재현 가능성 ↑ |
| **작은 베이스 이미지** | Alpine, distroless 사용 | `node:18-alpine` | 이미지 크기 90% ↓ |
| **레이어 최소화** | 불필요한 레이어 생성 방지 | RUN 명령어 결합 | 빌드 시간 ↓ |
| **캐시 활용** | 자주 변경되지 않는 부분 먼저 | 의존성 설치 먼저 | 빌드 시간 80% ↓ |
| **보안 업데이트** | 정기적인 베이스 이미지 업데이트 | `--no-cache` 옵션 | 보안 취약점 패치 |
| **멀티 스테이지** | 빌드와 실행 환경 분리 | 상단 예시 참조 | 이미지 크기 70% ↓ |

### 캐시 최적화 예시

```dockerfile
# 비효율적인 방법
FROM node
WORKDIR /app
COPY . .          # 소스 코드 변경 시 의존성 재설치
RUN npm install
RUN npm build

# 효율적인 방법
FROM node
WORKDIR /app
COPY package*.json ./  # 의존성 파일만 먼저
RUN npm install        # 의존성 변경 시만 재실행
COPY . .              # 소스 코드 복사
RUN npm build
```

### 보안 최적화

```bash
# 이미지 보안 스캔
docker scan nginx:latest

# 업데이트된 이미지로 재빌드
docker build --no-cache -t my-app:latest .

# 론너 사용자 설정
RUN adduser --disabled-password --uid 10001 appuser
USER appuser
```

## 마무리

Docker 이미지는 컨테이너의 기반이 되는 핵심 개념입니다. 레이어 구조를 이해하고 효율적으로 관리하는 것이 Docker를 잘 사용하는 첫걸음입니다.

### 주요 포인트 요약

```mermaid
mindmap
  root((Docker 이미지))
    레이어 구조
      SHA256식별자
      레이어 공유
      저장소 효율성
    최적화
      멀티스테이지
      Alpine 사용
      캐시 활용
    보안
      정기 업데이트
      비특권 사용자
      스캔 도구
```

### 실제 사용 시나리오

1. **개발 단계**: 일반 이미지 사용 (`node:18`)
2. **스테이징**: 중간 최적화 (`node:18-alpine`)
3. **프로덕션**: 멀티스테이지 + distroless

다음 편에서는 직접 이미지를 만들 수 있는 Dockerfile에 대해 알아보겠습니다.

## 다음 편 예고

### 다음에 다룰 내용

| 주제 | 내용 | 실습 |
|------|------|------|
| **Dockerfile 기본 문법** | FROM, RUN, COPY, CMD 등 | 간단한 웹앱 이미지 작성 |
| **빌드 최적화** | 레이어 캐싱, .dockerignore | 빌드 시간 단축 기법 |
| **멀티 스테이지** | 여러 스테이지 활용 | 프로덕션 준비 이미지 |
| **보안 베스트** | 비특권 사용자, 시크릿 | 보안 강화 전략 |

### 예습 과제

```bash
# 1. 비기너를 위한 이미지 비교
docker images | grep -E "ubuntu|alpine|node"

# 2. 이미지 레이어 분석
docker history nginx:alpine

# 3. 이미지 정리
docker image prune
```

이제 컨테이너뿐만 아니라 이미지도 자유자재로 다룰 수 있게 되었습니다!