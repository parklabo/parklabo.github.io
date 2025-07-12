---
layout: post
title: "ChatGPT PDF Exporter: Chrome 확장 프로그램 개발부터 스토어 배포까지"
date: 2025-07-12 21:00:00 +0900
categories: [Development, Chrome Extension]
tags: [chrome-extension, javascript, webstore, chatgpt, tutorial]
image:
  path: /assets/img/posts/chrome-extension-banner.png
  alt: Chrome Extension Development
---

## 프로젝트 소개

ChatGPT와의 대화를 PDF, HTML, 텍스트 파일로 저장하고 싶었던 적이 있으신가요? 저는 이런 필요를 느껴 **ChatGPT PDF Exporter**라는 Chrome 확장 프로그램을 개발했습니다. 

이 글에서는 아이디어부터 Chrome 웹 스토어 배포까지의 전 과정을 상세히 공유하겠습니다.

## 프로젝트 개요

### 주요 기능
- ChatGPT 대화를 PDF/HTML/텍스트로 내보내기
- 특정 메시지만 선택하여 내보내기
- 타임스탬프 포함/제외 옵션
- 다국어 지원 (한국어, 영어, 일본어)
- 로컬에서 모든 처리 (프라이버시 보호)

### 사용 기술
- Manifest V3
- JavaScript (ES6+)
- html2canvas, jsPDF
- Chrome Extension APIs

## 개발 과정

### 1. 프로젝트 구조 설정

```
chatgpt-pdf-exporter/
├── manifest.json          # 확장 프로그램 설정
├── background.js         # 백그라운드 스크립트
├── content.js           # 콘텐츠 스크립트
├── popup/               # 팝업 UI
│   ├── popup.html
│   ├── popup.css
│   └── popup.js
├── icons/               # 아이콘 파일들
│   ├── icon16.png
│   ├── icon48.png
│   └── icon128.png
└── lib/                 # 외부 라이브러리
    ├── html2canvas.min.js
    └── jspdf.umd.min.js
```

### 2. Manifest V3 설정

`manifest.json`은 확장 프로그램의 핵심 설정 파일입니다:

```json
{
  "manifest_version": 3,
  "name": "ChatGPT PDF Exporter",
  "version": "1.0.0",
  "description": "Export ChatGPT conversations to PDF, HTML, or text",
  
  "permissions": [
    "activeTab",
    "storage"
  ],
  
  "host_permissions": [
    "https://chatgpt.com/*",
    "https://chat.openai.com/*"
  ],
  
  "action": {
    "default_popup": "popup/popup.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },
  
  "content_scripts": [
    {
      "matches": ["https://chatgpt.com/*", "https://chat.openai.com/*"],
      "js": ["lib/html2canvas.min.js", "lib/jspdf.umd.min.js", "content.js"],
      "run_at": "document_end"
    }
  ],
  
  "background": {
    "service_worker": "background.js"
  }
}
```

### 3. 콘텐츠 스크립트 구현

ChatGPT 페이지에서 대화 내용을 추출하는 핵심 로직:

```javascript
// content.js
function extractConversation() {
  const messages = [];
  const messageElements = document.querySelectorAll('[data-message-author-role]');
  
  messageElements.forEach(element => {
    const role = element.getAttribute('data-message-author-role');
    const content = element.querySelector('.markdown-body')?.innerText || '';
    const timestamp = new Date().toLocaleString();
    
    messages.push({
      role: role === 'user' ? '사용자' : 'ChatGPT',
      content: content,
      timestamp: timestamp
    });
  });
  
  return messages;
}

// PDF 생성 함수
async function generatePDF(messages, options) {
  const { jsPDF } = window.jspdf;
  const pdf = new jsPDF();
  
  // 제목 추가
  pdf.setFontSize(16);
  pdf.text(options.title || 'ChatGPT 대화', 20, 20);
  
  // 메시지 추가
  let yPosition = 40;
  messages.forEach(msg => {
    pdf.setFontSize(12);
    pdf.text(`${msg.role}:`, 20, yPosition);
    
    if (options.includeTimestamp) {
      pdf.setFontSize(10);
      pdf.text(msg.timestamp, 150, yPosition);
    }
    
    yPosition += 10;
    
    // 긴 텍스트 줄바꿈 처리
    const lines = pdf.splitTextToSize(msg.content, 170);
    pdf.setFontSize(11);
    lines.forEach(line => {
      if (yPosition > 270) {
        pdf.addPage();
        yPosition = 20;
      }
      pdf.text(line, 20, yPosition);
      yPosition += 7;
    });
    
    yPosition += 10;
  });
  
  pdf.save(`chatgpt_${Date.now()}.pdf`);
}
```

### 4. 팝업 UI 구현

사용자 인터페이스를 위한 팝업 구현:

