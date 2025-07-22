---
layout: post
title: "[Docker 입문 #6] Docker 네트워크 기초"
date: 2025-07-20 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, tutorial, series, networking, docker-network, bridge, ports, beginner]
mermaid: true
---

Docker 컨테이너들이 서로 통신하려면 네트워크가 필요합니다. 이번 편에서는 Docker의 네트워크 개념과 컨테이너 간 통신 방법을 알아보겠습니다.

## Docker 네트워크 개요

Docker는 컨테이너를 위한 격리된 네트워크 환경을 제공합니다. 각 컨테이너는 고유한 IP 주소를 가지며, 다양한 네트워크 드라이버를 통해 통신할 수 있습니다.

### 왜 Docker 네트워크가 필요한가?

1. **격리**: 컨테이너 간 네트워크 격리
2. **통신**: 컨테이너 간 안전한 통신
3. **확장성**: 멀티 호스트 네트워킹
4. **유연성**: 다양한 네트워크 토폴로지 구성

## Docker 네트워크 드라이버

Docker는 여러 네트워크 드라이버를 제공합니다:

### 1. Bridge (기본값)

```bash
# 기본 bridge 네트워크 확인
docker network ls

# 컨테이너 실행 (기본 bridge 사용)
docker run -d --name web nginx
docker run -d --name db mysql:5.7
```

### 2. Host

호스트의 네트워크를 직접 사용합니다:

```bash
# host 네트워크 사용
docker run -d --network host --name web nginx
```

### 3. None

네트워크를 사용하지 않습니다:

```bash
# 네트워크 없이 실행
docker run -d --network none --name isolated alpine sleep 1000
```

### 4. Custom Bridge

사용자 정의 브리지 네트워크:

```bash
# 사용자 정의 네트워크 생성
docker network create my-network

# 생성한 네트워크에 컨테이너 연결
docker run -d --network my-network --name web nginx
docker run -d --network my-network --name db mysql:5.7

# IPv6 지원 네트워크 생성 (2025년 최신)
docker network create --ipv6 --subnet 2001:db8:1234::/64 my-ipv6-net

# IPv6 전용 네트워크
docker network create --ipv6 --ipv4=false v6net
```

### 5. Overlay

Docker Swarm에서 사용하는 멀티 호스트 네트워크:

```bash
# Overlay 네트워크 생성
docker network create -d overlay my-overlay

# attachable 옵션으로 standalone 컨테이너도 연결 가능
docker network create -d overlay --attachable my-attachable-overlay

# 서비스에 연결
docker service create \
  --name my-nginx \
  --network my-overlay \
  --replicas 3 \
  nginx

# IPv6 오버레이 네트워크 지원 (2025년 최신)
# IPv6 트랜스포트를 통한 오버레이 네트워크 구성 가능
```

## 네트워크 명령어

### 네트워크 관리

```bash
# 네트워크 목록 확인
docker network ls

# 네트워크 상세 정보
docker network inspect bridge

# 네트워크 생성
docker network create my-app-network

# 네트워크 삭제
docker network rm my-app-network

# 사용하지 않는 네트워크 정리
docker network prune
```

### 컨테이너 네트워크 연결/해제

```bash
# 실행 중인 컨테이너를 네트워크에 연결
docker network connect my-network my-container

# 네트워크에서 연결 해제
docker network disconnect my-network my-container
```

## 컨테이너 간 통신

### 1. 기본 Bridge 네트워크에서의 통신

```bash
# 두 컨테이너 실행
docker run -d --name container1 alpine sleep 1000
docker run -d --name container2 alpine sleep 1000

# IP 주소 확인
docker inspect container1 | grep IPAddress
docker inspect container2 | grep IPAddress

# container1에서 container2로 ping (IP 사용)
docker exec container1 ping -c 3 172.17.0.3
```

### 2. 사용자 정의 네트워크에서의 통신

```bash
# 사용자 정의 네트워크 생성
docker network create app-network

# 컨테이너를 사용자 정의 네트워크에 연결
docker run -d --network app-network --name web nginx
docker run -d --network app-network --name db mysql:5.7 -e MYSQL_ROOT_PASSWORD=secret

# 컨테이너 이름으로 통신 (DNS 자동 설정)
docker exec web ping -c 3 db
docker exec db ping -c 3 web
```

## 실습: 웹 애플리케이션과 데이터베이스 연결

### 1. 네트워크 생성

```bash
docker network create webapp-network
```

### 2. MySQL 데이터베이스 실행

```bash
docker run -d \
  --network webapp-network \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=myapp \
  -e MYSQL_USER=appuser \
  -e MYSQL_PASSWORD=apppass \
  mysql:5.7
```

### 3. 웹 애플리케이션 Dockerfile

