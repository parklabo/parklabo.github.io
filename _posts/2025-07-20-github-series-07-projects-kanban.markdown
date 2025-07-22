---
layout: post
title: "[GitHub 입문 #7] Projects로 프로젝트 관리: 칸반 보드 활용하기"
date: 2025-07-20 10:00:00 +0900
categories: [DevOps, GitHub]
tags: [github, projects, kanban, project-management, workflow, tutorial, series]
mermaid: true
---

## 들어가며

GitHub 입문 시리즈의 일곱 번째 시간입니다. 이번에는 GitHub Projects를 활용하여 프로젝트를 시각적으로 관리하는 방법을 알아보겠습니다. Projects는 이슈와 PR을 칸반 보드, 테이블, 로드맵 등 다양한 뷰로 관리할 수 있는 강력한 도구입니다.

## 1. GitHub Projects란?

GitHub Projects는 프로젝트 관리를 위한 유연하고 맞춤형 도구입니다. 2022년에 출시된 새로운 Projects는 다음과 같은 특징을 가집니다:

- **다양한 뷰**: 테이블, 보드, 로드맵 뷰 지원
- **커스텀 필드**: 프로젝트에 맞는 메타데이터 추가
- **자동화**: 워크플로우 자동화 지원
- **인사이트**: 차트와 분석 기능
- **통합**: Issues, PR과 완벽한 통합

## 2. Project 생성하기

### 새 프로젝트 만들기

1. GitHub 프로필 → Projects 탭
2. "New project" 클릭
3. 템플릿 선택 또는 빈 프로젝트 시작

### 프로젝트 템플릿

```yaml
기본 제공 템플릿:
- Team backlog: 스프린트 관리
- Feature planning: 기능 개발 계획
- Bug tracker: 버그 추적
- Roadmap: 로드맵 관리
```

### 프로젝트 설정

```yaml
프로젝트 이름: Product Development
설명: 제품 개발 진행 상황 관리
접근 권한: 
  - Private: 팀 멤버만
  - Public: 모든 사람이 볼 수 있음
README: 프로젝트 가이드라인 작성
```

## 3. Views (뷰) 활용하기

### Table View (테이블 뷰)

스프레드시트 형태로 데이터를 관리:

```yaml
열 구성 예시:
- Title: 이슈/PR 제목
- Status: Todo, In Progress, Done
- Assignees: 담당자
- Priority: High, Medium, Low
- Sprint: Sprint 1, Sprint 2
- Estimate: Story Points
- Labels: 라벨
```

### Board View (보드 뷰)

칸반 스타일의 시각적 관리:

```
┌─────────────┬─────────────┬─────────────┬─────────────┐
│   Backlog   │    Todo     │ In Progress │    Done     │
├─────────────┼─────────────┼─────────────┼─────────────┤
│ ○ Issue #1  │ ○ Issue #3  │ ○ Issue #5  │ ✓ Issue #7  │
│ ○ Issue #2  │ ○ Issue #4  │ ○ PR #6     │ ✓ Issue #8  │
│             │             │             │ ✓ PR #9     │
└─────────────┴─────────────┴─────────────┴─────────────┘
```

### Roadmap View (로드맵 뷰)

타임라인 기반 계획 관리:

```
2025 Q1 ─────────────────────────────────────────────────►
         Jan          Feb          Mar
         │            │            │
Feature A ████████████░░░░░░░░░░░░░
Feature B      ██████████████░░░░░
Feature C                 ████████████████
```

## 4. Custom Fields (커스텀 필드)

### 필드 타입

```yaml
Text: 자유 텍스트 입력
Number: 숫자 값
Date: 날짜 선택
Single select: 드롭다운 선택
Iteration: 스프린트/반복 주기
```

### 커스텀 필드 예시

```yaml
# 개발 프로젝트용 필드
Priority:
  type: single_select
  options: [🔴 Critical, 🟠 High, 🟡 Medium, 🟢 Low]

Story Points:
  type: number
  description: 작업 복잡도 (1-13)

Sprint:
  type: iteration
  duration: 2 weeks

Component:
  type: single_select
  options: [Frontend, Backend, Database, DevOps]

Review Status:
  type: single_select
  options: [Pending, In Review, Approved, Changes Requested]
```

