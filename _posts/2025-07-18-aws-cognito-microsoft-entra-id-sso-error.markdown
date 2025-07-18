---
layout: post
title: "[AWS Cognito] Microsoft Entra ID SSO 연동 시 AADSTS50011 에러 해결하기"
date: 2025-07-18 15:00:00 +0900
categories: [AWS, Security]
tags: [aws-cognito, microsoft-entra-id, azure-ad, sso, saml, oidc, authentication, troubleshooting]
mermaid: true
---

## 들어가며

최근 우리 서비스에서 AWS Cognito를 통해 Microsoft Entra ID (구 Azure AD) SSO 인증을 제공하던 중, 일부 고객사에서 `AADSTS50011` 에러가 발생했습니다. 이 에러는 redirect URI 불일치로 인해 발생하는데, 해결 과정에서 얻은 인사이트를 공유하고자 합니다.

## 1. AADSTS50011 에러란?

### 에러 메시지 분석

```
AADSTS50011: The redirect URI 'https://example.auth.ap-northeast-1.amazoncognito.com/oauth2/idpresponse' 
specified in the request does not match the redirect URIs configured for the application 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'. 
Make sure the redirect URI sent in the request matches one added to your application in the Azure portal.
```

### 에러 발생 원인

```mermaid
graph TD
    A[사용자 로그인 시도] --> B[AWS Cognito]
    B --> C[Microsoft Entra ID로 리다이렉트]
    C --> D{Redirect URI 검증}
    D -->|일치| E[인증 성공]
    D -->|불일치| F[AADSTS50011 에러]
    
    style F fill:#f96,stroke:#333,stroke-width:2px
    style E fill:#9f9,stroke:#333,stroke-width:2px
```

Microsoft Entra ID는 보안을 위해 사전에 등록된 redirect URI로만 인증 응답을 보냅니다. 요청된 URI가 등록된 URI와 정확히 일치하지 않으면 이 에러가 발생합니다.

## 2. AWS Cognito와 Microsoft Entra ID 통합 방식

### SAML vs OIDC 비교

| 구분 | SAML 2.0 | OIDC (OpenID Connect) |
|------|----------|----------------------|
| **Redirect URI 형식** | `/saml2/idpresponse` | `/oauth2/idpresponse` |
| **프로토콜** | XML 기반 | JSON/JWT 기반 |
| **복잡도** | 높음 | 중간 |
| **모바일 지원** | 제한적 | 우수 |
| **권장 사용** | 레거시 시스템 | 신규 구현 |

### 올바른 Redirect URI 설정

```mermaid
flowchart LR
    subgraph "Microsoft Entra ID"
        A[App Registration]
        A --> B[Authentication]
        B --> C[Redirect URIs]
    end
    
    subgraph "AWS Cognito"
        D[User Pool]
        D --> E[App Client]
        E --> F[Callback URLs]
    end
    
    C <--> F
    
    style C fill:#bbf,stroke:#333,stroke-width:2px
    style F fill:#bbf,stroke:#333,stroke-width:2px
```

## 3. 문제 해결 단계별 가이드

### Step 1: 정확한 에러 정보 수집

```bash
# 브라우저 개발자 도구에서 Network 탭 확인
# 실패한 요청의 redirect_uri 파라미터 복사

# 예시:
https://your-domain.auth.region.amazoncognito.com/oauth2/idpresponse
```

### Step 2: Microsoft Entra ID 설정 확인

