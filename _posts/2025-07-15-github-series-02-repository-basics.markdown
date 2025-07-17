---
layout: post
title: "[GitHub 입문 #2] Repository 기초: 생성, 클론, README 작성법"
date: 2025-07-15 10:00:00 +0900
categories: [GitHub, Tutorial]
tags: [github, git, repository, readme, tutorial, series]
---

## 들어가며

GitHub 입문 시리즈의 두 번째 시간입니다. 이번에는 GitHub의 핵심인 Repository(저장소)에 대해 알아보겠습니다. Repository는 프로젝트의 모든 파일과 변경 이력을 저장하는 공간으로, GitHub 활동의 기본 단위입니다.

## 1. Repository란?

Repository(줄여서 Repo)는 프로젝트의 파일들과 각 파일의 변경 이력을 저장하는 디렉토리입니다. Git이 관리하는 프로젝트 폴더라고 생각하면 쉽습니다.

### Repository의 구성 요소
- **파일과 폴더**: 프로젝트의 소스 코드, 문서, 이미지 등
- **커밋 히스토리**: 파일들의 변경 이력
- **브랜치**: 독립적인 작업 공간
- **이슈와 PR**: 협업을 위한 소통 도구
- **설정과 메타데이터**: 프로젝트 설정 정보

## 2. Repository 생성하기

### 웹에서 생성하기

1. GitHub 홈페이지 우측 상단 "+" 버튼 클릭
2. "New repository" 선택
3. 필수 정보 입력:

```yaml
Repository name: my-awesome-project
Description: 프로젝트에 대한 간단한 설명
Public/Private: 공개 여부 선택
Initialize with:
  - README file (권장)
  - .gitignore (프로젝트 언어에 맞게 선택)
  - License (오픈소스라면 필수)
```

### Repository 이름 규칙
- 소문자와 하이픈(-) 사용 권장
- 의미 있고 기억하기 쉬운 이름
- 프로젝트의 목적을 나타내는 이름

좋은 예시:
- `todo-app`
- `react-portfolio`
- `python-web-scraper`

피해야 할 예시:
- `test123`
- `my-project-final-final`
- `untitled-1`

## 3. Repository 클론하기

### HTTPS로 클론
```bash
git clone https://github.com/username/repository-name.git
cd repository-name
```

### SSH로 클론 (추천)
```bash
git clone git@github.com:username/repository-name.git
cd repository-name
```

### 특정 브랜치 클론
```bash
git clone -b branch-name https://github.com/username/repository-name.git
```

### 얕은 클론 (대용량 저장소)
```bash
git clone --depth 1 https://github.com/username/repository-name.git
```

## 4. README 파일 작성법

README.md는 프로젝트의 첫인상입니다. 방문자가 가장 먼저 보는 문서이므로 신경 써서 작성해야 합니다.

### 기본 구조

```markdown
# 프로젝트 이름

프로젝트에 대한 간단한 한 줄 설명

## 📋 목차
- [소개](#소개)
- [주요 기능](#주요-기능)
- [설치 방법](#설치-방법)
- [사용 방법](#사용-방법)
- [기여하기](#기여하기)
- [라이선스](#라이선스)

## 🎯 소개
프로젝트의 목적과 해결하려는 문제에 대한 설명

## ✨ 주요 기능
- 기능 1: 설명
- 기능 2: 설명
- 기능 3: 설명

## 🚀 설치 방법

### 요구사항
- Node.js 14.0 이상
- npm 또는 yarn

### 설치
```bash
# 저장소 클론
git clone https://github.com/username/project-name.git

# 디렉토리 이동
cd project-name

# 의존성 설치
npm install
```

## 💻 사용 방법

### 개발 서버 실행
```bash
npm run dev
```

### 빌드
```bash
npm run build
```

## 🤝 기여하기
프로젝트 기여는 언제나 환영입니다! 

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 📄 라이선스
이 프로젝트는 MIT 라이선스를 따릅니다. 자세한 내용은 [LICENSE](LICENSE) 파일을 참조하세요.

## 📞 연락처
- 이메일: your.email@example.com
- 블로그: https://your-blog.com
```

### README 작성 팁

#### 1. 뱃지 활용하기
```markdown
![GitHub stars](https://img.shields.io/github/stars/username/repo-name)
![GitHub forks](https://img.shields.io/github/forks/username/repo-name)
![GitHub issues](https://img.shields.io/github/issues/username/repo-name)
![GitHub license](https://img.shields.io/github/license/username/repo-name)
```

#### 2. 스크린샷/GIF 추가
```markdown
## 📸 스크린샷
![메인 화면](./screenshots/main.png)
![기능 데모](./screenshots/demo.gif)
```

#### 3. 기술 스택 시각화
```markdown
## 🛠️ 사용 기술
### Frontend
- React 18
- TypeScript
- Tailwind CSS

### Backend
- Node.js
- Express
- MongoDB

### DevOps
- Docker
- GitHub Actions
- AWS
```

## 5. .gitignore 파일 활용

