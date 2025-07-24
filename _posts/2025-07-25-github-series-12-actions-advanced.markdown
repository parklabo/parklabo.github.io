---
layout: post
title: "[GitHub 입문 #12] Actions 고급 활용: 복잡한 워크플로우 작성"
date: 2025-07-25 10:00:00 +0900
categories: [DevOps, GitHub]
tags: [github, actions, ci-cd, automation, advanced, tutorial, series]
mermaid: true
---

## 들어가며

GitHub 입문 시리즈의 열두 번째 시간입니다. 이번에는 GitHub Actions의 고급 기능들을 활용하여 복잡한 워크플로우를 작성하는 방법을 알아보겠습니다. 실제 프로덕션 환경에서 사용할 수 있는 고급 패턴들을 다룹니다.

## GitHub Actions 고급 기능 개요

| 기능 | 설명 | 사용 사례 | 복잡도 |
|------|------|-----------|---------|
| **복합 워크플로우** | 다단계 파이프라인 구성 | 복잡한 배포 프로세스 | 🔴 높음 |
| **커스텀 액션** | 재사용 가능한 액션 개발 | 팀 공통 작업 자동화 | 🔴 높음 |
| **동적 매트릭스** | 런타임 작업 생성 | 변경된 서비스만 배포 | 🟡 중간 |
| **워크플로우 체인** | 워크플로우 간 연결 | 순차적 파이프라인 | 🟡 중간 |
| **자체 호스팅 러너** | 전용 실행 환경 | 특수 환경 요구사항 | 🔴 높음 |
| **고급 캐싱** | 지능형 캐시 전략 | 빌드 시간 최적화 | 🟢 낮음 |

## 1. 복합 워크플로우 패턴

### 멀티 스테이지 배포 파이프라인

`.github/workflows/multi-stage-deploy.yml`:

{% raw %}
```yaml
name: Multi-Stage Deployment

on:
  push:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      skip-tests:
        description: 'Skip tests'
        type: boolean
        default: false
      target-env:
        description: 'Target environment'
        type: choice
        options:
          - dev
          - staging
          - production
        default: staging

concurrency:
  group: deploy-${{ github.ref }}-${{ inputs.target-env || 'staging' }}
  cancel-in-progress: false

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
      should-deploy: ${{ steps.check.outputs.should-deploy }}
      environments: ${{ steps.envs.outputs.environments }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Calculate version
        id: version
        run: |
          VERSION=$(git describe --tags --always --dirty)
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          
      - name: Check deployment conditions
        id: check
        run: |
          if [[ "${{ github.event_name }}" == "workflow_dispatch" ]] || 
             [[ "${{ github.ref }}" == "refs/heads/main" && "${{ github.event.head_commit.message }}" != *"[skip deploy]"* ]]; then
            echo "should-deploy=true" >> $GITHUB_OUTPUT
          else
            echo "should-deploy=false" >> $GITHUB_OUTPUT
          fi
          
      - name: Determine environments
        id: envs
        run: |
          if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
            echo "environments=[\"${{ inputs.target-env }}\"]" >> $GITHUB_OUTPUT
          else
            echo "environments=[\"dev\", \"staging\"]" >> $GITHUB_OUTPUT
          fi

  test:
    needs: prepare
    if: ${{ !inputs.skip-tests }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        test-suite: [unit, integration, e2e]
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup test environment
        run: |
          npm ci
          docker-compose up -d
          
      - name: Run ${{ matrix.test-suite }} tests
        run: npm run test:${{ matrix.test-suite }}
        
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results-${{ matrix.test-suite }}
          path: test-results/

  build:
    needs: [prepare, test]
    if: |
      always() && 
      needs.prepare.outputs.should-deploy == 'true' &&
      (needs.test.result == 'success' || needs.test.result == 'skipped')
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.docker.outputs.image-tag }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Build and push Docker image
        id: docker
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ needs.prepare.outputs.version }}
            ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ needs.prepare.outputs.version }}
            COMMIT_SHA=${{ github.sha }}
            
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/${{ github.repository }}:${{ needs.prepare.outputs.version }}
          format: spdx-json
          output-file: sbom.spdx.json
          
      - name: Security scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ needs.prepare.outputs.version }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          
      - name: Upload security results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  deploy:
    needs: [prepare, build]
    if: needs.prepare.outputs.should-deploy == 'true'
    strategy:
      matrix:
        environment: ${{ fromJson(needs.prepare.outputs.environments) }}
      max-parallel: 1
    runs-on: ubuntu-latest
    environment:
      name: ${{ matrix.environment }}
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to ${{ matrix.environment }}
        id: deploy
        run: |
          echo "Deploying version ${{ needs.prepare.outputs.version }} to ${{ matrix.environment }}"
          # Deployment logic here
          echo "url=https://${{ matrix.environment }}.example.com" >> $GITHUB_OUTPUT
          
      - name: Smoke tests
        run: |
          curl -f ${{ steps.deploy.outputs.url }}/health || exit 1
          
      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: |
            Deployment to ${{ matrix.environment }} ${{ job.status }}
            Version: ${{ needs.prepare.outputs.version }}
            URL: ${{ steps.deploy.outputs.url }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```
{% endraw %}

