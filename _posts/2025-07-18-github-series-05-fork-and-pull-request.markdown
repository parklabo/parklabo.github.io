---
layout: post
title: "[GitHub 입문 #5] Fork와 Pull Request: 오픈소스에 기여하는 방법"
date: 2025-07-18 10:00:00 +0900
categories: [DevOps, GitHub]
tags: [github, git, fork, pull-request, opensource, tutorial, series]
mermaid: true
---

## 들어가며

GitHub 입문 시리즈의 다섯 번째 시간입니다. 이번에는 오픈소스 프로젝트에 기여하는 핵심 메커니즘인 Fork와 Pull Request(PR)에 대해 알아보겠습니다. 이 두 기능을 통해 전 세계 개발자들과 협업할 수 있습니다.

## 1. Fork란 무엇인가?

Fork는 다른 사람의 저장소를 내 GitHub 계정으로 복사하는 기능입니다. 원본 저장소에 직접 접근 권한이 없어도 자유롭게 수정할 수 있습니다.

### Fork vs Clone의 차이

```
Original Repo (타인 소유)
      │
      ├─ Fork ─→ My Repo (내 GitHub)
      │               │
      │               └─ Clone ─→ Local Repo (내 컴퓨터)
      │
      └─ Clone ─→ Local Repo (읽기만 가능)
```

- **Fork**: GitHub 서버에서 저장소 복사
- **Clone**: 원격 저장소를 로컬로 복사

## 2. Fork 하기

### 웹에서 Fork

1. 기여하고 싶은 저장소 방문
2. 우측 상단 "Fork" 버튼 클릭
3. 내 계정 선택
4. Fork 완료!

### Fork 후 로컬 설정

```bash
# 1. Fork한 저장소 클론
git clone https://github.com/my-username/project-name.git
cd project-name

# 2. 원본 저장소를 upstream으로 추가
git remote add upstream https://github.com/original-owner/project-name.git

# 3. remote 확인
git remote -v
# origin    https://github.com/my-username/project-name.git (fetch)
# origin    https://github.com/my-username/project-name.git (push)
# upstream  https://github.com/original-owner/project-name.git (fetch)
# upstream  https://github.com/original-owner/project-name.git (push)
```

## 3. Pull Request란?

Pull Request(PR)는 내가 수정한 코드를 원본 저장소에 병합해달라고 요청하는 것입니다. GitHub의 핵심 협업 기능입니다.

### PR의 장점
- **코드 리뷰**: 변경사항 검토 및 피드백
- **토론**: 구현 방식에 대한 논의
- **테스트**: 자동화된 테스트 실행
- **문서화**: 변경 이유와 내용 기록

## 4. 첫 번째 Pull Request 만들기

### Step 1: 이슈 확인
```bash
# 기여할 이슈 찾기
# - "good first issue" 라벨
# - "help wanted" 라벨
# - "beginner friendly" 라벨
```

### Step 2: 브랜치 생성 및 작업
```bash
# 1. 최신 상태 동기화
git checkout main
git pull upstream main
git push origin main

# 2. 작업 브랜치 생성
git checkout -b fix/issue-123-typo

# 3. 코드 수정
echo "Fixed typo in README" >> README.md

# 4. 커밋
git add README.md
git commit -m "docs: Fix typo in installation guide

Fixes #123"

# 5. Fork 저장소에 푸시
git push origin fix/issue-123-typo
```

### Step 3: PR 생성

1. GitHub에서 Fork한 저장소 방문
2. "Compare & pull request" 버튼 클릭
3. PR 템플릿 작성:

```markdown
## Description
이 PR은 설치 가이드의 오타를 수정합니다.

## Changes
- README.md의 'instal' → 'install' 수정
- 문법 오류 수정

## Related Issue
Fixes #123

## Type of Change
- [x] Bug fix (오타/문법 수정)
- [ ] New feature
- [ ] Breaking change
- [x] Documentation update

## Checklist
- [x] 코드가 프로젝트 스타일 가이드를 따름
- [x] 변경사항을 테스트함
- [x] 관련 문서를 업데이트함
```

