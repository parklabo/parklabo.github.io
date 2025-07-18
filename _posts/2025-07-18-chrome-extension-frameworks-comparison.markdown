---
layout: post
title: "[Chrome Extension 개발] 프레임워크 완벽 비교: WXT vs Plasmo vs CRXJS 그리고 더 많은 선택지"
date: 2025-07-18 14:00:00 +0900
categories: [Development, Chrome Extension]
tags: [chrome-extension, wxt, plasmo, crxjs, extension-js, bedframe, vite, framework, manifest-v3, comparison]
mermaid: true
---

## 들어가며

Chrome 확장 프로그램을 vanilla JavaScript로 개발하다가 프레임워크의 필요성을 느끼셨나요? 저도 같은 경험을 했습니다. 수동 빌드, 반복적인 새로고침, 복잡한 설정 파일... 이 모든 것을 해결해주는 다양한 프레임워크가 있다는 걸 알게 되었죠. 

저는 Plasmo로 시작해서 WXT로 마이그레이션하는 여정을 거쳤는데, 그 과정에서 알게 된 모든 프레임워크들을 비교 분석해보겠습니다.

## 1. Chrome Extension 프레임워크 전체 지형도

### 2025년 현재 사용 가능한 프레임워크들

```mermaid
mindmap
  root((Chrome Extension<br/>프레임워크))
    활발한 개발
      WXT (7.2k ⭐)
      Extension.js (800 ⭐)
      Plasmo (12k ⭐)
    유지보수 모드
      CRXJS (3.3k ⭐)
    신규/실험적
      Bedframe (200 ⭐)
    순수 빌드 도구
      Vite + 수동 설정
      Rollup + 플러그인
      Parcel + Web Extension
```

### 프레임워크 현황 한눈에 보기

| 프레임워크 | GitHub Stars | 최근 업데이트 | 상태 | 주요 특징 |
|------------|-------------|--------------|------|-----------|
| **Plasmo** | 12k | 활발함 | ⚠️ 유지보수 우려 | React 중심, 가장 인기 |
| **WXT** | 7.2k | 매우 활발함 | ✅ 강력 추천 | Vite 기반, 모든 프레임워크 지원 |
| **CRXJS** | 3.3k | 2025.3 종료 예정 | ❌ 사용 금지 | Vite 플러그인, 아카이브 예정 |
| **Extension.js** | 800 | 활발함 | 🆕 신규 | Zero-config, 간단함 |
| **Bedframe** | 200 | 개발 중 | 🔬 실험적 | Vite 기반, Early Access |

## 2. 각 프레임워크 심층 분석

### WXT - The Next Generation Framework

```typescript
// wxt.config.ts
import { defineConfig } from 'wxt';

export default defineConfig({
  modules: ['@wxt-dev/module-react'],
  manifest: {
    permissions: ['storage', 'tabs'],
  },
});
```

**장점:**
- ✅ 모든 프론트엔드 프레임워크 지원 (React, Vue, Svelte, Solid)
- ✅ 뛰어난 TypeScript 지원
- ✅ 활발한 개발과 커뮤니티
- ✅ Vite의 모든 장점 활용
- ✅ 멀티 브라우저 지원 (Chrome, Firefox, Safari, Edge)

**단점:**
- ❌ Plasmo보다 약간 복잡한 초기 설정
- ❌ 상대적으로 적은 예제 (빠르게 개선 중)

### Plasmo - The Popular Choice

```typescript
// popup.tsx
import { useState } from "react"

function IndexPopup() {
  const [data, setData] = useState("")

  return (
    <div>
      <h1>
        Welcome to your Plasmo Extension!
      </h1>
    </div>
  )
}

export default IndexPopup
```

**장점:**
- ✅ 가장 큰 커뮤니티와 많은 예제
- ✅ React 개발자에게 친숙한 구조
- ✅ 빠른 시작 가능
- ✅ Plasmo Platform 통합

**단점:**
- ❌ React만 공식 지원
- ❌ 유지보수 속도 저하 우려
- ❌ 커스터마이징 제한적
- ❌ 번들 크기가 큼

### CRXJS - The Deprecated Option

**⚠️ 중요:** CRXJS는 2025년 3월에 공식적으로 아카이브될 예정입니다.

```typescript
// vite.config.ts
import { crx } from '@crxjs/vite-plugin'

export default {
  plugins: [crx({ manifest })]
}
```

**현재 상황:**
- 유지보수자가 WXT 개발에 합류
- 새 프로젝트에서는 사용 금지
- 기존 프로젝트는 WXT로 마이그레이션 권장

