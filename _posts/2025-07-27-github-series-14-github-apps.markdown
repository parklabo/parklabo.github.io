---
layout: post
title: "[GitHub 입문 #14] GitHub Apps 개발: 자동화 봇 만들기"
date: 2025-07-27 10:00:00 +0900
categories: [DevOps, GitHub]
tags: [github, apps, bot, automation, development, tutorial, series]
mermaid: true
---

## 들어가며

GitHub 입문 시리즈의 열네 번째 시간입니다. 이번에는 GitHub Apps를 개발하여 강력한 자동화 봇을 만드는 방법을 알아보겠습니다. GitHub Apps는 OAuth Apps보다 더 세밀한 권한 제어와 높은 API 제한을 제공합니다.

## 1. GitHub Apps vs OAuth Apps

### 비교표

| 특징 | GitHub Apps | OAuth Apps | 선택 기준 |
|------|-------------|------------|-----------|
| **인증 방식** | JWT + 설치 토큰 | OAuth 토큰 | 보안 수준 |
| **권한 모델** | 세분화된 권한 | 스코프 기반 | 최소 권한 필요성 |
| **API 제한** | 5,000/시간 (설치당) | 5,000/시간 (사용자당) | API 사용량 |
| **봇 계정** | 자동 생성 [bot] | 실제 사용자 필요 | 봇 운영 여부 |
| **웹훅** | 앱 레벨 중앙화 | 저장소별 설정 | 관리 편의성 |
| **마켓플레이스** | 지원 | 미지원 | 수익화 계획 |
| **설치 단위** | 조직/사용자 | 사용자 | 배포 범위 |

### 주요 차이점

```yaml
GitHub Apps:
  인증: JWT 기반, 설치별 토큰
  권한: 세분화된 권한 (Repository, Issue, PR 등)
  API 제한: 시간당 5,000 요청 (설치당)
  Webhook: 앱 레벨에서 중앙 관리
  봇 사용자: 자동 생성 ([bot] 배지)
  
OAuth Apps:
  인증: OAuth 토큰
  권한: 전체 스코프 기반
  API 제한: 시간당 5,000 요청 (사용자당)
  Webhook: 저장소별 설정
  봇 사용자: 실제 사용자 계정 필요
```

### 언제 GitHub App을 선택해야 할까?

```yaml
GitHub App이 적합한 경우:
  - 여러 조직/저장소에서 사용
  - 세밀한 권한 제어 필요
  - 높은 API 사용량
  - 봇 계정으로 활동
  - 마켓플레이스 배포 계획

OAuth App이 적합한 경우:
  - 사용자 대신 작업 수행
  - 간단한 개인 도구
  - 레거시 시스템 통합
```

## 2. GitHub App 생성하기

### GitHub App 개발 플로우

```mermaid
graph TD
    A[GitHub App 생성] --> B{개발 환경}
    B -->|로컬| C[ngrok/localtunnel]
    B -->|클라우드| D[Webhook URL 설정]
    
    C --> E[Webhook 수신]
    D --> E
    
    E --> F[이벤트 처리]
    F --> G{인증 필요?}
    
    G -->|Yes| H[JWT 생성]
    H --> I[설치 토큰 획득]
    I --> J[API 호출]
    
    G -->|No| K[직접 처리]
    
    J --> L[작업 수행]
    K --> L
    
    L --> M{결과}
    M -->|성공| N[응답 200]
    M -->|실패| O[에러 처리]
    
    O --> P[로깅]
    O --> Q[재시도]
```

### 앱 매니페스트 작성

`app-manifest.json`:

```json
{
  "name": "My Awesome Bot",
  "description": "자동화와 생산성을 위한 GitHub App",
  "url": "https://myapp.example.com",
  "hook_attributes": {
    "url": "https://myapp.example.com/webhook",
    "active": true
  },
  "redirect_url": "https://myapp.example.com/callback",
  "setup_url": "https://myapp.example.com/setup",
  "public": true,
  "default_permissions": {
    "issues": "write",
    "pull_requests": "write",
    "repository_projects": "read",
    "statuses": "write",
    "checks": "write",
    "metadata": "read",
    "contents": "read"
  },
  "default_events": [
    "issues",
    "issue_comment",
    "pull_request",
    "pull_request_review",
    "pull_request_review_comment",
    "push",
    "release",
    "check_suite",
    "check_run"
  ]
}
```

