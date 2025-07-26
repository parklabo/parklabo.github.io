---
layout: post
title: "[GitHub 입문 #10] Team 협업 설정: 권한 관리와 보호 규칙"
date: 2025-07-23 10:00:00 +0900
categories: [DevOps, GitHub]
tags: [github, team, collaboration, permissions, security, tutorial, series]
mermaid: true
---

## 들어가며

GitHub 입문 시리즈의 열 번째 시간입니다. 이번에는 팀 협업을 위한 권한 관리와 저장소 보호 규칙에 대해 알아보겠습니다. 효과적인 권한 관리는 보안을 유지하면서도 원활한 협업을 가능하게 하는 핵심 요소입니다.

## 1. GitHub 권한 체계 이해하기 (2025년 확장)

### AI 기반 권한 관리 (2025년 7월 최신)

```yaml
2025년 새로운 기능:
  AI 권한 추천:
    - 사용자 활동 패턴 분석
    - 최적 권한 레벨 제안
    - 이상 접근 감지
    
  Guest Collaborators (2025 신규):
    - 엔터프라이즈 환경 외부 협업
    - 세분화된 권한 제어
    - 자동 만료 관리
    
  팀 동기화:
    - SAML/SCIM 통합 강화
    - Azure AD 직접 연동
    - 자동 역할 매핑
```

### Organization 권한 레벨

```yaml
Owner:
  - 모든 권한 보유
  - 조직 설정 변경
  - 멤버 관리
  - 결제 정보 접근

Member:
  - 기본 멤버 권한
  - Public 저장소 생성
  - 팀 가입/탈퇴

Outside Collaborator:
  - 특정 저장소만 접근
  - 조직 멤버 아님
  - 제한된 권한
```

### Repository 권한 레벨 (2025년 7월 확장)

```yaml
Admin:
  - 저장소 완전 관리
  - 설정 변경
  - 협업자 추가/제거
  - 저장소 삭제
  - Actions 정책 관리 (2025 신규)

Maintain:
  - 저장소 관리 (삭제 제외)
  - 이슈/PR 관리
  - 브랜치 보호 규칙 제외
  - 보안 기능 설정 (2025 신규)

Write:
  - 코드 푸시
  - 이슈/PR 생성 및 관리
  - 브랜치 생성/삭제
  - Copilot 사용 (2025 신규)

Triage:
  - 이슈/PR 관리
  - 코드 푸시 불가
  - 라벨/마일스톤 관리
  - Projects 관리 (2025 신규)

Read:
  - 코드 읽기
  - 이슈 생성
  - PR 코멘트
  - 아티팩트 다운로드 (2025 신규)
```

## 2. Team 생성과 관리 (2025년 강화)

### AI Team Management (2025년 최신)

| 기능 | 설명 | 이점 |
|------|------|------|
| 팀 구성 최적화 | AI가 프로젝트 요구사항 분석하여 팀 구성 제안 | 30% 효율성 향상 |
| 자동 워크로드 분배 | 팀원 스킬과 가용성 기반 작업 할당 | 번아웃 50% 감소 |
| 협업 패턴 분석 | 팀 상호작용 분석 및 개선점 제안 | 커뮤니케이션 40% 개선 |

### Team 구조 설계

```yaml
Engineering:
  Frontend:
    - UI Team
    - UX Team
  Backend:
    - API Team
    - Database Team
  DevOps:
    - Infrastructure
    - Security

Product:
  - PM Team
  - Design Team
  
QA:
  - Manual Testing
  - Automation
```

### Team 생성하기

```bash
# GitHub CLI로 팀 생성
gh api orgs/{org}/teams \
  --method POST \
  -f name='Frontend Team' \
  -f description='Frontend 개발팀' \
  -f privacy='closed' \
  -f notification_setting='notifications_enabled'
```

### Team 권한 설정

```yaml
# 팀별 저장소 접근 권한
Frontend Team:
  - frontend-app: Write
  - design-system: Write
  - backend-api: Read
  - docs: Write

Backend Team:
  - backend-api: Write
  - database: Admin
  - frontend-app: Read
  - docs: Write

DevOps Team:
  - all repositories: Maintain
  - infrastructure: Admin
```