## 5. 코드 리뷰 프로세스

### 리뷰어 입장

```markdown
# 건설적인 피드백 예시

## 좋은 리뷰 ✅
"이 부분에서 `map` 대신 `forEach`를 사용하면 
더 명확할 것 같습니다. 반환값을 사용하지 않으니까요."

## 나쁜 리뷰 ❌
"코드가 별로네요. 다시 작성하세요."
```

### PR 작성자 입장

```bash
# 1. 리뷰 피드백 확인
# GitHub PR 페이지에서 리뷰 코멘트 확인

# 2. 요청된 변경사항 적용
git checkout fix/issue-123-typo
# 코드 수정...

# 3. 추가 커밋
git add .
git commit -m "refactor: Apply review feedback

- Use forEach instead of map
- Add error handling"

# 4. 푸시
git push origin fix/issue-123-typo
```

### 리뷰 도구 활용

GitHub PR 페이지에서:
- **Conversations**: 전체 토론
- **Commits**: 커밋 히스토리
- **Files changed**: 변경된 파일 검토
- **Review changes**: 승인/변경요청/코멘트

## 6. Fork 동기화 유지하기

### 원본 저장소의 변경사항 가져오기

```bash
# 1. upstream의 최신 변경사항 가져오기
git fetch upstream

# 2. 로컬 main 브랜치 업데이트
git checkout main
git merge upstream/main

# 3. Fork 저장소 업데이트
git push origin main

# 또는 한 번에 처리
git pull upstream main
git push origin main
```

### 작업 브랜치 리베이스

```bash
# 작업 중인 브랜치를 최신 main으로 리베이스
git checkout feature-branch
git rebase main

# 충돌 해결 후
git add .
git rebase --continue

# Fork에 강제 푸시 (주의!)
git push --force-with-lease origin feature-branch
```

## 7. 오픈소스 기여 가이드라인

### 1. 프로젝트 이해하기

```bash
# 필수 확인 문서
- README.md: 프로젝트 개요
- CONTRIBUTING.md: 기여 가이드라인
- CODE_OF_CONDUCT.md: 행동 강령
- LICENSE: 라이선스 확인
```

### 2. 좋은 PR의 조건

#### ✅ Do's

| 원칙 | 설명 | 예시 |
|------|------|---------|
| **작은 단위** | 리뷰하기 쉽게 | 50-200줄 이내 변경 |
| **명확한 제목** | 무엇을, 왜 변경했는지 | `feat: Add user login validation` |
| **테스트 포함** | 변경사항 검증 | 단위/통합 테스트 추가 |
| **문서 업데이트** | 필요시 문서도 수정 | README, API 문서 |
| **이슈 연결** | 추적 가능하게 | `Fixes #123`, `Closes #456` |

#### ❌ Don'ts

| 피할 점 | 이유 | 대안 |
|---------|------|---------|
| **거대한 PR** | 리뷰 어려움 | 여러 개의 작은 PR로 분할 |
| **관련 없는 변경** | 이슈 트래킹 어려움 | 기능별로 PR 분리 |
| **테스트 없는 기능** | 안정성 우려 | 테스트 코드 작성 |
| **스타일만 변경** | 의미 없는 노이즈 | 실제 기능과 함께 |
| **토론 없는 큰 변경** | 아키텍처 영향 | 이슈에서 사전 논의 |

### 3. 커밋 메시지 작성

```bash
# Angular 컨벤션 예시
feat: Add user authentication
fix: Resolve memory leak in data processor
docs: Update API documentation
style: Format code according to style guide
refactor: Simplify error handling logic
test: Add unit tests for auth service
chore: Update dependencies
```

## 8. 실전 시나리오

### 시나리오 1: 문서 오류 수정