### 프로그래밍 방식으로 앱 생성

```javascript
// create-github-app.js
const express = require('express');
const app = express();

// 매니페스트 플로우 시작
app.get('/create-app', (req, res) => {
  const manifest = {
    name: 'Auto PR Reviewer',
    description: 'Automated code review bot',
    url: 'https://mybot.example.com',
    hook_attributes: {
      url: 'https://mybot.example.com/webhook',
    },
    redirect_url: 'https://mybot.example.com/callback',
    public: false,
    default_permissions: {
      contents: 'read',
      issues: 'write',
      pull_requests: 'write',
      checks: 'write',
    },
    default_events: [
      'pull_request',
      'pull_request_review',
      'check_suite',
      'check_run',
    ],
  };

  const manifestString = JSON.stringify(manifest);
  const githubUrl = `https://github.com/settings/apps/new?manifest=${encodeURIComponent(manifestString)}`;
  
  res.redirect(githubUrl);
});

// 콜백 처리
app.get('/callback', async (req, res) => {
  const { code } = req.query;
  
  // GitHub에서 앱 정보 교환
  const response = await fetch(`https://api.github.com/app-manifests/${code}/conversions`, {
    method: 'POST',
    headers: {
      'Accept': 'application/vnd.github.v3+json',
    },
  });
  
  const appInfo = await response.json();
  
  // 앱 정보 저장
  console.log('App ID:', appInfo.id);
  console.log('App Slug:', appInfo.slug);
  console.log('Webhook Secret:', appInfo.webhook_secret);
  console.log('PEM:', appInfo.pem);
  
  // 환경 변수 파일 생성
  const envContent = `
GITHUB_APP_ID=${appInfo.id}
GITHUB_APP_SLUG=${appInfo.slug}
GITHUB_WEBHOOK_SECRET=${appInfo.webhook_secret}
GITHUB_APP_PRIVATE_KEY="${appInfo.pem.replace(/\n/g, '\\n')}"
  `;
  
  res.send('App created successfully! Check console for details.');
});
```

## 3. GitHub App 인증 구현

### JWT 인증과 설치 토큰

```javascript
// auth.js
const jwt = require('jsonwebtoken');
const { Octokit } = require('@octokit/rest');
const { createAppAuth } = require('@octokit/auth-app');

class GitHubAppAuth {
  constructor(appId, privateKey) {
    this.appId = appId;
    this.privateKey = privateKey;
    this.installations = new Map();
  }

  // JWT 토큰 생성
  generateJWT() {
    const now = Math.floor(Date.now() / 1000);
    const payload = {
      iat: now - 60,  // 과거 시간으로 설정 (시계 차이 보정)
      exp: now + 600, // 10분 후 만료
      iss: parseInt(this.appId),
    };

    return jwt.sign(payload, this.privateKey, { algorithm: 'RS256' });
  }

  // 앱 수준 Octokit 인스턴스
  getAppOctokit() {
    return new Octokit({
      authStrategy: createAppAuth,
      auth: {
        appId: this.appId,
        privateKey: this.privateKey,
      },
    });
  }

  // 설치별 Octokit 인스턴스
  async getInstallationOctokit(installationId) {
    const app = this.getAppOctokit();
    
    // 설치 액세스 토큰 생성
    const { data: { token } } = await app.apps.createInstallationAccessToken({
      installation_id: installationId,
    });

    return new Octokit({ auth: token });
  }

  // 저장소별 Octokit 인스턴스
  async getRepoOctokit(owner, repo) {
    const app = this.getAppOctokit();
    
    // 저장소의 설치 ID 찾기
    const { data: installation } = await app.apps.getRepoInstallation({
      owner,
      repo,
    });

    return this.getInstallationOctokit(installation.id);
  }

  // 사용 가능한 설치 목록
  async listInstallations() {
    const app = this.getAppOctokit();
    const { data } = await app.apps.listInstallations();
    
    return data.map(installation => ({
      id: installation.id,
      account: installation.account.login,
      type: installation.account.type,
      repositories: installation.repository_selection,
      permissions: installation.permissions,
      events: installation.events,
    }));
  }

