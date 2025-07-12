# Park-Labo Blog

[![Build and Deploy](https://github.com/parklabo/parklabo.github.io/actions/workflows/pages-deploy.yml/badge.svg)](https://github.com/parklabo/parklabo.github.io/actions/workflows/pages-deploy.yml)
[![Jekyll](https://img.shields.io/badge/Jekyll-4.3.4-blue)](https://jekyllrb.com/)
[![Theme](https://img.shields.io/badge/Theme-Chirpy-brightgreen)](https://github.com/cotes2020/jekyll-theme-chirpy)

개발자 박영수의 기술 블로그입니다.

🔗 **블로그 주소**: [https://parklabo.github.io](https://parklabo.github.io)

## 📝 About

프로그래밍, 개발 경험, 기술 트렌드를 공유하는 개인 기술 블로그입니다.

### 주요 주제
- 웹 개발 (JavaScript, React, Node.js)
- Chrome Extension 개발
- Jekyll & GitHub Pages
- 개발 도구 및 생산성
- 오픈소스 프로젝트

## 🛠️ Tech Stack

- **Static Site Generator**: [Jekyll](https://jekyllrb.com/) v4.3.4
- **Theme**: [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) v7.3.0
- **Hosting**: [GitHub Pages](https://pages.github.com/)
- **CI/CD**: GitHub Actions
- **Analytics**: Google Analytics (예정)

## 🚀 로컬 개발 환경 설정

### Prerequisites

- Ruby 3.0 이상
- Bundler
- Git

### 설치 및 실행

```bash
# 저장소 클론
git clone https://github.com/parklabo/parklabo.github.io.git
cd parklabo.github.io

# 의존성 설치
bundle install

# 로컬 서버 실행
bundle exec jekyll serve

# 브라우저에서 http://localhost:4000 접속
```

### 개발 명령어

```bash
# 사이트 빌드
bundle exec jekyll build

# 실시간 리로드와 함께 서버 실행
bundle exec jekyll serve --livereload

# 드래프트 포스트 포함하여 실행
bundle exec jekyll serve --drafts
```

## 📋 포스트 작성 가이드

### 새 포스트 생성

`_posts` 디렉토리에 다음 형식으로 파일을 생성합니다:

```
YYYY-MM-DD-post-title.markdown
```

### Front Matter 템플릿

```yaml
---
layout: post
title: "포스트 제목"
date: 2025-07-12 20:00:00 +0900
categories: [Category1, Category2]
tags: [tag1, tag2, tag3]
image:
  path: /assets/img/posts/post-image.png
  alt: 이미지 설명
---
```

### 이미지 추가

```markdown
![이미지 설명](/assets/img/posts/image.png)
```

### 코드 블록

````markdown
```javascript
console.log('Hello, World!');
```
````

## 🔧 커스터마이징

### 프로필 정보 수정

`_config.yml` 파일에서 다음 항목들을 수정:

- `title`: 블로그 제목
- `tagline`: 부제목
- `description`: 블로그 설명
- `url`: 블로그 URL
- `social`: 소셜 미디어 정보

### 아바타 이미지 변경

`/assets/img/avatar.jpg` 파일을 교체

### 파비콘 변경

`/assets/img/favicons/` 디렉토리의 파일들을 교체

## 📁 프로젝트 구조

```
parklabo.github.io/
├── _config.yml          # 사이트 설정
├── _data/              # 데이터 파일 (언어, 메뉴 등)
├── _includes/          # 재사용 가능한 컴포넌트
├── _layouts/           # 페이지 레이아웃
├── _posts/             # 블로그 포스트
├── _sass/              # SCSS 스타일시트
├── _tabs/              # 탭 페이지 (About, Archives 등)
├── assets/             # 정적 자원 (이미지, CSS, JS)
├── _javascript/        # JavaScript 소스 파일
└── .github/            # GitHub Actions 워크플로우
```

## 🤝 기여하기

이슈나 제안사항이 있으시면 [Issues](https://github.com/parklabo/parklabo.github.io/issues)에 등록해주세요.

## 📄 라이선스

이 블로그의 콘텐츠는 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 라이선스를 따릅니다.

Jekyll과 Chirpy 테마는 각각의 라이선스를 따릅니다:
- Jekyll: [MIT License](https://github.com/jekyll/jekyll/blob/master/LICENSE)
- Chirpy: [MIT License](https://github.com/cotes2020/jekyll-theme-chirpy/blob/master/LICENSE)

## 📞 Contact

- **Email**: finitenumber@gmail.com
- **GitHub**: [@parklabo](https://github.com/parklabo)
- **Twitter**: [@YoungsuPark6](https://twitter.com/YoungsuPark6)

---

Made with ❤️ by YonYon