## 3. Branch Protection Rules (2025년 고도화)

### AI 기반 보호 규칙 (2025년 최신)

```yaml
스마트 보호 규칙:
  AI 리스크 평가:
    - 변경사항 위험도 자동 분석
    - 동적 승인자 수 조정
    - 맥락 기반 규칙 적용
    
  자동 규칙 생성:
    - 프로젝트 타입별 템플릿
    - 업계 표준 자동 적용
    - 컴플라이언스 요구사항 반영
```

### 기본 보호 규칙 설정

```yaml
main branch protection:
  # PR 필수
  require_pull_request_reviews:
    required_approving_review_count: 2
    dismiss_stale_reviews: true
    require_review_from_codeowners: true
    require_last_push_approval: true
    
  # 상태 체크
  require_status_checks:
    strict: true  # 브랜치가 최신 상태여야 함
    contexts:
      - continuous-integration/travis-ci
      - coverage/coveralls
      - security/snyk
      
  # 추가 제한
  enforce_admins: true
  require_signed_commits: true
  require_linear_history: true
  require_conversation_resolution: true
  
  # 푸시 제한
  restrictions:
    users: []
    teams: ["release-team"]
    apps: ["dependabot"]
```

### 환경별 브랜치 전략

```yaml
Production (main):
  - 가장 엄격한 보호
  - 3명 이상 승인 필요
  - 모든 테스트 통과
  - 관리자도 규칙 적용

Staging (staging):
  - 2명 승인 필요
  - 주요 테스트 통과
  - QA 팀 승인 필수

Development (develop):
  - 1명 승인
  - 기본 테스트 통과
  - 개발자 자유도 높음

Feature branches:
  - 보호 규칙 없음
  - 자유로운 실험
  - PR 통해서만 병합
```

## 4. CODEOWNERS 파일 활용

### CODEOWNERS 파일 작성

`.github/CODEOWNERS`:

```bash
# 전체 기본 소유자
* @org/engineering-leads

# Frontend 관련
/src/components/ @org/frontend-team
/src/styles/ @org/design-team @org/frontend-team
*.css @org/design-team
*.scss @org/design-team

# Backend 관련
/api/ @org/backend-team
/src/services/ @org/backend-team @username
/database/ @org/database-team

# 설정 파일
/.github/ @org/devops-team @org/engineering-leads
/docker/ @org/devops-team
*.yml @org/devops-team
*.yaml @org/devops-team

# 문서
/docs/ @org/tech-writers @org/engineering
*.md @org/tech-writers
README.md @org/engineering-leads

# 보안 관련
/security/ @org/security-team
*.pem @org/security-team
*.key @org/security-team

# 특정 파일은 CTO 승인 필요
/src/core/auth.js @cto
/infrastructure/production.tf @cto @org/devops-leads
```

### CODEOWNERS 고급 패턴

```bash
# 여러 소유자 (모두의 승인 필요)
/critical/ @user1 @user2 @user3

# 팀과 개인 혼합
/src/payments/ @org/payment-team @payment-lead

# 파일 확장자 패턴
*.go @org/backend-team
*.js @org/frontend-team @org/backend-team
*.sql @org/database-team @dba-lead

# 제외 패턴 (! 사용)
/vendor/ @org/backend-team
!/vendor/critical-lib/ @security-expert

# 계층적 소유권
/app/ @org/frontend-team
/app/admin/ @org/admin-team @security-lead
```

## 5. Access Control 전략 (2025년 혁신)

### Zero Trust Security Model (2025년 최신)

```mermaid
graph TD
    A[사용자 요청] --> B[신원 확인]
    B --> C[컨텍스트 분석]
    C --> D[AI 위험 평가]
    D --> E{접근 허용?}
    E -->|예| F[임시 권한 부여]
    E -->|아니오| G[접근 거부]
    F --> H[활동 모니터링]
    H --> I[자동 권한 회수]
```

### Repository Visibility 관리