  // 권한 확인
  async checkPermissions(installationId, requiredPermissions) {
    const app = this.getAppOctokit();
    const { data: installation } = await app.apps.getInstallation({
      installation_id: installationId,
    });

    const currentPermissions = installation.permissions;
    const missingPermissions = [];

    for (const [resource, level] of Object.entries(requiredPermissions)) {
      const currentLevel = currentPermissions[resource];
      if (!currentLevel || 
          (level === 'write' && currentLevel === 'read') ||
          (level === 'admin' && currentLevel !== 'admin')) {
        missingPermissions.push(`${resource}:${level}`);
      }
    }

    return {
      hasPermissions: missingPermissions.length === 0,
      missingPermissions,
    };
  }
}

module.exports = GitHubAppAuth;
```

## 4. 실전 GitHub App 개발

### PR 자동 리뷰 봇

`pr-review-bot/index.js`:

```javascript
const { App } = require('@octokit/app');
const { Octokit } = require('@octokit/rest');
const express = require('express');
const bodyParser = require('body-parser');

// 리뷰 규칙 설정
const reviewRules = {
  filePatterns: {
    '*.test.js': { reviewer: 'test-team', required: true },
    'src/api/**': { reviewer: 'backend-team', required: true },
    'src/components/**': { reviewer: 'frontend-team', required: false },
    'docs/**': { reviewer: 'docs-team', required: false },
  },
  autoApprove: {
    authors: ['dependabot[bot]', 'renovate[bot]'],
    patterns: ['package-lock.json', '*.md'],
  },
  sizeLimit: {
    small: { lines: 100, label: 'size/S' },
    medium: { lines: 500, label: 'size/M' },
    large: { lines: 1000, label: 'size/L' },
    xlarge: { lines: Infinity, label: 'size/XL' },
  },
};

class PRReviewBot {
  constructor(app) {
    this.app = app;
    this.server = express();
    this.setupWebhooks();
  }

  setupWebhooks() {
    this.server.use(bodyParser.json());
    
    // Webhook 엔드포인트
    this.server.post('/webhook', async (req, res) => {
      const event = req.headers['x-github-event'];
      const payload = req.body;

      // 서명 검증
      if (!this.verifyWebhookSignature(req)) {
        return res.status(401).send('Unauthorized');
      }

      try {
        switch (event) {
          case 'pull_request':
            await this.handlePullRequest(payload);
            break;
          case 'pull_request_review':
            await this.handlePullRequestReview(payload);
            break;
          case 'check_suite':
            await this.handleCheckSuite(payload);
            break;
        }
        res.status(200).send('OK');
      } catch (error) {
        console.error('Error handling webhook:', error);
        res.status(500).send('Internal Server Error');
      }
    });
  }

  async handlePullRequest(payload) {
    const { action, pull_request, repository, installation } = payload;

    if (action !== 'opened' && action !== 'synchronize') {
      return;
    }

    const octokit = await this.app.getInstallationOctokit(installation.id);

    // 1. PR 크기 라벨 추가
    await this.addSizeLabel(octokit, repository, pull_request);

    // 2. 자동 리뷰어 할당
    await this.assignReviewers(octokit, repository, pull_request);

    // 3. 자동 승인 확인
    await this.checkAutoApprove(octokit, repository, pull_request);

    // 4. 체크 실행
    await this.runChecks(octokit, repository, pull_request);

    // 5. 환영 메시지 (첫 기여자)
    if (await this.isFirstTimeContributor(octokit, repository, pull_request)) {
      await this.postWelcomeMessage(octokit, repository, pull_request);
    }
  }

  async addSizeLabel(octokit, repository, pullRequest) {
    const { additions, deletions } = pullRequest;
    const totalLines = additions + deletions;

    let sizeLabel = 'size/XL';
    for (const [size, config] of Object.entries(reviewRules.sizeLimit)) {
      if (totalLines < config.lines) {
        sizeLabel = config.label;
        break;
      }
    }

    // 기존 크기 라벨 제거
    const { data: labels } = await octokit.issues.listLabelsOnIssue({
      owner: repository.owner.login,
      repo: repository.name,
      issue_number: pullRequest.number,
    });

    const sizeLabels = labels.filter(label => label.name.startsWith('size/'));
    for (const label of sizeLabels) {
      await octokit.issues.removeLabel({
        owner: repository.owner.login,
        repo: repository.name,
        issue_number: pullRequest.number,
        name: label.name,
      });
    }

    // 새 크기 라벨 추가
    await octokit.issues.addLabels({
      owner: repository.owner.login,
      repo: repository.name,
      issue_number: pullRequest.number,
      labels: [sizeLabel],
    });
  }

