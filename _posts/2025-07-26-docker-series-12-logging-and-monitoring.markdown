---
layout: post
title: "[Docker 입문 #12] Docker 로그와 모니터링"
date: 2025-07-26 10:00:00 +0900
categories: [DevOps, Docker, Monitoring]
tags: [docker, container, tutorial, series, logging, monitoring, observability, beginner]
mermaid: true
---

프로덕션 환경에서 컨테이너를 운영하려면 로그 관리와 모니터링이 필수입니다. 이번 편에서는 Docker 환경에서 효과적으로 로그를 수집하고 모니터링하는 방법을 알아보겠습니다.

## Docker 로깅과 모니터링 개요

### 주요 구성 요소

| 구성 요소 | 역할 | 주요 도구 | 특징 |
|-----------|------|-----------|------|
| **로그 수집** | 컨테이너 로그 수집 | Fluentd, Logstash | 중앙 집중화 |
| **로그 저장** | 로그 데이터 보관 | Elasticsearch, S3 | 검색 가능 |
| **시각화** | 대시보드 제공 | Kibana, Grafana | 실시간 모니터링 |
| **메트릭 수집** | 성능 지표 수집 | Prometheus, cAdvisor | 시계열 데이터 |
| **알림** | 이상 감지 및 알림 | Alertmanager, PagerDuty | 자동 대응 |

## Docker 로깅 기초

### 컨테이너 로그 확인

```bash
# 로그 보기
docker logs container-name

# 실시간 로그 (tail -f와 유사)
docker logs -f container-name

# 마지막 N줄만 보기
docker logs --tail 50 container-name

# 타임스탬프 포함
docker logs -t container-name

# 특정 시간 이후 로그
docker logs --since 2024-01-01T00:00:00 container-name
docker logs --since 10m container-name
```

### 로그 드라이버

Docker는 다양한 로그 드라이버를 지원합니다:

```bash
# 기본 로그 드라이버 확인
docker info | grep "Logging Driver"

# 컨테이너별 로그 드라이버 설정
docker run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  nginx
```

## 로그 드라이버 종류

### Docker 로그 드라이버 비교

| 드라이버 | 용도 | 장점 | 단점 | 사용 시나리오 |
|----------|------|------|------|---------------|
| **json-file** | 기본 로컬 저장 | 간단, docker logs 지원 | 로컬만 가능 | 개발/테스트 |
| **syslog** | 시스템 로그 통합 | 표준 프로토콜 | 구조화 제한 | 기존 인프라 통합 |
| **journald** | systemd 통합 | 시스템 통합 | Linux 전용 | systemd 환경 |
| **fluentd** | 유연한 라우팅 | 다양한 출력 | 추가 설정 필요 | 대규모 운영 |
| **awslogs** | AWS CloudWatch | AWS 통합 | AWS 전용 | AWS 환경 |
| **gcplogs** | GCP Logging | GCP 통합 | GCP 전용 | GCP 환경 |

### 1. json-file (기본값)

```bash
# 로그 옵션 설정
docker run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=5 \
  --log-opt compress=true \
  nginx

# 로그 파일 위치
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

### 2. syslog

```bash
# syslog로 전송
docker run -d \
  --log-driver syslog \
  --log-opt syslog-address=tcp://192.168.1.100:514 \
  --log-opt syslog-facility=daemon \
  --log-opt tag="{{.Name}}/{{.ID}}" \
  nginx
```

### 3. journald

```bash
# systemd journal로 로깅
docker run -d \
  --log-driver journald \
  --log-opt tag="docker/{{.Name}}" \
  nginx

# journalctl로 확인
journalctl CONTAINER_NAME=nginx -f
```

## 중앙 집중식 로깅: ELK Stack

### 1. docker-compose.yml

```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
    environment:
      - "LS_JAVA_OPTS=-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - elk
    depends_on:
      - elasticsearch

volumes:
  es_data:

networks:
  elk:
```

### 2. Logstash 설정

```ruby
# logstash/pipeline/logstash.conf
input {
  tcp {
    port => 5000
    codec => json
  }
  udp {
    port => 5000
    codec => json
  }
}

