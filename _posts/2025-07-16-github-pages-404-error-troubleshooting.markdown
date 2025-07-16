---
layout: post
title: "GitHub Pages 404 에러 해결기: Jekyll 플러그인 누락 문제"
date: 2025-07-16 10:00:00 +0900
categories: [Troubleshooting, Jekyll, GitHub Pages]
tags: [github-pages, jekyll, debugging, 404-error, build-error]
mermaid: true
---

## 문제 상황

오늘 블로그에 새로운 글들을 작성하고 푸시했는데, 이상하게도 모든 블로그 포스트에서 404 에러가 발생했습니다. 

```
https://parklabo.github.io/posts/docker-series-01-what-is-docker/
https://parklabo.github.io/posts/github-series-01-getting-started/
https://parklabo.github.io/posts/chrome-extension-development-guide/
```

위 URL들에 접속하면 모두 "404 - File not found" 페이지가 표시되었습니다.

더 이상한 점은:
- 로컬에서 `bundle exec jekyll serve`로 실행하면 **정상적으로 작동**
- GitHub에 파일도 모두 푸시되어 있음
- GitHub Actions도 "성공"으로 표시됨

## 문제 분석 과정

### 1. 첫 번째 의심: 파일이 푸시되지 않았나?

```bash
$ git status
# 모든 파일이 푸시됨 확인

$ git log --oneline
# 커밋 이력도 정상
```

파일은 모두 정상적으로 GitHub에 올라가 있었습니다.

### 2. 두 번째 의심: Jekyll 설정 문제?

`_config.yml` 확인:
```yaml
url: "https://parklabo.github.io"
baseurl: ""  # 비어있음 - 정상
theme: jekyll-theme-chirpy
future: true  # 미래 날짜 포스트도 빌드 - 정상
```

설정도 문제없어 보였습니다.

### 3. 진짜 원인 발견: GitHub Actions 로그

겉보기엔 "성공"으로 표시되었지만, 자세히 보니 **Build site** 단계에서 에러가 있었습니다!

```ruby
Dependency Error: Yikes! It looks like you don't have jekyll-feed 
or one of its dependencies installed. In order to use Jekyll as 
currently configured, you'll need to install this gem.
```

## 원인 분석

문제의 원인을 그림으로 표현하면:

```mermaid
graph TD
    A[_config.yml] -->|plugins 선언| B[jekyll-feed<br/>jekyll-seo-tag<br/>jekyll-sitemap 등]
    C[Gemfile] -->|플러그인 없음| D[❌ 빌드 실패]
    B --> D
    D --> E[404 에러 발생]
    
    style D fill:#ff6b6b,stroke:#c92a2a
    style E fill:#ff6b6b,stroke:#c92a2a
```

### 로컬 vs GitHub Actions 차이

| 환경 | 상황 | 결과 |
|------|------|------|
| **로컬** | 이전에 설치한 gem들이 캐시되어 있음 | ✅ 정상 작동 |
| **GitHub Actions** | Gemfile 기준으로 새로 설치 | ❌ 필요한 gem 없어서 실패 |

## 해결 방법

### 1단계: Gemfile에 누락된 플러그인 추가

```ruby
# Gemfile
source "https://rubygems.org"

gem "jekyll", "~> 4.3"
gem "jekyll-theme-chirpy", "~> 7.3"

# Jekyll plugins - 이 부분이 누락되어 있었음!
gem "jekyll-feed", "~> 0.12"
gem "jekyll-seo-tag", "~> 2.8"
gem "jekyll-sitemap", "~> 1.4"
gem "jekyll-paginate", "~> 1.1"
gem "jekyll-archives", "~> 2.2"
```

### 2단계: bundle install 실행

```bash
$ bundle install
# Gemfile.lock이 업데이트됨
```

### 3단계: 변경사항 커밋 & 푸시

```bash
$ git add Gemfile Gemfile.lock
$ git commit -m "fix: Add missing Jekyll plugins to Gemfile"
$ git push origin main
```

## 추가로 수정한 사항

404 에러를 해결하는 과정에서 추가로 발견한 문제들:

1. **404.html 위치 문제**
   - 기존: `/assets/404.html`
   - 수정: `/404.html` (루트 디렉토리)
   - GitHub Pages는 404 페이지를 루트에서 찾습니다

2. **_config.yml plugins 섹션 추가**
   ```yaml
   # _config.yml
   plugins:
     - jekyll-feed
     - jekyll-seo-tag
     - jekyll-sitemap
     - jekyll-paginate
     - jekyll-archives
   ```

## 교훈과 팁

### 1. GitHub Actions "성공" 표시를 믿지 마세요

```mermaid
graph LR
    A[Actions 성공 ✅] -->|하지만| B[내부 단계 실패 가능]
    B --> C[로그 상세 확인 필요!]
    
    style B fill:#ffe066,stroke:#f59f00
```

### 2. 로컬과 CI/CD 환경의 차이 인지

로컬에서 잘 돌아간다고 안심하면 안 됩니다. CI/CD 환경은:
- 매번 깨끗한 환경에서 시작
- Gemfile/package.json 등에 명시된 의존성만 설치
- 캐시나 전역 설치된 패키지 없음

### 3. 디버깅 체크리스트

Jekyll + GitHub Pages 404 에러 발생 시:

- [ ] 파일이 실제로 푸시되었는지 확인
- [ ] `_config.yml` 설정 확인 (url, baseurl)
- [ ] **GitHub Actions 빌드 로그 상세 확인**
- [ ] Gemfile에 필요한 플러그인이 모두 있는지 확인
- [ ] 404.html이 루트 디렉토리에 있는지 확인
- [ ] 브라우저 캐시 클리어 후 재시도

## 결론

"빌드 성공"이라는 초록색 체크 표시만 보고 안심하지 말고, 실제 로그를 확인하는 습관을 가져야 합니다. 특히 Jekyll 같은 정적 사이트 생성기는 플러그인 의존성 문제가 자주 발생하므로, `_config.yml`에 선언한 플러그인은 반드시 `Gemfile`에도 추가해야 합니다.

이번 경험을 통해 "왜 로컬에선 되는데 배포하면 안 돼요?"라는 고전적인 문제의 한 가지 패턴을 해결할 수 있었습니다. 

혹시 비슷한 문제를 겪고 계신 분들께 도움이 되길 바랍니다! 🚀

---

*이 글이 도움이 되셨다면, 다른 개발자들도 볼 수 있도록 공유해주세요!*