  async assignReviewers(octokit, repository, pullRequest) {
    const { data: files } = await octokit.pulls.listFiles({
      owner: repository.owner.login,
      repo: repository.name,
      pull_number: pullRequest.number,
    });

    const reviewers = new Set();
    const teams = new Set();

    for (const file of files) {
      for (const [pattern, config] of Object.entries(reviewRules.filePatterns)) {
        if (this.matchPattern(file.filename, pattern)) {
          if (config.reviewer.endsWith('-team')) {
            teams.add(config.reviewer);
          } else {
            reviewers.add(config.reviewer);
          }
        }
      }
    }

    // PR 작성자 제외
    reviewers.delete(pullRequest.user.login);

    if (reviewers.size > 0 || teams.size > 0) {
      await octokit.pulls.requestReviewers({
        owner: repository.owner.login,
        repo: repository.name,
        pull_number: pullRequest.number,
        reviewers: Array.from(reviewers),
        team_reviewers: Array.from(teams),
      });
    }
  }

  async runChecks(octokit, repository, pullRequest) {
    // 체크 실행 생성
    const { data: checkRun } = await octokit.checks.create({
      owner: repository.owner.login,
      repo: repository.name,
      name: 'PR Validation',
      head_sha: pullRequest.head.sha,
      status: 'in_progress',
      started_at: new Date().toISOString(),
    });

    try {
      const checks = [];

      // 1. 커밋 메시지 검사
      const commitCheck = await this.checkCommitMessages(octokit, repository, pullRequest);
      checks.push(commitCheck);

      // 2. 파일 이름 규칙 검사
      const fileCheck = await this.checkFileNaming(octokit, repository, pullRequest);
      checks.push(fileCheck);

      // 3. 금지된 파일 검사
      const forbiddenCheck = await this.checkForbiddenFiles(octokit, repository, pullRequest);
      checks.push(forbiddenCheck);

      // 4. PR 설명 검사
      const descriptionCheck = this.checkPRDescription(pullRequest);
      checks.push(descriptionCheck);

      // 결과 집계
      const failures = checks.filter(c => !c.passed);
      const conclusion = failures.length === 0 ? 'success' : 'failure';

      // 체크 완료
      await octokit.checks.update({
        owner: repository.owner.login,
        repo: repository.name,
        check_run_id: checkRun.id,
        status: 'completed',
        conclusion,
        completed_at: new Date().toISOString(),
        output: {
          title: `PR Validation ${conclusion === 'success' ? 'Passed' : 'Failed'}`,
          summary: this.generateCheckSummary(checks),
          annotations: this.generateAnnotations(failures),
        },
      });
    } catch (error) {
      await octokit.checks.update({
        owner: repository.owner.login,
        repo: repository.name,
        check_run_id: checkRun.id,
        status: 'completed',
        conclusion: 'failure',
        completed_at: new Date().toISOString(),
        output: {
          title: 'PR Validation Error',
          summary: `An error occurred: ${error.message}`,
        },
      });
    }
  }

  async checkCommitMessages(octokit, repository, pullRequest) {
    const { data: commits } = await octokit.pulls.listCommits({
      owner: repository.owner.login,
      repo: repository.name,
      pull_number: pullRequest.number,
    });

    const invalidCommits = [];
    const conventionalPattern = /^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,50}/;

    for (const commit of commits) {
      const firstLine = commit.commit.message.split('\n')[0];
      if (!conventionalPattern.test(firstLine)) {
        invalidCommits.push({
          sha: commit.sha,
          message: firstLine,
        });
      }
    }

