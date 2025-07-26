---
layout: post
title: "[GitHub 입문 #13] Webhooks와 API: 외부 서비스 연동"
date: 2025-07-26 10:00:00 +0900
categories: [DevOps, GitHub]
tags: [github, webhooks, api, integration, automation, tutorial, series]
mermaid: true
---

## 들어가며

GitHub 입문 시리즈의 열세 번째 시간입니다. 이번에는 GitHub Webhooks와 API를 활용하여 외부 서비스와 연동하는 방법을 알아보겠습니다. 이를 통해 GitHub 이벤트를 실시간으로 처리하고 자동화를 구현할 수 있습니다.

## Webhooks와 API 개요

### 주요 차이점

| 구분 | Webhooks | API |
|------|----------|-----|
| **통신 방식** | Push (GitHub → 서버) | Pull (서버 → GitHub) |
| **실시간성** | 즉시 알림 | 폴링 필요 |
| **용도** | 이벤트 기반 자동화 | 데이터 조회/조작 |
| **인증** | Secret 토큰 | OAuth/PAT |
| **제한** | 이벤트 타입별 | Rate Limit |

## 1. GitHub Webhooks 이해하기

Webhooks는 GitHub에서 특정 이벤트가 발생했을 때 외부 서버로 HTTP POST 요청을 보내는 기능입니다.

### Webhook 동작 원리

```
GitHub Repository          Your Server
      |                         |
      | Event occurs           |
      |----------------------->|
      | POST /webhook          |
      | {event data}          |
      |                        |
      |<----------------------|
      | 200 OK                |
```

### 주요 Webhook 이벤트

```mermaid
graph TD
    A[GitHub Events] --> B[Repository]
    A --> C[Pull Request]
    A --> D[Issues]
    A --> E[Actions]
    
    B --> B1[push]
    B --> B2[create/delete]
    B --> B3[release]
    
    C --> C1[opened/closed]
    C --> C2[review]
    C --> C3[merged]
    
    D --> D1[opened/closed]
    D --> D2[comment]
    D --> D3[labeled]
    
    E --> E1[workflow_run]
    E --> E2[deployment]
```

### Webhook 이벤트 상세 목록

| 카테고리 | 이벤트 | 설명 | 주요 용도 |
|----------|--------|------|-----------|
| **Repository** | push | 코드 푸시 | CI/CD 트리거 |
| | create | 브랜치/태그 생성 | 자동 브랜치 보호 |
| | release | 릴리즈 발행 | 배포 자동화 |
| **Pull Request** | pull_request | PR 상태 변경 | 코드 리뷰 봇 |
| | pull_request_review | 리뷰 제출 | 승인 프로세스 |
| **Issues** | issues | 이슈 상태 변경 | 이슈 트래킹 |
| | issue_comment | 코멘트 추가 | 자동 응답 |
| **Actions** | workflow_run | 워크플로우 완료 | 상태 모니터링 |
| | deployment_status | 배포 상태 | 배포 알림 |

## 2. Webhook 서버 구현

### Node.js Express 서버

