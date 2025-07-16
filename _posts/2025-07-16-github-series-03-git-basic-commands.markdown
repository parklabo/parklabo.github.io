---
layout: post
title: "[GitHub 마스터하기 #3] Git 기본 명령어: add, commit, push, pull 마스터하기"
date: 2025-07-16 10:00:00 +0900
categories: [GitHub, Tutorial]
tags: [github, git, commands, tutorial, series, version-control]
mermaid: true
---

## 들어가며

GitHub 마스터하기 시리즈의 세 번째 시간입니다. 이번에는 Git의 핵심 명령어들을 실습과 함께 자세히 알아보겠습니다. 이 명령어들은 Git을 사용하는 데 있어 가장 기본이 되는 도구들입니다.

## 1. Git의 작업 흐름 이해하기

### Git의 3단계 작업 영역

```mermaid
graph LR
    A[Working Directory<br/>작업 공간] -->|git add| B[Staging Area<br/>준비 영역]
    B -->|git commit| C[Local Repository<br/>로컬 저장소]
    C -->|git push| D[Remote Repository<br/>원격 저장소]
    D -->|git pull| A
    
    style A fill:#ffd,stroke:#333,stroke-width:2px
    style B fill:#dfd,stroke:#333,stroke-width:2px
    style C fill:#ddf,stroke:#333,stroke-width:2px
    style D fill:#fdd,stroke:#333,stroke-width:2px
```

### 각 영역의 특징

| 영역 | 역할 | 상태 | 주요 명령어 |
|------|------|------|------------|
| **Working Directory** | 실제 작업 공간 | • 파일 수정/생성/삭제<br>• Untracked/Modified | `git status`<br>`git diff` |
| **Staging Area** | 커밋 준비 영역 | • 커밋할 변경사항 준비<br>• Staged | `git add`<br>`git reset` |
| **Local Repository** | 로컬 저장소 | • 커밋 히스토리 저장<br>• Committed | `git commit`<br>`git log` |
| **Remote Repository** | 원격 저장소 | • 팀 공유 저장소<br>• Pushed | `git push`<br>`git pull` |

## 2. git add - 변경사항 준비하기

`git add`는 변경된 파일을 Staging Area로 옮기는 명령어입니다.

### git add 명령어 옵션 비교

| 명령어 | 설명 | 사용 시나리오 |
|--------|------|--------------|
| `git add <file>` | 특정 파일만 추가 | 선택적으로 커밋하고 싶을 때 |
| `git add .` | 현재 디렉토리의 모든 변경사항 | 모든 변경사항을 커밋할 때 |
| `git add -A` | 전체 저장소의 모든 변경사항 | 프로젝트 전체 변경사항 추가 |
| `git add -u` | 수정/삭제된 파일만 (새 파일 제외) | 기존 파일 변경사항만 추가 |
| `git add -p` | 대화형으로 일부분만 추가 | 세밀한 커밋 관리 |
| `git add *.js` | 특정 패턴의 파일들 | 특정 타입 파일만 추가 |

### git add 작동 과정

```mermaid
flowchart TD
    A[파일 수정] --> B{git add 실행}
    B --> C[Staging Area로 이동]
    C --> D[git status로 확인]
    D --> E{올바른 파일?}
    E -->|Yes| F[git commit 준비 완료]
    E -->|No| G[git reset으로 취소]
    G --> B
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style F fill:#9f9,stroke:#333,stroke-width:2px
```

### 실전 예제

```bash
# 1. 파일 생성
echo "Hello, Git!" > hello.txt
echo "console.log('Hello');" > app.js

# 2. 상태 확인
git status
# Output:
# Untracked files:
#   hello.txt
#   app.js

# 3. 파일 추가
git add hello.txt
git status
# Output:
# Changes to be committed:
#   new file: hello.txt
# Untracked files:
#   app.js

# 4. 모든 파일 추가
git add .
git status
# Output:
# Changes to be committed:
#   new file: hello.txt
#   new file: app.js
```

### 팁: 일부만 추가하기

```bash
# 파일의 일부분만 추가
git add -p filename.txt

# 대화형으로 선택
# y - 이 부분 추가
# n - 이 부분 건너뛰기
# s - 더 작은 단위로 나누기
# e - 수동으로 편집
```