```dockerfile
FROM node:14-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
```

### 4. Node.js 애플리케이션

```javascript
// app.js
const express = require('express');
const mysql = require('mysql2');

const app = express();

const connection = mysql.createConnection({
  host: 'mysql-db',  // 컨테이너 이름으로 접속
  user: 'appuser',
  password: 'apppass',
  database: 'myapp'
});

app.get('/', (req, res) => {
  connection.query('SELECT NOW() as time', (err, results) => {
    if (err) {
      res.status(500).json({ error: err.message });
    } else {
      res.json({ 
        message: 'Connected to MySQL!',
        time: results[0].time 
      });
    }
  });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### 5. 애플리케이션 실행

```bash
# 이미지 빌드
docker build -t webapp .

# 컨테이너 실행
docker run -d \
  --network webapp-network \
  --name webapp \
  -p 3000:3000 \
  webapp

# 테스트
curl http://localhost:3000
```

## 포트 매핑

### 포트 공개 및 매핑

```bash
# 단일 포트 매핑
docker run -d -p 8080:80 nginx

# 여러 포트 매핑
docker run -d -p 8080:80 -p 8443:443 nginx

# 호스트의 모든 인터페이스에 바인딩
docker run -d -p 0.0.0.0:8080:80 nginx

# 임의의 호스트 포트 사용
docker run -d -p 80 nginx

# UDP 포트 매핑
docker run -d -p 514:514/udp syslog-server
```

### 포트 확인

```bash
# 컨테이너의 포트 매핑 확인
docker port container-name

# 특정 포트 확인
docker port container-name 80
```

## 네트워크 보안

### 1. 네트워크 격리

```bash
# 프론트엔드 네트워크
docker network create frontend

# 백엔드 네트워크
docker network create backend

# 웹 서버는 두 네트워크에 연결
docker run -d --name web --network frontend nginx
docker network connect backend web

# DB는 백엔드 네트워크만
docker run -d --name db --network backend mysql:5.7
```

### 2. 네트워크 정책

```bash
# 서브넷 지정
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --ip-range 172.20.240.0/20 \
  secure-network
```

## 네트워크 문제 해결

### 연결 테스트

```bash
# 네트워크 연결 테스트
docker run --rm --network app-network alpine ping -c 3 my-container

# DNS 확인
docker run --rm --network app-network alpine nslookup my-container

# 포트 스캔
docker run --rm --network app-network alpine nc -zv my-container 80
```

### 네트워크 디버깅

```bash
# 컨테이너의 네트워크 정보 확인
docker exec container-name ip addr
docker exec container-name netstat -tulpn
docker exec container-name ss -tulpn

# 라우팅 테이블 확인
docker exec container-name route -n
```

## 실습: 3-Tier 애플리케이션

### 1. 네트워크 구성

```bash
# 프론트엔드 네트워크 (웹서버 - 앱서버)
docker network create frontend-net

# 백엔드 네트워크 (앱서버 - DB)
docker network create backend-net
```

### 2. 데이터베이스 계층

```bash
docker run -d \
  --name database \
  --network backend-net \
  -e POSTGRES_PASSWORD=secret \
  postgres:13
```

### 3. 애플리케이션 계층

```bash
docker run -d \
  --name app-server \
  --network backend-net \
  -e DATABASE_HOST=database \
  my-app:latest

# 프론트엔드 네트워크에도 연결
docker network connect frontend-net app-server
```

### 4. 웹 서버 계층

```bash
docker run -d \
  --name web-server \
  --network frontend-net \
  -p 80:80 \
  -e BACKEND_HOST=app-server \
  nginx:latest
```

## Docker 네트워크 베스트 프랙티스

1. **사용자 정의 네트워크 사용**: 기본 bridge보다 사용자 정의 네트워크 권장
2. **네트워크 격리**: 필요한 컨테이너만 같은 네트워크에 배치
3. **명시적 포트 매핑**: 필요한 포트만 외부에 노출
4. **DNS 이름 사용**: IP 주소 대신 컨테이너 이름 사용
5. **네트워크 정리**: 사용하지 않는 네트워크 정기적으로 정리

## 마무리

Docker 네트워크를 이해하면 복잡한 멀티 컨테이너 애플리케이션을 안전하고 효율적으로 구성할 수 있습니다. 다음 편에서는 컨테이너의 데이터를 영구적으로 저장하는 Docker 볼륨에 대해 알아보겠습니다.

## 다음 편 예고

- Docker 볼륨의 개념
- 볼륨 타입과 마운트
- 데이터 백업과 복원
- 볼륨 공유와 권한 관리

네트워크로 연결된 컨테이너들의 데이터를 안전하게 관리하는 방법을 배워봅시다!