```javascript
// webhook-server.js
const express = require('express');
const crypto = require('crypto');
const { Octokit } = require('@octokit/rest');

const app = express();
const PORT = process.env.PORT || 3000;
const WEBHOOK_SECRET = process.env.WEBHOOK_SECRET;
const GITHUB_TOKEN = process.env.GITHUB_TOKEN;

const octokit = new Octokit({ auth: GITHUB_TOKEN });

// Raw body를 위한 미들웨어
app.use(express.json({
  verify: (req, res, buf, encoding) => {
    req.rawBody = buf.toString(encoding || 'utf8');
  }
}));

// Webhook 서명 검증
function verifyWebhookSignature(req) {
  const signature = req.headers['x-hub-signature-256'];
  if (!signature) return false;

  const hmac = crypto.createHmac('sha256', WEBHOOK_SECRET);
  const digest = 'sha256=' + hmac.update(req.rawBody).digest('hex');
  
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(digest)
  );
}

// Webhook 엔드포인트
app.post('/webhook', async (req, res) => {
  // 서명 검증
  if (!verifyWebhookSignature(req)) {
    return res.status(401).send('Unauthorized');
  }

  const event = req.headers['x-github-event'];
  const payload = req.body;

  console.log(`Received ${event} event`);

  try {
    // 이벤트별 처리
    switch (event) {
      case 'push':
        await handlePushEvent(payload);
        break;
      case 'pull_request':
        await handlePullRequestEvent(payload);
        break;
      case 'issues':
        await handleIssueEvent(payload);
        break;
      case 'release':
        await handleReleaseEvent(payload);
        break;
      default:
        console.log(`Unhandled event type: ${event}`);
    }

    res.status(200).send('OK');
  } catch (error) {
    console.error('Error processing webhook:', error);
    res.status(500).send('Internal Server Error');
  }
});

// Push 이벤트 처리
async function handlePushEvent(payload) {
  const { repository, ref, commits, pusher } = payload;
  
  // main 브랜치 푸시만 처리
  if (ref !== 'refs/heads/main') {
    return;
  }

  console.log(`Push to main by ${pusher.name}`);
  console.log(`${commits.length} commits pushed`);

  // 커밋 메시지 분석
  for (const commit of commits) {
    if (commit.message.includes('[deploy]')) {
      console.log('Triggering deployment...');
      await triggerDeployment(repository, commit);
    }

    if (commit.message.includes('[notify]')) {
      await sendNotification(repository, commit);
    }
  }

  // 코드 품질 체크 트리거
  await triggerCodeQualityCheck(repository, payload.after);
}

// Pull Request 이벤트 처리
async function handlePullRequestEvent(payload) {
  const { action, pull_request, repository } = payload;

  switch (action) {
    case 'opened':
      // 자동 라벨 추가
      await addLabelsBasedOnFiles(repository, pull_request);
      
      // 리뷰어 자동 할당
      await assignReviewers(repository, pull_request);
      
      // 환영 메시지
      if (pull_request.user.type === 'User' && isFirstTimeContributor(pull_request.user)) {
        await postWelcomeComment(repository, pull_request);
      }
      break;

    case 'synchronize':
      // PR 업데이트 시 체크 재실행
      await runPRChecks(repository, pull_request);
      break;

    case 'closed':
      if (pull_request.merged) {
        // 머지 후 처리
        await handleMergedPR(repository, pull_request);
      }
      break;
  }
}

// 파일 변경 기반 라벨 추가
async function addLabelsBasedOnFiles(repository, pull_request) {
  const { data: files } = await octokit.pulls.listFiles({
    owner: repository.owner.login,
    repo: repository.name,
    pull_number: pull_request.number,
  });

  const labels = new Set();

  for (const file of files) {
    if (file.filename.startsWith('src/')) {
      labels.add('code');
    }
    if (file.filename.endsWith('.md')) {
      labels.add('documentation');
    }
    if (file.filename.includes('test')) {
      labels.add('tests');
    }
    if (file.filename.startsWith('.github/')) {
      labels.add('ci/cd');
    }
  }

  if (labels.size > 0) {
    await octokit.issues.addLabels({
      owner: repository.owner.login,
      repo: repository.name,
      issue_number: pull_request.number,
      labels: Array.from(labels),
    });
  }
}

// 리뷰어 자동 할당
async function assignReviewers(repository, pull_request) {
  // CODEOWNERS 파일 읽기
  try {
    const { data: codeowners } = await octokit.repos.getContent({
      owner: repository.owner.login,
      repo: repository.name,
      path: '.github/CODEOWNERS',
    });

    const content = Buffer.from(codeowners.content, 'base64').toString();
    const reviewers = parseCodeOwners(content, pull_request);

    if (reviewers.length > 0) {
      await octokit.pulls.requestReviewers({
        owner: repository.owner.login,
        repo: repository.name,
        pull_number: pull_request.number,
        reviewers: reviewers.filter(r => r !== pull_request.user.login),
      });
    }
  } catch (error) {
    console.log('No CODEOWNERS file found');
  }
}

// Issue 이벤트 처리
async function handleIssueEvent(payload) {
  const { action, issue, repository } = payload;

  if (action === 'opened') {
    // 이슈 템플릿 확인
    if (!issue.body || issue.body.trim().length < 50) {
      await octokit.issues.createComment({
        owner: repository.owner.login,
        repo: repository.name,
        issue_number: issue.number,
        body: '이슈 설명이 너무 짧습니다. 더 자세한 정보를 제공해주세요.',
      });
    }

    // 자동 분류
    await categorizeIssue(repository, issue);
  }
}

// Release 이벤트 처리
async function handleReleaseEvent(payload) {
  const { action, release, repository } = payload;

  if (action === 'published') {
    console.log(`New release: ${release.tag_name}`);

    // 릴리즈 노트 생성
    const releaseNotes = await generateReleaseNotes(repository, release);
    
    // 외부 서비스 알림
    await notifyExternalServices(release, releaseNotes);
    
    // 문서 업데이트 트리거
    await triggerDocsUpdate(repository, release);
  }
}

app.listen(PORT, () => {
  console.log(`Webhook server listening on port ${PORT}`);
});
```

### Python Flask 서버