## 3. git commit - 변경사항 저장하기

`git commit`은 Staging Area의 파일들을 Repository에 저장합니다.

### 기본 사용법

```bash
# 커밋 메시지와 함께 커밋
git commit -m "커밋 메시지"

# 에디터를 열어서 긴 메시지 작성
git commit

# 스테이징 없이 변경된 파일 모두 커밋
git commit -a -m "커밋 메시지"

# 이전 커밋 수정
git commit --amend

# 빈 커밋 만들기
git commit --allow-empty -m "빈 커밋"
```

### 좋은 커밋 메시지 작성법

#### 커밋 메시지 구조

```mermaid
graph TD
    A[커밋 메시지] --> B[제목 라인]
    A --> C[본문]
    A --> D[꼬리말]
    
    B --> B1["type(scope): subject"]
    B --> B2[50자 이내]
    
    C --> C1[상세 설명]
    C --> C2[72자 줄바꿈]
    C --> C3[무엇을, 왜]
    
    D --> D1[이슈 번호]
    D --> D2[Breaking Changes]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
```

#### 커밋 타입별 가이드

| 타입 | 설명 | 예시 |
|------|------|------|
| **feat** | 새로운 기능 추가 | `feat: 사용자 프로필 이미지 업로드 기능 추가` |
| **fix** | 버그 수정 | `fix: 로그인 시 null 포인터 예외 수정` |
| **docs** | 문서 수정 | `docs: API 사용 가이드 추가` |
| **style** | 코드 포맷팅, 세미콜론 누락 등 | `style: ESLint 규칙에 맞게 코드 정리` |
| **refactor** | 코드 리팩토링 | `refactor: 사용자 인증 로직 서비스 레이어로 분리` |
| **test** | 테스트 코드 추가/수정 | `test: 회원가입 통합 테스트 추가` |
| **chore** | 빌드 업무, 패키지 매니저 설정 등 | `chore: webpack 설정 업데이트` |
| **perf** | 성능 개선 | `perf: 이미지 레이지 로딩 구현` |

#### 좋은 커밋 메시지 예시

```bash
# 완전한 형식
git commit -m "feat(auth): JWT 기반 인증 시스템 구현

- Access Token과 Refresh Token 구조 구현
- 토큰 만료 시 자동 갱신 로직 추가
- 인증 실패 시 에러 핸들링 개선

Breaking Change: 기존 세션 기반 인증 제거
Closes #123, #124"
```

### 커밋 되돌리기

#### reset 옵션 비교

| 옵션 | HEAD 이동 | Staging Area | Working Directory | 사용 시나리오 |
|------|-----------|--------------|-------------------|--------------|
| `--soft` | ✓ | 유지 | 유지 | 커밋만 취소하고 변경사항 유지 |
| `--mixed` (기본) | ✓ | 초기화 | 유지 | 커밋과 스테이징 취소 |
| `--hard` | ✓ | 초기화 | 초기화 | 모든 변경사항 완전 삭제 |

```bash
# 마지막 커밋 취소 (파일은 그대로)
git reset --soft HEAD~1

# 마지막 커밋 취소 (파일도 unstaged 상태로)
git reset HEAD~1

# 마지막 커밋 완전히 취소 (파일 변경사항도 삭제)
git reset --hard HEAD~1

# 특정 커밋으로 되돌리기
git reset --hard <commit-hash>
```

## 4. git push - 원격 저장소에 업로드

`git push`는 로컬 커밋을 원격 저장소로 전송합니다.

### 기본 사용법

```bash
# 기본 푸시
git push

# 특정 원격 저장소와 브랜치 지정
git push origin main

# 새 브랜치 푸시
git push -u origin feature-branch

# 모든 브랜치 푸시
git push --all

# 태그 푸시
git push --tags

# 강제 푸시 (위험!)
git push --force
```

### 푸시 전 확인사항