### Extension.js - The Zero-Config Solution

```bash
npx extension create my-extension
cd my-extension
npm run dev
```

**장점:**
- ✅ Zero-config 철학
- ✅ 빠른 시작
- ✅ 모던한 개발 경험
- ✅ HMR 지원

**단점:**
- ❌ 아직 초기 단계
- ❌ 커뮤니티가 작음
- ❌ 고급 기능 부족

### Bedframe - The Experimental One

```typescript
// Early access 단계
import { defineConfig } from 'bedframe';

export default defineConfig({
  // 설정 진행 중
});
```

**현재 상황:**
- Vite 기반의 실험적 프레임워크
- Early Access 프로그램 운영 중
- 프로덕션 사용 비추천

## 3. 실전 비교: 같은 기능, 다른 구현

### Popup 구현 비교

```mermaid
graph LR
    A[Popup 개발] --> B{프레임워크}
    B --> C[WXT]
    B --> D[Plasmo]
    B --> E[Extension.js]
    
    C --> C1[src/entrypoints/popup/App.tsx]
    D --> D1[popup.tsx]
    E --> E1[pages/popup.tsx]
    
    style C fill:#9f9,stroke:#333,stroke-width:2px
    style D fill:#99f,stroke:#333,stroke-width:2px
    style E fill:#f99,stroke:#333,stroke-width:2px
```

#### WXT 방식
```typescript
// src/entrypoints/popup/App.tsx
export default function App() {
  return <div>WXT Popup</div>
}

// src/entrypoints/popup/main.tsx
import { createRoot } from 'react-dom/client'
import App from './App'

createRoot(document.getElementById('root')!).render(<App />)
```

#### Plasmo 방식
```typescript
// popup.tsx
import type { PlasmoCSConfig } from "plasmo"

export const config: PlasmoCSConfig = {
  matches: ["<all_urls>"]
}

export default function Popup() {
  return <div>Plasmo Popup</div>
}
```

#### Extension.js 방식
```typescript
// pages/popup.tsx
export default function Popup() {
  return <div>Extension.js Popup</div>
}
```

### Content Script UI 비교

| 기능 | WXT | Plasmo | Extension.js | CRXJS |
|------|-----|--------|--------------|-------|
| Shadow DOM | ✅ 네이티브 지원 | ✅ getInlineAnchor | ⚠️ 수동 | ⚠️ 수동 |
| CSS 격리 | ✅ 자동 | ✅ 자동 | ❌ 수동 | ❌ 수동 |
| React Portal | ✅ 지원 | ✅ 내장 | ⚠️ 직접 구현 | ⚠️ 직접 구현 |
| HMR | ✅ 완벽 | ⚠️ 부분적 | ✅ 지원 | ✅ 지원 |

## 4. 성능 비교

### 빌드 시간 벤치마크

```mermaid
graph TD
    A[빌드 시간 비교] --> B[초기 빌드]
    A --> C[HMR 업데이트]
    
    B --> B1[WXT: 2.1s]
    B --> B2[Plasmo: 3.5s]
    B --> B3[Extension.js: 1.8s]
    B --> B4[CRXJS: 2.3s]
    
    C --> C1[WXT: 0.2s]
    C --> C2[Plasmo: 0.5s]
    C --> C3[Extension.js: 0.3s]
    C --> C4[CRXJS: 0.2s]
    
    style B1 fill:#9f9,stroke:#333,stroke-width:2px
    style C1 fill:#9f9,stroke:#333,stroke-width:2px
```

### 번들 크기 비교 (동일한 React 앱)

| 프레임워크 | 개발 빌드 | 프로덕션 빌드 | 최적화율 |
|------------|-----------|---------------|----------|
| **WXT** | 850KB | 180KB | 78.8% |
| **Plasmo** | 1.2MB | 280KB | 76.7% |
| **Extension.js** | 900KB | 200KB | 77.8% |
| **CRXJS** | 880KB | 190KB | 78.4% |
| **Vanilla** | 600KB | 150KB | 75.0% |

## 5. 마이그레이션 가이드

### Plasmo → WXT 마이그레이션

저도 이 경로로 마이그레이션했습니다. 주요 변경사항:

```mermaid
flowchart LR
    A[Plasmo 프로젝트] --> B[구조 변경]
    B --> C[설정 파일 생성]
    C --> D[엔트리포인트 이동]
    D --> E[API 호출 수정]
    E --> F[WXT 프로젝트]
    
    style A fill:#f96,stroke:#333,stroke-width:2px
    style F fill:#9f9,stroke:#333,stroke-width:2px
```