```yaml
Public Repositories:
  용도:
    - 오픈소스 프로젝트
    - 공개 문서
    - 샘플 코드
  주의사항:
    - 민감한 정보 제거
    - 라이선스 명시
    - 보안 스캔 필수

Private Repositories:
  용도:
    - 상업용 코드
    - 내부 도구
    - 고객 프로젝트
  접근 관리:
    - Need-to-know 원칙
    - 정기적 권한 검토
    - 외부 협업자 최소화

Internal Repositories:
  용도:
    - 조직 내 공유 코드
    - 내부 문서
    - 팀 간 협업 프로젝트
  장점:
    - 조직 내 투명성
    - 쉬운 협업
    - 외부 노출 방지
```

### 세분화된 권한 관리

```yaml
# 역할 기반 접근 제어 (RBAC)
roles:
  junior_developer:
    repositories:
      - training-repo: Write
      - main-app: Read
      - docs: Write
    restrictions:
      - no_force_push
      - no_branch_deletion
      
  senior_developer:
    repositories:
      - all: Write
    restrictions:
      - no_force_push_to_main
      
  tech_lead:
    repositories:
      - all: Maintain
    additional:
      - branch_protection_override
      - emergency_merge
      
  devops_engineer:
    repositories:
      - all: Read
      - infrastructure: Admin
      - ci-cd: Admin
```

## 6. Security 설정 (2025년 최첨단)

### AI Security Assistant (2025년 7월 최신)

```yaml
Code Scanning 강화:
  대규모 활성화:
    - 여러 저장소 동시 설정
    - 기본 설정 자동 활성화
    - CodeQL 다중 언어 병렬 분석
    
  Dependabot 통합:
    - PR 자동 승인 워크플로우
    - Actions와 완벽 통합
    - 라이선스 검토 기능
    
  아티팩트 증명:
    - 공급망 보안 강화
    - SBOM 자동 생성
    - SARIF 업로드 통합
    
  엔터프라이즈 네트워크:
    - sec-GitHub-allowed-enterprise 헤더
    - Azure 프라이빗 네트워킹
    - 프록시/방화벽 고급 설정
```

### 보안 기능 활성화

```yaml
Security Features (2025년 확장):
  # Dependency scanning
  dependabot:
    enabled: true
    update_schedule: weekly
    automerge_minor: true
    
  # Code scanning
  code_scanning:
    enabled: true
    languages: [javascript, python, go]
    schedule: "0 0 * * 0"  # 매주 일요일
    
  # Secret scanning
  secret_scanning:
    enabled: true
    push_protection: true
    custom_patterns:
      - name: "Internal API Key"
        pattern: "INTERNAL_[A-Z0-9]{32}"
        
  # Security policies
  security_policy:
    location: .github/SECURITY.md
    vulnerability_reporting: enabled
```

### 2FA (Two-Factor Authentication) 강제

```yaml
Organization Settings:
  Security:
    - Require two-factor authentication: ✓
    - Grace period: 7 days
    - Notification: Email all members
    
  Enforcement:
    - Block access without 2FA
    - Remove members without 2FA after grace period
    - Audit 2FA status monthly
```

## 7. Audit와 Compliance (2025년 자동화)

### AI Compliance Engine (2025년 최신)

```yaml
자동 컴플라이언스:
  실시간 모니터링:
    - 24/7 규정 준수 체크
    - 이상 활동 즉시 감지
    - 예방적 조치 자동 실행
    
  AI 감사 보고서:
    - 자동 보고서 생성
    - 위험 요소 하이라이트
    - 개선 권장사항 제공
    
  규제 업데이트:
    - 새 규정 자동 반영
    - 정책 변경 알림
    - 준수 가이드 생성
```

### Audit Log 활용

```yaml
추적 가능한 이벤트:
  Repository:
    - 생성/삭제
    - 가시성 변경
    - 협업자 추가/제거
    - 브랜치 보호 규칙 변경
    
  Organization:
    - 멤버 초대/제거
    - 팀 생성/삭제
    - 권한 변경
    - 설정 변경
    
  Security:
    - 2FA 활성화/비활성화
    - SSH 키 추가/제거
    - PAT 생성/폐기
    - 보안 경고 해제
```

### Compliance 자동화

