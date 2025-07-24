---
layout: post
title: "[GitHub 입문 #11] GitHub Actions 입문: CI/CD 파이프라인 구축하기"
date: 2025-07-24 10:00:00 +0900
categories: [DevOps, GitHub]
tags: [github, actions, ci-cd, automation, devops, tutorial, series]
mermaid: true
---

## 들어가며

GitHub 입문 시리즈의 열한 번째 시간입니다. 이번에는 GitHub Actions를 활용하여 CI/CD 파이프라인을 구축하는 방법을 알아보겠습니다. GitHub Actions는 코드 빌드, 테스트, 배포를 자동화하는 강력한 도구입니다.

## 1. GitHub Actions란?

GitHub Actions는 GitHub에 내장된 CI/CD 플랫폼으로, 워크플로우를 자동화할 수 있습니다.

### 핵심 개념 비교

| 구성 요소 | 설명 | 특징 | 예시 |
|-----------|------|------|------|
| **Workflow** | 자동화된 프로세스 | `.github/workflows/` 디렉토리에 YAML 파일 | `ci.yml`, `deploy.yml` |
| **Job** | 워크플로우의 실행 단위 | 병렬 또는 순차 실행 가능 | `build`, `test`, `deploy` |
| **Step** | Job 내의 개별 작업 | Action 사용 또는 쉘 명령 실행 | `checkout`, `setup-node` |
| **Action** | 재사용 가능한 작업 단위 | 마켓플레이스에서 공유 | `actions/checkout@v4` |
| **Runner** | 워크플로우 실행 서버 | GitHub 호스팅 또는 자체 호스팅 | `ubuntu-latest`, `self-hosted` |
| **Event** | 워크플로우 트리거 | push, PR, schedule 등 | `on: push` |
| **Context** | 실행 환경 정보 | 변수와 시크릿 접근 | `${{ github.actor }}` |

### GitHub Actions 아키텍처

```mermaid
graph TB
    subgraph "GitHub Repository"
        A[Code Push] --> B[Event Trigger]
        C[Pull Request] --> B
        D[Schedule] --> B
        E[Manual] --> B
    end
    
    subgraph "GitHub Actions"
        B --> F[Workflow File]
        F --> G[Job Queue]
        G --> H{Runner Assignment}
    end
    
    subgraph "Runners"
        H --> I[GitHub-hosted Runner]
        H --> J[Self-hosted Runner]
        
        I --> K[Ubuntu]
        I --> L[Windows]
        I --> M[macOS]
    end
    
    subgraph "Execution"
        K --> N[Environment Setup]
        L --> N
        M --> N
        J --> N
        
        N --> O[Checkout Code]
        O --> P[Run Steps]
        P --> Q[Upload Artifacts]
        Q --> R[Report Status]
    end
    
    R --> S[GitHub UI]
    R --> T[Status Checks]
    R --> U[Notifications]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style B fill:#bbf,stroke:#333,stroke-width:2px
    style F fill:#bfb,stroke:#333,stroke-width:2px
    style R fill:#fbf,stroke:#333,stroke-width:2px
```

### 최신 기능 (2025년 7월 기준)

| 기능 | 설명 | 사용 예시 |
|------|------|----------|
| **GPU Runners** | GPU 지원 러너 | NVIDIA T4, A10G 지원 |
| **ARM64 Runners** | 네이티브 ARM 러너 | Graviton3 기반 |
| **Copilot Integration** | AI 기반 워크플로우 생성 | 자연어로 설명하면 YAML 생성 |
| **Merge Queue** | 병합 대기열 자동화 | 순차적 PR 병합 |
| **Repository Rules** | 고급 브랜치 보호 | 커밋 서명, 파일 패턴 제한 |
| **Workflow Delegation** | 워크플로우 권한 위임 | 팀별 배포 권한 |
| **Actions Importer** | 다른 CI/CD 마이그레이션 | Jenkins, GitLab CI 변환 |

## 2. 첫 번째 워크플로우 만들기