## 2. 커스텀 액션 만들기

### Composite Action

`action.yml`:

{% raw %}
```yaml
name: 'Deploy to Kubernetes'
description: 'Deploy application to Kubernetes cluster'
author: 'Your Name'

inputs:
  cluster-name:
    description: 'Kubernetes cluster name'
    required: true
  namespace:
    description: 'Kubernetes namespace'
    required: true
    default: 'default'
  image:
    description: 'Docker image to deploy'
    required: true
  replicas:
    description: 'Number of replicas'
    required: false
    default: '3'
  config-file:
    description: 'Kubernetes config file'
    required: false
    default: 'k8s/deployment.yaml'

outputs:
  deployment-name:
    description: 'Name of the deployment'
    value: ${{ steps.deploy.outputs.deployment-name }}
  service-url:
    description: 'Service URL'
    value: ${{ steps.get-url.outputs.url }}

runs:
  using: 'composite'
  steps:
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'latest'
        
    - name: Configure kubeconfig
      shell: bash
      run: |
        mkdir -p ~/.kube
        echo "${{ env.KUBE_CONFIG }}" | base64 -d > ~/.kube/config
        
    - name: Update deployment manifest
      shell: bash
      run: |
        sed -i "s|IMAGE_PLACEHOLDER|${{ inputs.image }}|g" ${{ inputs.config-file }}
        sed -i "s|REPLICAS_PLACEHOLDER|${{ inputs.replicas }}|g" ${{ inputs.config-file }}
        
    - name: Deploy to Kubernetes
      id: deploy
      shell: bash
      run: |
        kubectl apply -f ${{ inputs.config-file }} -n ${{ inputs.namespace }}
        DEPLOYMENT_NAME=$(kubectl get deployment -n ${{ inputs.namespace }} -o jsonpath='{.items[0].metadata.name}')
        echo "deployment-name=$DEPLOYMENT_NAME" >> $GITHUB_OUTPUT
        
    - name: Wait for rollout
      shell: bash
      run: |
        kubectl rollout status deployment/${{ steps.deploy.outputs.deployment-name }} \
          -n ${{ inputs.namespace }} \
          --timeout=5m
          
    - name: Get service URL
      id: get-url
      shell: bash
      run: |
        SERVICE_IP=$(kubectl get svc -n ${{ inputs.namespace }} -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
        echo "url=http://$SERVICE_IP" >> $GITHUB_OUTPUT

branding:
  icon: 'upload-cloud'
  color: 'blue'
```
{% endraw %}

### JavaScript Action

`index.js`:

```javascript
const core = require('@actions/core');
const github = require('@actions/github');
const exec = require('@actions/exec');

async function run() {
  try {
    // Get inputs
    const token = core.getInput('github-token', { required: true });
    const issueNumber = core.getInput('issue-number', { required: true });
    const environment = core.getInput('environment', { required: true });
    
    // Initialize Octokit
    const octokit = github.getOctokit(token);
    const context = github.context;
    
    // Validate deployment
    core.info(`Validating deployment for issue #${issueNumber}`);
    
    const { data: issue } = await octokit.rest.issues.get({
      owner: context.repo.owner,
      repo: context.repo.repo,
      issue_number: issueNumber,
    });
    
    // Check if issue has deployment label
    const hasDeployLabel = issue.labels.some(label => 
      label.name === `deploy:${environment}`
    );
    
    if (!hasDeployLabel) {
      core.setFailed(`Issue #${issueNumber} does not have deploy:${environment} label`);
      return;
    }
    
    // Parse deployment config from issue body
    const configMatch = issue.body.match(/```yaml\n([\s\S]*?)\n```/);
    if (!configMatch) {
      core.setFailed('No deployment configuration found in issue body');
      return;
    }
    
    const config = configMatch[1];
    
    // Create deployment
    const { data: deployment } = await octokit.rest.repos.createDeployment({
      owner: context.repo.owner,
      repo: context.repo.repo,
      ref: context.sha,
      environment,
      description: `Deploy from issue #${issueNumber}`,
      auto_merge: false,
      required_contexts: [],
    });
    
    // Set deployment status
    await octokit.rest.repos.createDeploymentStatus({
      owner: context.repo.owner,
      repo: context.repo.repo,
      deployment_id: deployment.id,
      state: 'in_progress',
      description: 'Deployment started',
    });
    
    try {
      // Execute deployment
      await exec.exec('deploy.sh', [environment, config]);
      
      // Update deployment status
      await octokit.rest.repos.createDeploymentStatus({
        owner: context.repo.owner,
        repo: context.repo.repo,
        deployment_id: deployment.id,
        state: 'success',
        description: 'Deployment completed',
        environment_url: `https://${environment}.example.com`,
      });
      
      // Comment on issue
      await octokit.rest.issues.createComment({
        owner: context.repo.owner,
        repo: context.repo.repo,
        issue_number: issueNumber,
        body: `✅ Deployment to ${environment} completed successfully!`,
      });
      
    } catch (error) {
      // Update deployment status on failure
      await octokit.rest.repos.createDeploymentStatus({
        owner: context.repo.owner,
        repo: context.repo.repo,
        deployment_id: deployment.id,
        state: 'failure',
        description: 'Deployment failed',
      });
      
      throw error;
    }
    
    // Set outputs
    core.setOutput('deployment-id', deployment.id);
    core.setOutput('environment-url', `https://${environment}.example.com`);
    
  } catch (error) {
    core.setFailed(error.message);
  }
}

run();
```

`action.yml`:

```yaml
name: 'Issue-based Deployment'
description: 'Deploy based on issue configuration'
inputs:
  github-token:
    description: 'GitHub token'
    required: true
  issue-number:
    description: 'Issue number containing deployment config'
    required: true
  environment:
    description: 'Target environment'
    required: true
outputs:
  deployment-id:
    description: 'GitHub deployment ID'
  environment-url:
    description: 'Environment URL'
runs:
  using: 'node16'
  main: 'dist/index.js'
```

## 3. 고급 워크플로우 제어

### 동적 Job 생성

`.github/workflows/dynamic-jobs.yml`:

{% raw %}
```yaml
name: Dynamic Job Generation

on:
  push:
    paths:
      - 'services/**'

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.detect.outputs.services }}
      matrix: ${{ steps.detect.outputs.matrix }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
          
      - name: Detect changed services
        id: detect
        run: |
          # Get changed files
          CHANGED_FILES=$(git diff --name-only HEAD^ HEAD)
          
          # Extract unique service names
          SERVICES=$(echo "$CHANGED_FILES" | grep '^services/' | cut -d'/' -f2 | sort -u)
          
          # Convert to JSON array
          SERVICES_JSON=$(echo "$SERVICES" | jq -R . | jq -s .)
          echo "services=$SERVICES_JSON" >> $GITHUB_OUTPUT
          
          # Create matrix
          MATRIX=$(echo "$SERVICES" | jq -R . | jq -s '{service: .}')
          echo "matrix=$MATRIX" >> $GITHUB_OUTPUT

  build-services:
    needs: detect-changes
    if: needs.detect-changes.outputs.services != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJson(needs.detect-changes.outputs.matrix) }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Build ${{ matrix.service }}
        run: |
          cd services/${{ matrix.service }}
          docker build -t ${{ matrix.service }}:latest .
          
      - name: Test ${{ matrix.service }}
        run: |
          cd services/${{ matrix.service }}
          npm test

  deploy-services:
    needs: [detect-changes, build-services]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJson(needs.detect-changes.outputs.matrix) }}
    steps:
      - name: Deploy ${{ matrix.service }}
        run: |
          echo "Deploying ${{ matrix.service }} to production"