    return {
      name: 'Commit Messages',
      passed: invalidCommits.length === 0,
      message: invalidCommits.length === 0 
        ? 'All commit messages follow conventional format'
        : `${invalidCommits.length} commit(s) don't follow conventional format`,
      details: invalidCommits,
    };
  }

  generateCheckSummary(checks) {
    let summary = '## PR Validation Results\n\n';
    
    for (const check of checks) {
      const icon = check.passed ? '✅' : '❌';
      summary += `${icon} **${check.name}**: ${check.message}\n`;
      
      if (!check.passed && check.details) {
        summary += '\n```\n';
        summary += JSON.stringify(check.details, null, 2);
        summary += '\n```\n\n';
      }
    }

    return summary;
  }

  matchPattern(filename, pattern) {
    if (pattern.includes('**')) {
      const regex = pattern
        .replace(/\*\*/g, '.*')
        .replace(/\*/g, '[^/]*')
        .replace(/\./g, '\\.');
      return new RegExp(`^${regex}$`).test(filename);
    }
    return filename === pattern || filename.endsWith(pattern);
  }

  verifyWebhookSignature(req) {
    const signature = req.headers['x-hub-signature-256'];
    const body = JSON.stringify(req.body);
    const secret = process.env.GITHUB_WEBHOOK_SECRET;
    
    const hmac = crypto.createHmac('sha256', secret);
    const digest = `sha256=${hmac.update(body).digest('hex')}`;
    
    return crypto.timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(digest)
    );
  }

  start(port = 3000) {
    this.server.listen(port, () => {
      console.log(`PR Review Bot listening on port ${port}`);
    });
  }
}

// 앱 초기화 및 시작
const app = new App({
  appId: process.env.GITHUB_APP_ID,
  privateKey: process.env.GITHUB_APP_PRIVATE_KEY,
  webhooks: {
    secret: process.env.GITHUB_WEBHOOK_SECRET,
  },
});

const bot = new PRReviewBot(app);
bot.start();
```

## 5. 고급 기능 구현

### 프로액티브 봇 행동

```javascript
// proactive-bot.js
class ProactiveBot {
  constructor(app) {
    this.app = app;
    this.scheduleProactiveTasks();
  }

  scheduleProactiveTasks() {
    // 매일 오전 9시 실행
    cron.schedule('0 9 * * *', () => {
      this.runDailyTasks();
    });

    // 1시간마다 실행
    cron.schedule('0 * * * *', () => {
      this.runHourlyTasks();
    });
  }

  async runDailyTasks() {
    const installations = await this.app.eachInstallation();
    
    for (const installation of installations) {
      const octokit = await this.app.getInstallationOctokit(installation.id);
      
      // 각 설치에 대해 작업 수행
      await this.checkStaleIssues(octokit, installation);
      await this.checkAbandonedPRs(octokit, installation);
      await this.generateDailyReport(octokit, installation);
    }
  }

  async checkStaleIssues(octokit, installation) {
    const repos = await this.getInstallationRepos(octokit);
    
    for (const repo of repos) {
      const { data: issues } = await octokit.issues.listForRepo({
        owner: repo.owner.login,
        repo: repo.name,
        state: 'open',
        sort: 'updated',
        direction: 'asc',
        per_page: 100,
      });

      const thirtyDaysAgo = new Date();
      thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

      for (const issue of issues) {
        if (new Date(issue.updated_at) < thirtyDaysAgo && !issue.pull_request) {
          // Stale 라벨 추가
          await octokit.issues.addLabels({
            owner: repo.owner.login,
            repo: repo.name,
            issue_number: issue.number,
            labels: ['stale'],
          });

          // 코멘트 추가
          await octokit.issues.createComment({
            owner: repo.owner.login,
            repo: repo.name,
            issue_number: issue.number,
            body: `이 이슈는 30일 이상 업데이트되지 않았습니다. 
7일 이내에 활동이 없으면 자동으로 닫힐 예정입니다.

이슈가 여전히 유효하다면 코멘트를 남겨주세요.`,
          });
        }
      }
    }
  }

  async generateDailyReport(octokit, installation) {
    const repos = await this.getInstallationRepos(octokit);
    const report = {
      date: new Date().toISOString().split('T')[0],
      repositories: [],
    };

    for (const repo of repos) {
      const stats = await this.getRepoStats(octokit, repo);
      report.repositories.push({
        name: repo.full_name,
        stats,
      });
    }

    // 보고서를 이슈로 생성
    const mainRepo = repos[0]; // 주 저장소 선택
    await octokit.issues.create({
      owner: mainRepo.owner.login,
      repo: mainRepo.name,
      title: `📊 Daily Report - ${report.date}`,
      body: this.formatDailyReport(report),
      labels: ['report', 'automated'],
    });
  }