### Hello World 워크플로우

`.github/workflows/hello-world.yml`:

```yaml
name: Hello World Workflow

# 워크플로우 트리거
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:  # 수동 실행 허용

# 작업 정의
jobs:
  hello:
    runs-on: ubuntu-latest
    
    steps:
      - name: Say Hello
        run: echo "Hello, GitHub Actions!"
        
      - name: Display date
        run: date
        
      - name: Multiline script
        run: |
          echo "This is line 1"
          echo "This is line 2"
          echo "Current directory: $(pwd)"
```

### 워크플로우 트리거 이벤트 완전 가이드

| 이벤트 타입 | 설명 | 사용 시나리오 | 예시 |
|------------|------|---------------|------|
| **push** | 코드 푸시 시 | CI 빌드, 테스트 | 브랜치, 태그, 경로 필터링 |
| **pull_request** | PR 생성/업데이트 | 코드 리뷰, 테스트 | opened, synchronize, closed |
| **schedule** | 정기 실행 | 정기 백업, 보안 스캔 | cron 표현식 |
| **workflow_dispatch** | 수동 실행 | 수동 배포, 긴급 작업 | 입력 파라미터 지원 |
| **release** | 릴리스 이벤트 | 배포, 문서 생성 | created, published |
| **issues** | 이슈 이벤트 | 자동 라벨링, 알림 | opened, labeled, closed |
| **discussion** | 디스커션 이벤트 | 커뮤니티 관리 | created, answered |
| **registry_package** | 패키지 이벤트 | 패키지 스캔, 배포 | published, updated |

### 고급 트리거 패턴

```yaml
# 복잡한 경로 필터링
on:
  push:
    branches:
      - main
      - 'releases/**'
    paths:
      - 'src/**'
      - 'package.json'
      - '!src/**/*.test.js'  # 테스트 파일 제외
    paths-ignore:
      - 'docs/**'
      - '**.md'

# PR 라벨 기반 트리거
on:
  pull_request:
    types: [labeled]

jobs:
  deploy-preview:
    if: github.event.label.name == 'deploy-preview'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying preview for PR #${{ github.event.number }}"

# 여러 워크플로우 연계
on:
  workflow_run:
    workflows: ["CI Pipeline"]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "CI passed, deploying..."
```

### 워크플로우 실행 흐름

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant WF as Workflow
    participant Runner as Runner
    participant Job as Job
    
    Dev->>GH: Push Code
    GH->>GH: Detect Event
    GH->>WF: Trigger Workflow
    WF->>Runner: Request Runner
    Runner->>Runner: Setup Environment
    Runner->>Job: Execute Job 1
    
    loop For Each Step
        Job->>Job: Run Step
        Job->>GH: Report Progress
    end
    
    Job->>Runner: Job Complete
    Runner->>WF: Report Status
    WF->>GH: Update PR/Commit Status
    GH->>Dev: Send Notification
    
    Note over Dev,GH: Parallel jobs run simultaneously
```

## 3. 실전 CI 파이프라인 구축

### 프로그래밍 언어별 CI 템플릿

| 언어/프레임워크 | 주요 도구 | 테스트 | 빌드 | 배포 대상 |
|----------------|----------|--------|------|----------|
| **Node.js** | npm/yarn/pnpm | Jest, Mocha | webpack, Vite | Vercel, Netlify |
| **Python** | pip/poetry | pytest, unittest | setuptools | PyPI, Heroku |
| **Java** | Maven/Gradle | JUnit, TestNG | jar/war | Maven Central |
| **Go** | go mod | go test | go build | Docker Hub |
| **Rust** | Cargo | cargo test | cargo build | crates.io |
| **.NET** | NuGet | xUnit, NUnit | dotnet build | NuGet, Azure |

### 포괄적 CI 파이프라인

`.github/workflows/comprehensive-ci.yml`:

{% raw %}
```yaml
name: Comprehensive CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  NODE_ENV: test
  CI: true