```python
# webhook_server.py
from flask import Flask, request, jsonify
import hmac
import hashlib
import json
import requests
from github import Github
import os

app = Flask(__name__)
WEBHOOK_SECRET = os.environ.get('WEBHOOK_SECRET', '').encode()
GITHUB_TOKEN = os.environ.get('GITHUB_TOKEN')

g = Github(GITHUB_TOKEN)

def verify_webhook_signature(data, signature):
    """Webhook 서명 검증"""
    if not signature:
        return False
    
    sha_name, signature = signature.split('=')
    if sha_name != 'sha256':
        return False
    
    mac = hmac.new(WEBHOOK_SECRET, msg=data, digestmod=hashlib.sha256)
    return hmac.compare_digest(mac.hexdigest(), signature)

@app.route('/webhook', methods=['POST'])
def github_webhook():
    # 서명 검증
    signature = request.headers.get('X-Hub-Signature-256')
    if not verify_webhook_signature(request.data, signature):
        return jsonify({'error': 'Invalid signature'}), 401
    
    event = request.headers.get('X-GitHub-Event')
    payload = request.json
    
    # 이벤트별 처리
    handlers = {
        'push': handle_push,
        'pull_request': handle_pull_request,
        'issues': handle_issues,
        'workflow_run': handle_workflow_run,
        'deployment_status': handle_deployment_status,
    }
    
    handler = handlers.get(event)
    if handler:
        try:
            handler(payload)
            return jsonify({'status': 'success'}), 200
        except Exception as e:
            app.logger.error(f'Error handling {event}: {str(e)}')
            return jsonify({'error': str(e)}), 500
    else:
        app.logger.info(f'Unhandled event: {event}')
        return jsonify({'status': 'ignored'}), 200

def handle_push(payload):
    """Push 이벤트 처리"""
    repo_name = payload['repository']['full_name']
    branch = payload['ref'].split('/')[-1]
    commits = payload['commits']
    
    # 특정 브랜치만 처리
    if branch not in ['main', 'master', 'develop']:
        return
    
    repo = g.get_repo(repo_name)
    
    # 커밋 분석
    for commit in commits:
        sha = commit['id']
        message = commit['message']
        
        # 커밋 메시지 패턴 확인
        if '[hotfix]' in message.lower():
            create_hotfix_issue(repo, commit)
        
        if '[breaking]' in message.lower():
            notify_breaking_change(repo, commit)
        
        # 보안 키워드 확인
        check_security_keywords(repo, commit)

def handle_pull_request(payload):
    """Pull Request 이벤트 처리"""
    action = payload['action']
    pr = payload['pull_request']
    repo_name = payload['repository']['full_name']
    
    repo = g.get_repo(repo_name)
    pull = repo.get_pull(pr['number'])
    
    if action == 'opened':
        # PR 크기 확인
        if pr['additions'] + pr['deletions'] > 500:
            pull.create_issue_comment(
                "⚠️ 이 PR이 너무 큽니다 (500줄 이상). "
                "더 작은 단위로 나누는 것을 고려해주세요."
            )
        
        # 브랜치 이름 규칙 확인
        if not is_valid_branch_name(pr['head']['ref']):
            pull.create_issue_comment(
                "❌ 브랜치 이름이 규칙을 따르지 않습니다.\n"
                "올바른 형식: feature/*, bugfix/*, hotfix/*"
            )
        
        # 관련 이슈 확인
        if not has_linked_issue(pr):
            pull.create_issue_comment(
                "📝 관련 이슈를 연결해주세요. "
                "PR 설명에 'Fixes #123' 형식을 사용하세요."
            )
    
    elif action == 'synchronize':
        # 충돌 확인
        if not pull.mergeable:
            pull.create_issue_comment(
                "🔄 병합 충돌이 발생했습니다. 해결해주세요."
            )

def handle_workflow_run(payload):
    """GitHub Actions 워크플로우 실행 결과 처리"""
    workflow_run = payload['workflow_run']
    conclusion = workflow_run['conclusion']
    
    if conclusion == 'failure':
        # 실패 알림
        send_notification({
            'type': 'workflow_failure',
            'workflow': workflow_run['name'],
            'run_id': workflow_run['id'],
            'url': workflow_run['html_url'],
            'repository': payload['repository']['full_name'],
        })
        
        # 자동 이슈 생성
        if workflow_run['name'] == 'Nightly Build':
            create_build_failure_issue(payload)

def create_build_failure_issue(payload):
    """빌드 실패 이슈 자동 생성"""
    repo_name = payload['repository']['full_name']
    workflow_run = payload['workflow_run']
    
    repo = g.get_repo(repo_name)
    
    # 중복 이슈 확인
    existing_issues = repo.get_issues(
        state='open',
        labels=['build-failure', 'automated']
    )
    
    for issue in existing_issues:
        if f"Run ID: {workflow_run['id']}" in issue.body:
            return  # 이미 이슈가 존재함
    
    # 새 이슈 생성
    issue = repo.create_issue(
        title=f"Build Failure: {workflow_run['name']}",
        body=f"""
## Build Failure Report

**Workflow**: {workflow_run['name']}
**Run ID**: {workflow_run['id']}
**Branch**: {workflow_run['head_branch']}
**Commit**: {workflow_run['head_sha'][:7]}
**Time**: {workflow_run['created_at']}

[View Workflow Run]({workflow_run['html_url']})

### Automated Issue
This issue was automatically created by the webhook handler.
        """,
        labels=['build-failure', 'automated', 'high-priority']
    )

if __name__ == '__main__':
    app.run(port=3000, debug=True)
```

## 3. GitHub API 활용

### REST API 기본 사용법

