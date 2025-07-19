---
layout: post
title: "[Docker 입문 #3] 첫 번째 컨테이너 실행하기"
date: 2025-07-17 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, tutorial, series, docker-run, hello-world, docker-basics, beginner]
mermaid: true
---

Docker를 설치했으니 이제 실제로 컨테이너를 실행해볼 차례입니다. 이번 편에서는 Docker 컨테이너의 기본적인 사용법을 실습을 통해 알아보겠습니다.

## Hello World 컨테이너 실행하기

가장 간단한 예제부터 시작해봅시다:

```bash
docker run hello-world
```

이 명령어를 실행하면 다음과 같은 프로세스가 진행됩니다:

```mermaid
flowchart TD
    A[docker run hello-world] --> B{로컬에 이미지 존재?}
    B -->|아니오| C[Docker Hub에서 다운로드]
    B -->|예| D[이미지로부터 컨테이너 생성]
    C --> D
    D --> E[컨테이너 실행]
    E --> F[Hello World 메시지 출력]
    F --> G[컨테이너 종료]
```

## Ubuntu 컨테이너 실행하기

이번에는 좀 더 실용적인 Ubuntu 컨테이너를 실행해봅시다:

```bash
# Ubuntu 이미지 다운로드
docker pull ubuntu:latest

# Ubuntu 컨테이너 실행
docker run ubuntu:latest echo "Hello from Ubuntu"
```

### 대화형 모드로 실행하기

컨테이너 내부에서 직접 명령어를 실행하려면 `-it` 옵션을 사용합니다:

```bash
docker run -it ubuntu:latest /bin/bash
```

| 옵션 | 의미 | 설명 |
|------|------|------|
| `-i` | interactive | 표준 입력(STDIN)을 열어둡니다 |
| `-t` | tty | 가상 터미널(pseudo-TTY)을 할당합니다 |
| `-it` | 두 옵션 결합 | 대화형 터미널 세션을 생성합니다 |

이제 Ubuntu 컨테이너 내부에 들어왔습니다! 다음 명령어들을 실행해보세요:

```bash
# 컨테이너 내부에서
ls
pwd
whoami
cat /etc/os-release
```

컨테이너에서 나가려면 `exit` 명령어를 입력하거나 `Ctrl + D`를 누릅니다.

## 백그라운드에서 컨테이너 실행하기

웹 서버처럼 계속 실행되어야 하는 서비스는 `-d` 옵션으로 백그라운드에서 실행합니다:

```bash
# Nginx 웹 서버를 백그라운드에서 실행
docker run -d --name my-nginx -p 8080:80 nginx:latest
```

### Docker Run 주요 옵션 (2025년 7월 기준)

| 옵션 | 설명 | 예시 |
|------|------|------|
| `-d, --detach` | 백그라운드에서 실행 | `docker run -d nginx` |
| `--name` | 컨테이너 이름 지정 | `docker run --name web nginx` |
| `-p, --publish` | 포트 매핑 | `-p 8080:80` |
| `-v, --volume` | 볼륨 마운트 | `-v /data:/app/data` |
| `-e, --env` | 환경 변수 설정 | `-e NODE_ENV=production` |
| `--rm` | 종료 시 자동 삭제 | `docker run --rm ubuntu` |
| `--network` | 네트워크 지정 | `--network my-net` |
| `--memory` | 메모리 제한 | `--memory 512m` |
| `--cpus` | CPU 제한 | `--cpus 0.5` |

브라우저에서 `http://localhost:8080`에 접속하면 Nginx 환영 페이지를 볼 수 있습니다.

## 컨테이너 관리하기

### 실행 중인 컨테이너 확인

```bash
# 실행 중인 컨테이너만 표시
docker ps

# 모든 컨테이너 표시 (중지된 것 포함)
docker ps -a
```

```mermaid
stateDiagram-v2
    [*] --> Created: docker create
    Created --> Running: docker start
    Running --> Paused: docker pause
    Paused --> Running: docker unpause
    Running --> Stopped: docker stop
    Stopped --> Running: docker start
    Running --> Stopped: docker kill
    Stopped --> Removed: docker rm
    Removed --> [*]
```

### 컨테이너 중지 및 시작

```bash
# 컨테이너 중지
docker stop my-nginx

# 컨테이너 시작
docker start my-nginx

# 컨테이너 재시작
docker restart my-nginx
```

### 컨테이너 삭제

```bash
# 중지된 컨테이너 삭제
docker rm my-nginx

# 실행 중인 컨테이너 강제 삭제
docker rm -f my-nginx
```

## 컨테이너 내부 살펴보기

### 실행 중인 컨테이너에 접속

```bash
# 새로운 Nginx 컨테이너 실행
docker run -d --name web-server nginx:latest

# 실행 중인 컨테이너에 접속
docker exec -it web-server /bin/bash
```

컨테이너 내부에서 파일을 확인하거나 수정할 수 있습니다:

```bash
# Nginx 설정 파일 확인
cat /etc/nginx/nginx.conf

# 웹 루트 디렉토리 확인
ls /usr/share/nginx/html/
```

### 컨테이너 로그 확인

```bash
# 컨테이너 로그 보기
docker logs web-server

# 실시간으로 로그 보기
docker logs -f web-server

# 마지막 10줄만 보기
docker logs --tail 10 web-server
```

### 컨테이너 상세 정보 확인

```bash
# 컨테이너 상세 정보
docker inspect web-server

# 특정 정보만 추출
docker inspect -f '{{.NetworkSettings.IPAddress}}' web-server
```