  formatDailyReport(report) {
    let body = `# Daily Activity Report\n\n`;
    body += `**Date**: ${report.date}\n\n`;

    for (const repo of report.repositories) {
      body += `## ${repo.name}\n\n`;
      body += `- Open Issues: ${repo.stats.openIssues}\n`;
      body += `- Open PRs: ${repo.stats.openPRs}\n`;
      body += `- New Issues (24h): ${repo.stats.newIssues}\n`;
      body += `- Closed Issues (24h): ${repo.stats.closedIssues}\n`;
      body += `- Merged PRs (24h): ${repo.stats.mergedPRs}\n\n`;
    }

    return body;
  }
}
```

### GitHub App 마켓플레이스 준비

```javascript
// marketplace-handler.js
class MarketplaceHandler {
  constructor(app) {
    this.app = app;
  }

  // 마켓플레이스 구매 이벤트 처리
  async handleMarketplacePurchase(payload) {
    const { action, marketplace_purchase } = payload;

    switch (action) {
      case 'purchased':
        await this.onPurchase(marketplace_purchase);
        break;
      case 'cancelled':
        await this.onCancellation(marketplace_purchase);
        break;
      case 'changed':
        await this.onPlanChange(marketplace_purchase);
        break;
    }
  }

  async onPurchase(purchase) {
    // 새 구매 처리
    console.log(`New purchase: ${purchase.account.login} - ${purchase.plan.name}`);
    
    // 환영 이메일 발송
    await this.sendWelcomeEmail(purchase.account);
    
    // 데이터베이스에 구독 정보 저장
    await this.saveSubscription({
      accountId: purchase.account.id,
      accountLogin: purchase.account.login,
      planId: purchase.plan.id,
      planName: purchase.plan.name,
      seats: purchase.plan.unit_count,
      startDate: new Date(),
    });

    // 프리미엄 기능 활성화
    await this.enablePremiumFeatures(purchase.account.id);
  }

  async onPlanChange(purchase) {
    const oldPlan = purchase.previous_marketplace_purchase.plan;
    const newPlan = purchase.plan;

    console.log(`Plan change: ${oldPlan.name} -> ${newPlan.name}`);

    // 기능 조정
    if (newPlan.price_model === 'FREE') {
      await this.disablePremiumFeatures(purchase.account.id);
    } else {
      await this.updatePremiumFeatures(purchase.account.id, newPlan);
    }
  }

  // 플랜별 제한 확인
  async checkPlanLimits(installationId) {
    const subscription = await this.getSubscription(installationId);
    
    const limits = {
      FREE: {
        repositories: 3,
        checks_per_day: 100,
        auto_merge: false,
      },
      BASIC: {
        repositories: 10,
        checks_per_day: 1000,
        auto_merge: true,
      },
      PRO: {
        repositories: -1, // 무제한
        checks_per_day: -1,
        auto_merge: true,
        priority_support: true,
      },
    };

    return limits[subscription.planName] || limits.FREE;
  }
}
```

## 6. 배포와 운영

### Docker 컨테이너화

`Dockerfile`:

```dockerfile
FROM node:18-alpine

# 보안 업데이트
RUN apk update && apk upgrade

# 앱 사용자 생성
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# 작업 디렉토리
WORKDIR /app

# 의존성 설치
COPY package*.json ./
RUN npm ci --only=production

# 앱 코드 복사
COPY --chown=nodejs:nodejs . .

# 사용자 전환
USER nodejs

# 헬스체크
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

EXPOSE 3000

CMD ["node", "index.js"]
```

### Kubernetes 배포

`k8s-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-app-bot
  labels:
    app: github-app-bot
spec:
  replicas: 3
  selector:
    matchLabels:
      app: github-app-bot
  template:
    metadata:
      labels:
        app: github-app-bot
    spec:
      containers:
      - name: bot
        image: myregistry/github-app-bot:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: GITHUB_APP_ID
          valueFrom:
            secretKeyRef:
              name: github-app-secrets
              key: app-id
        - name: GITHUB_WEBHOOK_SECRET
          valueFrom:
            secretKeyRef:
              name: github-app-secrets
              key: webhook-secret
        - name: GITHUB_APP_PRIVATE_KEY
          valueFrom:
            secretKeyRef:
              name: github-app-secrets
              key: private-key
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: github-app-bot-service
spec:
  selector:
    app: github-app-bot
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: LoadBalancer
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: github-app-bot-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: github-app-bot
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 모니터링과 로깅

