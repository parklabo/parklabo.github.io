---
layout: post
title: "[Docker 입문 #1] Docker란 무엇인가?"
date: 2025-07-15 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, tutorial, series, docker-basics, virtualization, containerization, beginner]
mermaid: true
---

Docker는 개발자들 사이에서 가장 인기 있는 컨테이너 기술 중 하나입니다. 이번 시리즈에서는 Docker를 처음 접하는 분들을 위해 기초부터 차근차근 알아보겠습니다.

## Docker가 필요한 이유

개발을 하다 보면 이런 경험을 해보신 적이 있을 겁니다:

- "내 컴퓨터에서는 잘 돌아가는데 왜 서버에서는 안 되지?"
- "팀원 컴퓨터에서는 에러가 나는데 내 컴퓨터에서는 잘 돌아가요"
- "개발 환경 설정하는데 하루가 다 갔어요"

이런 문제들이 발생하는 이유는 각 컴퓨터마다 운영체제, 설치된 프로그램 버전, 환경 설정 등이 다르기 때문입니다. Docker는 바로 이런 문제를 해결하기 위해 만들어졌습니다.

## Docker란?

Docker는 **컨테이너 기반의 오픈소스 가상화 플랫폼**입니다. 쉽게 말해, 애플리케이션과 그 애플리케이션이 실행되는 데 필요한 모든 것(코드, 런타임, 시스템 도구, 라이브러리 등)을 하나의 패키지로 만들어주는 도구입니다.

```mermaid
flowchart LR
    A[애플리케이션 코드] --> D[Docker Container]
    B[라이브러리/의존성] --> D
    C[환경 설정] --> D
    D --> E[어디서든 실행 가능!]
    
    E --> F[개발자 노트북]
    E --> G[테스트 서버]
    E --> H[프로덕션 서버]
    E --> I[클라우드]
    
    style D fill:#bbf,stroke:#333,stroke-width:4px
    style E fill:#bfb,stroke:#333,stroke-width:2px
```

### 컨테이너(Container)란?

컨테이너는 애플리케이션을 실행하기 위한 격리된 환경입니다. 마치 화물 컨테이너처럼:

- **표준화**: 어디서든 같은 방식으로 실행됩니다
- **이식성**: 한 곳에서 다른 곳으로 쉽게 옮길 수 있습니다
- **격리성**: 다른 컨테이너나 호스트 시스템에 영향을 주지 않습니다

## Docker vs 가상머신(VM)

Docker를 이해하기 위해 가상머신과 비교해보겠습니다:

```mermaid
graph TB
    subgraph "가상머신 아키텍처"
        A[하드웨어] --> B[하이퍼바이저]
        B --> C[Guest OS 1]
        B --> D[Guest OS 2]
        B --> E[Guest OS 3]
        C --> F[App 1]
        D --> G[App 2]
        E --> H[App 3]
    end
    
    subgraph "Docker 아키텍처"
        I[하드웨어] --> J[Host OS]
        J --> K[Docker Engine]
        K --> L[Container 1<br/>App 1]
        K --> M[Container 2<br/>App 2]
        K --> N[Container 3<br/>App 3]
    end
```

### 상세 비교표

| 특징 | 가상머신 (VM) | Docker 컨테이너 |
|------|---------------|-----------------|
| **가상화 수준** | 하드웨어 수준 가상화 | OS 수준 가상화 |
| **Guest OS** | 각 VM마다 전체 OS 필요 | Host OS 커널 공유 |
| **시작 시간** | 수 분 | 수 초 |
| **크기** | 수 GB | 수십 MB |
| **성능** | 오버헤드 높음 | 네이티브에 가까운 성능 |
| **격리 수준** | 완전한 격리 | 프로세스 수준 격리 |
| **리소스 사용** | 높음 | 낮음 |
| **이식성** | 제한적 | 매우 높음 |

## Docker의 주요 구성 요소

```mermaid
graph LR
    A[Dockerfile] -->|build| B[Docker Image]
    B -->|run| C[Docker Container]
    B -->|push| D[Docker Hub/Registry]
    D -->|pull| B
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style B fill:#bbf,stroke:#333,stroke-width:2px
    style C fill:#bfb,stroke:#333,stroke-width:2px
    style D fill:#fbf,stroke:#333,stroke-width:2px
```

### Docker 구성 요소 설명