```
{% endraw %}

### 워크플로우 체인과 의존성

`.github/workflows/workflow-chain.yml`:

{% raw %}
```yaml
name: Workflow Chain Controller

on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly
  workflow_dispatch:

jobs:
  trigger-sequence:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Trigger data pipeline
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.WORKFLOW_TOKEN }}
          script: |
            // Trigger data processing workflow
            const dataPipeline = await github.rest.actions.createWorkflowDispatch({
              owner: context.repo.owner,
              repo: context.repo.repo,
              workflow_id: 'data-pipeline.yml',
              ref: 'main',
              inputs: {
                mode: 'full',
                date: new Date().toISOString().split('T')[0]
              }
            });
            
            // Wait for completion
            let status = 'in_progress';
            while (status === 'in_progress') {
              await new Promise(resolve => setTimeout(resolve, 30000)); // 30s
              
              const runs = await github.rest.actions.listWorkflowRuns({
                owner: context.repo.owner,
                repo: context.repo.repo,
                workflow_id: 'data-pipeline.yml',
                per_page: 1
              });
              
              status = runs.data.workflow_runs[0].status;
            }
            
            if (status !== 'completed') {
              throw new Error('Data pipeline failed');
            }
            
            // Trigger ML training
            await github.rest.actions.createWorkflowDispatch({
              owner: context.repo.owner,
              repo: context.repo.repo,
              workflow_id: 'ml-training.yml',
              ref: 'main',
              inputs: {
                dataset: 'latest',
                epochs: '100'
              }
            });
```
{% endraw %}

## 4. 고급 시크릿 관리

### 환경별 시크릿 로테이션

`.github/workflows/secret-rotation.yml`:

{% raw %}
```yaml
name: Secret Rotation

on:
  schedule:
    - cron: '0 0 1 * *'  # Monthly
  workflow_dispatch:

jobs:
  rotate-secrets:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, production]
    environment: ${{ matrix.environment }}-admin
    steps:
      - name: Generate new credentials
        id: generate
        run: |
          # Generate new password
          NEW_PASSWORD=$(openssl rand -base64 32)
          echo "::add-mask::$NEW_PASSWORD"
          echo "password=$NEW_PASSWORD" >> $GITHUB_OUTPUT
          
          # Generate new API key
          NEW_API_KEY=$(uuidgen | tr -d '-' | tr '[:lower:]' '[:upper:]')
          echo "::add-mask::$NEW_API_KEY"
          echo "api-key=$NEW_API_KEY" >> $GITHUB_OUTPUT
          
      - name: Update cloud provider secrets
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          # Update AWS Secrets Manager
          aws secretsmanager update-secret \
            --secret-id ${{ matrix.environment }}/db-password \
            --secret-string "${{ steps.generate.outputs.password }}"
            
          aws secretsmanager update-secret \
            --secret-id ${{ matrix.environment }}/api-key \
            --secret-string "${{ steps.generate.outputs.api-key }}"
            
      - name: Update GitHub secrets
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.ADMIN_TOKEN }}
          script: |
            const sodium = require('tweetsodium');
            
            // Get public key
            const { data: { key, key_id } } = await github.rest.actions.getRepoPublicKey({
              owner: context.repo.owner,
              repo: context.repo.repo,
            });
            
            // Encrypt secrets
            const encryptSecret = (secret) => {
              const messageBytes = Buffer.from(secret);
              const keyBytes = Buffer.from(key, 'base64');
              const encryptedBytes = sodium.seal(messageBytes, keyBytes);
              return Buffer.from(encryptedBytes).toString('base64');
            };
            
            // Update repository secrets
            await github.rest.actions.createOrUpdateRepoSecret({
              owner: context.repo.owner,
              repo: context.repo.repo,
              secret_name: `${matrix.environment.toUpperCase()}_DB_PASSWORD`,
              encrypted_value: encryptSecret('${{ steps.generate.outputs.password }}'),
              key_id: key_id,
            });
            
      - name: Restart services
        run: |
          # Trigger service restart to use new credentials
          kubectl rollout restart deployment/api -n ${{ matrix.environment }}
          kubectl rollout restart deployment/worker -n ${{ matrix.environment }}