jobs:
  # 코드 품질 검사
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # SonarCloud 분석을 위해
      
      - name: Code Quality Analysis
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      
      - name: Check code formatting
        run: |
          npm ci
          npm run format:check
      
      - name: Security audit
        run: |
          npm audit --audit-level=moderate
          npx snyk test

  # 멀티 플랫폼 테스트
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18.x, 20.x, 21.x]
        exclude:
          - os: windows-latest
            node-version: 21.x  # Windows에서 불안정
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run linter
        run: npm run lint
        
      - name: Type checking
        run: npm run type-check
        
      - name: Run unit tests
        run: npm test -- --coverage
        
      - name: Run integration tests
        run: npm run test:integration
        
      - name: Run E2E tests
        if: matrix.os == 'ubuntu-latest'  # E2E는 Linux에서만
        run: |
          npx playwright install --with-deps
          npm run test:e2e
        
      - name: Upload coverage
        if: matrix.os == 'ubuntu-latest' && matrix.node-version == '20.x'
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          file: ./coverage/lcov.info
          flags: unittests
          name: codecov-umbrella

  # 빌드 및 번들 분석
  build:
    needs: [quality, test]
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Build project
        run: |
          npm run build
          npm run build:analyze > bundle-stats.json
        
      - name: Check bundle size
        uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          skip_step: install
        
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-${{ github.sha }}
          path: |
            dist/
            bundle-stats.json
          retention-days: 30
          
      - name: Upload to CDN (Preview)
        if: github.event_name == 'pull_request'
        run: |
          echo "Uploading to preview CDN..."
          # CDN upload logic here

  # 성능 테스트
  performance:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-${{ github.sha }}
          
      - name: Lighthouse CI
        uses: treosh/lighthouse-ci-action@v9
        with:
          urls: |
            http://localhost:3000
            http://localhost:3000/about
          uploadArtifacts: true
          temporaryPublicStorage: true
```
{% endraw %}

### CI 파이프라인 시각화

```mermaid
graph LR
    subgraph "Quality Gates"
        A[Code Push] --> B[Code Quality]
        B --> C[Security Scan]
        C --> D[Format Check]
    end
    
    subgraph "Testing Matrix"
        D --> E[Linux Tests]
        D --> F[Windows Tests]
        D --> G[macOS Tests]
        
        E --> H[Unit Tests]
        F --> H
        G --> H
        
        H --> I[Integration Tests]
        I --> J[E2E Tests]
    end
    
    subgraph "Build & Analysis"
        J --> K[Build App]
        K --> L[Bundle Analysis]
        L --> M[Size Check]
        M --> N[Performance Test]
    end
    
    subgraph "Deployment"
        N --> O{Branch?}
        O -->|main| P[Production]
        O -->|develop| Q[Staging]
        O -->|PR| R[Preview]
    end
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style P fill:#9f9,stroke:#333,stroke-width:2px
    style Q fill:#ff9,stroke:#333,stroke-width:2px
    style R fill:#9ff,stroke:#333,stroke-width:2px