```javascript
// github-api-client.js
const { Octokit } = require('@octokit/rest');
const { retry } = require('@octokit/plugin-retry');
const { throttling } = require('@octokit/plugin-throttling');

// 플러그인이 적용된 Octokit 생성
const MyOctokit = Octokit.plugin(retry, throttling);

const octokit = new MyOctokit({
  auth: process.env.GITHUB_TOKEN,
  throttle: {
    onRateLimit: (retryAfter, options) => {
      console.warn(`Rate limit hit for ${options.method} ${options.url}`);
      if (options.request.retryCount === 0) {
        console.log(`Retrying after ${retryAfter} seconds!`);
        return true;
      }
    },
    onAbuseLimit: (retryAfter, options) => {
      console.warn(`Abuse limit hit for ${options.method} ${options.url}`);
    },
  },
});

// Repository 정보 조회
async function getRepositoryInfo(owner, repo) {
  try {
    const { data } = await octokit.repos.get({ owner, repo });
    return {
      name: data.name,
      description: data.description,
      stars: data.stargazers_count,
      forks: data.forks_count,
      language: data.language,
      topics: data.topics,
      created_at: data.created_at,
      updated_at: data.updated_at,
    };
  } catch (error) {
    console.error('Error fetching repository:', error.message);
    throw error;
  }
}

// 모든 이슈 가져오기 (페이지네이션)
async function getAllIssues(owner, repo, state = 'all') {
  const issues = await octokit.paginate(octokit.issues.listForRepo, {
    owner,
    repo,
    state,
    per_page: 100,
  });
  
  return issues.filter(issue => !issue.pull_request);
}

// PR 통계 생성
async function generatePRStats(owner, repo, days = 30) {
  const since = new Date();
  since.setDate(since.getDate() - days);
  
  const pulls = await octokit.paginate(octokit.pulls.list, {
    owner,
    repo,
    state: 'all',
    sort: 'created',
    direction: 'desc',
    per_page: 100,
  });
  
  const recentPulls = pulls.filter(pr => 
    new Date(pr.created_at) > since
  );
  
  const stats = {
    total: recentPulls.length,
    open: recentPulls.filter(pr => pr.state === 'open').length,
    merged: recentPulls.filter(pr => pr.merged_at).length,
    avgTimeToMerge: 0,
    contributors: new Set(),
  };
  
  let totalMergeTime = 0;
  let mergedCount = 0;
  
  for (const pr of recentPulls) {
    stats.contributors.add(pr.user.login);
    
    if (pr.merged_at) {
      const created = new Date(pr.created_at);
      const merged = new Date(pr.merged_at);
      totalMergeTime += merged - created;
      mergedCount++;
    }
  }
  
  if (mergedCount > 0) {
    stats.avgTimeToMerge = totalMergeTime / mergedCount / (1000 * 60 * 60); // hours
  }
  
  stats.contributors = stats.contributors.size;
  
  return stats;
}

// 자동화된 릴리즈 생성
async function createAutomatedRelease(owner, repo, tagName, options = {}) {
  // 최근 릴리즈 확인
  const { data: latestRelease } = await octokit.repos.getLatestRelease({
    owner,
    repo,
  }).catch(() => ({ data: null }));
  
  // 변경사항 수집
  const compareBase = latestRelease ? latestRelease.tag_name : 'HEAD~20';
  const { data: comparison } = await octokit.repos.compareCommits({
    owner,
    repo,
    base: compareBase,
    head: 'HEAD',
  });
  
  // 릴리즈 노트 생성
  const releaseNotes = generateReleaseNotes(comparison.commits);
  
  // 릴리즈 생성
  const { data: release } = await octokit.repos.createRelease({
    owner,
    repo,
    tag_name: tagName,
    name: options.name || tagName,
    body: releaseNotes,
    draft: options.draft || false,
    prerelease: options.prerelease || false,
    target_commitish: options.target || 'main',
  });
  
  return release;
}

function generateReleaseNotes(commits) {
  const notes = {
    features: [],
    fixes: [],
    others: [],
  };
  
  for (const commit of commits) {
    const message = commit.commit.message;
    const firstLine = message.split('\n')[0];
    
    if (firstLine.startsWith('feat:')) {
      notes.features.push(firstLine);
    } else if (firstLine.startsWith('fix:')) {
      notes.fixes.push(firstLine);
    } else {
      notes.others.push(firstLine);
    }
  }
  
  let releaseBody = '## What\'s Changed\n\n';
  
  if (notes.features.length > 0) {
    releaseBody += '### 🚀 New Features\n';
    notes.features.forEach(feat => {
      releaseBody += `- ${feat}\n`;
    });
    releaseBody += '\n';
  }
  
  if (notes.fixes.length > 0) {
    releaseBody += '### 🐛 Bug Fixes\n';
    notes.fixes.forEach(fix => {
      releaseBody += `- ${fix}\n`;
    });
    releaseBody += '\n';
  }
  
  if (notes.others.length > 0) {
    releaseBody += '### 📝 Other Changes\n';
    notes.others.forEach(other => {
      releaseBody += `- ${other}\n`;
    });
  }
  
  return releaseBody;
}

module.exports = {
  octokit,
  getRepositoryInfo,
  getAllIssues,
  generatePRStats,
  createAutomatedRelease,
};
```

### GraphQL API 사용

```javascript
// github-graphql-client.js
const { graphql } = require('@octokit/graphql');

const graphqlWithAuth = graphql.defaults({
  headers: {
    authorization: `token ${process.env.GITHUB_TOKEN}`,
  },
});

// 복잡한 쿼리 예제
async function getRepositoryDetails(owner, name) {
  const query = `
    query GetRepoDetails($owner: String!, $name: String!) {
      repository(owner: $owner, name: $name) {
        name
        description
        createdAt
        updatedAt
        primaryLanguage {
          name
          color
        }
        languages(first: 10) {
          edges {
            node {
              name
              color
            }
            size
          }
          totalSize
        }
        issues(states: OPEN) {
          totalCount
        }
        pullRequests(states: OPEN) {
          totalCount
        }
        stargazers {
          totalCount
        }
        forks {
          totalCount
        }
        releases(first: 5, orderBy: {field: CREATED_AT, direction: DESC}) {
          nodes {
            tagName
            createdAt
            author {
              login
            }
          }
        }
        collaborators(first: 10) {
          edges {
            node {
              login
              avatarUrl
            }
            permission
          }
        }
        defaultBranchRef {
          name
          target {
            ... on Commit {
              history(first: 1) {
                nodes {
                  message
                  committedDate
                  author {
                    name
                  }
                }
              }
            }
          }
        }
      }
    }
  `;
  
  try {
    const result = await graphqlWithAuth(query, { owner, name });
    return result.repository;
  } catch (error) {
    console.error('GraphQL query failed:', error.message);
    throw error;
  }
}

// 사용자 기여도 분석
async function analyzeUserContributions(username, days = 365) {
  const since = new Date();
  since.setDate(since.getDate() - days);
  
  const query = `
    query GetUserContributions($username: String!, $since: DateTime!) {
      user(login: $username) {
        login
        name
        bio
        contributionsCollection(from: $since) {
          totalCommitContributions
          totalIssueContributions
          totalPullRequestContributions
          totalPullRequestReviewContributions
          contributionCalendar {
            totalContributions
            weeks {
              contributionDays {
                contributionCount
                date
                color
              }
            }
          }
          commitContributionsByRepository(maxRepositories: 10) {
            repository {
              name
              owner {
                login
              }
            }
            contributions {
              totalCount
            }
          }
        }
        repositories(first: 100, ownerAffiliations: OWNER) {
          totalCount
          nodes {
            name
            stargazers {
              totalCount
            }
          }
        }
        followers {
          totalCount
        }
        following {
          totalCount
        }
      }
    }
  `;
  
  const result = await graphqlWithAuth(query, {
    username,
    since: since.toISOString(),
  });
  
  return result.user;
}

// 조직 분석
async function analyzeOrganization(orgName) {
  const query = `
    query AnalyzeOrg($orgName: String!) {
      organization(login: $orgName) {
        name
        description
        createdAt
        repositories(first: 100, privacy: PUBLIC) {
          totalCount
          nodes {
            name
            primaryLanguage {
              name
            }
            stargazers {
              totalCount
            }
            forks {
              totalCount
            }
            issues(states: OPEN) {
              totalCount
            }
          }
        }
        membersWithRole {
          totalCount
        }
        teams(first: 20) {
          totalCount
          nodes {
            name
            description
            members {
              totalCount
            }
          }
        }
      }
    }
  `;
  
  const result = await graphqlWithAuth(query, { orgName });
  
  // 통계 계산
  const stats = {
    totalRepos: result.organization.repositories.totalCount,
    totalStars: 0,
    totalForks: 0,
    totalOpenIssues: 0,
    languageDistribution: {},
    topRepos: [],
  };
  
  result.organization.repositories.nodes.forEach(repo => {
    stats.totalStars += repo.stargazers.totalCount;
    stats.totalForks += repo.forks.totalCount;
    stats.totalOpenIssues += repo.issues.totalCount;
    
    if (repo.primaryLanguage) {
      const lang = repo.primaryLanguage.name;
      stats.languageDistribution[lang] = (stats.languageDistribution[lang] || 0) + 1;
    }
  });
  
  stats.topRepos = result.organization.repositories.nodes
    .sort((a, b) => b.stargazers.totalCount - a.stargazers.totalCount)
    .slice(0, 5)
    .map(repo => ({
      name: repo.name,
      stars: repo.stargazers.totalCount,
    }));
  
  return {
    organization: result.organization,
    stats,
  };
}

module.exports = {
  graphqlWithAuth,
  getRepositoryDetails,
  analyzeUserContributions,
  analyzeOrganization,
};
```

