# GitHub Actions Workflow Guide - Step by Step

## Understanding GitHub Actions Architecture

### Core Components

```
Repository
└── .github/
    └── workflows/
        ├── ci.yml          # Workflow file 1
        ├── deploy.yml      # Workflow file 2
        └── release.yml     # Workflow file 3
```

### Workflow Hierarchy

```
Workflow (YAML file)
├── Trigger (on: push, pull_request, etc.)
├── Environment Variables (env:)
└── Jobs (can run in parallel)
    └── Job 1
        ├── Runs on a runner (OS)
        ├── Environment (dev, staging, prod)
        └── Steps (sequential)
            ├── Step 1: Checkout code
            ├── Step 2: Setup Java
            ├── Step 3: Build
            └── Step 4: Test
```

---

## Basic Workflow Structure

### Minimal Workflow

```yaml
name: Workflow Name           # Shows in GitHub UI

on: [push]                    # When to trigger

jobs:                         # List of jobs
  job-name:                   # Job identifier
    runs-on: ubuntu-latest    # Runner OS
    steps:                    # List of steps
      - name: Step name       # Step description
        run: echo "Hello"     # Command to run
```

---

## Step-by-Step: Writing a Workflow

### Step 1: Choose Triggers

#### Trigger on Push
```yaml
name: CI

on:
  push:
    branches:
      - main           # Only main branch
      - develop        # And develop branch
    paths:
      - 'src/**'       # Only when src/ files change
      - 'pom.xml'      # Or when pom.xml changes
```

#### Trigger on Pull Request
```yaml
on:
  pull_request:
    branches: [main]
    types:
      - opened         # PR opened
      - synchronize    # PR updated with new commits
      - reopened       # PR reopened
```

#### Trigger on Schedule (Cron)
```yaml
on:
  schedule:
    - cron: '0 2 * * *'    # Every day at 2 AM UTC
    - cron: '0 0 * * 0'    # Every Sunday at midnight
    # Format: minute hour day month weekday
```

#### Manual Trigger
```yaml
on:
  workflow_dispatch:           # Adds "Run workflow" button
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
```

#### Multiple Triggers
```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'
  workflow_dispatch:
```

### Step 2: Define Jobs

#### Single Job
```yaml
jobs:
  build:                       # Job name
    runs-on: ubuntu-latest     # Runner
    steps:
      - run: echo "Building"
```

#### Multiple Jobs (Parallel)
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: mvn package
  
  test:                        # Runs in parallel with build
    runs-on: ubuntu-latest
    steps:
      - run: mvn test
```

#### Sequential Jobs
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: mvn package
  
  test:
    needs: build               # Wait for build to complete
    runs-on: ubuntu-latest
    steps:
      - run: mvn test
  
  deploy:
    needs: [build, test]       # Wait for both
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### Step 3: Choose Runner

```yaml
runs-on: ubuntu-latest         # Linux (most common)
runs-on: windows-latest        # Windows
runs-on: macos-latest          # macOS

# Specific versions
runs-on: ubuntu-22.04
runs-on: windows-2022
runs-on: macos-12

# Matrix for multiple OS
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
runs-on: ${{ matrix.os }}
```

### Step 4: Add Steps

#### Checkout Repository
```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4           # Official GitHub action
    with:
      fetch-depth: 0                    # Full history (for SonarQube)
      submodules: true                  # Include submodules
```

#### Run Shell Commands
```yaml
  - name: Run multiple commands
    run: |
      echo "First command"
      mkdir build
      cd build
      echo "In build directory"
```

#### Use Actions from Marketplace
```yaml
  - name: Setup Java
    uses: actions/setup-java@v4
    with:
      java-version: '17'
      distribution: 'temurin'
      cache: 'maven'                    # Cache Maven dependencies
```

#### Set Environment Variables
```yaml
  - name: Set variables
    run: |
      echo "BUILD_DATE=$(date +'%Y-%m-%d')" >> $GITHUB_ENV
      echo "VERSION=1.2.3" >> $GITHUB_ENV
  
  - name: Use variables
    run: |
      echo "Building version $VERSION on $BUILD_DATE"
