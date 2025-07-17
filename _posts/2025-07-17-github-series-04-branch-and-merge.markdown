---
layout: post
title: "[GitHub 입문 #4] Branch와 Merge: 효율적인 협업을 위한 브랜치 전략"
date: 2025-07-17 10:00:00 +0900
categories: [GitHub, Tutorial]
tags: [github, git, branch, merge, workflow, tutorial, series]
mermaid: true
---

## 들어가며

GitHub 입문 시리즈의 네 번째 시간입니다. 이번에는 Git의 강력한 기능인 브랜치(Branch)와 병합(Merge)에 대해 알아보겠습니다. 브랜치는 독립적인 작업 공간을 만들어 여러 기능을 동시에 개발할 수 있게 해주는 Git의 핵심 기능입니다.

## 1. Branch란 무엇인가?

브랜치는 독립적인 작업 공간으로, 메인 코드베이스에서 분기하여 별도의 작업을 수행할 수 있게 해줍니다.

### 브랜치를 사용하는 이유

```mermaid
gitGraph
    commit id: "Initial commit"
    commit id: "Add header"
    branch feature
    checkout feature
    commit id: "Add login form"
    commit id: "Add validation"
    checkout main
    commit id: "Update styles"
    branch bugfix
    checkout bugfix
    commit id: "Fix navbar"
    commit id: "Fix responsive"
    checkout main
    merge feature
    merge bugfix
```

| 이점 | 설명 |
|------|------|
| **안전한 개발** | 메인 코드를 건드리지 않고 새 기능 개발 |
| **동시 작업** | 여러 개발자가 독립적으로 작업 |
| **실험** | 새로운 아이디어를 부담 없이 시도 |
| **버전 관리** | 릴리즈 버전별로 브랜치 관리 |
| **코드 리뷰** | PR을 통한 체계적인 코드 리뷰 |

## 2. Branch 기본 명령어 (2025년 7월 최신)

### 브랜치 생성과 이동

| 명령어 | 설명 | Git 버전 |
|---------|------|----------|
| `git branch` | 브랜치 목록 보기 | 모든 버전 |
| `git branch <name>` | 새 브랜치 생성 | 모든 버전 |
| `git checkout <branch>` | 브랜치 이동 | 모든 버전 |
| `git checkout -b <branch>` | 생성 + 이동 | 모든 버전 |
| `git switch <branch>` | 브랜치 이동 (권장) | 2.23+ |
| `git switch -c <branch>` | 생성 + 이동 (권장) | 2.23+ |

```bash
# 예시: 새로운 기능 브랜치 생성
git switch -c feature/user-authentication

# 브랜치 목록 확인
git branch -v  # 최신 커밋 포함
```

### 브랜치 관리

```bash
# 모든 브랜치 보기 (원격 포함)
git branch -a

# 원격 브랜치만 보기
git branch -r

# 브랜치 이름 변경
git branch -m old-name new-name

# 브랜치 삭제
git branch -d feature-login  # 병합된 브랜치만 삭제
git branch -D feature-login  # 강제 삭제

# 원격 브랜치 삭제
git push origin --delete feature-login
```

## 3. 브랜치 작업 실습

### 실습 시나리오: 로그인 기능 개발

```bash
# 1. 메인 브랜치에서 시작
git checkout main

# 2. 새 기능 브랜치 생성
git checkout -b feature/login

# 3. 로그인 파일 생성
echo "function login() { }" > login.js
git add login.js
git commit -m "feat: 로그인 함수 추가"

# 4. 추가 작업
echo "function logout() { }" >> login.js
git add login.js
git commit -m "feat: 로그아웃 함수 추가"

# 5. 브랜치 히스토리 확인
git log --oneline --graph

# 6. 메인 브랜치로 돌아가기
git checkout main
```

## 4. Merge - 브랜치 병합하기

### Merge의 종류

#### 1. Fast-Forward Merge
메인 브랜치에 새로운 커밋이 없을 때 발생하는 단순한 병합

```bash
# main에 새 커밋이 없는 상태
#
# main ────●
#          │
# feature  └──●──●──●
#
# 병합 후:
# main ────●──●──●──●

git checkout main
git merge feature/login
```

#### 2. 3-Way Merge
메인 브랜치와 기능 브랜치 모두에 새로운 커밋이 있을 때

```bash
# main ────●────●────●
#          │         │
# feature  └──●──●   │
#                │   │
#                └───● (merge commit)

git checkout main
git merge feature/login
# 자동으로 병합 커밋 생성
```

#### 3. Squash Merge
여러 커밋을 하나로 합쳐서 병합

```bash
# 여러 커밋을 하나로 정리
git checkout main
git merge --squash feature/login
git commit -m "feat: 로그인 기능 구현"
```

### 병합 충돌 해결하기

```bash
# 1. 충돌 발생 시나리오
# main과 feature에서 같은 파일의 같은 부분 수정

# 2. 병합 시도
git checkout main
git merge feature/login
# CONFLICT (content): Merge conflict in app.js

# 3. 충돌 파일 확인
git status
# both modified: app.js

# 4. 충돌 내용 확인
<<<<<<< HEAD
const title = "Main Title";
=======
const title = "Feature Title";
>>>>>>> feature/login

# 5. 수동으로 해결
const title = "Feature Title";  # 원하는 내용 선택

# 6. 해결 완료
git add app.js
git commit -m "merge: feature/login 병합 및 충돌 해결"
```