```javascript
// monitoring.js
const prometheus = require('prom-client');
const winston = require('winston');

// Prometheus 메트릭 설정
const register = new prometheus.Registry();

const httpRequestDuration = new prometheus.Histogram({
  name: 'github_app_http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10],
});

const webhookProcessingTime = new prometheus.Histogram({
  name: 'github_app_webhook_processing_seconds',
  help: 'Time to process webhook events',
  labelNames: ['event_type', 'action'],
});

const apiCallsTotal = new prometheus.Counter({
  name: 'github_app_api_calls_total',
  help: 'Total number of GitHub API calls',
  labelNames: ['endpoint', 'status'],
});

register.registerMetric(httpRequestDuration);
register.registerMetric(webhookProcessingTime);
register.registerMetric(apiCallsTotal);

// Winston 로거 설정
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json(),
  ),
  defaultMeta: { service: 'github-app-bot' },
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple(),
      ),
    }),
    new winston.transports.File({ 
      filename: 'error.log', 
      level: 'error' 
    }),
    new winston.transports.File({ 
      filename: 'combined.log' 
    }),
  ],
});

// 메트릭 엔드포인트
app.get('/metrics', (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(register.metrics());
});

module.exports = {
  register,
  httpRequestDuration,
  webhookProcessingTime,
  apiCallsTotal,
  logger,
};
```

## 7. 보안 베스트 프랙티스

### GitHub App 권한 매트릭스

| 권한 | Read | Write | Admin | 용도 |
|------|------|-------|-------|------|
| **Actions** | 워크플로우 조회 | 워크플로우 실행 | - | CI/CD 자동화 |
| **Checks** | 체크 결과 조회 | 체크 생성/수정 | - | PR 검증 |
| **Contents** | 코드 읽기 | 커밋 생성 | - | 코드 분석/수정 |
| **Issues** | 이슈 조회 | 이슈 생성/수정 | - | 이슈 관리 |
| **Metadata** | 기본 정보 | - | - | 필수 권한 |
| **Pull requests** | PR 조회 | PR 생성/수정 | - | PR 자동화 |
| **Webhooks** | - | 웹훅 관리 | - | 이벤트 수신 |

### 2025년 GitHub App 보안 체크리스트

| 영역 | 체크 항목 | 중요도 | 구현 방법 |
|------|-----------|--------|-----------|
| **인증** | JWT 만료 시간 설정 | 높음 | 10분 이내 |
| **인증** | 설치 토큰 캐싱 | 중간 | Redis 활용 |
| **권한** | 최소 권한 원칙 | 높음 | 필요한 권한만 요청 |
| **시크릿** | 환경 변수 암호화 | 높음 | KMS/Vault 사용 |
| **웹훅** | 서명 검증 | 높음 | HMAC-SHA256 |
| **API** | Rate Limit 처리 | 중간 | 적응형 스로틀링 |
| **로깅** | 감사 로그 | 중간 | 모든 작업 기록 |

### 시크릿 관리

```javascript
// secret-manager.js
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');
const AWS = require('aws-sdk');

class SecretManager {
  constructor(provider = 'gcp') {
    this.provider = provider;
    
    if (provider === 'gcp') {
      this.client = new SecretManagerServiceClient();
    } else if (provider === 'aws') {
      this.client = new AWS.SecretsManager();
    }
  }

  async getSecret(secretName) {
    if (this.provider === 'gcp') {
      const [version] = await this.client.accessSecretVersion({
        name: `projects/${process.env.GCP_PROJECT}/secrets/${secretName}/versions/latest`,
      });
      return version.payload.data.toString('utf8');
    } else if (this.provider === 'aws') {
      const data = await this.client.getSecretValue({
        SecretId: secretName,
      }).promise();
      return data.SecretString;
    }
    
    // 로컬 개발용 폴백
    return process.env[secretName];
  }

  async rotateAppPrivateKey() {
    // 새 키 쌍 생성
    const { privateKey, publicKey } = await this.generateKeyPair();
    
    // GitHub App 설정 업데이트
    // (수동으로 수행해야 함)
    console.log('New public key:', publicKey);
    
    // 새 private key 저장
    await this.saveSecret('GITHUB_APP_PRIVATE_KEY', privateKey);
    
    // 이전 키 백업
    const timestamp = new Date().toISOString();
    await this.saveSecret(`GITHUB_APP_PRIVATE_KEY_BACKUP_${timestamp}`, 
      await this.getSecret('GITHUB_APP_PRIVATE_KEY'));
    
    return { privateKey, publicKey };
  }

  async generateKeyPair() {
    const { generateKeyPairSync } = require('crypto');
    const { privateKey, publicKey } = generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: {
        type: 'spki',
        format: 'pem',
      },
      privateKeyEncoding: {
        type: 'pkcs8',
        format: 'pem',
      },
    });
    
    return { privateKey, publicKey };
  }
}
```

