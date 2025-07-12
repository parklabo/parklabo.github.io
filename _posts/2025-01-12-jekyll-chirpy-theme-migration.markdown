---
layout: post
title: "Jekyll 블로그를 Chirpy 테마로 전환하기: 완벽 가이드"
date: 2025-07-12 20:00:00 +0900
categories: [Blog, Jekyll]
tags: [jekyll, chirpy, github-pages, blog, tutorial]
image:
  path: /assets/img/posts/chirpy-theme.png
  alt: Jekyll Chirpy Theme
---

## 들어가며

3년 전에 만든 Jekyll 블로그가 있었습니다. 기본 Minima 테마를 사용하고 있었는데, 시간이 지나면서 더 현대적이고 기능이 풍부한 테마로 바꾸고 싶어졌습니다. 여러 테마를 검토한 끝에 **Chirpy**를 선택했습니다.

이 글에서는 기존 Jekyll 블로그를 Chirpy 테마로 전환하는 과정과 겪었던 문제들, 그리고 해결 방법을 상세히 공유하겠습니다.

## Chirpy 테마를 선택한 이유

1. **깔끔한 디자인**: 미니멀하면서도 필요한 기능은 모두 갖춘 디자인
2. **다크 모드**: 시스템 설정에 따라 자동으로 전환
3. **강력한 기능**: TOC, 검색, 카테고리/태그 분류 등
4. **SEO 최적화**: 구조화된 데이터와 메타 태그 자동 생성
5. **PWA 지원**: 오프라인에서도 접근 가능

## 전환 과정

### 1. 현재 상태 백업

가장 먼저 할 일은 백업입니다. 혹시 모를 상황에 대비해 현재 블로그 콘텐츠를 안전하게 보관합니다.

```bash
# 백업 폴더 생성
mkdir -p backup

# 중요 파일들 백업
cp -r _posts backup/
cp _config.yml backup/
cp -r assets backup/
```

### 2. Jekyll 및 의존성 업데이트

3년이나 된 블로그라 먼저 Jekyll과 모든 의존성을 최신 버전으로 업데이트했습니다.

```bash
# Gemfile 수정
gem "jekyll", "~> 4.3.3"
gem "jekyll-feed", "~> 0.17"
gem "webrick", "~> 1.8"

# Ruby 3.4 경고 해결을 위한 gem 추가
gem "csv"
gem "base64"
```

```bash
# 의존성 업데이트
bundle update
```

### 3. Chirpy 테마 설치

#### Chirpy 테마 다운로드

```bash
# Chirpy 테마 최신 버전(v7.3.0) 다운로드
git clone --depth 1 --branch v7.3.0 https://github.com/cotes2020/jekyll-theme-chirpy.git chirpy-temp
```

#### 기존 테마 파일 제거 및 새 파일 복사

```bash
# 기존 테마 관련 파일 제거
rm -rf _layouts _includes _sass assets _data _tabs _plugins index.html 404.html about.markdown

# Chirpy 테마 파일 복사
cp -r chirpy-temp/_layouts .
cp -r chirpy-temp/_includes .
cp -r chirpy-temp/_sass .
cp -r chirpy-temp/assets .
cp -r chirpy-temp/_data .
cp -r chirpy-temp/_tabs .
cp -r chirpy-temp/_plugins .
cp -r chirpy-temp/_javascript .
cp chirpy-temp/index.html .
```

### 4. 설정 파일 구성

`_config.yml` 파일을 Chirpy 테마에 맞게 수정합니다:

```yaml
# 테마 설정
theme: jekyll-theme-chirpy

# 기본 정보
lang: ko-KR
timezone: Asia/Seoul

title: Park-Labo
tagline: 개발자의 기술 블로그
description: >-
  개발자 박영수의 기술 블로그입니다. 
  프로그래밍, 개발 경험, 기술 공유를 다룹니다.

url: "https://parklabo.github.io"

# 소셜 미디어
github:
  username: parklabo
twitter:
  username: YoungsuPark6

social:
  name: Youngsu Park
  email: finitenumber@gmail.com
  links:
    - https://twitter.com/YoungsuPark6
    - https://github.com/parklabo
```