```

### 언어별 최적화된 CI 예시

#### Python 프로젝트 (Poetry + 고급 기능)

`.github/workflows/python-advanced-ci.yml`:

{% raw %}
```yaml
name: Python Advanced CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.9', '3.10', '3.11', '3.12']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Poetry
        uses: snok/install-poetry@v1
        with:
          version: 1.7.1
          virtualenvs-create: true
          virtualenvs-in-project: true
          
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'poetry'
          
      - name: Install dependencies
        run: poetry install --no-interaction --no-root
        
      - name: Install project
        run: poetry install --no-interaction
        
      - name: Run pre-commit hooks
        run: |
          poetry run pre-commit install
          poetry run pre-commit run --all-files
          
      - name: Type checking
        run: |
          poetry run mypy src/ --strict
          poetry run pyright src/
          
      - name: Linting
        run: |
          poetry run ruff check src/ tests/
          poetry run black --check src/ tests/
          poetry run isort --check-only src/ tests/
          
      - name: Security scanning
        run: |
          poetry run bandit -r src/
          poetry run safety check
          poetry run pip-audit
          
      - name: Run tests with coverage
        run: |
          poetry run pytest tests/ \
            --cov=src \
            --cov-report=xml \
            --cov-report=html \
            --cov-report=term-missing \
            --benchmark-disable
          
      - name: Run benchmarks
        if: matrix.python-version == '3.11' && matrix.os == 'ubuntu-latest'
        run: |
          poetry run pytest tests/ \
            --benchmark-only \
            --benchmark-json=benchmark.json
          
      - name: Upload coverage
        if: matrix.python-version == '3.11' && matrix.os == 'ubuntu-latest'
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          flags: unittests
          
      - name: Build package
        run: poetry build
        
      - name: Check package
        run: |
          poetry run twine check dist/*
          poetry run check-wheel-contents dist/*.whl
```
{% endraw %}

#### Java/Spring Boot 프로젝트

`.github/workflows/java-springboot-ci.yml`:

{% raw %}
```yaml
name: Java Spring Boot CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
          
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Shallow clones disabled for SonarQube
          
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
          
      - name: Cache SonarQube packages
        uses: actions/cache@v3
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar
          
      - name: Run tests
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: postgres
          SPRING_DATASOURCE_PASSWORD: postgres
          SPRING_REDIS_HOST: localhost
          SPRING_REDIS_PORT: 6379
        run: |
          ./mvnw clean verify
          ./mvnw jacoco:report
          
      - name: SonarQube analysis
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          ./mvnw sonar:sonar \
            -Dsonar.projectKey=myproject \
            -Dsonar.host.url=https://sonarcloud.io \
            -Dsonar.organization=myorg
            
      - name: Build Docker image
        run: |
          ./mvnw spring-boot:build-image \
            -Dspring-boot.build-image.imageName=${{ github.repository }}:${{ github.sha }}
```
{% endraw %}

## 4. CD 파이프라인 구축

### 클라우드 플랫폼별 배포 전략

| 플랫폼 | 배포 방식 | 장점 | 사용 사례 |
|--------|-----------|------|----------|
| **AWS** | ECS, Lambda, S3 | 완전한 제어, 확장성 | 엔터프라이즈 앱 |
| **Azure** | App Service, AKS | MS 생태계 통합 | .NET 애플리케이션 |
| **GCP** | Cloud Run, GKE | 빠른 시작, AI/ML | 컨테이너 기반 앱 |
| **Vercel** | 자동 배포 | 간편함, 빠른 속도 | Next.js, 정적 사이트 |
| **Netlify** | Git 기반 배포 | 간편한 설정 | JAMstack 앱 |
| **Heroku** | Git push 배포 | 개발자 친화적 | 프로토타입, MVP |

### 모던 배포 워크플로우

`.github/workflows/deploy-aws.yml`:

{% raw %}
```yaml
name: Deploy to AWS

on:
  push:
    branches: [ main ]
  workflow_dispatch:

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: my-app
  ECS_SERVICE: my-app-service
  ECS_CLUSTER: my-cluster
  ECS_TASK_DEFINITION: task-definition.json

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
          
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
        
      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
          
      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ${{ env.ECS_TASK_DEFINITION }}
          container-name: my-app
          image: ${{ steps.build-image.outputs.image }}
          
      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```
{% endraw %}

### GitHub Pages 배포

`.github/workflows/deploy-pages.yml`:

{% raw %}
```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [ main ]

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Build
        run: npm run build
        env:
          PUBLIC_URL: /${{ github.event.repository.name }}
          
      - name: Setup Pages
        uses: actions/configure-pages@v3
        
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v2
        with:
          path: ./dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v2
```
{% endraw %}

## 5. 재사용 가능한 워크플로우

### Reusable Workflow 정의

`.github/workflows/reusable-test.yml`:

{% raw %}
```yaml
name: Reusable Test Workflow

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
      environment:
        required: false
        type: string
        default: 'test'
    secrets:
      npm-token:
        required: true

jobs:
  test:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          registry-url: 'https://registry.npmjs.org'
          
      - name: Install dependencies
        run: npm ci
        env:
          NODE_AUTH_TOKEN: ${{ secrets.npm-token }}
          
      - name: Run tests
        run: npm test
```
{% endraw %}

### Reusable Workflow 사용

`.github/workflows/ci.yml`:

{% raw %}
```yaml
name: CI

on: [push, pull_request]

jobs:
  test-node-16:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '16.x'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
      
  test-node-18:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '18.x'
      environment: 'staging'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```
{% endraw %}

## 6. 환경 변수와 시크릿

### 환경 변수 사용

{% raw %}
```yaml
name: Environment Variables

env:
  # 워크플로우 레벨 환경 변수
  GLOBAL_VAR: 'workflow-level'
  NODE_ENV: 'production'

jobs:
  example:
    runs-on: ubuntu-latest
    env:
      # Job 레벨 환경 변수
      JOB_VAR: 'job-level'
      
    steps:
      - name: Use environment variables
        env:
          # Step 레벨 환경 변수
          STEP_VAR: 'step-level'
        run: |
          echo "Global: $GLOBAL_VAR"
          echo "Job: $JOB_VAR"
          echo "Step: $STEP_VAR"
          echo "GitHub: ${{ github.repository }}"
          
      - name: Set dynamic environment variable
        run: echo "DYNAMIC_VAR=value-$(date +%s)" >> $GITHUB_ENV
        
      - name: Use dynamic variable
        run: echo "Dynamic: $DYNAMIC_VAR"
```
{% endraw %}

### 시크릿 관리

{% raw %}
```yaml
name: Using Secrets

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
          API_TOKEN: ${{ secrets.API_TOKEN }}
        run: |
          # 시크릿은 로그에 마스킹됨
          echo "Deploying with masked secrets"
          
      - name: Create secret file
        run: |
          echo "${{ secrets.CONFIG_FILE }}" > config.json
          # 파일 내용 확인 (시크릿은 마스킹됨)
          cat config.json
          
      - name: Conditional secret usage
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        env:
          PROD_SECRET: ${{ secrets.PRODUCTION_KEY }}
        run: |
          echo "Using production secret"
```
{% endraw %}

## 7. 매트릭스 전략과 병렬 처리

### 복잡한 매트릭스 구성

{% raw %}
```yaml
name: Matrix Build

on: [push]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false  # 하나 실패해도 나머지 계속 실행
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
        include:
          # 특정 조합 추가
          - os: ubuntu-latest
            node: 21
            experimental: true
        exclude:
          # 특정 조합 제외
          - os: windows-latest
            node: 16
            
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          
      - name: Install and test
        run: |
          npm ci
          npm test
        continue-on-error: ${{ matrix.experimental == true }}
```
{% endraw %}

### 동적 매트릭스 생성

{% raw %}
```yaml
name: Dynamic Matrix

on: [push]

jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v4
      
      - id: set-matrix
        run: |
          # 동적으로 매트릭스 생성
          MATRIX=$(cat <<EOF
          {
            "include": [
              {"project": "app1", "node": "16"},
              {"project": "app2", "node": "18"},
              {"project": "app3", "node": "20"}
            ]
          }
          EOF
          )
          echo "matrix=$(echo $MATRIX | jq -c .)" >> $GITHUB_OUTPUT

  test:
    needs: setup
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJson(needs.setup.outputs.matrix) }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Test ${{ matrix.project }}
        run: |
          cd ${{ matrix.project }}
          npm test
```
{% endraw %}

## 8. 조건부 실행과 제어 흐름

### 조건문 활용

{% raw %}
```yaml
name: Conditional Execution

on:
  push:
  pull_request:
  workflow_dispatch:

jobs:
  check:
    runs-on: ubuntu-latest
    outputs:
      should-deploy: ${{ steps.check-deploy.outputs.should-deploy }}
    steps:
      - name: Check if should deploy
        id: check-deploy
        run: |
          if [[ "${{ github.event_name }}" == "push" && "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "should-deploy=true" >> $GITHUB_OUTPUT
          else
            echo "should-deploy=false" >> $GITHUB_OUTPUT
          fi

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run tests
        run: npm test
        
      - name: Upload coverage (PR only)
        if: github.event_name == 'pull_request'
        uses: codecov/codecov-action@v3

  deploy:
    needs: [check, test]
    if: needs.check.outputs.should-deploy == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: echo "Deploying to production"
```
{% endraw %}

### 에러 처리와 재시도

{% raw %}
```yaml
name: Error Handling

on: [push]

jobs:
  resilient-job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Flaky test with retry
        uses: nick-fields/retry@v2
        with:
          timeout_minutes: 10
          max_attempts: 3
          retry_wait_seconds: 30
          command: npm test
          
      - name: Continue on error
        continue-on-error: true
        run: |
          # This might fail but won't stop the workflow
          npm run optional-task
          
      - name: Conditional failure
        run: |
          if [[ "${{ failure() }}" == "true" ]]; then
            echo "Previous step failed, handling error"
          fi
          
      - name: Always run cleanup
        if: always()
        run: |
          echo "Cleaning up resources"
          rm -rf temp/
```
{% endraw %}

## 9. 캐싱과 아티팩트

### 효과적인 캐싱 전략

{% raw %}
```yaml
name: Caching Strategy

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Node.js 의존성 캐싱
      - name: Cache node modules
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
            
      # 빌드 결과 캐싱
      - name: Cache build
        uses: actions/cache@v3
        with:
          path: |
            dist
            .next/cache
          key: ${{ runner.os }}-build-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-build-
            
      # 커스텀 캐시 관리
      - name: Get cache metadata
        id: cache-metadata
        run: |
          CACHE_KEY="custom-cache-$(date +%Y%m%d)"
          echo "cache-key=$CACHE_KEY" >> $GITHUB_OUTPUT
          
      - name: Custom cache
        uses: actions/cache@v3
        with:
          path: .cache
          key: ${{ steps.cache-metadata.outputs.cache-key }}
```
{% endraw %}

### 아티팩트 활용

{% raw %}
```yaml
name: Artifact Management

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build application
        run: |
          npm ci
          npm run build
          
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-${{ github.sha }}
          path: |
            dist/
            !dist/**/*.map
          retention-days: 30
          
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: test-results/
          
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-${{ github.sha }}
          path: dist/
          
      - name: Deploy artifacts
        run: |
          ls -la dist/
          # Deploy logic here