## 5. Rebase - 커밋 히스토리 정리하기

### Merge vs Rebase

```mermaid
flowchart TD
    subgraph "Merge 방식"
        A1[main: commit A] --> B1[main: commit B]
        A1 --> C1[feature: commit C]
        C1 --> D1[feature: commit D]
        B1 --> M1[merge commit]
        D1 --> M1
    end
    
    subgraph "Rebase 방식"
        A2[main: commit A] --> B2[main: commit B]
        B2 --> C2[feature: commit C']
        C2 --> D2[feature: commit D']
    end
```

| 차이점 | Merge | Rebase |
|------|-------|--------|
| **히스토리** | 병합 커밋 생성 | 선형 히스토리 |
| **장점** | 안전, 원본 보존 | 깔끔한 히스토리 |
| **단점** | 복잡한 그래프 | 공유 브랜치에 위험 |
| **사용 시기** | 팀 작업, PR | 개인 작업 |

### Rebase 사용법

```bash
# 1. feature 브랜치에서 작업
git checkout feature/login

# 2. main의 최신 변경사항을 기반으로 재정렬
git rebase main

# 3. 충돌 발생 시 해결
# 충돌 해결 후
git add .
git rebase --continue

# 4. rebase 취소
git rebase --abort
```

### Interactive Rebase

```bash
# 최근 3개 커밋 정리
git rebase -i HEAD~3

# 에디터에서 작업 선택
# pick abc123 첫 번째 커밋
# squash def456 두 번째 커밋  # 첫 번째와 합치기
# reword ghi789 세 번째 커밋  # 메시지 수정

# 작업 옵션:
# pick   = 커밋 사용
# reword = 커밋 메시지 수정
# edit   = 커밋 수정
# squash = 이전 커밋과 합치기
# fixup  = squash와 같지만 메시지 버림
# drop   = 커밋 삭제
```

## 6. 브랜치 전략 (Branching Strategies)

### Git Flow (2025년 기준 업데이트)

```mermaid
gitGraph
    commit tag: "v1.0"
    branch develop
    checkout develop
    commit id: "dev-1"
    branch feature/login
    checkout feature/login
    commit id: "login-1"
    commit id: "login-2"
    checkout develop
    merge feature/login
    branch release/1.1
    checkout release/1.1
    commit id: "release-prep"
    checkout main
    merge release/1.1 tag: "v1.1"
    checkout develop
    merge release/1.1
    checkout main
    branch hotfix/security
    checkout hotfix/security
    commit id: "security-fix"
    checkout main
    merge hotfix/security tag: "v1.1.1"
    checkout develop
    merge hotfix/security
```