filter {
  if [docker][name] {
    mutate {
      add_field => { "container_name" => "%{[docker][name]}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "docker-logs-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

### 3. 애플리케이션 로깅 설정

```yaml
# app-compose.yml
version: '3.8'

services:
  web:
    image: nginx:alpine
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://localhost:5000"
        tag: "{{.Name}}/{{.ID}}"
    ports:
      - "80:80"

  app:
    build: .
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://localhost:5000"
        tag: "{{.Name}}/{{.ID}}"
    environment:
      - LOG_LEVEL=info
```

## 메트릭 모니터링: Prometheus + Grafana

### 1. 모니터링 스택 구성

```yaml
# monitoring-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /dev/disk:/dev/disk:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
```

### 2. Prometheus 설정

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'docker'
    static_configs:
      - targets: ['172.17.0.1:9323']
```

### 3. Docker 메트릭 활성화

```json
// /etc/docker/daemon.json
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}
```

재시작:
```bash
sudo systemctl restart docker
```

## 애플리케이션 레벨 모니터링

### Node.js 애플리케이션 예시

```javascript
// app.js
const express = require('express');
const promClient = require('prom-client');

const app = express();
const register = new promClient.Registry();

// 기본 메트릭 수집
promClient.collectDefaultMetrics({ register });

// 커스텀 메트릭
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.5, 1, 2, 5]
});
register.registerMetric(httpRequestDuration);

// 미들웨어
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on('finish', () => {
    end({ 
      method: req.method, 
      route: req.route?.path || req.path,
      status_code: res.statusCode 
    });
  });
  next();
});