```html
<!-- popup.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <link rel="stylesheet" href="popup.css">
</head>
<body>
  <div class="container">
    <h2>ChatGPT Exporter</h2>
    
    <div class="options">
      <label>
        <input type="checkbox" id="includeTimestamp" checked>
        타임스탬프 포함
      </label>
      
      <label>
        제목: <input type="text" id="exportTitle" placeholder="대화 제목">
      </label>
    </div>
    
    <div class="buttons">
      <button id="exportPDF">PDF로 내보내기</button>
      <button id="exportHTML">HTML로 내보내기</button>
      <button id="exportText">텍스트로 내보내기</button>
    </div>
  </div>
  
  <script src="popup.js"></script>
</body>
</html>
```

### 5. 다국어 지원

`_locales` 폴더를 통한 다국어 지원:

```json
// _locales/ko/messages.json
{
  "appName": {
    "message": "ChatGPT PDF 내보내기"
  },
  "appDescription": {
    "message": "ChatGPT 대화를 PDF, HTML, 텍스트로 내보내기"
  }
}
```

## Chrome 웹 스토어 배포 과정

### 1. 개발자 계정 등록

1. [Chrome Web Store Developer Dashboard](https://chrome.google.com/webstore/developer/dashboard) 접속
2. Google 계정으로 로그인
3. 일회성 등록비 $5 결제

### 2. 스토어 등록 정보 준비

필요한 자료들:
- **앱 이름**: 32자 이내
- **설명**: 상세 설명 (132자) + 전체 설명
- **아이콘**: 128x128px PNG
- **스크린샷**: 1280x800 또는 640x400 (최소 1개, 최대 5개)
- **프로모션 타일**: 440x280px (선택사항)
- **카테고리**: 생산성, 도구 등 선택
- **언어**: 지원하는 언어 선택

### 3. 확장 프로그램 패키징

```bash
# 개발 파일 정리
rm -rf .git/ .gitignore README.md

# ZIP 파일 생성
zip -r chatgpt-pdf-exporter.zip .
```

### 4. 스토어에 업로드

1. Developer Dashboard에서 "새 항목" 클릭
2. ZIP 파일 업로드
3. 스토어 등록 정보 입력
4. 스크린샷 업로드
5. 개인정보처리방침 URL 입력 (필수)

### 5. 개인정보처리방침 작성

```markdown
# ChatGPT PDF Exporter 개인정보처리방침

## 수집하는 정보
본 확장 프로그램은 어떠한 개인정보도 수집하지 않습니다.

## 데이터 처리
- 모든 데이터는 사용자의 브라우저에서 로컬로 처리됩니다
- 외부 서버로 데이터를 전송하지 않습니다
- ChatGPT 대화 내용은 내보내기 시에만 일시적으로 처리됩니다

## 권한 사용
- activeTab: 현재 탭의 콘텐츠에 접근하여 대화 내용을 추출
- storage: 사용자 설정 저장 (로컬 저장소만 사용)
```

### 6. 검토 및 게시

1. 모든 정보 입력 후 "검토를 위해 제출" 클릭
2. Google의 검토 대기 (보통 1-3일 소요)
3. 승인되면 이메일 통지
4. 스토어에 자동 게시

## 배포 후 관리

### 업데이트 방법

1. `manifest.json`의 버전 번호 증가
2. 변경사항 적용
3. 새 ZIP 파일 생성
4. Dashboard에서 "패키지 업로드" 클릭
5. 업데이트 노트 작성

### 사용자 피드백 관리

- 리뷰와 평점 모니터링
- 버그 리포트 대응
- 기능 요청 수집
- 정기적인 업데이트

## 개발 팁과 주의사항

### 1. Manifest V3 마이그레이션
- `background.js`를 Service Worker로 변경
- `chrome.runtime` API 활용
- 동적 스크립트 실행 제한 준수

### 2. 보안 고려사항
- Content Security Policy 설정
- 외부 스크립트 최소화
- 민감한 정보 처리 주의

### 3. 성능 최적화
- 큰 파일은 청크 단위로 처리
- 메모리 사용량 모니터링
- 비동기 처리 활용

### 4. 사용자 경험
- 직관적인 UI 디자인
- 명확한 권한 설명
- 빠른 응답 속도

## 마무리

Chrome 확장 프로그램 개발은 웹 개발 지식만 있다면 누구나 도전할 수 있습니다. 특히 ChatGPT처럼 자주 사용하는 서비스에 필요한 기능을 직접 추가할 수 있다는 점이 매력적입니다.

이 프로젝트의 전체 소스 코드는 [GitHub](https://github.com/pyxym/chatgpt-pdf-exporter)에서 확인하실 수 있습니다.

여러분도 일상에서 불편한 점을 해결하는 확장 프로그램을 만들어보는 건 어떨까요?

## 참고 자료

- [Chrome Extension 공식 문서](https://developer.chrome.com/docs/extensions/mv3/)
- [Chrome Web Store 개발자 가이드](https://developer.chrome.com/docs/webstore/)
- [Manifest V3 마이그레이션 가이드](https://developer.chrome.com/docs/extensions/mv3/intro/mv3-migration/)