## 4. Webhook과 API 통합 예제

### 자동 이슈 관리 시스템

```javascript
// issue-automation.js
const express = require('express');
const { Octokit } = require('@octokit/rest');
const natural = require('natural');

const app = express();
const octokit = new Octokit({ auth: process.env.GITHUB_TOKEN });

// NLP 분류기 초기화
const classifier = new natural.BayesClassifier();

// 학습 데이터
const trainingData = [
  { text: 'app crashes when clicking button', category: 'bug' },
  { text: 'add dark mode support', category: 'enhancement' },
  { text: 'update installation guide', category: 'documentation' },
  { text: 'improve performance of search', category: 'performance' },
  { text: 'security vulnerability in dependencies', category: 'security' },
];

trainingData.forEach(item => {
  classifier.addDocument(item.text, item.category);
});
classifier.train();

// 이슈 자동 분류 및 처리
app.post('/webhook/issues', async (req, res) => {
  const { action, issue, repository } = req.body;
  
  if (action !== 'opened') {
    return res.status(200).send('OK');
  }
  
  try {
    // 1. 이슈 내용 분석
    const category = classifier.classify(issue.title + ' ' + issue.body);
    const sentiment = analyzeSentiment(issue.body);
    const priority = calculatePriority(issue, category, sentiment);
    
    // 2. 라벨 추가
    const labels = [category];
    if (priority === 'high') labels.push('high-priority');
    if (sentiment < -0.5) labels.push('urgent');
    
    await octokit.issues.addLabels({
      owner: repository.owner.login,
      repo: repository.name,
      issue_number: issue.number,
      labels,
    });
    
    // 3. 자동 할당
    const assignee = await findBestAssignee(repository, category);
    if (assignee) {
      await octokit.issues.addAssignees({
        owner: repository.owner.login,
        repo: repository.name,
        issue_number: issue.number,
        assignees: [assignee],
      });
    }
    
    // 4. 프로젝트 보드에 추가
    await addToProjectBoard(repository, issue, category);
    
    // 5. 자동 응답
    const response = generateAutoResponse(category, issue);
    await octokit.issues.createComment({
      owner: repository.owner.login,
      repo: repository.name,
      issue_number: issue.number,
      body: response,
    });
    
    // 6. 관련 이슈 찾기
    const relatedIssues = await findRelatedIssues(repository, issue);
    if (relatedIssues.length > 0) {
      await octokit.issues.createComment({
        owner: repository.owner.login,
        repo: repository.name,
        issue_number: issue.number,
        body: `관련 이슈: ${relatedIssues.map(i => `#${i.number}`).join(', ')}`,
      });
    }
    
    res.status(200).send('Processed');
  } catch (error) {
    console.error('Error processing issue:', error);
    res.status(500).send('Error');
  }
});

// 최적의 담당자 찾기
async function findBestAssignee(repository, category) {
  // 최근 활동 기록 조회
  const { data: events } = await octokit.activity.listRepoEvents({
    owner: repository.owner.login,
    repo: repository.name,
    per_page: 100,
  });
  
  // 카테고리별 전문가 찾기
  const experts = {};
  
  for (const event of events) {
    if (event.type === 'PullRequestEvent' && event.payload.pull_request.merged) {
      const files = await octokit.pulls.listFiles({
        owner: repository.owner.login,
        repo: repository.name,
        pull_number: event.payload.pull_request.number,
      });
      
      // 파일 패턴 기반 전문성 판단
      const expertise = determineExpertise(files.data);
      if (expertise === category) {
        experts[event.actor.login] = (experts[event.actor.login] || 0) + 1;
      }
    }
  }
  
  // 가장 활발한 전문가 선택
  const sortedExperts = Object.entries(experts)
    .sort(([, a], [, b]) => b - a)
    .map(([login]) => login);
  
  return sortedExperts[0] || null;
}

// 자동 응답 생성
function generateAutoResponse(category, issue) {
  const templates = {
    bug: `안녕하세요 @${issue.user.login}님,