## 5. 워크플로우 자동화

### 기본 자동화

Projects 설정에서 활성화:

```yaml
자동화 규칙:
- Item added to project → Set status to "Todo"
- Item closed → Set status to "Done"
- Pull request merged → Set status to "Done"
- Item reopened → Set status to "In Progress"
```

### GitHub Actions와 연동

```yaml
# .github/workflows/project-automation.yml
name: Project Card Automation

on:
  issues:
    types: [opened, closed, assigned]
  pull_request:
    types: [opened, closed, review_requested]

jobs:
  update-project:
    runs-on: ubuntu-latest
    steps:
      - name: Update project card
        uses: actions/github-script@v6
        with:
          script: |
            const projectNumber = 1;
            const fieldName = 'Status';
            
            if (context.payload.action === 'opened') {
              // 새 이슈/PR을 Todo로 설정
              await github.graphql(`
                mutation {
                  updateProjectV2ItemFieldValue(
                    input: {
                      projectId: "PROJECT_ID"
                      itemId: "ITEM_ID"
                      fieldId: "FIELD_ID"
                      value: { singleSelectOptionId: "TODO_OPTION_ID" }
                    }
                  ) {
                    projectV2Item {
                      id
                    }
                  }
                }
              `);
            }
```

## 6. 프로젝트 관리 전략

### 스프린트 기반 관리

```yaml
Sprint 1 (2025-01-15 ~ 2025-01-29):
  Todo:
    - [ ] 사용자 인증 API
    - [ ] 로그인 UI
    - [ ] 테스트 작성
  In Progress:
    - [ ] 데이터베이스 설계
  Done:
    - [x] 요구사항 분석
    
  Velocity: 21 points
  Burndown: ▓▓▓▓▓▓▓░░░░░░░
```

### 칸반 플로우

```yaml
WIP Limits:
  Todo: ∞
  In Progress: 3 per person
  Review: 5 total
  Done: Archive after sprint

Flow Rules:
  - One assignee per card
  - Move to Review when PR created
  - Require approval before Done
  - Daily standup review
```

## 7. 필터와 그룹화

### 필터링 예시

```sql
# 내가 담당한 진행 중인 작업
assignee:@me status:"In Progress"

# 이번 스프린트의 높은 우선순위
sprint:"Sprint 3" priority:"High"

# 리뷰 대기 중인 PR
is:pr status:"Review"

# 기한이 지난 작업
status:!"Done" due:<@today
```

### 그룹화 옵션

```yaml
Group by:
  - Status: 진행 상태별 그룹
  - Assignee: 담당자별 그룹
  - Priority: 우선순위별 그룹
  - Sprint: 스프린트별 그룹
  - Repository: 저장소별 그룹
```

## 8. Insights와 분석

### 번다운 차트

```
Story Points
40 │\
35 │ \
30 │  \____
25 │       \___
20 │           \
15 │            \___
10 │                \
5  │                 \__
0  └─────────────────────►
   Day 1    5    10    14
   
이상적 번다운 ----
실제 번다운 ────
```

### 벨로시티 차트

```
Points Completed
30 │    ┌─┐
25 │ ┌─┐│ │  ┌─┐
20 │ │ ││ │┌─┐│ │
15 │ │ ││ ││ ││ │
10 │ │ ││ ││ ││ │
5  │ │ ││ ││ ││ │
0  └─┴─┴┴─┴┴─┴┴─┴─
   S1 S2 S3 S4 S5
   
평균 벨로시티: 22 points
```

## 9. 팀 협업 시나리오

### 시나리오 1: 스프린트 플래닝