```

---

## Complete Examples

### Example 1: Java Maven Build

```yaml
name: Java CI with Maven

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  JAVA_VERSION: '17'
  MAVEN_OPTS: '-Xmx1024m'

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    # Step 1: Get the code
    - name: Checkout repository
      uses: actions/checkout@v4
    
    # Step 2: Setup Java
    - name: Set up JDK ${{ env.JAVA_VERSION }}
      uses: actions/setup-java@v4
      with:
        java-version: ${{ env.JAVA_VERSION }}
        distribution: 'temurin'
        cache: 'maven'
    
    # Step 3: Build
    - name: Build with Maven
      run: mvn clean compile
    
    # Step 4: Run tests
    - name: Run unit tests
      run: mvn test
    
    # Step 5: Package
    - name: Package application
      run: mvn package -DskipTests
    
    # Step 6: Upload artifact
    - name: Upload JAR file
      uses: actions/upload-artifact@v3
      with:
        name: application-jar
        path: target/*.jar
        retention-days: 30
```

### Example 2: Node.js Application

```yaml
name: Node.js CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Setup Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linter
      run: npm run lint
    
    - name: Run tests
      run: npm test
    
    - name: Build
      run: npm run build
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage/lcov.info
```

### Example 3: Docker Build and Push

```yaml
name: Docker Build and Push

on:
  push:
    branches: [main]
    tags:
      - 'v*'

env:
  REGISTRY: docker.io
  IMAGE_NAME: myusername/myapp

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha
    
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

### Example 4: Deploy to AWS

```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Build and push image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: myapp
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
    
    - name: Deploy to ECS
      run: |
        aws ecs update-service \
          --cluster production-cluster \
          --service myapp-service \
          --force-new-deployment
```

---

## Advanced Workflow Features

### 1. Using Secrets

```yaml
steps:
  - name: Use secret
    env:
      API_KEY: ${{ secrets.API_KEY }}        # Repository secret
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    run: |
      echo "API_KEY is set"
      # The secret value is masked in logs
      ./deploy.sh
```

**Setting secrets**: GitHub → Repository → Settings → Secrets → New secret

### 2. Conditional Execution

```yaml
steps:
  - name: Deploy to production
    if: github.ref == 'refs/heads/main'      # Only on main branch
    run: ./deploy-prod.sh
  
  - name: Deploy to staging
    if: github.ref == 'refs/heads/develop'   # Only on develop
    run: ./deploy-staging.sh
  
  - name: Run on PR
    if: github.event_name == 'pull_request'
    run: echo "This is a PR"
  
  - name: Run on success
    if: success()                            # Previous steps succeeded
    run: echo "All good!"
  
  - name: Run on failure
    if: failure()                            # Previous step failed
    run: echo "Something failed"
```

### 3. Matrix Strategy

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        java-version: ['11', '17', '21']
        include:
          - os: ubuntu-latest
            java-version: '17'
            experimental: false
        exclude:
          - os: macos-latest
            java-version: '11'
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
      - run: mvn test
```

This creates jobs for:
- ubuntu-latest × [11, 17, 21] = 3 jobs
- windows-latest × [11, 17, 21] = 3 jobs  
- macos-latest × [17, 21] = 2 jobs (11 excluded)
- **Total: 8 jobs running in parallel**

### 4. Reusable Workflows

#### Main workflow (.github/workflows/deploy.yml)
```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    uses: ./.github/workflows/deploy-reusable.yml
    with:
      environment: staging
    secrets: inherit
  
  deploy-production:
    needs: deploy-staging
    uses: ./.github/workflows/deploy-reusable.yml
    with:
      environment: production
    secrets: inherit
```

#### Reusable workflow (.github/workflows/deploy-reusable.yml)
```yaml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      AWS_ACCESS_KEY_ID:
        required: true
      AWS_SECRET_ACCESS_KEY:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to ${{ inputs.environment }}
        run: ./deploy.sh ${{ inputs.environment }}
```

### 5. Outputs Between Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get-version.outputs.version }}
      artifact-path: ${{ steps.build.outputs.path }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Get version
        id: get-version
        run: |
          VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
          echo "version=$VERSION" >> $GITHUB_OUTPUT
      
      - name: Build
        id: build
        run: |
          mvn package
          echo "path=target/myapp.jar" >> $GITHUB_OUTPUT
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - name: Use outputs from build
        run: |
          echo "Version: ${{ needs.build.outputs.version }}"
          echo "Artifact: ${{ needs.build.outputs.artifact-path }}"
```

### 6. Composite Actions

Create custom reusable action: `.github/actions/setup-java-maven/action.yml`

```yaml
name: 'Setup Java and Maven'
description: 'Setup Java with Maven caching'

inputs:
  java-version:
    description: 'Java version'
    required: false
    default: '17'

runs:
  using: 'composite'
  steps:
    - name: Setup JDK
      uses: actions/setup-java@v4
      with:
        java-version: ${{ inputs.java-version }}
        distribution: 'temurin'
        cache: 'maven'
    
    - name: Cache Maven packages
      uses: actions/cache@v3
      with:
        path: ~/.m2/repository
        key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
        restore-keys: |
          ${{ runner.os }}-maven-
    
    - name: Install Maven
      shell: bash
      run: |
        mvn --version
```

Use it in workflow:
```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-java-maven
    with:
      java-version: '17'
```

---

## GitHub Actions Context Variables

### Available Context Objects

```yaml
steps:
  - name: Print context info
    run: |
      echo "Repository: ${{ github.repository }}"
      echo "Ref: ${{ github.ref }}"                    # refs/heads/main
      echo "SHA: ${{ github.sha }}"                    # Commit SHA
      echo "Actor: ${{ github.actor }}"                # User who triggered
      echo "Event: ${{ github.event_name }}"           # push, pull_request, etc
      echo "Run ID: ${{ github.run_id }}"
      echo "Run Number: ${{ github.run_number }}"
      echo "Workflow: ${{ github.workflow }}"
      
      echo "Runner OS: ${{ runner.os }}"               # Linux, Windows, macOS
      echo "Runner Temp: ${{ runner.temp }}"
      
      echo "Job Status: ${{ job.status }}"             # success, failure, cancelled
```

### Accessing Environment Variables

```yaml
env:
  GLOBAL_VAR: 'global'

jobs:
  example:
    runs-on: ubuntu-latest
    env:
      JOB_VAR: 'job-level'
    
    steps:
      - name: Step with env
        env:
          STEP_VAR: 'step-level'
        run: |
          echo $GLOBAL_VAR
          echo $JOB_VAR
          echo $STEP_VAR
          echo ${{ env.GLOBAL_VAR }}        # Alternative syntax
```

---

## Common Workflow Patterns

### Pattern 1: Build Once, Deploy Many

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mvn package
      - uses: actions/upload-artifact@v3
        with:
          name: app-jar
          path: target/*.jar
  
  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: app-jar
      - run: ./deploy.sh dev
  
  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: app-jar
      - run: ./deploy.sh staging
  
  deploy-prod:
    needs: [build, deploy-staging]
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: app-jar
      - run: ./deploy.sh prod
```

### Pattern 2: PR Checks

```yaml
name: PR Checks

on:
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint
  
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
  
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
  
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit
  
  comment-pr:
    needs: [lint, test, build, security]
    runs-on: ubuntu-latest
    steps:
      - name: Comment on PR
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: '✅ All checks passed! Ready to merge.'
            })
```

### Pattern 3: Nightly Build

```yaml
name: Nightly Build

on:
  schedule:
    - cron: '0 2 * * *'    # 2 AM daily
  workflow_dispatch:       # Manual trigger

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mvn clean install
      
      - name: Run integration tests
        run: mvn verify -Pintegration-tests
      
      - name: Generate reports
        run: mvn site
      
      - name: Upload reports
        uses: actions/upload-artifact@v3
        with:
          name: test-reports
          path: target/site/
      
      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "❌ Nightly build failed!",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Nightly Build Failed*\nCheck the logs: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                  }
                }
              ]
            }
```

---

## Debugging Workflows

### 1. Enable Debug Logging

Repository → Settings → Secrets → Add:
- `ACTIONS_RUNNER_DEBUG` = `true`
- `ACTIONS_STEP_DEBUG` = `true`

### 2. Add Debug Steps

```yaml
- name: Debug information
  run: |
    echo "GitHub Context:"
    echo "${{ toJSON(github) }}"
    
    echo "Runner Context:"
    echo "${{ toJSON(runner) }}"
    
    echo "Environment Variables:"
    env
    
    echo "Files in workspace:"
    ls -la
```

### 3. Use tmate for SSH Access

```yaml
- name: Setup tmate session
  uses: mxschmitt/action-tmate@v3
  if: failure()    # Only on failure
```

---

## Best Practices

### 1. Pin Action Versions
```yaml
# Good
uses: actions/checkout@v4
uses: actions/setup-java@v4

# Better (with commit SHA for security)
uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1
```

### 2. Cache Dependencies
```yaml
- uses: actions/cache@v3
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
```

### 3. Use Environment Protection
Repository → Settings → Environments → New environment

```yaml
jobs:
  deploy:
    environment: production    # Requires manual approval
```

### 4. Limit Workflow Concurrency
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true     # Cancel old runs
```

### 5. Set Timeouts
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30        # Cancel after 30 min
```

---

## Troubleshooting Common Issues

### Issue 1: Permission Denied

```yaml
# Add execute permission
- name: Make script executable
  run: chmod +x ./script.sh
```

### Issue 2: Path Not Found

```yaml
# Always checkout first
- uses: actions/checkout@v4
# Then run commands
- run: ./my-script.sh
```

### Issue 3: Secrets Not Working

```yaml
# Verify secret name matches exactly
env:
  TOKEN: ${{ secrets.GITHUB_TOKEN }}    # Correct
  # TOKEN: ${{ secrets.github_token }}  # Wrong (case-sensitive)
```

### Issue 4: Workflow Not Triggering

```yaml
# Check YAML syntax
# Verify branch names match
on:
  push:
    branches:
      - main       # Must match exactly
      # - master  # Wrong if branch is 'main'
```

---

## Quick Reference

### Workflow Template

```yaml
name: Workflow Name

on:
  push:
    branches: [main]

env:
  VARIABLE: value

jobs:
  job-name:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Step name
        run: command here
```

### Useful GitHub Actions

- `actions/checkout@v4` - Checkout code
- `actions/setup-java@v4` - Setup Java
- `actions/setup-node@v3` - Setup Node.js
- `actions/setup-python@v4` - Setup Python
- `actions/cache@v3` - Cache dependencies
- `actions/upload-artifact@v3` - Upload files
- `actions/download-artifact@v3` - Download files
- `docker/build-push-action@v4` - Build Docker
- `aws-actions/configure-aws-credentials@v2` - AWS auth