버그 리포트를 제출해 주셔서 감사합니다. 저희 팀에서 확인 후 빠르게 대응하겠습니다.

추가로 다음 정보를 제공해 주시면 더 빠른 해결에 도움이 됩니다:
- 운영체제 및 버전
- 브라우저 정보 (웹 애플리케이션인 경우)
- 에러 메시지나 스크린샷
- 재현 가능한 최소 코드 예제`,
    
    enhancement: `안녕하세요 @${issue.user.login}님,

좋은 제안 감사합니다! 저희 팀에서 검토 후 구현 가능성을 평가하겠습니다.

다음 정보를 추가로 제공해 주시면 도움이 됩니다:
- 제안하신 기능의 구체적인 사용 사례
- 예상되는 사용자 경험
- 참고할 만한 다른 프로젝트나 구현 사례`,
    
    documentation: `안녕하세요 @${issue.user.login}님,

문서 개선 제안 감사합니다. 문서팀에서 검토 후 업데이트하겠습니다.

혹시 직접 기여하고 싶으시다면 PR을 제출해 주셔도 좋습니다!`,
  };
  
  return templates[category] || `감사합니다. 곧 검토하겠습니다.`;
}
```

### CI/CD 상태 대시보드

```javascript
// ci-dashboard.js
const express = require('express');
const { graphql } = require('@octokit/graphql');

const app = express();

// 실시간 CI/CD 상태 모니터링
app.get('/api/ci-status/:owner/:repo', async (req, res) => {
  const { owner, repo } = req.params;
  
  const query = `
    query GetCIStatus($owner: String!, $repo: String!) {
      repository(owner: $owner, name: $repo) {
        defaultBranchRef {
          name
          target {
            ... on Commit {
              statusCheckRollup {
                state
                contexts(first: 100) {
                  nodes {
                    __typename
                    ... on CheckRun {
                      name
                      status
                      conclusion
                      startedAt
                      completedAt
                      detailsUrl
                    }
                    ... on StatusContext {
                      state
                      description
                      context
                      targetUrl
                    }
                  }
                }
              }
            }
          }
        }
        pullRequests(first: 10, states: OPEN, orderBy: {field: UPDATED_AT, direction: DESC}) {
          nodes {
            number
            title
            author {
              login
            }
            commits(last: 1) {
              nodes {
                commit {
                  statusCheckRollup {
                    state
                  }
                }
              }
            }
          }
        }
      }
    }
  `;
  
  try {
    const result = await graphql(query, {
      owner,
      repo,
      headers: {
        authorization: `token ${process.env.GITHUB_TOKEN}`,
      },
    });
    
    const status = {
      mainBranch: {
        name: result.repository.defaultBranchRef.name,
        status: result.repository.defaultBranchRef.target.statusCheckRollup?.state || 'UNKNOWN',
        checks: result.repository.defaultBranchRef.target.statusCheckRollup?.contexts.nodes || [],
      },
      pullRequests: result.repository.pullRequests.nodes.map(pr => ({
        number: pr.number,
        title: pr.title,
        author: pr.author.login,
        status: pr.commits.nodes[0]?.commit.statusCheckRollup?.state || 'UNKNOWN',
      })),
    };
    
    res.json(status);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Webhook으로 실시간 업데이트
app.post('/webhook/status', async (req, res) => {
  const { state, sha, repository, context, target_url } = req.body;
  
  // WebSocket으로 클라이언트에 실시간 전송
  io.emit('ci-status-update', {
    repository: repository.full_name,
    commit: sha,
    context,
    state,
    url: target_url,
    timestamp: new Date().toISOString(),
  });
  
  // 실패 시 알림
  if (state === 'failure') {
    await sendFailureNotification({
      repository: repository.full_name,
      context,
      sha,
      url: target_url,
    });
  }
  
  res.status(200).send('OK');
});
```

## 5. 보안 고려사항

### Webhook 보안 강화

```javascript
// webhook-security.js
const crypto = require('crypto');
const ipRangeCheck = require('ip-range-check');

// GitHub의 IP 범위
const GITHUB_IP_RANGES = [
  '192.30.252.0/22',
  '185.199.108.0/22',
  '140.82.112.0/20',
  '143.55.64.0/20',
];

// IP 화이트리스트 검증
function verifyGitHubIP(req) {
  const clientIP = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
  return ipRangeCheck(clientIP, GITHUB_IP_RANGES);
}

// 서명 검증 (타이밍 공격 방지)
function verifySignature(payload, signature, secret) {
  if (!signature) return false;
  
  const [algo, hash] = signature.split('=');
  if (algo !== 'sha256') return false;
  
  const expectedHash = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  
  return crypto.timingSafeEqual(
    Buffer.from(hash),
    Buffer.from(expectedHash)
  );
}

// 재전송 공격 방지
const processedDeliveries = new Set();

function preventReplay(deliveryId) {
  if (processedDeliveries.has(deliveryId)) {
    return false;
  }
  
  processedDeliveries.add(deliveryId);
  
  // 1시간 후 제거
  setTimeout(() => {
    processedDeliveries.delete(deliveryId);
  }, 60 * 60 * 1000);
  
  return true;
}

// 보안 미들웨어
function webhookSecurity(secret) {
  return (req, res, next) => {
    // 1. IP 검증
    if (!verifyGitHubIP(req)) {
      return res.status(403).send('Forbidden: Invalid IP');
    }
    
    // 2. 서명 검증
    const signature = req.headers['x-hub-signature-256'];
    if (!verifySignature(req.rawBody, signature, secret)) {
      return res.status(401).send('Unauthorized: Invalid signature');
    }
    
    // 3. 재전송 공격 방지
    const deliveryId = req.headers['x-github-delivery'];
    if (!preventReplay(deliveryId)) {
      return res.status(409).send('Conflict: Duplicate delivery');
    }
    
    // 4. 이벤트 타입 검증
    const event = req.headers['x-github-event'];
    const allowedEvents = ['push', 'pull_request', 'issues', 'release'];
    if (!allowedEvents.includes(event)) {
      return res.status(400).send('Bad Request: Unsupported event');
    }
    
    next();
  };
}

// 사용 예제
app.post('/webhook',
  express.raw({ type: 'application/json' }),
  webhookSecurity(process.env.WEBHOOK_SECRET),
  async (req, res) => {
    // 안전하게 처리
    const payload = JSON.parse(req.body);
    // ...
  }
);
```

