---
layout: post
title: "[Docker 입문 #7] Docker 볼륨과 데이터 관리"
date: 2025-07-21 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, tutorial, series, volumes, data-persistence, bind-mounts, storage, beginner]
mermaid: true
---

컨테이너는 기본적으로 임시적(ephemeral)입니다. 컨테이너가 삭제되면 내부의 데이터도 함께 사라집니다. Docker 볼륨을 사용하면 데이터를 영구적으로 저장하고 관리할 수 있습니다.

## 왜 볼륨이 필요한가?

### 컨테이너의 데이터 문제

```bash
# MySQL 컨테이너 실행
docker run -d --name mydb -e MYSQL_ROOT_PASSWORD=secret mysql:5.7

# 데이터베이스에 데이터 추가
docker exec -it mydb mysql -p
# CREATE DATABASE testdb;
# exit

# 컨테이너 삭제
docker rm -f mydb

# 새 컨테이너 실행
docker run -d --name mydb -e MYSQL_ROOT_PASSWORD=secret mysql:5.7

# 데이터가 사라졌습니다!
docker exec -it mydb mysql -p -e "SHOW DATABASES;"
```

## Docker 스토리지 옵션

Docker는 세 가지 데이터 저장 방식을 제공합니다:

1. **볼륨(Volumes)**: Docker가 관리하는 스토리지
2. **바인드 마운트(Bind Mounts)**: 호스트 파일시스템 직접 연결
3. **tmpfs 마운트**: 메모리에 임시 저장

## 볼륨(Volumes)

### 볼륨 생성 및 관리

```bash
# 볼륨 생성
docker volume create my-volume

# 볼륨 목록 확인
docker volume ls

# 볼륨 상세 정보
docker volume inspect my-volume

# 볼륨 삭제
docker volume rm my-volume

# 사용하지 않는 볼륨 모두 삭제
docker volume prune
```

### 볼륨 사용하기

```bash
# 볼륨을 마운트하여 컨테이너 실행
docker run -d \
  --name mysql-db \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:5.7

# 볼륨 위치 확인
docker volume inspect mysql-data
```

### 익명 볼륨

```bash
# 이름 없이 볼륨 생성 (익명 볼륨)
docker run -d \
  --name web \
  -v /usr/share/nginx/html \
  nginx

# 익명 볼륨 확인
docker inspect web | grep -A 10 Mounts
```

## 바인드 마운트(Bind Mounts)

호스트의 특정 디렉토리를 컨테이너에 마운트합니다:

```bash
# 현재 디렉토리를 마운트
docker run -d \
  --name web-dev \
  -v $(pwd)/html:/usr/share/nginx/html \
  -p 8080:80 \
  nginx

# Windows에서는
docker run -d \
  --name web-dev \
  -v ${PWD}/html:/usr/share/nginx/html \
  -p 8080:80 \
  nginx
```

### 읽기 전용 마운트

```bash
# 읽기 전용으로 마운트
docker run -d \
  --name web-ro \
  -v $(pwd)/config:/etc/nginx/conf.d:ro \
  nginx
```

## 실습: WordPress와 MySQL 데이터 영속성

### 1. 네트워크 생성

```bash
docker network create wordpress-net
```

### 2. MySQL with Volume

```bash
# MySQL 볼륨 생성
docker volume create mysql-data

# MySQL 실행
docker run -d \
  --name wordpress-db \
  --network wordpress-net \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wordpress \
  -e MYSQL_PASSWORD=wordpass \
  mysql:5.7
```

### 3. WordPress with Volume

```bash
# WordPress 볼륨 생성
docker volume create wordpress-data

# WordPress 실행
docker run -d \
  --name wordpress-app \
  --network wordpress-net \
  -v wordpress-data:/var/www/html \
  -p 8080:80 \
  -e WORDPRESS_DB_HOST=wordpress-db \
  -e WORDPRESS_DB_USER=wordpress \
  -e WORDPRESS_DB_PASSWORD=wordpass \
  -e WORDPRESS_DB_NAME=wordpress \
  wordpress:latest
```

### 4. 데이터 영속성 테스트

```bash
# WordPress 설정 후 컨테이너 삭제
docker rm -f wordpress-app wordpress-db

# 동일한 볼륨으로 다시 실행
docker run -d \
  --name wordpress-db \
  --network wordpress-net \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  mysql:5.7

docker run -d \
  --name wordpress-app \
  --network wordpress-net \
  -v wordpress-data:/var/www/html \
  -p 8080:80 \
  -e WORDPRESS_DB_HOST=wordpress-db \
  -e WORDPRESS_DB_USER=wordpress \
  -e WORDPRESS_DB_PASSWORD=wordpass \
  -e WORDPRESS_DB_NAME=wordpress \
  wordpress:latest

# 데이터가 유지됨!
```

