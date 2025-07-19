---
layout: post
title: "[Docker 입문 #2] Docker 설치 및 환경 설정"
date: 2025-07-16 10:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, tutorial, series, installation, setup, docker-desktop, beginner]
mermaid: true
---

지난 편에서 Docker가 무엇인지 알아보았습니다. 이번에는 실제로 Docker를 설치하고 기본적인 환경 설정을 해보겠습니다.

## Docker 설치 전 확인사항

Docker를 설치하기 전에 시스템 요구사항을 확인해야 합니다:

### 시스템 요구사항 비교표

| 운영체제 | 최소 요구사항 | 권장 사양 | 추가 요구사항 |
|---------|-------------|----------|-------------|
| **Windows** | • Windows 10 64-bit (Build 15063+)<br>• Pro/Enterprise/Education<br>• 4GB RAM | • Windows 11<br>• 8GB RAM 이상<br>• SSD 스토리지 | • WSL 2 활성화<br>• Hyper-V 지원<br>• BIOS 가상화 활성화 |
| **macOS** | • macOS 10.15 이상<br>• 4GB RAM<br>• Intel/Apple Silicon | • macOS 12 이상<br>• 8GB RAM 이상<br>• Apple Silicon 추천 | • 4GB 이상 여유 공간 |
| **Linux** | • 64-bit 커널 3.10+<br>• 2GB RAM | • 최신 LTS 버전<br>• 4GB RAM 이상 | • iptables 1.4+<br>• Git 1.7+<br>• xz-utils |

### Docker 설치 방법 선택 가이드

```mermaid
graph TD
    A[Docker 설치 시작] --> B{운영체제는?}
    B -->|Windows| C[Windows 버전 확인]
    B -->|macOS| D[칩셋 확인]
    B -->|Linux| E[배포판 확인]
    
    C --> F{Windows 10/11<br/>Pro/Enterprise?}
    F -->|Yes| G[Docker Desktop<br/>추천]
    F -->|No| H[WSL2 +<br/>Docker Desktop]
    
    D --> I{Apple Silicon?}
    I -->|Yes| J[Docker Desktop<br/>Apple Silicon]
    I -->|No| K[Docker Desktop<br/>Intel]
    
    E --> L{GUI 필요?}
    L -->|Yes| M[Docker Desktop<br/>for Linux]
    L -->|No| N[Docker Engine<br/>CLI]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style G fill:#bfb,stroke:#333,stroke-width:2px
    style H fill:#bfb,stroke:#333,stroke-width:2px
    style J fill:#bfb,stroke:#333,stroke-width:2px
    style K fill:#bfb,stroke:#333,stroke-width:2px
    style M fill:#bfb,stroke:#333,stroke-width:2px
    style N fill:#bfb,stroke:#333,stroke-width:2px
```

## Windows에서 Docker 설치하기

### Windows 설치 과정 흐름도

```mermaid
flowchart LR
    A[시작] --> B[Docker Desktop<br/>다운로드]
    B --> C[WSL 2 확인]
    C --> D{WSL 2<br/>설치됨?}
    D -->|No| E[WSL 2 설치]
    D -->|Yes| F[Docker Desktop<br/>설치]
    E --> F
    F --> G[재시작]
    G --> H[Docker 실행]
    H --> I[완료!]
    
    style A fill:#f96,stroke:#333,stroke-width:2px
    style I fill:#9f6,stroke:#333,stroke-width:2px
```