```yaml
# .github/workflows/compliance-check.yml
name: Compliance Check

on:
  schedule:
    - cron: '0 9 * * 1'  # 매주 월요일 9시
  workflow_dispatch:

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - name: Check branch protection
        uses: actions/github-script@v6
        with:
          script: |
            const repos = await github.paginate(
              github.rest.repos.listForOrg,
              { org: context.repo.owner }
            );
            
            for (const repo of repos) {
              const protection = await github.rest.repos.getBranchProtection({
                owner: context.repo.owner,
                repo: repo.name,
                branch: 'main'
              }).catch(() => null);
              
              if (!protection) {
                core.warning(`No protection on ${repo.name}`);
              }
            }
            
      - name: Check 2FA compliance
        run: |
          gh api orgs/${{ github.repository_owner }}/members \
            --jq '.[] | select(.two_factor_authentication == false) | .login' \
            > non-2fa-users.txt
            
      - name: Generate report
        run: |
          echo "# Compliance Report" > report.md
          echo "Date: $(date)" >> report.md
          # ... 추가 보고서 생성 로직
```

## 8. 외부 협업자 관리 (2025년 스마트화)

### Guest Collaborators 기능 (2025년 7월 신규)

```mermaid
graph TD
    A[Guest 초대] --> B[엔터프라이즈 정책 확인]
    B --> C[세분화된 권한 설정]
    C --> D[특정 저장소만 접근]
    D --> E[시간 제한 적용]
    E --> F[활동 모니터링]
    F --> G[자동 만료 처리]
    
    H[Guest 유형]
    H --> I[외부 개발자]
    H --> J[파트너사 직원]
    H --> K[컨설턴트]
```

### Outside Collaborator 전략

```yaml
협업자 추가 프로세스:
  1. 요청:
     - 목적 명시
     - 필요 권한 수준
     - 접근 기간
     
  2. 승인:
     - 팀 리드 승인
     - 보안팀 검토 (민감한 저장소)
     
  3. 권한 부여:
     - 최소 필요 권한
     - 특정 저장소만
     - 기간 제한
     
  4. 모니터링:
     - 활동 로그 확인
     - 정기적 권한 검토
     - 자동 만료 설정
```

### 임시 접근 관리

```bash
#!/bin/bash
# 임시 협업자 권한 부여 스크립트

REPO="org/repo"
USER="contractor-username"
PERMISSION="pull"  # pull, push, admin
DAYS=30

# 권한 부여
gh api repos/$REPO/collaborators/$USER \
  --method PUT \
  -f permission=$PERMISSION

# 캘린더에 만료일 추가
echo "권한 만료: $USER on $REPO" | at now + $DAYS days

# Slack 알림
curl -X POST $SLACK_WEBHOOK \
  -H 'Content-type: application/json' \
  --data "{\"text\":\"임시 권한 부여: $USER -> $REPO ($DAYS days)\"}"
```

## 9. 팀 워크플로우 자동화 (2025년 AI 통합)

### GitHub Copilot Workspace (2025년 7월 최신)

```yaml
Copilot Coding Agent:
  자율 작업:
    - Issue에서 작업 자동 픽업
    - PR 자동 생성 및 제출
    - 코멘트 기반 반복 개선
    
  지원 작업:
    - 버그 수정
    - 기능 개발
    - 문서화
    - 유지보수 작업
    
  멀티 모델 지원:
    - GPT-4.5, Claude 4.0 (프리뷰)
    - Gemini 2.5 Pro (프리뷰)
    - O3, O4 Mini (프리뷰)
    
  커스텀 설정:
    - .github/copilot-instructions.md
    - 팀별 AI 가이드라인
    - 프로젝트 특화 동작
```

### 자동 팀 할당

```yaml
# .github/ISSUE_TEMPLATE/config.yml
contact_links:
  - name: Frontend Issue
    url: https://github.com/org/repo/issues/new?labels=frontend&assignees=@org/frontend-team
    about: Frontend 관련 이슈
    
  - name: Backend Issue
    url: https://github.com/org/repo/issues/new?labels=backend&assignees=@org/backend-team
    about: Backend 관련 이슈
```

### PR 자동 리뷰어 할당

```yaml
# .github/auto_assign.yml
addReviewers: true
addAssignees: false

reviewers:
  defaults:
    - review-team
  groups:
    frontend:
      - frontend-reviewer-1
      - frontend-reviewer-2
    backend:
      - backend-reviewer-1
      - backend-reviewer-2

files:
  "src/components/**":
    - frontend
  "api/**":
    - backend

numberOfReviewers: 2
```

