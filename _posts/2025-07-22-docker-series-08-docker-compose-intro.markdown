---
layout: post
title: "[Docker 입문 #8] Docker Compose 입문"
date: 2025-07-22 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, tutorial, series, docker-compose, yaml, orchestration, beginner]
mermaid: true
---

지금까지 개별 컨테이너를 다루는 방법을 배웠습니다. 하지만 실제 애플리케이션은 여러 컨테이너가 함께 동작해야 합니다. Docker Compose는 멀티 컨테이너 애플리케이션을 정의하고 실행하는 도구입니다.

## Docker Compose란?

Docker Compose는 YAML 파일을 사용해 여러 컨테이너로 구성된 애플리케이션을 정의하고 한 번에 실행할 수 있게 해주는 도구입니다.

### Docker Compose의 장점

1. **간단한 명령어**: 복잡한 docker run 명령어를 YAML로 관리
2. **일관성**: 개발, 테스트, 프로덕션 환경의 일관성 유지
3. **버전 관리**: 설정 파일을 Git으로 관리 가능
4. **쉬운 확장**: 서비스 추가/제거가 간편

## Docker Compose 설치 확인

Docker Desktop을 설치했다면 Docker Compose가 이미 포함되어 있습니다:

```bash
# Docker Compose 버전 확인
docker compose version

# 짧은 버전 표시 (2025년 최신)
docker compose version --short

# JSON 형식으로 출력
docker compose version --format json
```

## 첫 번째 docker-compose.yml

간단한 웹 애플리케이션을 만들어봅시다:

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
```

HTML 파일 생성:
```bash
mkdir html
echo "<h1>Hello Docker Compose!</h1>" > html/index.html
```

실행:
```bash
# 서비스 시작
docker compose up

# 백그라운드에서 실행
docker compose up -d

# 서비스 중지
docker compose down
```

## docker-compose.yml 구조

### 기본 구조

```yaml
version: '3.8'  # Compose 파일 버전

services:       # 서비스 정의
  service1:
    # 서비스1 설정
  service2:
    # 서비스2 설정

networks:       # 네트워크 정의 (선택사항)
  network1:

volumes:        # 볼륨 정의 (선택사항)
  volume1:
```

## 실습: WordPress 블로그 구축

### 1. docker-compose.yml 작성

```yaml
version: '3.8'

services:
  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
      - "8000:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html

volumes:
  db_data:
  wordpress_data:
```

### 2. 실행 및 관리

```bash
# 서비스 시작
docker compose up -d

# 실행 중인 서비스 확인
docker compose ps

# 로그 확인
docker compose logs
docker compose logs -f wordpress

# 특정 서비스만 재시작
docker compose restart wordpress

# 서비스 중지 및 삭제
docker compose down

# 볼륨까지 삭제
docker compose down -v
```

## 주요 설정 옵션

### 프로필 사용 (2025년 최신)

```yaml
services:
  web:
    image: nginx
    profiles: ["frontend"]
  
  api:
    image: myapp:latest
    profiles: ["backend"]
  
  debug:
    image: debug-tools
    profiles: ["debug"]
```

특정 프로필만 실행:
```bash
# frontend 프로필만 실행
docker compose --profile frontend up

# 여러 프로필 동시 실행
docker compose --profile frontend --profile debug up
```

### 이미지와 빌드

```yaml
services:
  # 이미지 사용
  web:
    image: nginx:latest
  
  # Dockerfile로 빌드
  app:
    build: .
  
  # 상세 빌드 설정
  custom:
    build:
      context: ./dir
      dockerfile: Dockerfile.prod
      args:
        NODE_VERSION: 16
```

### 포트 매핑

```yaml
services:
  web:
    ports:
      - "8080:80"        # HOST:CONTAINER
      - "443:443"
      - "3000-3005:3000-3005"  # 포트 범위
```

### 볼륨

```yaml
services:
  app:
    volumes:
      # 명명된 볼륨
      - app_data:/data
      # 바인드 마운트
      - ./src:/app/src
      # 읽기 전용
      - ./config:/app/config:ro

volumes:
  app_data:
```

### 환경 변수

```yaml
services:
  app:
    # 직접 정의
    environment:
      NODE_ENV: production
      DB_HOST: database
    
    # 파일에서 로드
    env_file:
      - .env
      - .env.production
```

### 네트워크

```yaml
services:
  frontend:
    networks:
      - frontend
  
  backend:
    networks:
      - frontend
      - backend
  
  database:
    networks:
      - backend

networks:
  frontend:
  backend:
```

### 의존성 관리

```yaml
services:
  web:
    depends_on:
      - db
      - redis
  
  db:
    image: postgres
  
  redis:
    image: redis
```

## 실습: 풀스택 애플리케이션

### 1. 프로젝트 구조

```
myapp/
├── docker-compose.yml
├── frontend/
│   ├── Dockerfile
│   └── src/
├── backend/
│   ├── Dockerfile
│   └── src/
└── nginx/
    └── default.conf