```
{% endraw %}

## 10. 모니터링과 알림

### 알림 채널 통합

| 채널 | 용도 | 설정 난이도 | 특징 |
|------|------|-------------|------|
| **Slack** | 팀 협업 알림 | 쉬움 | Webhook, 리치 메시지 |
| **Discord** | 커뮤니티 알림 | 쉬움 | Webhook, 임베드 |
| **Email** | 공식 알림 | 쉬움 | 개인별 설정 |
| **Teams** | 기업 협업 | 보통 | Office 365 통합 |
| **Telegram** | 실시간 알림 | 보통 | Bot API |
| **Jira** | 이슈 트래킹 | 어려움 | 자동 이슈 생성 |

### 통합 알림 워크플로우

{% raw %}
```yaml
name: Deployment with Notifications

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy application
        id: deploy
        run: |
          # Deployment logic
          echo "deploy-url=https://app.example.com" >> $GITHUB_OUTPUT
          
      - name: Notify Slack on Success
        if: success()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "Deployment Successful! 🎉",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Successful!* 🎉\n*Repository:* ${{ github.repository }}\n*Branch:* ${{ github.ref_name }}\n*Deployed by:* ${{ github.actor }}\n*URL:* ${{ steps.deploy.outputs.deploy-url }}"
                  }
                }
              ]
            }
            
      - name: Notify Slack on Failure
        if: failure()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "Deployment Failed! ❌",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Failed!* ❌\n*Repository:* ${{ github.repository }}\n*Branch:* ${{ github.ref_name }}\n*Failed by:* ${{ github.actor }}\n*Job URL:* ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                  }
                }
              ]
            }