| 브랜치 종류 | 목적 | 생명주기 |
|-------------|------|----------|
| **main/master** | 프로덕션 코드 | 영구적 |
| **develop** | 개발 통합 브랜치 | 영구적 |
| **feature/** | 기능 개발 | 임시 (develop에 병합 후 삭제) |
| **release/** | 배포 준비 | 임시 (main과 develop에 병합 후 삭제) |
| **hotfix/** | 긴급 버그 수정 | 임시 (main과 develop에 병합 후 삭제) |

```bash
# 1. 개발 브랜치에서 기능 브랜치 생성
git checkout develop
git checkout -b feature/new-feature

# 2. 작업 완료 후 develop에 병합
git checkout develop
git merge --no-ff feature/new-feature

# 3. 릴리즈 준비
git checkout -b release/1.0 develop

# 4. 릴리즈 완료
git checkout master
git merge --no-ff release/1.0
git tag -a v1.0 -m "Version 1.0"

# 5. develop에도 병합
git checkout develop
git merge --no-ff release/1.0
```

### GitHub Flow

더 간단한 브랜치 전략

```
main
  │
  ├── feature/login
  ├── feature/signup
  └── fix/bug-123
```

```bash
# 1. main에서 브랜치 생성
git checkout -b feature/new-feature

# 2. 작업 및 푸시
git push -u origin feature/new-feature

# 3. Pull Request 생성 및 리뷰

# 4. main에 병합
```

### 브랜치 네이밍 규칙 (2025년 베스트 프랙티스)

| 접두사 | 사용 목적 | 예시 |
|--------|----------|------|
| `feature/` | 새 기능 개발 | `feature/user-authentication` |
| `bugfix/` | 버그 수정 (개발 환경) | `bugfix/login-validation` |
| `hotfix/` | 긴급 버그 수정 (프로덕션) | `hotfix/critical-security-patch` |
| `refactor/` | 코드 개선 | `refactor/api-optimization` |
| `docs/` | 문서 작업 | `docs/api-documentation` |
| `test/` | 테스트 추가/수정 | `test/integration-tests` |
| `chore/` | 유지보수 작업 | `chore/update-dependencies` |
| `release/` | 릴리즈 준비 | `release/v2.1.0` |

```mermaid
flowchart LR
    A[main] --> B[feature/oauth-login]
    A --> C[bugfix/header-layout]
    A --> D[hotfix/xss-vulnerability]
    B --> E[PR #123]
    C --> F[PR #124]
    D --> G[Direct merge]
    E --> A
    F --> A
    G --> A
```

## 7. 실전 팁과 주의사항

### 브랜치 보호 규칙 (GitHub 2025년 7월 기능)

```mermaid
flowchart TD
    A[Settings] --> B[Branches]
    B --> C[Add rule]
    C --> D[Branch name pattern: main]
    D --> E[Protection rules]
    E --> F[Require PR reviews]
    E --> G[Status checks]
    E --> H[Dismiss stale reviews]
    E --> I[Require branches up to date]
    E --> J[Require conversation resolution]
    E --> K[Require signed commits]
    E --> L[Lock branch]
    E --> M[Restrict deletions]
```

| 보호 규칙 | 설명 | 권장 설정 |
|-----------|------|------------|
| **PR 리뷰 필수** | 최소 리뷰어 수 지정 | 1-2명 |
| **상태 체크** | CI/CD 통과 필수 | 테스트, 빌드 |
| **대화 해결** | 모든 PR 코멘트 해결 | 필수 |
| **최신 상태 유지** | 병합 전 최신 동기화 | 권장 |
| **서명된 커밋** | GPG 서명 필수 | 보안 필요시 |
| **관리자 포함** | 관리자도 규칙 적용 | 필수 |

### 자주 하는 실수와 해결법

#### 실수 1: 잘못된 브랜치에서 작업
```bash
# 현재 브랜치 확인 습관화
git branch --show-current

# 작업 전 항상 확인
git status
```

#### 실수 2: 원격에 푸시한 브랜치 rebase
```bash
# 절대 하지 말 것!
git rebase main
git push --force  # 위험!

# 대신 새 브랜치로 작업
git checkout -b feature/new-branch
```

#### 실수 3: 병합 전 테스트 안 함
```bash
# 병합 전 테스트 브랜치 생성
git checkout -b test-merge main
git merge feature/login
# 테스트 수행
# 문제 없으면 실제 병합
```

### 유용한 명령어 모음

```bash
# 브랜치 간 차이 확인
git diff main..feature/login

# 브랜치에만 있는 커밋 보기
git log main..feature/login

# 브랜치 생성 시점 찾기
git merge-base main feature/login

# 모든 브랜치의 최신 커밋 보기
git branch -v

# 병합된 브랜치 정리
git branch --merged | grep -v "\*\|main" | xargs -n 1 git branch -d
```

## 8. 협업 시나리오

### 시나리오: 팀 프로젝트에서 기능 개발

```bash
# 1. 최신 main 가져오기
git checkout main
git pull origin main

# 2. 기능 브랜치 생성
git checkout -b feature/shopping-cart

# 3. 작업 및 정기적 커밋
git add .
git commit -m "feat: 장바구니 추가 기능 구현"

# 4. 원격에 푸시
git push -u origin feature/shopping-cart

# 5. 작업 중 main 업데이트 반영
git checkout main
git pull origin main
git checkout feature/shopping-cart
git merge main  # 또는 rebase

# 6. Pull Request 생성
# GitHub 웹에서 PR 생성

# 7. 리뷰 후 병합
```

## 마무리

브랜치와 병합은 Git의 가장 강력한 기능 중 하나입니다. 처음에는 복잡해 보일 수 있지만, 꾸준히 사용하다 보면 자연스럽게 익숙해집니다.

### 핵심 체크리스트

| ✅ | 항목 | 설명 |
|---|------|------|
| ☐ | 브랜치 전략 수립 | Git Flow, GitHub Flow 중 선택 |
| ☐ | 네이밍 규칙 문서화 | 팀 위키에 기록 |
| ☐ | 보호 규칙 설정 | main, develop 브랜치 |
| ☐ | PR 템플릿 작성 | 일관된 형식 |
| ☐ | 충돌 해결 가이드 | 팀원 교육 |

다음 편에서는 오픈소스 기여의 핵심인 Fork와 Pull Request에 대해 알아보겠습니다.

### 학습 리소스 (2025년 7월 추천)

| 리소스 | 설명 | 링크 |
|---------|------|------|
| **GitHub Skills** | GitHub 공식 학습 플랫폼 | [skills.github.com](https://skills.github.com) |
| **Learn Git Branching** | 인터랙티브 브랜치 학습 | [learngitbranching.js.org](https://learngitbranching.js.org) |
| **Pro Git Book** | Git 공식 문서 | [git-scm.com/book](https://git-scm.com/book) |

## 시리즈 목차
1. [GitHub 시작하기](/posts/github-series-01-getting-started/)
2. [Repository 기초](/posts/github-series-02-repository-basics/)
3. [Git 기본 명령어](/posts/github-series-03-git-basic-commands/)
4. **Branch와 Merge** (현재 글)
5. Fork와 Pull Request (예정)

---

*질문이나 피드백이 있으시다면 댓글로 남겨주세요!*