| 구성 요소 | 설명 | 주요 특징 |
|-----------|------|----------|
| **Docker Image** | 컨테이너 실행을 위한 읽기 전용 템플릿 | • 레이어 구조<br>• 재사용 가능<br>• 버전 관리 |
| **Docker Container** | 이미지를 실행한 인스턴스 | • 격리된 프로세스<br>• 상태 저장<br>• 생명주기 관리 |
| **Docker Hub** | 이미지 저장소 (클라우드) | • 공식 이미지<br>• 커뮤니티 이미지<br>• Private 저장소 |
| **Dockerfile** | 이미지 빌드 설정 파일 | • 텍스트 기반<br>• 버전 관리 가능<br>• 자동화 |

### Docker 이미지 레이어 구조

```mermaid
graph TD
    subgraph "Docker Image Layers"
        A[Base OS Layer] --> B[Dependencies Layer]
        B --> C[Application Layer]
        C --> D[Config Layer]
    end
    
    E[Container Layer<br/>읽기/쓰기 가능] -.-> D
    
    style A fill:#ffd,stroke:#333,stroke-width:2px
    style B fill:#dfd,stroke:#333,stroke-width:2px
    style C fill:#ddf,stroke:#333,stroke-width:2px
    style D fill:#fdd,stroke:#333,stroke-width:2px
    style E fill:#fff,stroke:#333,stroke-width:2px,stroke-dasharray: 5 5
```

## Docker의 장점

```mermaid
mindmap
  root((Docker 장점))
    환경
      일관된 환경
      개발/운영 동일
      환경 설정 자동화
    성능
      빠른 시작
      경량화
      효율적 리소스
    배포
      빠른 배포
      롤백 용이
      버전 관리
    확장성
      수평 확장
      마이크로서비스
      오케스트레이션
```

### Docker 도입 효과

| 영역 | 기존 방식 | Docker 사용 시 | 개선 효과 |
|------|-----------|----------------|-----------|
| **환경 구성** | 수동 설정, 문서화 필요 | Dockerfile로 자동화 | 80% 시간 단축 |
| **배포 시간** | 30분~1시간 | 1~5분 | 90% 단축 |
| **리소스 사용** | VM당 1~2GB RAM | 컨테이너당 100~200MB | 80% 절약 |
| **확장성** | 수동 서버 추가 | 자동 스케일링 | 즉시 대응 |
| **일관성** | 환경별 차이 발생 | 완전히 동일 | 100% 일관성 |

## Docker 사용 사례

```mermaid
graph TD
    subgraph "마이크로서비스 아키텍처"
        A[API Gateway] --> B[User Service]
        A --> C[Order Service]
        A --> D[Payment Service]
        B --> E[(User DB)]
        C --> F[(Order DB)]
        D --> G[(Payment DB)]
    end
    
    subgraph "CI/CD 파이프라인"
        H[Code Push] --> I[Build]
        I --> J[Test]
        J --> K[Deploy]
    end
    
    style A fill:#f96,stroke:#333,stroke-width:2px
    style H fill:#9f6,stroke:#333,stroke-width:2px
```

### 실제 활용 예시

| 사용 사례 | 설명 | 주요 이점 |
|-----------|------|----------|
| **마이크로서비스** | 각 서비스를 독립 컨테이너로 운영 | • 독립적 배포<br>• 기술 스택 자유<br>• 장애 격리 |
| **CI/CD** | 빌드/테스트/배포 자동화 | • 일관된 환경<br>• 빠른 피드백<br>• 자동화 |
| **개발 환경** | 로컬 개발 환경 표준화 | • 빠른 온보딩<br>• 환경 일치<br>• 쉬운 공유 |
| **테스트 환경** | 격리된 테스트 환경 제공 | • 병렬 테스트<br>• 깨끗한 환경<br>• 재현 가능 |

## 마무리

Docker는 현대 소프트웨어 개발에서 필수적인 도구가 되었습니다.
컨테이너 기술을 통해 "내 컴퓨터에서는 되는데?"라는 문제를 해결하고, 개발부터 배포까지의 과정을 더 효율적으로 만들어줍니다.

다음 편에서는 Docker를 직접 설치하고 환경을 설정하는 방법을 알아보겠습니다. Docker의 세계로 함께 떠나봅시다!

## 다음 편 예고

- Docker 설치 방법 (Windows, Mac, Linux)
- Docker Desktop 소개
- 기본 설정 및 확인
- 첫 번째 Docker 명령어 실행

이 시리즈를 통해 Docker를 자신있게 사용할 수 있게 되시길 바랍니다!