1. [Azure Portal](https://portal.azure.com) 접속
2. Microsoft Entra ID → App registrations
3. 해당 애플리케이션 선택
4. Authentication → Platform configurations

### Step 3: Redirect URI 추가/수정

```yaml
올바른 형식:
  OIDC: https://<cognito-domain>.auth.<region>.amazoncognito.com/oauth2/idpresponse
  SAML: https://<cognito-domain>.auth.<region>.amazoncognito.com/saml2/idpresponse

잘못된 예시:
  - http:// (HTTPS 필수)
  - 마지막 슬래시 추가 (/oauth2/idpresponse/)
  - 대소문자 오류 (OAuth2 X, oauth2 O)
  - 포트 번호 누락 (:443 제외)
```

### Step 4: AWS Cognito 설정 동기화

```javascript
// Cognito User Pool 설정 예시
{
  "CallbackURLs": [
    "https://app.example.com/callback",
    "http://localhost:3000/callback"  // 개발 환경
  ],
  "LogoutURLs": [
    "https://app.example.com/logout",
    "http://localhost:3000/logout"
  ],
  "AllowedOAuthFlows": ["code"],
  "AllowedOAuthScopes": ["openid", "email", "profile"]
}
```

## 4. 자주 발생하는 실수와 해결법

### 흔한 실수 체크리스트

| 실수 유형 | 잘못된 예 | 올바른 예 | 영향도 |
|----------|----------|-----------|---------|
| **프로토콜 불일치** | `http://` | `https://` | 🔴 높음 |
| **후행 슬래시** | `/idpresponse/` | `/idpresponse` | 🔴 높음 |
| **대소문자** | `/OAuth2/` | `/oauth2/` | 🔴 높음 |
| **도메인 오타** | `.amazoncognito.com` | `.amazoncognito.com` | 🔴 높음 |
| **리전 불일치** | `us-east-1` | `ap-northeast-1` | 🔴 높음 |
| **환경 구분 미비** | 개발/운영 미구분 | 환경별 등록 | 🟡 중간 |

### 디버깅 팁

```bash
# 1. Cognito 도메인 확인
aws cognito-idp describe-user-pool-domain \
  --domain your-domain-prefix \
  --region ap-northeast-1

# 2. 현재 설정된 Callback URLs 확인
aws cognito-idp describe-user-pool-client \
  --user-pool-id ap-northeast-1_xxxxxxxx \
  --client-id xxxxxxxxxxxxxxxxxxxxxxxxx \
  --region ap-northeast-1 | jq '.UserPoolClient.CallbackURLs'
```

## 5. 환경별 설정 관리

### 멀티 환경 구성

```mermaid
graph TD
    A[Microsoft Entra ID<br/>App Registration] --> B[Redirect URIs]
    
    B --> C[개발 환경<br/>http://localhost:3000/*]
    B --> D[스테이징<br/>https://staging.example.com/*]
    B --> E[프로덕션<br/>https://app.example.com/*]
    
    C --> F[Cognito Dev Pool]
    D --> G[Cognito Staging Pool]
    E --> H[Cognito Prod Pool]
    
    style A fill:#bbf,stroke:#333,stroke-width:2px
    style H fill:#9f9,stroke:#333,stroke-width:2px
```

### 환경별 설정 예시

```typescript
// config/auth.config.ts
const authConfig = {
  development: {
    domain: 'dev-example.auth.ap-northeast-1.amazoncognito.com',
    clientId: 'dev-client-id',
    redirectUri: 'http://localhost:3000/callback',
    idpResponseUri: 'https://dev-example.auth.ap-northeast-1.amazoncognito.com/oauth2/idpresponse'
  },
  staging: {
    domain: 'staging-example.auth.ap-northeast-1.amazoncognito.com',
    clientId: 'staging-client-id',
    redirectUri: 'https://staging.example.com/callback',
    idpResponseUri: 'https://staging-example.auth.ap-northeast-1.amazoncognito.com/oauth2/idpresponse'
  },
  production: {
    domain: 'prod-example.auth.ap-northeast-1.amazoncognito.com',
    clientId: 'prod-client-id',
    redirectUri: 'https://app.example.com/callback',
    idpResponseUri: 'https://prod-example.auth.ap-northeast-1.amazoncognito.com/oauth2/idpresponse'
  }
}

export default authConfig[process.env.NODE_ENV || 'development'];
```

## 6. 모니터링과 에러 처리

### 로깅 구현

```typescript
// utils/auth-logger.ts
export class AuthLogger {
  static logRedirectError(error: any) {
    if (error.error === 'AADSTS50011') {
      console.error('Redirect URI Mismatch:', {
        timestamp: new Date().toISOString(),
        requestedUri: error.redirect_uri,
        applicationId: error.application_id,
        suggestion: 'Check Azure AD App Registration redirect URIs'
      });
      
      // 사용자에게 친화적인 메시지 표시
      return {
        userMessage: 'SSO 설정에 문제가 있습니다. 관리자에게 문의해주세요.',
        adminMessage: `Azure AD에 다음 URI를 추가하세요: ${error.redirect_uri}`
      };
    }
  }
}
```

### 사용자 경험 개선

```typescript
// components/SSOErrorHandler.tsx
import React from 'react';

const SSOErrorHandler: React.FC<{ error: string }> = ({ error }) => {
  if (error.includes('AADSTS50011')) {
    return (
      <div className="error-container">
        <h2>SSO 로그인 오류</h2>
        <p>현재 SSO 설정에 문제가 있어 로그인할 수 없습니다.</p>
        <details>
          <summary>관리자용 정보</summary>
          <pre>
            에러 코드: AADSTS50011
            해결 방법: Azure AD App Registration에서 
            Redirect URI를 확인하고 추가해주세요.
          </pre>
        </details>
        <button onClick={() => window.location.href = '/login'}>
          다른 방법으로 로그인
        </button>
      </div>
    );
  }
  
  return <div>로그인 중 오류가 발생했습니다.</div>;
};
```

## 7. 베스트 프랙티스

### 설정 관리 자동화

```yaml
# .github/workflows/validate-sso-config.yml
name: Validate SSO Configuration

on:
  push:
    paths:
      - 'config/auth.config.ts'
      - 'infrastructure/cognito/*'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Check Redirect URIs
        run: |
          # Cognito와 Azure AD 설정 비교
          aws cognito-idp describe-user-pool-client \
            --user-pool-id ${{ secrets.USER_POOL_ID }} \
            --client-id ${{ secrets.CLIENT_ID }} \
            | jq '.UserPoolClient.CallbackURLs' > cognito-urls.json
          
          # Azure AD 설정과 비교 (Azure CLI 사용)
          az ad app show --id ${{ secrets.AZURE_APP_ID }} \
            --query "web.redirectUris" > azure-urls.json
          
          # 불일치 확인
          diff cognito-urls.json azure-urls.json
```

### 문서화 템플릿

```markdown
## SSO 설정 문서

### Microsoft Entra ID 설정
- Application ID: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`
- Redirect URIs:
  - 개발: `http://localhost:3000/callback`
  - 운영: `https://prod-example.auth.ap-northeast-1.amazoncognito.com/oauth2/idpresponse`

### AWS Cognito 설정
- User Pool ID: `ap-northeast-1_xxxxxxxxx`
- App Client ID: `xxxxxxxxxxxxxxxxxxxxxxxxx`
- Domain: `prod-example`

### 테스트 체크리스트
- [ ] 개발 환경 SSO 로그인
- [ ] 스테이징 환경 SSO 로그인
- [ ] 프로덕션 환경 SSO 로그인
- [ ] 에러 처리 동작 확인
```

## 8. 트러블슈팅 체크리스트

### 빠른 진단 가이드

```mermaid
flowchart TD
    A[AADSTS50011 에러 발생] --> B{브라우저 개발자 도구<br/>Network 탭 확인}
    B --> C[redirect_uri 파라미터 복사]
    C --> D{Azure Portal에<br/>동일한 URI 존재?}
    D -->|No| E[Azure AD에 URI 추가]
    D -->|Yes| F{대소문자, 슬래시<br/>정확히 일치?}
    F -->|No| G[정확한 형식으로 수정]
    F -->|Yes| H{5분 대기 후<br/>재시도}
    H -->|성공| I[완료]
    H -->|실패| J[Cognito 설정 확인]
    
    style A fill:#f96,stroke:#333,stroke-width:2px
    style I fill:#9f9,stroke:#333,stroke-width:2px
```

## 마무리

AADSTS50011 에러는 단순해 보이지만, 정확한 문자열 매칭이 필요하기 때문에 사소한 실수로도 발생할 수 있습니다. 이 글에서 제시한 체크리스트와 자동화 방법을 활용하면 이러한 에러를 사전에 방지하고, 발생 시 빠르게 해결할 수 있습니다.

### 핵심 포인트
- ✅ Redirect URI는 **정확히** 일치해야 함 (대소문자, 슬래시 포함)
- ✅ 환경별로 별도 설정 관리 필수
- ✅ 변경사항 반영에 3-5분 소요
- ✅ 자동화와 모니터링으로 사전 예방

SSO 통합은 초기 설정이 까다롭지만, 한 번 제대로 구성하면 사용자에게 편리한 인증 경험을 제공할 수 있습니다.

## 참고 자료

- [Microsoft: AADSTS50011 에러 문서](https://learn.microsoft.com/troubleshoot/entra/entra-id/app-integration/error-code-aadsts50011-redirect-uri-mismatch)
- [AWS Cognito SAML IdP 설정 가이드](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-saml-idp.html)
- [Azure AD와 AWS Cognito 통합](https://docs.microsoft.com/azure/active-directory/saas-apps/amazon-web-service-tutorial)

---

*SSO 통합 관련 경험이나 다른 에러 해결 사례가 있으시다면 댓글로 공유해주세요!*