```

### 2. docker-compose.yml

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - frontend
      - backend
    networks:
      - app-network

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    volumes:
      - ./frontend/src:/app/src
    environment:
      - REACT_APP_API_URL=http://localhost/api
    networks:
      - app-network

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    volumes:
      - ./backend/src:/app/src
    environment:
      - DB_HOST=database
      - DB_USER=myuser
      - DB_PASS=mypass
      - DB_NAME=myapp
    depends_on:
      - database
    networks:
      - app-network

  database:
    image: postgres:13
    environment:
      - POSTGRES_USER=myuser
      - POSTGRES_PASSWORD=mypass
      - POSTGRES_DB=myapp
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - app-network

networks:
  app-network:

volumes:
  db_data:
```

### 3. Nginx 설정

```nginx
# nginx/default.conf
server {
    listen 80;

    location / {
        proxy_pass http://frontend:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api {
        proxy_pass http://backend:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## 환경별 설정

### 개발 환경

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  backend:
    build:
      target: development
    volumes:
      - ./backend:/app
    environment:
      - NODE_ENV=development
    command: npm run dev
```

### 프로덕션 환경

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  backend:
    build:
      target: production
    environment:
      - NODE_ENV=production
    restart: always
```

### 환경별 실행

```bash
# 개발 환경
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# 프로덕션 환경
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

## 유용한 명령어

### 서비스 관리

```bash
# 특정 서비스만 시작
docker compose up backend

# 서비스 스케일링
docker compose up -d --scale backend=3

# 서비스 일시정지/재개
docker compose pause
docker compose unpause

# 설정 검증
docker compose config

# 드라이 런 모드 (2025년 최신)
# 실제 작업을 수행하지 않고 확인
docker compose --dry-run up --build -d

# 서비스 대기 (2025년 최신)
# 서비스가 실행/헬스 상태가 될 때까지 대기
docker compose up --wait --wait-timeout 300

# Watch 모드로 파일 변경 감지 (2025년 최신)
docker compose up --watch
```

### 실행 및 로그

```bash
# 서비스에서 명령 실행
docker compose exec backend npm test

# 새 컨테이너에서 명령 실행
docker compose run backend npm install

# 서비스 포트로 명령 실행 (2025년 최신)
# 서비스에 정의된 포트를 호스트에 매핑
docker compose run --service-ports web python manage.py shell

# 실시간 로그
docker compose logs -f --tail=100

# 타임스탬프 표시 (2025년 최신)
docker compose logs --timestamps

# 컨테이너 차듨 (2025년 최신)
# 로그 제한을 위해 특정 서비스만 연결
docker compose up --attach web --no-attach db
```

### 정리

```bash
# 컨테이너만 중지
docker compose stop

# 컨테이너 삭제
docker compose rm

# 전체 정리 (네트워크, 볼륨 포함)
docker compose down --volumes --remove-orphans

# 실행 중인 프로젝트 목록 (2025년 최신)
docker compose ls

# 모든 프로젝트 표시 (중지된 것 포함)
docker compose ls --all

# JSON 형식으로 출력
docker compose ls --format json
```

## 실습: 마이크로서비스 아키텍처

```yaml
version: '3.8'

services:
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    environment:
      - USER_SERVICE_URL=http://user-service:3001
      - ORDER_SERVICE_URL=http://order-service:3002
      - PRODUCT_SERVICE_URL=http://product-service:3003

  user-service:
    build: ./services/user
    environment:
      - DB_HOST=user-db
      - REDIS_HOST=redis

  order-service:
    build: ./services/order
    environment:
      - DB_HOST=order-db
      - KAFKA_BROKER=kafka:9092

  product-service:
    build: ./services/product
    environment:
      - DB_HOST=product-db
      - CACHE_HOST=redis

  user-db:
    image: postgres:13
    environment:
      - POSTGRES_DB=users

  order-db:
    image: postgres:13
    environment:
      - POSTGRES_DB=orders

  product-db:
    image: postgres:13
    environment:
      - POSTGRES_DB=products

  redis:
    image: redis:alpine

  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
```

## 베스트 프랙티스

1. **버전 명시**: 이미지 태그를 명확히 지정
2. **환경 분리**: 개발/운영 환경 설정 분리
3. **.env 파일**: 민감한 정보는 환경 변수로 관리
4. **헬스체크**: 서비스 상태 모니터링 설정
5. **리소스 제한**: CPU/메모리 제한 설정
6. **프로필 활용**: 선택적 서비스 실행을 위한 프로필 사용
7. **Watch 모드**: 개발 시 파일 변경 자동 감지

```yaml
services:
  app:
    image: myapp:1.0.0  # 버전 명시
    env_file: .env      # 환경 변수
    healthcheck:        # 헬스체크
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:             # 리소스 제한
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    develop:            # Watch 모드 설정 (2025년 최신)
      watch:
        - action: rebuild
          path: ./src
        - action: sync
          path: ./static
          target: /app/static
```

## 마무리

Docker Compose를 사용하면 복잡한 멀티 컨테이너 애플리케이션도 쉽게 관리할 수 있습니다. YAML 파일 하나로 전체 애플리케이션 스택을 정의하고 버전 관리할 수 있다는 것이 큰 장점입니다. 다음 편에서는 실제 멀티 컨테이너 애플리케이션을 구축해보겠습니다.

## 다음 편 예고

- 실제 멀티 컨테이너 애플리케이션 구축
- 서비스 간 통신 패턴
- 로드 밸런싱과 스케일링
- 실전 프로젝트 예제

Docker Compose로 프로덕션 레벨의 애플리케이션을 만들어봅시다! 🚀