```bash
# 1. 현재 브랜치 확인
git branch
# * main

# 2. 원격 저장소 확인
git remote -v
# origin  git@github.com:username/repo.git (fetch)
# origin  git@github.com:username/repo.git (push)

# 3. 푸시할 커밋 확인
git log origin/main..HEAD --oneline
# abc123 feat: 새 기능 추가
# def456 fix: 버그 수정

# 4. 푸시
git push origin main
```

### 푸시 실패 시 해결법

```mermaid
flowchart TD
    A[git push 실패] --> B{실패 원인}
    B -->|원격에 새 커밋| C[git pull 실행]
    B -->|권한 없음| D[저장소 권한 확인]
    B -->|브랜치 보호| E[PR 생성]
    
    C --> F{충돌 발생?}
    F -->|Yes| G[충돌 해결]
    F -->|No| H[자동 병합]
    
    G --> I[git add/commit]
    H --> I
    I --> J[git push 재시도]
    
    style A fill:#f99,stroke:#333,stroke-width:2px
    style J fill:#9f9,stroke:#333,stroke-width:2px
```

#### 푸시 옵션 비교

| 옵션 | 설명 | 위험도 | 사용 시나리오 |
|------|------|--------|--------------|
| `git push` | 기본 푸시 | 안전 | 일반적인 경우 |
| `git push -u origin main` | 업스트림 설정 | 안전 | 새 브랜치 첫 푸시 |
| `git push --force` | 강제 푸시 | 위험 | 절대 피해야 함 |
| `git push --force-with-lease` | 안전한 강제 푸시 | 중간 | 리베이스 후 사용 |

## 5. git pull - 원격 저장소에서 가져오기

`git pull`은 원격 저장소의 변경사항을 가져와 현재 브랜치에 병합합니다.

### 기본 사용법

```bash
# 기본 풀
git pull

# 특정 원격 저장소와 브랜치에서 풀
git pull origin main

# 리베이스로 풀
git pull --rebase

# 병합 없이 가져오기만 (fetch)
git fetch

# 모든 브랜치 업데이트
git pull --all
```

### pull = fetch + merge

```mermaid
graph LR
    A[git pull] --> B[git fetch]
    B --> C[원격 변경사항<br/>다운로드]
    C --> D[git merge]
    D --> E[로컬 브랜치에<br/>병합]
    
    F[git pull --rebase] --> B
    B --> C
    C --> G[git rebase]
    G --> H[커밋 히스토리<br/>재정렬]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style F fill:#9ff,stroke:#333,stroke-width:2px
```

#### pull vs fetch 비교

| 명령어 | 동작 | 로컬 변경 | 사용 시나리오 |
|--------|------|-----------|--------------|
| `git fetch` | 원격 변경사항만 가져옴 | 변경 없음 | 확인 후 병합하고 싶을 때 |
| `git pull` | fetch + merge | 자동 병합 | 바로 병합해도 안전할 때 |
| `git pull --rebase` | fetch + rebase | 히스토리 재정렬 | 깔끔한 히스토리 유지 |

### 충돌 해결하기

```bash
# 1. 풀 시도
git pull origin main
# CONFLICT (content): Merge conflict in file.txt

# 2. 충돌 확인
git status
# both modified: file.txt

# 3. 파일 편집하여 충돌 해결
# <<<<<<< HEAD
# 내 변경사항
# =======
# 원격 저장소의 변경사항
# >>>>>>> origin/main

# 4. 해결 후 커밋
git add file.txt
git commit -m "merge: 충돌 해결"
git push origin main
```

## 6. 실전 워크플로우

### 일일 작업 플로우

```mermaid
sequenceDiagram
    participant W as Working Directory
    participant S as Staging Area
    participant L as Local Repo
    participant R as Remote Repo
    
    Note over W,R: 작업 시작
    R->>W: git pull (최신 상태 동기화)
    W->>W: 파일 수정/생성
    W->>S: git add
    S->>L: git commit
    R->>W: git pull (충돌 확인)
    L->>R: git push
    Note over W,R: 작업 완료
```

### 작업 플로우 체크리스트