## 10. 모범 사례와 팁 (2025년 AI 시대)

### GitHub Projects 최신 기능 (2025년 7월)

```yaml
Projects 고급 필터링:
  새로운 필터:
    - @me: 나에게 할당/리뷰 요청
    - assignee:username
    - reviewer:username
    - label:"bug"
    - milestone:"v2.0"
    - 커스텀 필드 필터링
    
  자동화 워크플로우:
    - 고급 필터로 자동 추가
    - GraphQL API 통합
    - CLI project 스코프
    
  Actions 통합:
    - 프로젝트 상태 자동 업데이트
    - 이슈/PR 자동 분류
    - 팀 성과 메트릭 추적
```

### 권한 관리 체크리스트

```markdown
## 월간 권한 검토 체크리스트

### 조직 레벨
- [ ] 비활성 멤버 제거
- [ ] 2FA 미설정 사용자 확인
- [ ] 외부 협업자 권한 검토
- [ ] 팀 구성원 업데이트

### 저장소 레벨
- [ ] CODEOWNERS 파일 최신화
- [ ] 브랜치 보호 규칙 검토
- [ ] 불필요한 접근 권한 제거
- [ ] Deploy keys 확인

### 보안
- [ ] 만료된 토큰 제거
- [ ] 보안 경고 확인
- [ ] Audit log 검토
- [ ] 컴플라이언스 확인
```

### 효과적인 팀 구조

```yaml
Small Team (< 10):
  structure: flat
  teams:
    - developers
    - maintainers
  
Medium Team (10-50):
  structure: functional
  teams:
    - frontend
    - backend
    - devops
    - qa
    
Large Team (50+):
  structure: hierarchical
  teams:
    - engineering
      - frontend
        - web
        - mobile
      - backend
        - api
        - services
    - platform
      - infrastructure
      - security
```

### 권한 에스컬레이션 프로세스 (2025년 자동화)

```markdown
## AI 기반 긴급 권한 관리 (2025년 최신)

1. **요청 생성**
   - Issue 템플릿으로 요청
   - 목적과 기간 명시
   - 영향 범위 설명

2. **승인 프로세스**
   - 직속 매니저 승인
   - 보안팀 검토 (production 접근 시)
   - CTO 최종 승인 (critical 시스템)

3. **권한 부여**
   - 최소 필요 권한만 부여
   - 시간 제한 설정
   - 모든 활동 로깅

4. **사후 처리**
   - 자동 권한 회수
   - 활동 보고서 생성
   - 개선사항 도출
```

## 마무리

2025년 현재 GitHub의 팀 협업 기능은 AI의 도입으로 완전히 새로운 차원에 도달했습니다. 자동화된 권한 관리, 지능형 보안 시스템, 그리고 예측적 팀 관리가 가능해졌습니다.

### 2025년 7월 핵심 변화

| 영역 | 이전 | 2025년 현재 |
|------|------|-------------|
| 권한 관리 | 수동 설정 | Guest Collaborators + SAML/SCIM |
| 보안 | 기본 스캔 | CodeQL 다중언어 + 아티팩트 증명 |
| Copilot | 코드 제안 | 자율 Coding Agent |
| Projects | 기본 필터 | @me + GraphQL API |
| Actions | CI/CD | 엔터프라이즈 네트워크 + OIDC |

효과적인 팀 협업 설정은 보안과 생산성의 균형을 맞추는 것이 핵심입니다. 

중요 포인트:
- 최소 권한 원칙 적용
- 정기적인 권한 검토
- 자동화를 통한 일관성
- 명확한 프로세스와 문서화

잘 설계된 권한 체계는 팀의 성장과 함께 확장 가능하며, 보안 사고를 예방하면서도 개발 속도를 유지할 수 있게 합니다.

다음 편에서는 GitHub Actions를 활용한 CI/CD 파이프라인 구축에 대해 알아보겠습니다.

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
10. **Team 협업 설정** (현재 글)
11. GitHub Actions 입문 (예정)

---

*질문이나 피드백이 있으시다면 댓글로 남겨주세요!*