```
{% endraw %}

## 5. 고급 캐싱 전략

### 지능형 캐시 관리

`.github/workflows/smart-cache.yml`:

{% raw %}
```yaml
name: Smart Caching

on: [push, pull_request]

env:
  CACHE_VERSION: v1

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Calculate cache keys
        id: cache-keys
        run: |
          # Base cache key
          BASE_KEY="${{ runner.os }}-${{ env.CACHE_VERSION }}"
          
          # Dependencies cache key
          DEPS_HASH=$(cat package-lock.json yarn.lock 2>/dev/null | sha256sum | cut -d' ' -f1)
          echo "deps-key=$BASE_KEY-deps-$DEPS_HASH" >> $GITHUB_OUTPUT
          
          # Build cache key
          SRC_HASH=$(find src -type f -name '*.js' -o -name '*.ts' | xargs cat | sha256sum | cut -d' ' -f1)
          echo "build-key=$BASE_KEY-build-$SRC_HASH" >> $GITHUB_OUTPUT
          
          # Test cache key
          TEST_HASH=$(find tests -type f | xargs cat | sha256sum | cut -d' ' -f1)
          echo "test-key=$BASE_KEY-test-$TEST_HASH" >> $GITHUB_OUTPUT
          
      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: |
            node_modules
            ~/.npm
            ~/.yarn/cache
          key: ${{ steps.cache-keys.outputs.deps-key }}
          restore-keys: |
            ${{ runner.os }}-${{ env.CACHE_VERSION }}-deps-
            
      - name: Cache build artifacts
        uses: actions/cache@v3
        with:
          path: |
            dist
            .next/cache
            build
          key: ${{ steps.cache-keys.outputs.build-key }}
          restore-keys: |
            ${{ runner.os }}-${{ env.CACHE_VERSION }}-build-
            
      - name: Conditional install
        run: |
          if [ ! -d "node_modules" ]; then
            echo "Cache miss - installing dependencies"
            npm ci
          else
            echo "Using cached dependencies"
          fi
          
      - name: Conditional build
        run: |
          if [ ! -d "dist" ]; then
            echo "Cache miss - building project"
            npm run build
          else
            echo "Using cached build"
            # Verify build integrity
            npm run build:verify
          fi
          
      - name: Cleanup old caches
        if: github.ref == 'refs/heads/main'
        uses: actions/github-script@v6
        with:
          script: |
            const caches = await github.rest.actions.getActionsCacheList({
              owner: context.repo.owner,
              repo: context.repo.repo,
            });
            
            const oneWeekAgo = new Date();
            oneWeekAgo.setDate(oneWeekAgo.getDate() - 7);
            
            for (const cache of caches.data.actions_caches) {
              if (new Date(cache.last_accessed_at) < oneWeekAgo) {
                await github.rest.actions.deleteActionsCacheById({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  cache_id: cache.id,
                });
                console.log(`Deleted old cache: ${cache.key}`);
              }
            }
```
{% endraw %}

## 6. 모니터링과 관찰성

### 워크플로우 메트릭 수집

`.github/workflows/collect-metrics.yml`:

{% raw %}
```yaml
name: Workflow Metrics Collector

on:
  workflow_run:
    workflows: ["*"]
    types: [completed]