| 단계 | 명령어 | 목적 | 주의사항 |
|------|--------|------|----------|
| 1. 동기화 | `git pull origin main` | 최신 코드 받기 | 작업 시작 전 필수 |
| 2. 작업 | 파일 수정/생성 | 기능 구현 | 작은 단위로 작업 |
| 3. 확인 | `git status` | 변경사항 확인 | 의도한 파일만 수정? |
| 4. 스테이징 | `git add .` | 커밋 준비 | 불필요한 파일 제외 |
| 5. 커밋 | `git commit -m "..."` | 변경사항 저장 | 명확한 메시지 작성 |
| 6. 동기화 | `git pull origin main` | 충돌 방지 | 푸시 전 필수 |
| 7. 푸시 | `git push origin main` | 공유 | 팀원에게 알림 |

### 실수 복구 시나리오

#### 시나리오 1: 잘못된 파일을 add 했을 때
```bash
# 특정 파일 unstage
git reset HEAD file.txt

# 모든 파일 unstage
git reset HEAD
```

#### 시나리오 2: 잘못된 커밋 메시지
```bash
# 마지막 커밋 메시지 수정
git commit --amend -m "올바른 커밋 메시지"
```

#### 시나리오 3: 민감한 정보를 커밋했을 때
```bash
# 파일에서 민감한 정보 제거
echo "안전한 내용" > config.txt

# 수정사항 커밋
git add config.txt
git commit --amend --no-edit

# 강제 푸시 (주의!)
git push --force-with-lease
```

## 7. 유용한 단축키와 별칭

### Git 별칭 설정

```bash
# 자주 사용하는 명령어 별칭 설정
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.cm commit
git config --global alias.ps push
git config --global alias.pl pull

# 로그 보기 별칭
git config --global alias.lg "log --graph --oneline --all --decorate"

# 사용 예
git st  # git status
git cm -m "커밋 메시지"  # git commit -m "커밋 메시지"
git lg  # 예쁜 로그 보기
```

## 8. 상태 확인 명령어들

### 상태 확인 명령어 정리

| 명령어 | 용도 | 주요 옵션 | 출력 내용 |
|--------|------|-----------|-----------|
| `git status` | 작업 상태 확인 | `-s` 간단히<br>`-sb` 브랜치 포함 | 변경된 파일, 스테이징 상태 |
| `git log` | 커밋 히스토리 | `--oneline` 한 줄<br>`--graph` 그래프 | 커밋 이력 |
| `git diff` | 변경사항 비교 | `--staged` 스테이징<br>`--stat` 통계 | 코드 차이점 |
| `git show` | 특정 커밋 확인 | `--stat` 요약 | 커밋 상세 정보 |

### Git 상태 플로우

```mermaid
stateDiagram-v2
    [*] --> Untracked: 새 파일 생성
    Untracked --> Staged: git add
    
    [*] --> Unmodified: 기존 파일
    Unmodified --> Modified: 파일 수정
    Modified --> Staged: git add
    
    Staged --> Unmodified: git commit
    
    Modified --> Unmodified: git restore
    Staged --> Modified: git restore --staged
    
    Unmodified --> [*]: git rm
```

### 유용한 로그 포맷

```bash
# 커스텀 로그 포맷
git log --pretty=format:"%h - %an, %ar : %s"

# 그래프와 함께 보기
git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit

# 통계와 함께 보기
git log --stat --pretty=format:'%C(yellow)%h%Creset %s %C(green)(%ar)%Creset'
```

## 마무리

Git의 기본 명령어들은 버전 관리의 핵심입니다. add, commit, push, pull 이 네 가지 명령어만 잘 이해해도 대부분의 Git 작업을 수행할 수 있습니다. 

중요한 것은 이론보다 실습입니다. 개인 프로젝트를 만들어서 자유롭게 실험해보고, 실수하고, 복구하는 과정을 통해 Git에 익숙해지세요.

다음 편에서는 브랜치(Branch)와 병합(Merge)에 대해 자세히 알아보겠습니다.

## 시리즈 목차
1. [GitHub 시작하기](/posts/github-series-01-getting-started/)
2. [Repository 기초](/posts/github-series-02-repository-basics/)
3. **Git 기본 명령어** (현재 글)
4. Branch와 Merge (예정)
5. Fork와 Pull Request (예정)

---

*질문이나 피드백이 있으시다면 댓글로 남겨주세요!*