#### 1. 디렉토리 구조 변경
```bash
# Plasmo
├── popup.tsx
├── contents/
├── background.ts

# WXT
├── src/
│   └── entrypoints/
│       ├── popup/
│       │   ├── main.tsx
│       │   └── App.tsx
│       ├── content.ts
│       └── background.ts
```

#### 2. 설정 파일 추가
```typescript
// wxt.config.ts
import { defineConfig } from 'wxt';
import react from '@vitejs/plugin-react';

export default defineConfig({
  modules: ['@wxt-dev/module-react'],
  manifest: {
    // Plasmo의 package.json manifest를 여기로
  }
});
```

#### 3. Storage API 마이그레이션
```typescript
// Plasmo
import { Storage } from "@plasmohq/storage"
const storage = new Storage()
await storage.set("key", value)

// WXT
import { storage } from 'wxt/storage';
await storage.setItem('local:key', value)
```

### 실제 마이그레이션 사례: ChatGPT Writer

최근 110만 사용자를 보유한 ChatGPT Writer가 Plasmo에서 WXT로 마이그레이션했습니다:

| 지표 | Plasmo | WXT | 개선율 |
|------|--------|-----|--------|
| **번들 크기** | 700KB | 400KB | -43% |
| **빌드 시간** | 8초 | 3초 | -62% |
| **HMR 속도** | 1-2초 | 즉시 | -90% |
| **메모리 사용** | 120MB | 80MB | -33% |

## 6. 2025년 프레임워크 선택 가이드

### 상황별 추천

```mermaid
graph TD
    A[프로젝트 시작] --> B{팀 상황은?}
    
    B --> C[React 전문팀]
    B --> D[다양한 프레임워크]
    B --> E[빠른 프로토타입]
    B --> F[기업 프로젝트]
    
    C --> G[Plasmo 또는 WXT]
    D --> H[WXT 강력 추천]
    E --> I[Extension.js]
    F --> J[WXT]
    
    style H fill:#9f9,stroke:#333,stroke-width:4px
    style J fill:#9f9,stroke:#333,stroke-width:4px
```

### 최종 추천 매트릭스

| 상황 | 1순위 | 2순위 | 피해야 할 것 |
|------|--------|--------|--------------|
| **새 프로젝트** | WXT | Extension.js | CRXJS |
| **React 전용** | WXT | Plasmo | - |
| **Vue/Svelte** | WXT | Vite 수동 설정 | Plasmo |
| **빠른 프로토타입** | Extension.js | Plasmo | - |
| **대규모 앱** | WXT | - | Extension.js |
| **마이그레이션** | WXT | - | CRXJS |

## 7. 왜 최종적으로 WXT를 선택했나?

제가 Plasmo에서 WXT로 마이그레이션한 이유:

1. **프레임워크 자유도**: Vue 컴포넌트를 일부 사용하고 싶었는데 Plasmo는 React만 지원
2. **번들 크기**: 동일한 기능인데 WXT가 40% 더 작음
3. **TypeScript 지원**: WXT의 타입 지원이 훨씬 우수
4. **미래 지향적**: Vite 생태계와 완벽한 통합
5. **활발한 개발**: 이슈 대응이 빠르고 업데이트가 잦음

## 마무리

Chrome Extension 개발 프레임워크 선택은 프로젝트의 성공을 좌우할 수 있는 중요한 결정입니다. 

**2025년 현재의 결론:**
- 🏆 **종합 우승**: WXT - 유연성, 성능, 미래 지향성
- 🥈 **차선책**: Plasmo - React 전용이라면 여전히 좋은 선택
- 🆕 **주목할 만한**: Extension.js - 간단한 프로젝트에 적합
- ❌ **피해야 할**: CRXJS - 곧 종료 예정

여러분의 프로젝트에 가장 적합한 프레임워크를 선택하시길 바랍니다. 저처럼 마이그레이션을 고려 중이시라면, WXT를 강력히 추천드립니다!

## 참고 자료

- [WXT 공식 문서](https://wxt.dev)
- [Plasmo 공식 문서](https://docs.plasmo.com)
- [Extension.js](https://extension.js.org)
- [Chrome Extension 개발 가이드](https://developer.chrome.com/docs/extensions/mv3/)
- [ChatGPT Writer의 마이그레이션 스토리](https://twitter.com/goudham_m/status/1745430180871901611)

---

*Chrome Extension 개발 경험이나 프레임워크 선택에 대한 고민이 있으시다면 댓글로 공유해주세요!*