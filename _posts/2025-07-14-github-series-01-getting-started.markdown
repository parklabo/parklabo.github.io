---
layout: post
title: "[GitHub 입문 #1] GitHub 시작하기: 계정 생성부터 프로필 꾸미기까지"
date: 2025-07-14 22:00:00 +0900
categories: [DevOps, GitHub]
tags: [github, git, tutorial, series, beginner]
mermaid: true
---

## 시리즈 소개

"GitHub 입문" 시리즈의 첫 번째 글입니다. 이 시리즈에서는 GitHub의 기초부터 고급 기능까지 단계별로 알아보겠습니다. 오늘은 GitHub 여정의 첫 걸음인 계정 생성과 프로필 설정에 대해 자세히 다뤄보겠습니다.

## 1. GitHub이란?

GitHub은 Git 버전 관리 시스템을 기반으로 하는 웹 호스팅 서비스입니다. 개발자들이 코드를 저장, 관리, 공유하고 협업할 수 있는 플랫폼으로, 오픈소스 프로젝트의 중심지이자 개발자 포트폴리오의 대명사가 되었습니다.

### 왜 GitHub을 사용해야 할까요?

- **버전 관리**: 코드의 변경 이력을 체계적으로 관리
- **협업**: 팀원들과 효율적인 코드 공유 및 리뷰
- **포트폴리오**: 개발 실력을 보여줄 수 있는 최고의 플랫폼
- **오픈소스**: 전 세계 개발자들의 프로젝트 참여 및 기여
- **무료**: 개인 프로젝트는 무제한 무료 사용 가능

## 2. GitHub 계정 생성하기

### Step 1: GitHub 홈페이지 접속
1. [github.com](https://github.com)에 접속합니다
2. 우측 상단의 "Sign up" 버튼을 클릭합니다

### Step 2: 계정 정보 입력
```
Username: 고유한 사용자명 (변경 가능하지만 신중히 선택)
Email: 알림을 받을 이메일 주소
Password: 15자 이상 또는 8자 이상의 복잡한 비밀번호
```

> **Tip**: Username은 나중에 `github.com/username` 형태로 프로필 URL이 되므로 전문적이고 기억하기 쉬운 이름을 선택하세요.

### Step 3: 이메일 인증
- 가입 시 입력한 이메일로 전송된 인증 코드를 입력합니다
- 인증이 완료되면 GitHub 사용을 시작할 수 있습니다

## 3. 기본 설정하기

### 프로필 사진 설정
1. 우측 상단 프로필 아이콘 클릭 → "Settings"
2. "Profile" 섹션에서 "Upload a photo" 클릭
3. 전문적인 프로필 사진 업로드 (권장: 정사각형, 최소 200x200px)

### 기본 정보 입력
```yaml
Name: 실명 또는 활동명
Bio: 간단한 자기소개 (최대 160자)
Company: 소속 회사 또는 학교
Location: 위치 정보
Website: 개인 웹사이트 또는 블로그
```

### 이메일 설정
- Settings → Emails에서 추가 이메일 등록 가능
- "Keep my email addresses private" 옵션으로 이메일 주소 보호
- "Block command line pushes that expose my email" 활성화 권장

## 4. 프로필 꾸미기

### README 프로필 만들기

GitHub은 사용자명과 동일한 이름의 특별한 저장소를 지원합니다. 이 저장소의 README.md 파일은 프로필 페이지에 표시됩니다.

1. 새 저장소 생성 (이름: 본인의 username과 동일)
2. "Add a README file" 체크
3. README.md 파일 편집

### 기본 README 템플릿

```markdown
# 안녕하세요! 👋

저는 [이름]입니다.

## 🚀 About Me
- 🔭 현재 작업 중인 프로젝트: [프로젝트명]
- 🌱 현재 학습 중: [기술/언어]
- 👯 협업하고 싶은 분야: [관심 분야]
- 💬 질문 환영: [전문 분야]
- 📫 연락처: [이메일/SNS]

## 🛠️ Tech Stack
- Languages: Python, JavaScript, Java
- Frontend: React, Vue.js
- Backend: Node.js, Django
- Database: MySQL, MongoDB
- Tools: Git, Docker, AWS

## 📊 GitHub Stats
![GitHub stats](https://github-readme-stats.vercel.app/api?username=YOUR_USERNAME&show_icons=true&theme=radical)

## 📝 Latest Blog Posts
<!-- BLOG-POST-LIST:START -->
<!-- BLOG-POST-LIST:END -->
```

### 프로필 뱃지 추가하기

shields.io를 활용하여 기술 스택을 시각적으로 표현할 수 있습니다:

```markdown
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)
![React](https://img.shields.io/badge/React-61DAFB?style=for-the-badge&logo=react&logoColor=black)
```

## 5. 보안 설정

### 2단계 인증(2FA) 활성화
1. Settings → Password and authentication
2. "Enable two-factor authentication" 클릭
3. 인증 앱(Google Authenticator, Authy 등) 사용 권장

### SSH 키 설정
터미널에서 SSH 키 생성:
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

생성된 공개 키를 GitHub에 등록:
1. Settings → SSH and GPG keys
2. "New SSH key" 클릭
3. 공개 키 내용 붙여넣기

## 6. 첫 저장소 만들기

### 저장소 생성
1. 우측 상단 "+" 버튼 → "New repository"
2. Repository name: 의미 있는 이름 선택
3. Description: 프로젝트 설명 추가
4. Public/Private 선택
5. "Initialize with README" 체크 권장

### 첫 커밋하기
```bash
git clone https://github.com/username/repository-name.git
cd repository-name
echo "# My First Project" >> README.md
git add README.md
git commit -m "docs: Update README"
git push origin main
```

## 7. 프로필 최적화 팁

### 활동 내역 채우기
- 매일 조금씩이라도 커밋하여 잔디 심기
- 오픈소스 프로젝트에 기여
- 자신의 프로젝트를 꾸준히 업데이트

### 팔로우 & 스타
- 관심 있는 개발자와 프로젝트 팔로우
- 유용한 저장소에 스타 표시
- 트렌딩 저장소 확인하며 최신 기술 동향 파악

## 마무리

GitHub 계정을 만들고 프로필을 설정하는 것은 개발자로서의 온라인 정체성을 구축하는 첫 걸음입니다. 꾸준히 활동하고 프로젝트를 공유하다 보면 자연스럽게 네트워크가 형성되고 기회가 찾아올 것입니다.

다음 편에서는 Repository를 효과적으로 관리하는 방법과 README 작성법에 대해 자세히 알아보겠습니다.

## 시리즈 목차
1. **GitHub 시작하기** (현재 글)
2. [Repository 기초](/posts/github-series-02-repository-basics/)
3. Git 기본 명령어 (예정)
4. Branch와 Merge (예정)
5. Fork와 Pull Request (예정)

---

*질문이나 피드백이 있으시다면 댓글로 남겨주세요!*