```
{% endraw %}

### 상태 배지 추가

README.md에 추가:

```markdown
# My Project

![CI Status](https://github.com/username/repo/workflows/CI/badge.svg)
![Deploy Status](https://github.com/username/repo/workflows/Deploy/badge.svg?branch=main)
![Coverage](https://codecov.io/gh/username/repo/branch/main/graph/badge.svg)
![License](https://img.shields.io/github/license/username/repo)

## Custom Badge

![Custom](https://img.shields.io/endpoint?url=https://your-api.com/badge-data)
```

## 11. GitHub Actions 비용 최적화

### 무료 티어 및 요금제 (2025년 7월 기준)

| 플랜 | 무료 시간/월 | 스토리지 | 동시 실행 | GPU 시간 | 추가 비용 |
|------|-------------|----------|-----------|----------|----------|
| **Free** | 2,000분 (Linux) | 500MB | 20개 | 0분 | 불가 |
| **Pro** | 3,000분 | 2GB | 40개 | 0분 | $0.008/분 |
| **Team** | 10,000분 | 4GB | 100개 | 500분 | $0.008/분 |
| **Enterprise** | 50,000분 | 100GB | 1000개 | 5000분 | $0.004/분 |

### 비용 절감 전략

```yaml
name: Cost-Optimized Workflow 2025

on:
  push:
    branches: [ main ]
  merge_group:  # Merge Queue 사용

jobs:
  # 1. 스마트 체인지 감지
  smart-trigger:
    runs-on: ubuntu-latest
    outputs:
      skip: ${{ steps.check.outputs.skip }}
      strategy: ${{ steps.check.outputs.strategy }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
          
      - id: check
        uses: github/skip-duplicate-actions@v2
        with:
          concurrent_skipping: 'same_content_newer'
          
      - name: AI 기반 워크플로우 최적화
        uses: github/copilot-actions@v1
        with:
          task: optimize-workflow
          context: ${{ toJson(github) }}
  
  # 2. 동적 러너 선택
  build:
    needs: smart-trigger
    if: needs.smart-trigger.outputs.skip != 'true'
    runs-on: ${{ fromJSON(needs.smart-trigger.outputs.strategy).runner }}
    steps:
      - uses: actions/checkout@v4
      
      - name: 캐시 최적화
        uses: actions/cache@v4
        with:
          key: ${{ runner.os }}-build-${{ hashFiles('**/lockfiles') }}
          restore-keys: |
            ${{ runner.os }}-build-
          enableCrossOsArchive: true  # 2025 신기능
      
      - name: 빌드 및 테스트
        run: |
          npm ci --cache .npm
          npm run build
          
  # 3. GPU 러너 활용 (ML/AI 작업)
  ml-tasks:
    if: contains(github.event.head_commit.message, '[ML]')
    runs-on: [gpu, nvidia-t4]  # GPU 러너
    steps:
      - name: 모델 학습
        run: python train.py
```

## 12. 베스트 프랙티스 체크리스트

### 보안 (2025 강화)
- [ ] 시크릿을 코드에 하드코딩하지 않기
- [ ] 최소 권한 원칙 적용 (permissions 명시)
- [ ] OIDC 사용으로 장기 자격 증명 제거
- [ ] Dependabot + Copilot 보안 자동화
- [ ] Advanced Security 활성화 (CodeQL, Secret Scanning, Push Protection)
- [ ] SLSA Level 3 준수
- [ ] Sigstore/Cosign으로 워크플로우 서명

### 성능
- [ ] 병렬 작업 최대한 활용
- [ ] 캐싱 전략 최적화
- [ ] 불필요한 단계 제거
- [ ] 조건부 실행 활용
- [ ] 아티팩트 보관 기간 최적화

### 유지보수
- [ ] 재사용 가능한 워크플로우 작성
- [ ] 명확한 이름과 설명 사용
- [ ] 버전 태그 사용 (actions/checkout@v4)
- [ ] 실패 시 디버깅 정보 수집
- [ ] 정기적인 워크플로우 검토

## 13. 트러블슈팅 가이드

### 일반적인 문제와 해결법

| 문제 | 원인 | 해결 방법 (2025) |
|------|------|----------------|
| **권한 오류** | 토큰 권한 부족 | Fine-grained PAT 사용 |
| **캐시 미스** | 키 변경 | 크로스 OS 캐시 활성화 |
| **느린 빌드** | 러너 할당 대기 | Larger runners 사용 |
| **비용 초과** | 과도한 사용 | Copilot으로 최적화 |
| **병합 충돌** | 동시 PR | Merge Queue 사용 |
| **서명 실패** | 인증 문제 | Sigstore OIDC 사용 |

### 디버깅 팁

```yaml
# 디버깅을 위한 유용한 스텝들
steps:
  - name: Dump GitHub context
    env:
      GITHUB_CONTEXT: ${{ toJson(github) }}
    run: echo "$GITHUB_CONTEXT"
    
  - name: Dump job context
    env:
      JOB_CONTEXT: ${{ toJson(job) }}
    run: echo "$JOB_CONTEXT"
    
  - name: Dump runner context
    env:
      RUNNER_CONTEXT: ${{ toJson(runner) }}
    run: echo "$RUNNER_CONTEXT"
    
  - name: Debug with tmate
    if: ${{ failure() }}
    uses: mxschmitt/action-tmate@v3
    timeout-minutes: 15
```

## 마무리

GitHub Actions는 2025년 7월 현재 CI/CD 플랫폼의 새로운 표준으로 자리잡았습니다. GPU 러너, AI 기반 최적화, Merge Queue 등 혁신적인 기능들이 추가되어 개발 워크플로우를 한층 강화했습니다.

### 핵심 포인트 (2025년 7월)
- 🤖 **AI 통합**: Copilot으로 워크플로우 자동 생성
- 🎮 **GPU 러너**: ML/AI 워크플로우 지원
- 🔄 **Merge Queue**: 안전한 순차 병합
- 💲 **비용 최적화**: 스마트 트리거링
- 🔐 **강화된 보안**: SLSA, Sigstore 통합
- 🌍 **크로스 플랫폼**: ARM64 네이티브 지원

### 시작하기 위한 단계
1. 간단한 Hello World 워크플로우 생성
2. 프로젝트에 맞는 CI 파이프라인 구축
3. 캐싱과 병렬 처리로 최적화
4. CD 파이프라인으로 자동 배포
5. 모니터링과 알림 설정

다음 편에서는 GitHub Actions의 2025년 최신 기능들과 AI 기반 CI/CD 파이프라인 구축 방법을 더 자세히 알아보겠습니다.

## 시리즈 목차
1. [GitHub 시작하기](/posts/github-series-01-getting-started/)
2. [Repository 기초](/posts/github-series-02-repository-basics/)
3. [Git 기본 명령어](/posts/github-series-03-git-basic-commands/)
4. [Branch와 Merge](/posts/github-series-04-branch-and-merge/)
5. [Fork와 Pull Request](/posts/github-series-05-fork-and-pull-request/)
6. [Issues 활용법](/posts/github-series-06-issues-management/)
7. [Projects로 프로젝트 관리](/posts/github-series-07-projects-kanban/)
8. [Code Review 잘하기](/posts/github-series-08-code-review/)
9. [GitHub Discussions](/posts/github-series-09-discussions/)
10. [Team 협업 설정](/posts/github-series-10-team-collaboration/)
11. **GitHub Actions 입문** (현재 글)
12. Actions 고급 활용 (예정)

---

*질문이나 피드백이 있으시다면 댓글로 남겨주세요!*