### 레이트 리밋 처리

```javascript
// rate-limiter.js
class RateLimiter {
  constructor() {
    this.limits = new Map();
  }

  async checkRateLimit(octokit) {
    const { data: rateLimit } = await octokit.rateLimit.get();
    
    return {
      core: {
        limit: rateLimit.resources.core.limit,
        remaining: rateLimit.resources.core.remaining,
        reset: new Date(rateLimit.resources.core.reset * 1000),
      },
      graphql: {
        limit: rateLimit.resources.graphql.limit,
        remaining: rateLimit.resources.graphql.remaining,
        reset: new Date(rateLimit.resources.graphql.reset * 1000),
      },
    };
  }

  async waitIfNeeded(octokit, minRemaining = 100) {
    const limits = await this.checkRateLimit(octokit);
    
    if (limits.core.remaining < minRemaining) {
      const waitTime = limits.core.reset - new Date();
      console.log(`Rate limit low. Waiting ${waitTime / 1000} seconds...`);
      await new Promise(resolve => setTimeout(resolve, waitTime));
    }
  }

  // 적응형 속도 조절
  async adaptiveThrottle(octokit, callback) {
    const limits = await this.checkRateLimit(octokit);
    const remainingRatio = limits.core.remaining / limits.core.limit;
    
    // 남은 할당량에 따라 지연 시간 조정
    let delay = 0;
    if (remainingRatio < 0.1) {
      delay = 5000; // 5초
    } else if (remainingRatio < 0.3) {
      delay = 1000; // 1초
    } else if (remainingRatio < 0.5) {
      delay = 100; // 100ms
    }
    
    if (delay > 0) {
      await new Promise(resolve => setTimeout(resolve, delay));
    }
    
    return callback();
  }
}
```

## 마무리

GitHub Apps는 강력하고 안전한 자동화를 구현할 수 있는 최선의 방법입니다.

### 핵심 체크포인트

| 항목 | 설명 | 중요도 |
|------|------|--------|
| **권한 설계** | 최소 권한 원칙 적용 | ⭐⭐⭐ |
| **인증 구현** | JWT + 설치 토큰 활용 | ⭐⭐⭐ |
| **웹훅 처리** | 비동기 이벤트 처리 | ⭐⭐⭐ |
| **에러 처리** | 재시도 및 로깅 | ⭐⭐ |
| **모니터링** | 메트릭 및 알림 | ⭐⭐ |

### GitHub App 아키텍처 모범 사례

```mermaid
graph TB
    subgraph "GitHub"
        A[GitHub Events] --> B[Webhook]
    end
    
    subgraph "App Infrastructure"
        B --> C[Load Balancer]
        C --> D[App Instances]
        D --> E[Queue]
        E --> F[Workers]
        
        D --> G[Cache]
        F --> G
        
        D --> H[Database]
        F --> H
        
        I[Monitoring] --> D
        I --> F
    end
    
    subgraph "External"
        F --> J[GitHub API]
        F --> K[Notifications]
    end
```

잘 설계된 GitHub App은 개발 워크플로우를 획기적으로 개선하고 팀의 생산성을 높일 수 있습니다.

다음 편에서는 GitHub Pages를 활용한 정적 사이트 호스팅에 대해 알아보겠습니다.

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
10. [Team 협업 설정](/posts/github-series-10-team-collaboration/)
11. [GitHub Actions 입문](/posts/github-series-11-github-actions-intro/)
12. [Actions 고급 활용](/posts/github-series-12-actions-advanced/)
13. [Webhooks와 API](/posts/github-series-13-webhooks-and-api/)
14. **GitHub Apps 개발** (현재 글)
15. GitHub Pages (예정)

---

*질문이나 피드백이 있으시다면 댓글로 남겨주세요!*