```bash
# 1. Fork & Clone
git clone https://github.com/my-username/awesome-project.git
cd awesome-project

# 2. upstream 설정
git remote add upstream https://github.com/original/awesome-project.git

# 3. 브랜치 생성
git checkout -b docs/fix-readme-typo

# 4. 수정 & 커밋
# README.md 수정
git add README.md
git commit -m "docs: Fix typo in installation section"

# 5. Push & PR
git push origin docs/fix-readme-typo
# GitHub에서 PR 생성
```

### 시나리오 2: 새 기능 추가

```bash
# 1. 이슈 생성 또는 확인
# "새로운 정렬 옵션 추가" #456

# 2. 최신 코드 동기화
git checkout main
git pull upstream main
git push origin main

# 3. 기능 브랜치 생성
git checkout -b feat/456-add-sort-option

# 4. 개발 & 테스트
# 코드 작성...
# 테스트 작성...

# 5. 커밋 (의미 있는 단위로)
git add src/sort.js
git commit -m "feat: Add alphabetical sort option"

git add tests/sort.test.js
git commit -m "test: Add tests for alphabetical sort"

git add docs/api.md
git commit -m "docs: Document new sort option"

# 6. PR 생성 전 최종 확인
git diff main...feat/456-add-sort-option

# 7. Push & PR
git push origin feat/456-add-sort-option
```

## 9. PR 체크리스트

PR을 생성하기 전 확인사항:

```markdown
### 코드 품질
- [ ] 코드가 프로젝트 스타일 가이드를 따르는가?
- [ ] 불필요한 주석이나 디버그 코드가 없는가?
- [ ] 변수명과 함수명이 명확한가?

### 기능
- [ ] 의도한 대로 작동하는가?
- [ ] 엣지 케이스를 고려했는가?
- [ ] 기존 기능을 깨뜨리지 않는가?

### 테스트
- [ ] 새로운 테스트를 추가했는가?
- [ ] 모든 테스트가 통과하는가?
- [ ] 테스트 커버리지가 유지되는가?

### 문서
- [ ] README 업데이트가 필요한가?
- [ ] API 문서 업데이트가 필요한가?
- [ ] 변경 로그 업데이트가 필요한가?

### PR 설명
- [ ] 제목이 명확한가?
- [ ] 변경 이유를 설명했는가?
- [ ] 관련 이슈를 연결했는가?
- [ ] 스크린샷이 필요한 경우 추가했는가?
```

## 10. Fork 관리 팁

### 오래된 Fork 정리하기

```bash
# 1. 사용하지 않는 브랜치 삭제
git branch -d old-feature
git push origin --delete old-feature

# 2. 병합된 브랜치 정리
git branch --merged | grep -v "\*\|main" | xargs -n 1 git branch -d

# 3. Fork 자체를 삭제하려면
# GitHub Settings → Danger Zone → Delete this repository
```

### 여러 PR 동시 작업

```bash
# PR별로 별도 브랜치 생성
git checkout -b pr/feature-a
# 작업...

git checkout main
git checkout -b pr/feature-b
# 다른 작업...

# 각각 독립적으로 PR 생성 가능
```

## 마무리

Fork와 Pull Request는 오픈소스 생태계의 핵심입니다. 이를 통해:
- 권한 없이도 프로젝트에 기여 가능
- 체계적인 코드 리뷰 프로세스
- 전 세계 개발자들과 협업

처음에는 어려워 보일 수 있지만, 작은 기여부터 시작하면 금방 익숙해집니다. "good first issue"부터 도전해보세요!

다음 편에서는 Issues를 활용한 프로젝트 관리에 대해 알아보겠습니다.

## 시리즈 목차
1. [GitHub 시작하기](/posts/github-series-01-getting-started/)
2. [Repository 기초](/posts/github-series-02-repository-basics/)
3. [Git 기본 명령어](/posts/github-series-03-git-basic-commands/)
4. [Branch와 Merge](/posts/github-series-04-branch-and-merge/)
5. **Fork와 Pull Request** (현재 글)
6. Issues 활용법 (예정)

---

*질문이나 피드백이 있으시다면 댓글로 남겨주세요!*