## 볼륨 백업과 복원

### 볼륨 백업

```bash
# 볼륨을 tar 파일로 백업
docker run --rm \
  -v mysql-data:/data \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/mysql-backup.tar.gz -C /data .
```

### 볼륨 복원

```bash
# 새 볼륨 생성
docker volume create mysql-data-restored

# 백업에서 복원
docker run --rm \
  -v mysql-data-restored:/data \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/mysql-backup.tar.gz -C /data
```

## 볼륨 드라이버

### 로컬 드라이버 옵션

```bash
# 특정 디바이스 사용
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/mnt/data \
  --opt o=bind \
  my-local-volume

# NFS 볼륨
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/path/to/nfs/share \
  nfs-volume
```

## 실습: 개발 환경 구성

### 1. Node.js 개발 환경

```bash
# 프로젝트 디렉토리 생성
mkdir node-dev && cd node-dev

# package.json 생성
cat > package.json << EOF
{
  "name": "node-dev",
  "version": "1.0.0",
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.0"
  }
}
EOF

# app.js 생성
cat > app.js << EOF
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello from Docker Dev Environment!');
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF

# Dockerfile
cat > Dockerfile << EOF
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]
EOF
```

### 2. 개발 모드로 실행

```bash
# 소스 코드를 바인드 마운트
docker run -d \
  --name node-dev \
  -v $(pwd):/app \
  -v /app/node_modules \
  -p 3000:3000 \
  node:16-alpine \
  sh -c "cd /app && npm install && npm run dev"
```

## 볼륨 공유

### 여러 컨테이너에서 볼륨 공유

```bash
# 공유 볼륨 생성
docker volume create shared-data

# 첫 번째 컨테이너
docker run -d \
  --name writer \
  -v shared-data:/data \
  alpine \
  sh -c "while true; do date >> /data/log.txt; sleep 5; done"

# 두 번째 컨테이너
docker run -d \
  --name reader \
  -v shared-data:/data \
  alpine \
  sh -c "tail -f /data/log.txt"

# 로그 확인
docker logs reader
```

## tmpfs 마운트

메모리에 임시 데이터 저장:

```bash
# tmpfs 마운트 사용
docker run -d \
  --name tmp-test \
  --tmpfs /tmp:rw,size=100m \
  alpine \
  sh -c "dd if=/dev/zero of=/tmp/test bs=1M count=50 && sleep 1000"

# 메모리 사용량 확인
docker stats tmp-test
```

## 볼륨 베스트 프랙티스

### 1. 명명된 볼륨 사용

```bash
# 좋은 예: 명명된 볼륨
docker run -v mysql-data:/var/lib/mysql mysql

# 피해야 할 예: 익명 볼륨
docker run -v /var/lib/mysql mysql
```

### 2. 볼륨 라벨 사용

```bash
# 라벨로 볼륨 관리
docker volume create \
  --label project=myapp \
  --label env=production \
  myapp-data

# 라벨로 필터링
docker volume ls --filter label=project=myapp
```

### 3. 정기적인 백업

```bash
# 백업 스크립트 예시
#!/bin/bash
BACKUP_DIR="/backup/volumes"
DATE=$(date +%Y%m%d_%H%M%S)

for volume in $(docker volume ls -q); do
  docker run --rm \
    -v $volume:/data \
    -v $BACKUP_DIR:/backup \
    alpine \
    tar czf /backup/${volume}_${DATE}.tar.gz -C /data .
done
```

## 문제 해결

### 권한 문제

```bash
# 사용자 ID 확인
docker run --rm alpine id

# 권한 설정
docker run -d \
  --name web \
  -v $(pwd)/data:/data \
  --user $(id -u):$(id -g) \
  nginx
```

### 볼륨 정리

```bash
# 사용하지 않는 볼륨 확인
docker volume ls -f dangling=true

# 안전하게 정리
docker volume prune

# 강제 정리 (주의!)
docker volume prune -f
```

## 마무리

Docker 볼륨을 사용하면 컨테이너의 생명주기와 독립적으로 데이터를 관리할 수 있습니다. 데이터베이스, 파일 업로드, 설정 파일 등 영구 보존이 필요한 모든 데이터에 볼륨을 활용하세요. 다음 편에서는 여러 컨테이너를 쉽게 관리할 수 있는 Docker Compose에 대해 알아보겠습니다.

## 다음 편 예고

- Docker Compose 소개
- docker-compose.yml 작성법
- 멀티 컨테이너 오케스트레이션
- 환경별 설정 관리

복잡한 애플리케이션도 간단하게 관리하는 방법을 배워봅시다!