### 1. Docker Desktop 다운로드
1. [Docker 공식 웹사이트](https://www.docker.com/products/docker-desktop)에서 Docker Desktop for Windows를 다운로드합니다
2. 다운로드한 `Docker Desktop Installer.exe`를 실행합니다

### 2. WSL 2 설정

#### WSL 2 설치 단계별 명령어

| 단계 | 명령어 | 설명 |
|-----|--------|------|
| 1. WSL 활성화 | `dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart` | Windows Subsystem for Linux 기능 활성화 |
| 2. VM 기능 활성화 | `dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart` | 가상 머신 플랫폼 활성화 |
| 3. 재시작 | 컴퓨터 재시작 | 변경사항 적용 |
| 4. WSL 2 설정 | `wsl --set-default-version 2` | WSL 2를 기본 버전으로 설정 |
| 5. 커널 업데이트 | [WSL 2 커널 다운로드](https://aka.ms/wsl2kernel) | Linux 커널 업데이트 패키지 설치 |

```powershell
# PowerShell을 관리자 권한으로 실행하여 한 번에 실행
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

### 3. Docker Desktop 설치
1. 설치 마법사의 지시에 따라 설치를 진행합니다
2. "Use WSL 2 instead of Hyper-V" 옵션을 선택합니다
3. 설치 완료 후 컴퓨터를 재시작합니다

### Windows 설치 시 체크리스트

- [ ] Windows 버전이 요구사항을 충족하는가?
- [ ] BIOS에서 가상화가 활성화되어 있는가?
- [ ] WSL 2가 설치되어 있는가?
- [ ] 충분한 디스크 공간(10GB 이상)이 있는가?
- [ ] 관리자 권한으로 설치를 진행하는가?

## macOS에서 Docker 설치하기

### macOS 칩셋별 버전 선택

| 칩셋 타입 | 확인 방법 | Docker 버전 | 특징 |
|----------|----------|------------|------|
| **Intel** | 메뉴 → 이 Mac에 관하여<br>프로세서: Intel... | Docker Desktop Intel | • Rosetta 2 불필요<br>• 기존 x86 이미지 호환 |
| **Apple Silicon**<br>(M1, M2, M3) | 메뉴 → 이 Mac에 관하여<br>칩: Apple M... | Docker Desktop Apple Silicon | • 네이티브 성능<br>• ARM64 이미지 우선<br>• x86 에뮬레이션 가능 |

### macOS 설치 프로세스

```mermaid
graph TD
    A[Docker.dmg 다운로드] --> B[DMG 파일 실행]
    B --> C[Docker를 Applications로 드래그]
    C --> D[Docker 첫 실행]
    D --> E{권한 요청}
    E -->|승인| F[Docker 초기화]
    F --> G[상태바에 Docker 아이콘]
    G --> H[설치 완료!]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style H fill:#9f6,stroke:#333,stroke-width:2px
```

### 설치 단계
1. **다운로드**: [Docker 공식 웹사이트](https://www.docker.com/products/docker-desktop)에서 본인의 칩에 맞는 버전 다운로드
2. **설치**: 다운로드한 `Docker.dmg` 파일을 더블클릭
3. **복사**: Docker 아이콘을 Applications 폴더로 드래그
4. **실행**: Applications에서 Docker를 실행
5. **권한**: 권한 요청이 나타나면 승인

### Apple Silicon 사용자를 위한 팁

```bash
# 현재 아키텍처 확인
uname -m
# arm64: Apple Silicon
# x86_64: Intel

# Rosetta 2 설치 (x86 이미지 실행 시 필요)
softwareupdate --install-rosetta
```

## Linux(Ubuntu)에서 Docker 설치하기

### 1. 기존 Docker 제거
```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

### 2. 필요한 패키지 설치
```bash
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

### 3. Docker 공식 GPG 키 추가
```bash
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### 4. Docker 저장소 설정
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 5. Docker Engine 설치
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 6. 사용자를 docker 그룹에 추가
```bash
sudo usermod -aG docker $USER
# 로그아웃 후 다시 로그인하거나 다음 명령 실행
newgrp docker
```

## Docker 설치 확인

### 설치 검증 단계

```mermaid
flowchart TD
    A[Docker 설치 완료] --> B[docker --version]
    B --> C{버전 출력?}
    C -->|Yes| D[docker info]
    C -->|No| E[설치 문제 해결]
    D --> F{시스템 정보?}
    F -->|Yes| G[docker run hello-world]
    F -->|No| E
    G --> H{Hello 메시지?}
    H -->|Yes| I[설치 성공!]
    H -->|No| E
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style I fill:#9f6,stroke:#333,stroke-width:2px
    style E fill:#f99,stroke:#333,stroke-width:2px
```

### 설치 확인 명령어

| 명령어 | 설명 | 예상 출력 |
|--------|------|----------|
| `docker --version` | Docker 버전 확인 | `Docker version 24.0.x, build xxxxx` |
| `docker info` | 시스템 정보 확인 | 컨테이너, 이미지, 스토리지 정보 등 |
| `docker run hello-world` | 테스트 컨테이너 실행 | Hello from Docker! 메시지 |

```bash
# Docker 버전 확인
docker --version

# Docker 시스템 정보 확인
docker info

# Hello World 컨테이너 실행
docker run hello-world
```

정상적으로 설치되었다면 다음과 같은 메시지가 출력됩니다:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image.
 4. The Docker daemon streamed that output to the Docker client.
```

## Docker Desktop 기본 설정

### Docker Desktop 설정 메뉴 구조

```mermaid
graph TD
    A[Docker Desktop 설정] --> B[General]
    A --> C[Resources]
    A --> D[Docker Engine]
    A --> E[Experimental Features]
    
    B --> B1[Start Docker Desktop<br/>when you log in]
    B --> B2[Use Docker Compose V2]
    
    C --> C1[Advanced]
    C --> C2[File Sharing]
    C --> C3[Proxies]
    
    C1 --> C11[CPUs]
    C1 --> C12[Memory]
    C1 --> C13[Swap]
    C1 --> C14[Disk image size]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
```

### 권장 리소스 설정

| 리소스 | 최소값 | 권장값 | 설명 |
|--------|--------|--------|------|
| **CPU** | 2 cores | 4+ cores | 빌드 속도와 성능에 영향 |
| **Memory** | 2 GB | 8+ GB | 동시 실행 컨테이너 수에 영향 |
| **Swap** | 1 GB | 2 GB | 메모리 부족 시 사용 |
| **Disk** | 20 GB | 60+ GB | 이미지와 컨테이너 저장 공간 |

### 주요 설정 항목

#### 1. 리소스 설정
- **위치**: Settings → Resources → Advanced
- **설정 방법**: 슬라이더로 조정
- **적용**: Apply & Restart 클릭

#### 2. 파일 공유 설정
- **Windows/Mac**: Settings → Resources → File Sharing
- **용도**: 호스트-컨테이너 간 파일 공유
- **팁**: 개발 프로젝트 폴더 추가

#### 3. 네트워크 프록시
- **위치**: Settings → Resources → Proxies
- **설정 항목**: HTTP/HTTPS 프록시 주소
- **기업 환경**: 사내 프록시 서버 정보 입력

## Docker 기본 명령어 익히기

### Docker 명령어 구조

```mermaid
graph LR
    A[docker] --> B[명령어]
    B --> C[옵션]
    C --> D[대상]
    
    A2[docker] --> B2[container]
    B2 --> C2[run]
    C2 --> D2[-it]
    D2 --> E2[ubuntu]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style A2 fill:#9f9,stroke:#333,stroke-width:2px
```

### 필수 Docker 명령어 정리

| 분류 | 명령어 | 설명 | 예시 |
|-----|--------|------|------|
| **정보 확인** | `docker version` | Docker 버전 정보 | `docker version` |
| | `docker info` | 시스템 전체 정보 | `docker info` |
| | `docker --help` | 도움말 표시 | `docker --help` |
| **이미지** | `docker images` | 로컬 이미지 목록 | `docker images` |
| | `docker pull` | 이미지 다운로드 | `docker pull nginx:latest` |
| | `docker search` | 이미지 검색 | `docker search ubuntu` |
| **컨테이너** | `docker ps` | 실행 중인 컨테이너 | `docker ps` |
| | `docker ps -a` | 모든 컨테이너 | `docker ps -a` |
| | `docker run` | 컨테이너 실행 | `docker run hello-world` |

### 첫 번째 연습 명령어

```bash
# 1. Docker 버전 확인
docker version

# 2. nginx 이미지 다운로드
docker pull nginx:alpine

# 3. 다운로드된 이미지 확인
docker images

# 4. 컨테이너 실행
docker run -d -p 8080:80 nginx:alpine

# 5. 실행 중인 컨테이너 확인
docker ps

# 6. 브라우저에서 http://localhost:8080 접속
```

## 문제 해결

### 운영체제별 주요 문제와 해결 방법

| 문제 유형 | Windows | macOS | Linux |
|----------|---------|-------|-------|
| **설치 실패** | • Windows 업데이트<br>• WSL 2 커널 업데이트 | • macOS 버전 확인<br>• 디스크 공간 확인 | • 의존성 패키지 설치<br>• GPG 키 재설정 |
| **권한 오류** | • 관리자 권한 실행<br>• WSL 재시작 | • 시스템 환경설정 승인<br>• Docker 재시작 | • docker 그룹 추가<br>• `sudo usermod -aG docker $USER` |
| **가상화 오류** | • BIOS에서 VT-x 활성화<br>• Hyper-V 활성화 | • 자동으로 처리됨 | • KVM 모듈 확인<br>• `lsmod \| grep kvm` |
| **네트워크 오류** | • Windows Defender 확인<br>• VPN 충돌 확인 | • 방화벽 설정 확인 | • iptables 규칙 확인<br>• DNS 설정 확인 |

### 문제 해결 플로우차트

```mermaid
graph TD
    A[Docker 오류 발생] --> B{어떤 오류?}
    B -->|Cannot connect to<br/>Docker daemon| C[Docker 서비스 확인]
    B -->|Permission denied| D[권한 문제 해결]
    B -->|Image not found| E[이미지 다운로드]
    B -->|Port already in use| F[포트 충돌 해결]
    
    C --> C1[systemctl status docker]
    C --> C2[Docker Desktop 재시작]
    
    D --> D1[docker 그룹 확인]
    D --> D2[sudo 권한 사용]
    
    E --> E1[docker pull 명령어]
    E --> E2[인터넷 연결 확인]
    
    F --> F1[사용 중인 포트 확인]
    F --> F2[다른 포트 사용]
    
    style A fill:#f99,stroke:#333,stroke-width:2px
```

### 유용한 디버깅 명령어

```bash
# Docker 데몬 로그 확인 (Linux)
sudo journalctl -u docker.service -f

# Docker 데몬 로그 확인 (Mac/Windows)
# Docker Desktop → Troubleshoot → View Logs

# 시스템 리소스 확인
docker system df

# Docker 정리
docker system prune -a
```

## 마무리

Docker 설치와 기본 설정을 완료했습니다! 이제 Docker를 사용할 준비가 되었습니다. 다음 편에서는 첫 번째 컨테이너를 실행하고 기본적인 조작 방법을 배워보겠습니다.

## 다음 편 예고

- Docker 컨테이너 실행하기
- 컨테이너 내부 접속하기
- 컨테이너 생명주기 관리
- 기본 Docker 명령어 실습

Docker의 세계에 오신 것을 환영합니다! 🐳