## 파일 복사하기

호스트와 컨테이너 간에 파일을 복사할 수 있습니다:

```bash
# 호스트에서 컨테이너로 복사
echo "Hello Docker" > hello.txt
docker cp hello.txt web-server:/tmp/

# 컨테이너에서 호스트로 복사
docker cp web-server:/etc/nginx/nginx.conf ./nginx.conf
```

## 컨테이너 리소스 모니터링

```bash
# 컨테이너 리소스 사용량 확인
docker stats

# 특정 컨테이너만 모니터링
docker stats web-server
```

### Docker Stats 출력 필드 설명

| 필드 | 설명 | 예시 값 |
|------|------|----------|
| **CONTAINER ID** | 컨테이너 ID | a1b2c3d4 |
| **NAME** | 컨테이너 이름 | web-server |
| **CPU %** | CPU 사용률 | 0.15% |
| **MEM USAGE / LIMIT** | 메모리 사용량/제한 | 64MiB / 1.952GiB |
| **MEM %** | 메모리 사용률 | 3.28% |
| **NET I/O** | 네트워크 I/O | 1.2kB / 0B |
| **BLOCK I/O** | 디스크 I/O | 0B / 0B |
| **PIDS** | 프로세스 수 | 2 |

## 실습: 간단한 웹 페이지 배포하기

이제 배운 내용을 활용해 간단한 웹 페이지를 배포해봅시다:

1. **HTML 파일 생성**
```bash
cat > index.html << EOF
<!DOCTYPE html>
<html>
<head>
    <title>My Docker Website</title>
</head>
<body>
    <h1>안녕하세요, Docker!</h1>
    <p>첫 번째 Docker 웹사이트입니다.</p>
</body>
</html>
EOF
```

2. **Nginx 컨테이너 실행**
```bash
docker run -d --name my-website -p 8081:80 nginx:latest
```

3. **HTML 파일 복사**
```bash
docker cp index.html my-website:/usr/share/nginx/html/
```

4. **브라우저에서 확인**
`http://localhost:8081`에 접속하면 방금 만든 웹 페이지를 볼 수 있습니다!

## 유용한 Docker 명령어 정리 (2025년 7월 최신)

### 컨테이너 생명주기 명령어

```mermaid
graph LR
    A[docker create] --> B[Created]
    B --> C[docker start]
    C --> D[Running]
    D --> E[docker stop]
    E --> F[Stopped]
    D --> G[docker pause]
    G --> H[Paused]
    H --> I[docker unpause]
    I --> D
    F --> J[docker rm]
    J --> K[Removed]
```

| 분류 | 명령어 | 설명 | 예시 |
|------|---------|------|------|
| **생명주기** | `docker create` | 컨테이너 생성 | `docker create nginx` |
| | `docker start` | 컨테이너 시작 | `docker start my-nginx` |
| | `docker stop` | 컨테이너 중지 | `docker stop my-nginx` |
| | `docker restart` | 컨테이너 재시작 | `docker restart my-nginx` |
| | `docker pause` | 컨테이너 일시정지 | `docker pause my-nginx` |
| | `docker unpause` | 컨테이너 재개 | `docker unpause my-nginx` |
| | `docker rm` | 컨테이너 삭제 | `docker rm my-nginx` |
| **정보 확인** | `docker ps` | 컨테이너 목록 | `docker ps -a` |
| | `docker logs` | 컨테이너 로그 | `docker logs -f my-nginx` |
| | `docker inspect` | 상세 정보 | `docker inspect my-nginx` |
| | `docker stats` | 리소스 사용량 | `docker stats --no-stream` |
| | `docker top` | 프로세스 목록 | `docker top my-nginx` |
| **상호작용** | `docker exec` | 명령 실행 | `docker exec -it my-nginx bash` |
| | `docker attach` | 컨테이너 연결 | `docker attach my-nginx` |
| | `docker cp` | 파일 복사 | `docker cp file.txt my-nginx:/tmp/` |

## 정리하기

실습이 끝나면 생성한 컨테이너와 이미지를 정리합니다:

```bash
# 모든 컨테이너 중지
docker stop $(docker ps -aq)

# 모든 컨테이너 삭제
docker rm $(docker ps -aq)

# 사용하지 않는 이미지 삭제
docker image prune -a
```

## Docker 최신 동향 (2025년 7월)

### Docker Desktop 주요 기능 업데이트

| 기능 | 설명 | 버전 |
|------|------|------|
| **Docker Init** | Dockerfile 자동 생성 | 4.20+ |
| **Docker Scout** | 보안 취약점 스캔 | 4.17+ |
| **Docker Extensions** | IDE 통합 확장 | 4.8+ |
| **Resource Saver** | 자동 리소스 최적화 | 4.24+ |
| **Compose Watch** | 파일 변경 감지 및 자동 재시작 | 4.25+ |

## 마무리

첫 번째 Docker 컨테이너를 성공적으로 실행했습니다! 이번 편에서는 컨테이너의 기본적인 생명주기와 관리 방법을 배웠습니다. 다음 편에서는 Docker 이미지에 대해 더 자세히 알아보겠습니다.

## 다음 편 예고

- Docker 이미지의 구조
- 이미지 레이어 이해하기
- 이미지 태그와 버전 관리
- Docker Hub에서 이미지 찾기

Docker의 핵심인 이미지에 대해 깊이 있게 알아봅시다!