### API 토큰 관리

```javascript
// token-manager.js
const crypto = require('crypto');
const jwt = require('jsonwebtoken');

class TokenManager {
  constructor() {
    this.tokens = new Map();
    this.rotationInterval = 30 * 24 * 60 * 60 * 1000; // 30일
  }
  
  // 앱 설치 토큰 생성 (GitHub App)
  async generateInstallationToken(installationId) {
    const appId = process.env.GITHUB_APP_ID;
    const privateKey = process.env.GITHUB_APP_PRIVATE_KEY;
    
    // JWT 생성
    const now = Math.floor(Date.now() / 1000);
    const payload = {
      iat: now - 60,
      exp: now + 600,
      iss: appId,
    };
    
    const jwtToken = jwt.sign(payload, privateKey, {
      algorithm: 'RS256',
    });
    
    // 설치 토큰 요청
    const response = await fetch(
      `https://api.github.com/app/installations/${installationId}/access_tokens`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${jwtToken}`,
          'Accept': 'application/vnd.github.v3+json',
        },
      }
    );
    
    const data = await response.json();
    
    // 토큰 저장 및 만료 추적
    this.tokens.set(installationId, {
      token: data.token,
      expiresAt: new Date(data.expires_at),
    });
    
    return data.token;
  }
  
  // 토큰 갱신
  async refreshToken(installationId) {
    const tokenData = this.tokens.get(installationId);
    if (!tokenData) {
      return this.generateInstallationToken(installationId);
    }
    
    const now = new Date();
    const expiresIn = tokenData.expiresAt - now;
    
    // 5분 이내 만료 시 갱신
    if (expiresIn < 5 * 60 * 1000) {
      return this.generateInstallationToken(installationId);
    }
    
    return tokenData.token;
  }
  
  // Fine-grained PAT 검증
  async validateToken(token) {
    try {
      const response = await fetch('https://api.github.com/user', {
        headers: {
          'Authorization': `token ${token}`,
        },
      });
      
      if (!response.ok) {
        return { valid: false, reason: 'Invalid token' };
      }
      
      // 권한 확인
      const scopes = response.headers.get('x-oauth-scopes');
      const rateLimit = {
        limit: response.headers.get('x-ratelimit-limit'),
        remaining: response.headers.get('x-ratelimit-remaining'),
        reset: new Date(response.headers.get('x-ratelimit-reset') * 1000),
      };
      
      return {
        valid: true,
        scopes: scopes ? scopes.split(', ') : [],
        rateLimit,
      };
    } catch (error) {
      return { valid: false, reason: error.message };
    }
  }
}

module.exports = new TokenManager();
```

## 6. 실전 통합 예제

### PR 자동 리뷰 봇

```javascript
// pr-review-bot.js
const { Octokit } = require('@octokit/rest');
const { minimatch } = require('minimatch');

class PRReviewBot {
  constructor(token) {
    this.octokit = new Octokit({ auth: token });
    this.rules = this.loadRules();
  }
  