.gitignore 파일은 Git이 추적하지 않을 파일들을 지정합니다.

### 기본 .gitignore 예시
```gitignore
# 운영체제 관련
.DS_Store
Thumbs.db

# IDE 관련
.vscode/
.idea/
*.swp
*.swo

# 의존성
node_modules/
vendor/
venv/

# 빌드 결과물
dist/
build/
*.pyc
__pycache__/

# 환경 설정
.env
.env.local
config/secrets.yml

# 로그
*.log
logs/

# 임시 파일
*.tmp
*.temp
.cache/
```

### 언어별 .gitignore 템플릿
GitHub에서 제공하는 템플릿 활용:
- [github.com/github/gitignore](https://github.com/github/gitignore)

## 6. 라이선스 선택하기

오픈소스 프로젝트라면 라이선스는 필수입니다.

### 주요 라이선스 비교

| 라이선스 | 상업적 사용 | 수정 | 배포 | 특허 사용 | 개인 사용 |
|---------|-----------|-----|-----|----------|---------|
| MIT | ✅ | ✅ | ✅ | ❌ | ✅ |
| Apache 2.0 | ✅ | ✅ | ✅ | ✅ | ✅ |
| GPL v3 | ✅ | ✅ | ✅ | ✅ | ✅ |
| BSD 3 | ✅ | ✅ | ✅ | ❌ | ✅ |

### 라이선스 추가 방법
1. Repository 메인 페이지에서 "Add file" → "Create new file"
2. 파일명: `LICENSE`
3. "Choose a license template" 버튼 클릭
4. 적절한 라이선스 선택

## 7. Repository 설정 최적화

### 기본 설정
1. Settings 탭 진입
2. 주요 설정 항목:
   - **Description**: 프로젝트 설명 추가
   - **Website**: 관련 웹사이트 URL
   - **Topics**: 검색 가능한 태그 추가

### 기능 설정
- **Issues**: 버그 리포트와 기능 요청
- **Projects**: 프로젝트 관리 보드
- **Wiki**: 상세 문서화
- **Discussions**: 커뮤니티 토론

### 보안 설정
- **Security policy**: 보안 정책 작성
- **Dependabot**: 의존성 자동 업데이트
- **Code scanning**: 코드 취약점 검사

## 8. 첫 커밋 만들기

### 로컬에서 작업하기
```bash
# 파일 생성/수정
echo "# Hello World" > hello.md

# 변경사항 확인
git status

# 스테이징
git add hello.md

# 커밋
git commit -m "docs: Add hello.md"

# 푸시
git push origin main
```

### 커밋 메시지 컨벤션
```
<type>: <subject>

<body>

<footer>
```

Type 예시:
- `feat`: 새로운 기능
- `fix`: 버그 수정
- `docs`: 문서 수정
- `style`: 코드 포맷팅
- `refactor`: 코드 리팩토링
- `test`: 테스트 코드
- `chore`: 빌드 업무, 패키지 매니저 설정 등

## 9. Repository 관리 팁

### 이슈 템플릿 만들기
`.github/ISSUE_TEMPLATE/` 디렉토리에 템플릿 파일 생성:

```markdown
---
name: Bug report
about: Create a report to help us improve
title: '[BUG] '
labels: bug
assignees: ''
---

## 버그 설명
버그에 대한 명확하고 간결한 설명

## 재현 단계
1. '...'로 이동
2. '...' 클릭
3. '...' 까지 스크롤
4. 오류 확인

## 예상 동작
예상했던 동작에 대한 설명

## 스크린샷
해당되는 경우 스크린샷 추가

## 환경
- OS: [예: iOS]
- Browser: [예: chrome, safari]
- Version: [예: 22]
```

### PR 템플릿 만들기
`.github/pull_request_template.md`:

```markdown
## 변경 사항
- 변경 사항에 대한 설명

## 변경 유형
- [ ] 버그 수정
- [ ] 새로운 기능
- [ ] 문서 업데이트
- [ ] 성능 개선

## 체크리스트
- [ ] 코드가 프로젝트 스타일 가이드를 따름
- [ ] 셀프 리뷰 완료
- [ ] 코드에 주석 추가 (특히 복잡한 부분)
- [ ] 문서 업데이트 완료
- [ ] 테스트 추가/업데이트 완료
```

## 마무리

Repository는 GitHub에서 모든 작업의 중심입니다. 잘 구성된 Repository는 프로젝트의 신뢰도를 높이고 협업을 원활하게 만듭니다. 특히 README 파일은 프로젝트의 얼굴이므로 시간을 들여 잘 작성하는 것이 중요합니다.

다음 편에서는 Git의 기본 명령어들을 실습과 함께 자세히 알아보겠습니다.

## 시리즈 목차
1. [GitHub 시작하기](/posts/github-series-01-getting-started/)
2. **Repository 기초** (현재 글)
3. Git 기본 명령어 (예정)
4. Branch와 Merge (예정)
5. Fork와 Pull Request (예정)

---

*질문이나 피드백이 있으시다면 댓글로 남겨주세요!*