### 5. Gemfile 수정

Chirpy 테마를 사용하기 위해 Gemfile을 수정합니다:

```ruby
source "https://rubygems.org"

gem "jekyll", "~> 4.3"
gem "jekyll-theme-chirpy", "~> 7.3"
gem "html-proofer", "~> 5.0", group: :test

# Windows 지원
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Ruby 3.4 warnings 해결
gem "csv"
gem "base64"
```

### 6. 기존 포스트 마이그레이션

기존 포스트들을 Chirpy 형식에 맞게 수정합니다:

```yaml
# 기존 형식
---
layout: post
title: "제목"
date: 2023-06-07 09:00:00 +0900
categories: jekyll update
---

# Chirpy 형식
---
layout: post
title: "제목"
date: 2023-06-07 09:00:00 +0900
categories: [Git, GitHub]  # 배열 형식으로 변경
tags: [git, github, tutorial]  # 태그 추가
---
```

## 트러블슈팅

### 문제 1: GitHub Pages 빌드 실패

GitHub에 푸시했더니 다음과 같은 에러가 발생했습니다:

```
The jekyll-theme-chirpy theme could not be found.
```

**원인**: GitHub Pages는 제한된 테마만 지원하며, Chirpy는 지원 목록에 없습니다.

**해결**: GitHub Actions를 사용한 커스텀 배포 워크플로우를 생성합니다.

### 문제 2: GitHub Actions 워크플로우 설정

`.github/workflows/pages-deploy.yml` 파일을 생성합니다:

```yaml
name: "Build and Deploy"
on:
  push:
    branches:
      - main
      - master
    paths-ignore:
      - .gitignore
      - README.md
      - LICENSE

  workflow_dispatch:

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
        with:
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.3
          bundler-cache: true

      - name: Build site
        run: bundle exec jekyll b -d "_site${{ steps.pages.outputs.base_path }}"
        env:
          JEKYLL_ENV: "production"

      - name: Test site
        run: |
          bundle exec htmlproofer _site \
            --disable-external \
            --ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/" \
            --enforce-https=false \
            || true

      - name: Upload site artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: "_site${{ steps.pages.outputs.base_path }}"

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 문제 3: GitHub Pages 소스 설정

워크플로우를 추가했는데도 여전히 빌드가 실패한다면, GitHub 저장소 설정을 확인해야 합니다.

1. 저장소 Settings → Pages로 이동
2. Source를 "Deploy from a branch"에서 **"GitHub Actions"**로 변경
3. 저장

이 설정을 하지 않으면 GitHub이 기본 Jekyll 빌드를 시도하여 Chirpy 테마를 찾지 못합니다.

### 문제 4: Sass Deprecation 경고

빌드는 성공하지만 다음과 같은 경고가 나타납니다:

```
DEPRECATION WARNING: Sass @import rules are deprecated and will be removed in Dart Sass 3.0.0.
```

이는 Minima 테마의 오래된 Sass 문법 때문입니다. Chirpy로 완전히 전환하면 해결됩니다.

## 성과

전환 후 얻은 이점들:

1. **현대적인 디자인**: 훨씬 깔끔하고 읽기 좋은 레이아웃
2. **향상된 기능**: 검색, TOC, 다크 모드 등
3. **더 나은 성능**: 최적화된 에셋 로딩
4. **PWA 지원**: 오프라인에서도 블로그 접근 가능

## 마무리

Jekyll 블로그를 Chirpy 테마로 전환하는 과정은 생각보다 복잡했지만, 결과적으로 매우 만족스럽습니다. 특히 GitHub Actions를 활용한 배포 방식은 더 많은 커스터마이징이 가능하게 해줍니다.

혹시 비슷한 전환을 계획하고 계신다면, 이 글이 도움이 되길 바랍니다. 문제가 발생하면 댓글로 질문해주세요!

## 참고 자료

- [Chirpy 공식 문서](https://github.com/cotes2020/jekyll-theme-chirpy)
- [GitHub Actions 문서](https://docs.github.com/en/actions)
- [Jekyll 공식 문서](https://jekyllrb.com/docs/)