jobs:
  collect-metrics:
    runs-on: ubuntu-latest
    steps:
      - name: Collect workflow metrics
        uses: actions/github-script@v6
        with:
          script: |
            const workflow_run = context.payload.workflow_run;
            
            // Calculate duration
            const started = new Date(workflow_run.created_at);
            const completed = new Date(workflow_run.updated_at);
            const duration = (completed - started) / 1000; // seconds
            
            // Get job details
            const jobs = await github.rest.actions.listJobsForWorkflowRun({
              owner: context.repo.owner,
              repo: context.repo.repo,
              run_id: workflow_run.id,
            });
            
            // Calculate metrics
            const metrics = {
              workflow_name: workflow_run.name,
              workflow_id: workflow_run.workflow_id,
              run_id: workflow_run.id,
              status: workflow_run.conclusion,
              duration_seconds: duration,
              trigger: workflow_run.event,
              branch: workflow_run.head_branch,
              commit: workflow_run.head_sha,
              jobs: jobs.data.jobs.map(job => ({
                name: job.name,
                status: job.conclusion,
                duration: (new Date(job.completed_at) - new Date(job.started_at)) / 1000,
                steps: job.steps.length,
              })),
              timestamp: new Date().toISOString(),
            };
            
            // Send to monitoring service
            await fetch('https://metrics.example.com/workflows', {
              method: 'POST',
              headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${{ secrets.METRICS_API_KEY }}`,
              },
              body: JSON.stringify(metrics),
            });
            
            // Log to GitHub
            console.log('Workflow Metrics:', JSON.stringify(metrics, null, 2));
            
            // Create summary
            await core.summary
              .addHeading('Workflow Metrics')
              .addTable([
                [{data: 'Metric', header: true}, {data: 'Value', header: true}],
                ['Workflow', workflow_run.name],
                ['Status', workflow_run.conclusion],
                ['Duration', `${duration}s`],
                ['Jobs', jobs.data.jobs.length.toString()],
              ])
              .write();
```
{% endraw %}

## 7. 자체 호스팅 러너 관리

### 자동 스케일링 러너

`runner-autoscale.yml`:

{% raw %}
```yaml
name: Runner Autoscaling

on:
  workflow_dispatch:
  schedule:
    - cron: '*/5 * * * *'  # Every 5 minutes

jobs:
  scale-runners:
    runs-on: ubuntu-latest
    steps:
      - name: Check queue depth
        id: queue
        uses: actions/github-script@v6
        with:
          script: |
            // Get pending runs
            const runs = await github.rest.actions.listWorkflowRunsForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
              status: 'queued',
            });
            
            const pendingJobs = runs.data.workflow_runs.length;
            console.log(`Pending jobs: ${pendingJobs}`);
            
            // Calculate required runners
            const runnersPerJob = 1;
            const minRunners = 2;
            const maxRunners = 10;
            
            let desiredRunners = Math.ceil(pendingJobs * runnersPerJob);
            desiredRunners = Math.max(minRunners, Math.min(maxRunners, desiredRunners));
            
            core.setOutput('desired-runners', desiredRunners);
            core.setOutput('pending-jobs', pendingJobs);
            
      - name: Scale runner deployment
        env:
          KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
        run: |
          echo "$KUBE_CONFIG" | base64 -d > /tmp/kubeconfig
          export KUBECONFIG=/tmp/kubeconfig
          
          # Update runner deployment
          kubectl scale deployment github-runner \
            --replicas=${{ steps.queue.outputs.desired-runners }} \
            -n github-runners
            
          # Wait for scale
          kubectl rollout status deployment/github-runner -n github-runners
          
      - name: Update metrics
        run: |
          cat << EOF > metrics.json
          {
            "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
            "pending_jobs": ${{ steps.queue.outputs.pending-jobs }},
            "desired_runners": ${{ steps.queue.outputs.desired-runners }},
            "action": "scale"
          }
          EOF
          
          curl -X POST https://metrics.example.com/runners \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${{ secrets.METRICS_API_KEY }}" \
            -d @metrics.json
```
{% endraw %}

### 러너 이미지 빌드

`Dockerfile.runner`:

```dockerfile
FROM ubuntu:22.04

# Install dependencies
RUN apt-get update && apt-get install -y \
    curl \
    git \
    jq \
    build-essential \
    libssl-dev \
    libffi-dev \
    python3 \
    python3-venv \
    python3-dev \
    python3-pip \
    docker.io \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js
RUN curl -fsSL https://deb.nodesource.com/setup_18.x | bash - \
    && apt-get install -y nodejs

# Create runner user
RUN useradd -m -s /bin/bash runner \
    && usermod -aG docker runner

# Install GitHub Runner
WORKDIR /home/runner
ARG RUNNER_VERSION=2.311.0
RUN curl -O -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
    && tar xzf actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
    && rm actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
    && chown -R runner:runner /home/runner

# Install additional tools
RUN npm install -g yarn pnpm
RUN pip3 install --upgrade pip setuptools wheel
RUN pip3 install aws-cli azure-cli

# Custom entrypoint
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

USER runner
ENTRYPOINT ["/entrypoint.sh"]
```

## 8. 보안 강화 워크플로우

### SAST/DAST 통합

`.github/workflows/security-scan.yml`:

{% raw %}
```yaml
name: Security Scanning

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 0 * * 1'  # Weekly

permissions:
  contents: read
  security-events: write

jobs:
  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run CodeQL Analysis
        uses: github/codeql-action/analyze@v2
        with:
          languages: javascript, python
          
      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          
      - name: Semgrep Scan
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten
            
      - name: License Scan
        uses: fossas/fossa-action@main
        with:
          api-key: ${{ secrets.FOSSA_API_KEY }}

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: docker build -t scan-target:${{ github.sha }} .
        
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: scan-target:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH,MEDIUM'
          
      - name: Snyk Container Scan
        uses: snyk/actions/docker@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          image: scan-target:${{ github.sha }}
          args: --severity-threshold=high
          
      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  dynamic-analysis:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        run: |
          # Deploy application to staging environment
          ./deploy.sh staging
          
      - name: OWASP ZAP Scan
        uses: zaproxy/action-full-scan@v0.4.0
        with:
          target: 'https://staging.example.com'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'
          
      - name: Nuclei Scan
        uses: projectdiscovery/nuclei-action@main
        with:
          target: 'https://staging.example.com'
          flags: '-severity critical,high,medium'
          
      - name: SSL/TLS Check
        run: |
          pip install sslyze
          python -m sslyze --regular staging.example.com:443 \
            --json_out sslyze-results.json
```
{% endraw %}

## 9. 복잡한 배포 전략

### Blue-Green 배포

`.github/workflows/blue-green-deploy.yml`:

{% raw %}
```yaml
name: Blue-Green Deployment

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Determine target environment
        id: target
        run: |
          # Get current active environment
          ACTIVE=$(curl -s https://api.example.com/health | jq -r '.environment')
          
          if [[ "$ACTIVE" == "blue" ]]; then
            echo "target=green" >> $GITHUB_OUTPUT
            echo "current=blue" >> $GITHUB_OUTPUT
          else
            echo "target=blue" >> $GITHUB_OUTPUT
            echo "current=green" >> $GITHUB_OUTPUT
          fi
          
      - name: Deploy to ${{ steps.target.outputs.target }}
        run: |
          # Deploy new version to inactive environment
          ./deploy.sh ${{ steps.target.outputs.target }} ${{ github.sha }}
          
          # Wait for health check
          for i in {1..30}; do
            if curl -f https://${{ steps.target.outputs.target }}.example.com/health; then
              echo "Health check passed"
              break
            fi
            sleep 10
          done
          
      - name: Run smoke tests
        run: |
          npm run test:smoke -- --url=https://${{ steps.target.outputs.target }}.example.com
          
      - name: Switch traffic
        run: |
          # Update load balancer to point to new environment
          aws elbv2 modify-listener \
            --listener-arn ${{ secrets.ALB_LISTENER_ARN }} \
            --default-actions Type=forward,TargetGroupArn=${{ secrets[format('TARGET_GROUP_{0}', steps.target.outputs.target)] }}
            
      - name: Monitor metrics
        run: |
          # Monitor error rates for 5 minutes
          END=$(($(date +%s) + 300))
          while [ $(date +%s) -lt $END ]; do
            ERROR_RATE=$(curl -s https://metrics.example.com/error-rate | jq -r '.rate')
            if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
              echo "Error rate too high: $ERROR_RATE"
              exit 1
            fi
            sleep 30
          done
          
      - name: Cleanup old environment
        if: success()
        run: |
          # Scale down old environment after 30 minutes
          echo "Scheduling cleanup of ${{ steps.target.outputs.current }} environment"
          at now + 30 minutes <<< "./cleanup.sh ${{ steps.target.outputs.current }}"
```
{% endraw %}

## 10. 워크플로우 최적화

### 병렬화와 최적화

`.github/workflows/optimized-pipeline.yml`:

{% raw %}
```yaml
name: Optimized Pipeline

on: [push, pull_request]

jobs:
  # Quick checks that can fail fast
  quick-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Commit lint
        run: |
          npm install -g @commitlint/cli @commitlint/config-conventional
          commitlint --from HEAD~1 --to HEAD
          
      - name: File size check
        run: |
          # Check for large files
          find . -type f -size +1M | grep -v -E '\.(jpg|png|gif|pdf)$' | while read f; do
            echo "Error: $f is larger than 1MB"
            exit 1
          done
          
      - name: Security pre-check
        run: |
          # Quick security checks
          grep -r "password.*=.*['\"]" --include="*.js" --include="*.py" . || true

  # Parallel static analysis
  analyze:
    needs: quick-checks
    runs-on: ubuntu-latest
    strategy:
      matrix:
        tool: [eslint, prettier, typescript, security]
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci --only=dev
        
      - name: Run ${{ matrix.tool }}
        run: |
          case "${{ matrix.tool }}" in
            eslint)
              npm run lint
              ;;
            prettier)
              npm run format:check
              ;;
            typescript)
              npm run type-check
              ;;
            security)
              npm audit --production
              ;;
          esac

  # Optimized test execution
  test:
    needs: quick-checks
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run tests (shard ${{ matrix.shard }}/4)
        run: |
          npm run test -- \
            --shard=${{ matrix.shard }}/4 \
            --coverage \
            --coverageReporters=json
            
      - name: Upload coverage
        uses: actions/upload-artifact@v3
        with:
          name: coverage-${{ matrix.shard }}
          path: coverage/coverage-final.json

  # Merge coverage reports
  coverage:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Download all coverage reports
        uses: actions/download-artifact@v3
        with:
          pattern: coverage-*
          path: coverage
          
      - name: Merge coverage reports
        run: |
          npx nyc merge coverage coverage/merged.json
          npx nyc report -t coverage --report-dir coverage/final
          
      - name: Upload to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/final/lcov.info
```
{% endraw %}

## 2025년 GitHub Actions 최신 업데이트

### 새로운 기능들

| 기능 | 설명 | 사용 예시 |
|------|------|----------|
| **GPU 러너** | GPU 지원 러너 | ML 모델 학습, 그래픽 렌더링 |
| **대규모 러너** | 최대 64 코어, 256GB RAM | 대규모 빌드, 테스트 |
| **ARM64 지원** | ARM 아키텍처 네이티브 지원 | M1/M2 Mac, ARM 서버 빌드 |
| **개선된 캐싱** | 10GB 캐시 용량 | 대용량 의존성 캐싱 |
| **워크플로우 인사이트** | 상세한 성능 분석 | 병목 현상 식별 |

### 성능 최적화 팁

```mermaid
graph TD
    A[워크플로우 최적화] --> B[병렬화]
    A --> C[캐싱]
    A --> D[조건부 실행]
    
    B --> E[매트릭스 전략]
    B --> F[동시 작업]
    
    C --> G[의존성 캐시]
    C --> H[빌드 캐시]
    
    D --> I[경로 필터]
    D --> J[조건문 활용]
```

## 마무리

GitHub Actions의 고급 기능들을 활용하면 매우 복잡하고 강력한 자동화 워크플로우를 구축할 수 있습니다.

### 핵심 포인트 정리

| 영역 | 주요 내용 | 효과 |
|------|-----------|------|
| **재사용성** | Composite/Custom Actions | 코드 중복 제거, 유지보수성 향상 |
| **유연성** | 동적 매트릭스, 조건부 실행 | 상황별 최적화된 실행 |
| **안정성** | 보안 스캔, 모니터링 | 프로덕션 품질 보장 |
| **성능** | 캐싱, 병렬화 | 실행 시간 50-80% 단축 |
| **배포** | Blue-Green, Canary | 무중단 서비스 제공 |

이러한 고급 패턴들을 적절히 조합하여 프로젝트에 맞는 최적의 CI/CD 파이프라인을 구축하세요.

다음 편에서는 Webhooks와 API를 활용한 외부 서비스 연동에 대해 알아보겠습니다.

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
12. **Actions 고급 활용** (현재 글)
13. Webhooks와 API (예정)

---

*질문이나 피드백이 있으시다면 댓글로 남겨주세요!*