```markdown
## Sprint Planning Meeting

1. **백로그 리뷰**
   - Projects의 Backlog 컬럼 확인
   - 우선순위 기준 정렬

2. **스토리 포인트 추정**
   - 각 이슈에 Story Points 필드 설정
   - Planning Poker 진행

3. **스프린트 할당**
   - Sprint 필드를 현재 스프린트로 설정
   - Todo 컬럼으로 이동

4. **담당자 지정**
   - Assignee 설정
   - 작업 부하 확인

5. **스프린트 시작**
   - 스프린트 기간 설정
   - 번다운 차트 초기화
```

### 시나리오 2: 일일 스탠드업

```yaml
Daily Standup View:
  Filter: sprint:current
  Group by: Assignee
  Sort: Updated (newest first)

각 팀원별 확인사항:
  - Yesterday: Done 컬럼의 작업
  - Today: In Progress 컬럼의 작업
  - Blockers: Blocked 라벨이 있는 작업
```

## 10. 프로젝트 템플릿 만들기

### 커스텀 템플릿 생성

```yaml
name: Product Development Template
description: 제품 개발을 위한 표준 템플릿

views:
  - name: Sprint Board
    type: board
    group_by: Status
    
  - name: Planning Table  
    type: table
    fields: [Title, Status, Priority, Sprint, Points, Assignee]
    
  - name: Roadmap
    type: roadmap
    date_field: Target Date

fields:
  - name: Status
    type: single_select
    options: [Backlog, Todo, In Progress, Review, Done]
    
  - name: Priority
    type: single_select
    options: [Critical, High, Medium, Low]
    
  - name: Story Points
    type: number
    
  - name: Sprint
    type: iteration
    duration: 2 weeks

automation:
  - when: item_added
    set: status = "Backlog"
    
  - when: item_closed
    set: status = "Done"
```

### 템플릿 공유

```bash
# 프로젝트 설정 내보내기
1. Project settings → Menu → Export
2. JSON 파일 다운로드
3. 팀과 공유

# 템플릿으로 새 프로젝트 생성
1. New project → Import
2. JSON 파일 업로드
3. 프로젝트 이름 설정
```

## 11. 모범 사례

### 프로젝트 구조화

```yaml
프로젝트 계층:
  Organization Projects:
    - Company Roadmap (전체 로드맵)
    - Product Development (제품 개발)
    
  Team Projects:
    - Frontend Team Board
    - Backend Team Board
    - DevOps Pipeline
    
  Personal Projects:
    - My Tasks
    - Learning Goals
```

### 효과적인 운영 팁

```markdown
1. **명확한 정의**
   - Done의 정의 문서화
   - 각 상태의 의미 명확화

2. **정기적인 정리**
   - 완료된 작업 아카이브
   - 오래된 이슈 검토

3. **일관성 유지**
   - 필드 사용 규칙 준수
   - 네이밍 컨벤션 통일

4. **자동화 활용**
   - 반복 작업 자동화
   - 알림 설정

5. **지속적 개선**
   - 회고를 통한 프로세스 개선
   - 메트릭 기반 의사결정
```

## 마무리

GitHub Projects는 단순한 작업 관리를 넘어 팀의 전체적인 개발 프로세스를 시각화하고 최적화할 수 있는 강력한 도구입니다. 

핵심은:
- 팀에 맞는 워크플로우 설계
- 적절한 자동화 활용
- 지속적인 프로세스 개선
- 데이터 기반 의사결정

다음 편에서는 코드 리뷰 문화와 효과적인 PR 리뷰 방법에 대해 알아보겠습니다.

## 시리즈 목차
1. [GitHub 시작하기](/posts/github-series-01-getting-started/)
2. [Repository 기초](/posts/github-series-02-repository-basics/)
3. [Git 기본 명령어](/posts/github-series-03-git-basic-commands/)
4. [Branch와 Merge](/posts/github-series-04-branch-and-merge/)
5. [Fork와 Pull Request](/posts/github-series-05-fork-and-pull-request/)
6. [Issues 활용법](/posts/github-series-06-issues-management/)
7. **Projects로 프로젝트 관리** (현재 글)
8. Code Review 잘하기 (예정)

---

*질문이나 피드백이 있으시다면 댓글로 남겨주세요!*