  loadRules() {
    return {
      security: {
        patterns: [
          /api[_-]?key/i,
          /secret/i,
          /password/i,
          /token/i,
        ],
        message: '🔐 보안 경고: 민감한 정보가 포함되어 있을 수 있습니다.',
      },
      performance: {
        patterns: [
          /SELECT \* FROM/i,
          /\.forEach\(.+\.forEach/,
          /async.+await.+for/,
        ],
        message: '⚡ 성능: 이 패턴은 성능 문제를 일으킬 수 있습니다.',
      },
      codeQuality: {
        maxFileSize: 500,
        maxFunctionLength: 50,
        maxComplexity: 10,
      },
    };
  }
  
  async reviewPR(owner, repo, prNumber) {
    // PR 정보 가져오기
    const { data: pr } = await this.octokit.pulls.get({
      owner,
      repo,
      pull_number: prNumber,
    });
    
    // 변경된 파일 가져오기
    const { data: files } = await this.octokit.pulls.listFiles({
      owner,
      repo,
      pull_number: prNumber,
    });
    
    const comments = [];
    const problems = [];
    
    for (const file of files) {
      // 파일 크기 체크
      if (file.additions > this.rules.codeQuality.maxFileSize) {
        problems.push({
          path: file.filename,
          message: `파일이 너무 큽니다 (${file.additions}줄). 분할을 고려해주세요.`,
        });
      }
      
      // 파일 내용 분석
      if (file.patch) {
        const issues = await this.analyzeFile(file);
        comments.push(...issues);
      }
    }
    
    // 리뷰 제출
    if (comments.length > 0 || problems.length > 0) {
      await this.submitReview(owner, repo, prNumber, pr.head.sha, comments, problems);
    }
    
    // 자동 라벨 추가
    await this.addLabels(owner, repo, prNumber, files);
    
    return { comments: comments.length, problems: problems.length };
  }
  
  async analyzeFile(file) {
    const comments = [];
    const lines = file.patch.split('\n');
    
    lines.forEach((line, index) => {
      // 추가된 라인만 검사
      if (!line.startsWith('+')) return;
      
      // 보안 패턴 검사
      for (const pattern of this.rules.security.patterns) {
        if (pattern.test(line)) {
          comments.push({
            path: file.filename,
            line: this.getLineNumber(file.patch, index),
            body: this.rules.security.message,
          });
        }
      }
      
      // 성능 패턴 검사
      for (const pattern of this.rules.performance.patterns) {
        if (pattern.test(line)) {
          comments.push({
            path: file.filename,
            line: this.getLineNumber(file.patch, index),
            body: this.rules.performance.message,
          });
        }
      }
      
      // TODO 주석 찾기
      if (/TODO|FIXME|HACK/i.test(line)) {
        comments.push({
          path: file.filename,
          line: this.getLineNumber(file.patch, index),
          body: '📝 TODO 주석이 발견되었습니다. 이슈로 등록하는 것을 고려해주세요.',
        });
      }
    });
    
    return comments;
  }
  
  getLineNumber(patch, patchLineIndex) {
    const lines = patch.split('\n');
    let lineNumber = 0;
    
    for (let i = 0; i <= patchLineIndex; i++) {
      const line = lines[i];
      if (line.startsWith('@@')) {
        const match = line.match(/@@ -\d+,?\d* \+(\d+)/);
        if (match) {
          lineNumber = parseInt(match[1]) - 1;
        }
      } else if (!line.startsWith('-')) {
        lineNumber++;
      }
    }
    
    return lineNumber;
  }
  
  async submitReview(owner, repo, prNumber, commitSha, comments, problems) {
    const body = problems.length > 0
      ? `## 전체적인 피드백\n\n${problems.map(p => `- **${p.path}**: ${p.message}`).join('\n')}`
      : '';
    
    await this.octokit.pulls.createReview({
      owner,
      repo,
      pull_number: prNumber,
      commit_id: commitSha,
      body,
      event: 'COMMENT',
      comments: comments.map(c => ({
        path: c.path,
        line: c.line,
        body: c.body,
      })),
    });
  }
  
  async addLabels(owner, repo, prNumber, files) {
    const labels = new Set();
    
    files.forEach(file => {
      if (file.filename.includes('test')) {
        labels.add('tests');
      }
      if (file.filename.endsWith('.md')) {
        labels.add('documentation');
      }
      if (file.filename.startsWith('src/')) {
        labels.add('source-code');
      }
      if (file.filename.includes('package.json')) {
        labels.add('dependencies');
      }
    });
    
    if (labels.size > 0) {
      await this.octokit.issues.addLabels({
        owner,
        repo,
        issue_number: prNumber,
        labels: Array.from(labels),
      });
    }
  }
}

// Webhook 핸들러
app.post('/webhook/pr', async (req, res) => {
  const { action, pull_request, repository } = req.body;
  
  if (action === 'opened' || action === 'synchronize') {
    const bot = new PRReviewBot(process.env.GITHUB_TOKEN);
    
    try {
      const result = await bot.reviewPR(
        repository.owner.login,
        repository.name,
        pull_request.number
      );
      
      console.log(`Reviewed PR #${pull_request.number}: ${result.comments} comments, ${result.problems} problems`);
      res.status(200).send('Review completed');
    } catch (error) {
      console.error('Review failed:', error);
      res.status(500).send('Review failed');
    }
  } else {
    res.status(200).send('Ignored');
  }
});
```

## 2025년 최신 API 기능

### GitHub API 버전별 비교

| 버전 | REST API v3 | GraphQL v4 | REST API v4 (2025) |
|------|-------------|------------|-------------------|
| **프로토콜** | REST | GraphQL | REST + GraphQL |
| **데이터 페칭** | 과다 페칭 | 필요한 것만 | 하이브리드 |
| **Rate Limit** | 5,000/시간 | 5,000 포인트/시간 | 10,000/시간 |
| **실시간 기능** | 없음 | Subscription | WebSocket 지원 |
| **캐싱** | HTTP 캐싱 | 제한적 | 스마트 캐싱 |

### 통합 자동화 아키텍처

```mermaid
graph TB
    A[GitHub] --> B[Webhook Event]
    B --> C[Event Router]
    
    C --> D[CI/CD Pipeline]
    C --> E[Issue Automation]
    C --> F[Code Review Bot]
    C --> G[Notification Service]
    
    D --> H[Deploy]
    E --> I[Jira Sync]
    F --> J[AI Analysis]
    G --> K[Slack/Teams]
```

## 마무리

GitHub Webhooks와 API는 강력한 자동화와 통합을 가능하게 하는 핵심 도구입니다.

### 핵심 포인트 정리

| 영역 | 주요 내용 | 효과 |
|------|-----------|------|
| **실시간 처리** | Webhooks 이벤트 기반 자동화 | 즉각적인 대응 |
| **데이터 접근** | REST/GraphQL API 활용 | 유연한 데이터 조작 |
| **보안** | 서명 검증, 토큰 관리 | 안전한 통신 |
| **확장성** | Rate Limit 관리, 병렬 처리 | 대규모 운영 가능 |
| **통합** | 다양한 서비스 연동 | 워크플로우 최적화 |

이러한 도구들을 활용하여 개발 워크플로우를 자동화하고, 생산성을 크게 향상시킬 수 있습니다.

다음 편에서는 GitHub Apps 개발에 대해 자세히 알아보겠습니다.

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
13. **Webhooks와 API** (현재 글)
14. GitHub Apps 개발 (예정)

---

*질문이나 피드백이 있으시다면 댓글로 남겨주세요!*