// 메트릭 엔드포인트
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// 애플리케이션 라우트
app.get('/', (req, res) => {
  res.json({ message: 'Hello World' });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

## 실시간 로그 분석: Fluentd

### 1. Fluentd 설정

```dockerfile
# fluentd/Dockerfile
FROM fluent/fluentd:v1.16-1

USER root
RUN gem install fluent-plugin-elasticsearch fluent-plugin-prometheus
USER fluent
```

### 2. Fluentd 구성

```
# fluentd/fluent.conf
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<filter docker.**>
  @type parser
  key_name log
  <parse>
    @type json
  </parse>
</filter>

<match docker.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix docker
  <buffer>
    @type file
    path /fluentd/log/docker.*.buffer
    flush_interval 10s
  </buffer>
</match>
```

### 3. Docker Compose 통합

```yaml
version: '3.8'

services:
  fluentd:
    build: ./fluentd
    volumes:
      - ./fluentd/fluent.conf:/fluentd/etc/fluent.conf
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    networks:
      - logging

  web:
    image: nginx
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: docker.nginx
    ports:
      - "80:80"
    networks:
      - logging

networks:
  logging:
```

## 모니터링 스택 아키텍처

```mermaid
graph TB
    A[Docker Containers] --> B[cAdvisor]
    A --> C[Node Exporter]
    B --> D[Prometheus]
    C --> D
    D --> E[Grafana]
    
    A --> F[Fluentd/Logstash]
    F --> G[Elasticsearch]
    G --> H[Kibana]
    
    D --> I[Alertmanager]
    I --> J[Slack/Email/PagerDuty]
```

## 로그 관리 베스트 프랙티스

### 로깅 레벨별 활용 가이드

| 레벨 | 용도 | 예시 | 보관 기간 |
|------|------|------|-----------|
| **DEBUG** | 개발 디버깅 | 변수 값, 함수 호출 | 1-3일 |
| **INFO** | 일반 정보 | 서비스 시작, 요청 처리 | 7-14일 |
| **WARN** | 경고 사항 | 리트라이, 성능 저하 | 30일 |
| **ERROR** | 오류 발생 | 예외, 실패 | 90일 |
| **FATAL** | 치명적 오류 | 서비스 중단 | 1년 |

### 1. 구조화된 로깅

```javascript
// 좋은 예: JSON 형식
const winston = require('winston');

const logger = winston.createLogger({
  format: winston.format.json(),
  transports: [
    new winston.transports.Console()
  ]
});

logger.info('User logged in', {
  userId: '12345',
  timestamp: new Date().toISOString(),
  ip: req.ip
});
```

### 2. 로그 레벨 활용

```yaml
# Docker Compose에서 로그 레벨 설정
services:
  app:
    environment:
      - LOG_LEVEL=info  # debug, info, warn, error
```

### 3. 로그 로테이션

```bash
# json-file 드라이버 로테이션
docker run -d \
  --log-opt max-size=10m \
  --log-opt max-file=5 \
  --log-opt compress=true \
  myapp
```

## 알림 설정

### Prometheus Alertmanager

```yaml
# alertmanager/config.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'webhook'

receivers:
- name: 'webhook'
  webhook_configs:
  - url: 'http://localhost:5001/webhook'
```

### 알림 규칙

```yaml
# prometheus/alerts.yml
groups:
- name: container_alerts
  rules:
  - alert: ContainerDown
    expr: up{job="cadvisor"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Container is down"
      
  - alert: HighMemoryUsage
    expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Container memory usage is above 90%"
```

## 디버깅과 트러블슈팅

### 1. 컨테이너 내부 로그

```bash
# 컨테이너 내부 접속
docker exec -it container-name sh

# 로그 파일 확인
tail -f /var/log/nginx/error.log

# 프로세스 확인
ps aux
top
```

### 2. Docker 이벤트 모니터링

```bash
# 실시간 이벤트
docker events

# 특정 이벤트 필터링
docker events --filter event=start --filter event=stop

# JSON 형식
docker events --format '{{json .}}'
```

### 3. 시스템 리소스 확인

```bash
# 컨테이너 리소스 사용량
docker stats

# 특정 컨테이너
docker stats container-name

# 시스템 정보
docker system df
docker system events
```

## 종합 모니터링 대시보드

### Grafana 대시보드 설정

```json
// grafana/provisioning/dashboards/docker.json
{
  "dashboard": {
    "title": "Docker Monitoring",
    "panels": [
      {
        "title": "Container CPU Usage",
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total[5m]) * 100"
          }
        ]
      },
      {
        "title": "Container Memory Usage",
        "targets": [
          {
            "expr": "container_memory_usage_bytes / container_spec_memory_limit_bytes * 100"
          }
        ]
      }
    ]
  }
}
```

## 2025년 최신 모니터링 도구

### 클라우드 네이티브 도구 비교

| 도구 | 유형 | 특징 | 장점 | 단점 | 가격 |
|------|------|------|------|------|------|
| **Datadog** | SaaS | 올인원 플랫폼 | 쉬운 통합 | 비용 | $15/호스트 |
| **New Relic** | SaaS | APM 중심 | 상세 분석 | 복잡도 | $99/월 |
| **Elastic Stack** | Self/Cloud | 로그 중심 | 강력한 검색 | 리소스 필요 | 무료/유료 |
| **Prometheus** | Open Source | 메트릭 중심 | 무료, 확장성 | 학습 곡선 | 무료 |
| **Loki** | Open Source | 로그 집계 | 가벼움 | 기능 제한 | 무료 |

### 통합 모니터링 구성

```mermaid
graph LR
    A[Application] --> B[OpenTelemetry]
    B --> C[Traces]
    B --> D[Metrics]
    B --> E[Logs]
    
    C --> F[Jaeger/Tempo]
    D --> G[Prometheus]
    E --> H[Loki]
    
    F --> I[Grafana]
    G --> I
    H --> I
```

## 마무리

효과적인 로깅과 모니터링은 안정적인 컨테이너 운영의 핵심입니다. 적절한 도구를 선택하고 구조화된 로그를 생성하며, 의미 있는 메트릭을 수집하여 문제를 빠르게 파악하고 해결할 수 있습니다. 

### 핵심 체크리스트
- [ ] 중앙 집중식 로그 수집 구성
- [ ] 실시간 메트릭 모니터링 설정
- [ ] 알림 규칙 정의
- [ ] 로그 보관 정책 수립
- [ ] 대시보드 구성

다음 편에서는 Docker로 Node.js 애플리케이션을 배포하는 실전 예제를 다루겠습니다.

## 다음 편 예고

- Node.js 애플리케이션 컨테이너화
- 개발/운영 환경 구성
- 성능 최적화
- 실전 배포 전략

모니터링이 잘 갖춰진 Node.js 애플리케이션을 만들어봅시다! 📊