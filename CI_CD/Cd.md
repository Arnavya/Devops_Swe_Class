# Continuous Deployment & Delivery (CD) - Deep Dive

## What is CD?

### Continuous Delivery (CD)
**Definition**: Code is always in a deployable state. Deployments require manual approval.

### Continuous Deployment (CD)
**Definition**: Every change that passes automated tests is automatically deployed to production.

```
Continuous Delivery:    Build → Test → [Manual Approval] → Deploy
Continuous Deployment:  Build → Test → Deploy (fully automated)
```

---

## Why Continuous Deployment?

### Traditional Deployments ❌
- Deploy once a month/quarter
- Large, risky releases
- Hard to track which change caused issues
- Long rollback times

### Continuous Deployment ✅
- Deploy multiple times per day
- Small, low-risk changes
- Easy to identify problematic changes
- Fast rollback (minutes, not hours)

**Real-world example**: Amazon deploys every 11.7 seconds on average.

---

## Deployment Environments

### Standard Environment Flow

```
Developer's Machine
    ↓
Development/Integration Environment (auto-deploy on commit)
    ↓
Testing/QA Environment (auto-deploy after tests pass)
    ↓
Staging/Pre-production (mirrors production, manual approval)
    ↓
Production (manual or auto-deploy)
```

### Environment Characteristics

| Environment | Purpose | Who Uses | Deploy Frequency |
|-------------|---------|----------|------------------|
| **Dev** | Active development | Developers | Every commit |
| **QA/Test** | Quality assurance | QA team | Daily |
| **Staging** | Pre-production testing | QA + Stakeholders | Before prod |
| **Production** | Real users | End users | Multiple/day or week |

---

## Deployment Strategies

### 1. **Blue-Green Deployment**

**How it works**:
- Two identical environments (Blue = current, Green = new)
- Deploy to Green while Blue serves traffic
- Switch traffic from Blue to Green instantly
- Keep Blue as quick rollback option

```yaml
# Simplified blue-green deployment
- name: Deploy to Green Environment
  run: |
    kubectl apply -f green-deployment.yaml
    kubectl wait --for=condition=available deployment/app-green
    
- name: Run Smoke Tests on Green
  run: ./smoke-tests.sh https://green.example.com
    
- name: Switch Traffic to Green
  run: kubectl patch service app-service -p '{"spec":{"selector":{"version":"green"}}}'
    
- name: Keep Blue for 1 Hour (Quick Rollback)
  run: sleep 3600 && kubectl delete deployment app-blue
```

**Pros**:
- ✅ Zero downtime
- ✅ Instant rollback (switch back to Blue)
- ✅ Test in production-like environment

**Cons**:
- ❌ Need 2x infrastructure
- ❌ Database migrations are complex

**Use when**: You need zero-downtime deployments and can afford duplicate infrastructure.

### 2. **Canary Deployment**

**How it works**:
- Deploy to small % of servers/users first (5-10%)
- Monitor metrics (errors, latency, CPU)
- Gradually increase traffic to new version
- Rollback if issues detected

```yaml
# Canary deployment with traffic shifting
- name: Deploy Canary (10% traffic)
  run: |
    kubectl apply -f canary-deployment.yaml
    kubectl set image deployment/app-canary app=myapp:v2
    
- name: Route 10% Traffic to Canary
  run: |
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: app-route
    spec:
      http:
      - match:
        - headers:
            canary:
              exact: "true"
        route:
        - destination:
            host: app-canary
          weight: 10
        - destination:
            host: app-stable
          weight: 90
    EOF

- name: Monitor Canary for 30 Minutes
  run: |
    python monitor-canary.py --duration 30m \
      --error-threshold 5% \
      --latency-threshold 500ms

- name: Promote Canary if Healthy
  if: success()
  run: |
    kubectl set image deployment/app-stable app=myapp:v2
    kubectl delete deployment app-canary
```

**Monitoring script example**:
```python
# monitor-canary.py
import time
import requests

def check_health():
    metrics = requests.get('http://prometheus/api/v1/query',
        params={'query': 'error_rate{version="canary"}'})
    error_rate = float(metrics.json()['data']['result'][0]['value'][1])
    
    if error_rate > 0.05:  # 5% threshold
        print("❌ Canary failed: High error rate")
        return False
    return True

# Monitor for 30 minutes
for i in range(30):
    if not check_health():
        exit(1)
    time.sleep(60)
print("✅ Canary healthy, promoting to production")
```

**Pros**:
- ✅ Low risk (only affects small % of users)
- ✅ Real production testing
- ✅ Gradual rollout

**Cons**:
- ❌ Complex setup (requires load balancer/service mesh)
- ❌ Requires good monitoring

**Use when**: You want to minimize blast radius of issues.

### 3. **Rolling Deployment**

**How it works**:
- Update servers/pods one at a time
- Old version and new version run simultaneously
- No downtime

```yaml
# Kubernetes rolling update
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Max 1 pod down at a time
      maxSurge: 2        # Max 2 extra pods during update
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

**What happens**:
1. Create 2 new pods with v2
2. Wait for them to be ready (readiness probe)
3. Terminate 1 old pod
4. Create 1 new pod
5. Repeat until all pods are v2

**Pros**:
- ✅ No downtime
- ✅ No extra infrastructure needed
- ✅ Built into Kubernetes

**Cons**:
- ❌ Both versions run simultaneously (compatibility issues)
- ❌ Slower rollback than blue-green

**Use when**: Using Kubernetes and want simple zero-downtime updates.

### 4. **Recreate Deployment**

**How it works**:
- Stop all old version instances
- Start new version instances
- Brief downtime (seconds to minutes)

```yaml
# Kubernetes recreate strategy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    type: Recreate  # Stop all, then start all
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2
```

**Pros**:
- ✅ Simple
- ✅ Never run two versions simultaneously
- ✅ Good for stateful apps with compatibility issues

**Cons**:
- ❌ Downtime during deployment

**Use when**: Downtime is acceptable, or versions can't run together.

---

## Complete CD Pipeline Example

### GitHub Actions Deployment Workflow

```yaml
name: CD Pipeline - Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:  # Manual trigger

env:
  DOCKER_IMAGE: mycompany/myapp
  KUBE_NAMESPACE: production

jobs:
  deploy-to-staging:
    runs-on: ubuntu-latest
    environment: staging  # GitHub environment with secrets
    
    steps:
    - name: Checkout Code
      uses: actions/checkout@v4
    
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
    
    - name: Login to ECR
      run: |
        aws ecr get-login-password --region us-east-1 | \
        docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
    
    - name: Build and Push Docker Image
      run: |
        docker build -t ${{ env.DOCKER_IMAGE }}:${{ github.sha }} .
        docker tag ${{ env.DOCKER_IMAGE }}:${{ github.sha }} ${{ env.DOCKER_IMAGE }}:staging
        docker push ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
        docker push ${{ env.DOCKER_IMAGE }}:staging
    
    - name: Deploy to Staging EKS
      run: |
        aws eks update-kubeconfig --region us-east-1 --name staging-cluster
        kubectl set image deployment/myapp \
          app=${{ env.DOCKER_IMAGE }}:${{ github.sha }} \
          -n staging
        kubectl rollout status deployment/myapp -n staging
    
    - name: Run Smoke Tests
      run: |
        ./scripts/smoke-tests.sh https://staging.example.com
    
    - name: Send Slack Notification
      uses: slackapi/slack-github-action@v1.24.0
      with:
        webhook-url: ${{ secrets.SLACK_WEBHOOK }}
        payload: |
          {
            "text": "✅ Deployed to Staging: ${{ github.sha }}",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Deployment to Staging Successful*\nCommit: `${{ github.sha }}`\nAuthor: ${{ github.actor }}"
                }
              }
            ]
          }
  
  deploy-to-production:
    needs: deploy-to-staging
    runs-on: ubuntu-latest
    environment: 
      name: production
      url: https://example.com
    
    steps:
    - name: Checkout Code
      uses: actions/checkout@v4
    
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
    
    - name: Login to ECR
      run: |
        aws ecr get-login-password --region us-east-1 | \
        docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
    
    - name: Tag for Production
      run: |
        docker pull ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
        docker tag ${{ env.DOCKER_IMAGE }}:${{ github.sha }} ${{ env.DOCKER_IMAGE }}:production
        docker tag ${{ env.DOCKER_IMAGE }}:${{ github.sha }} ${{ env.DOCKER_IMAGE }}:v${{ github.run_number }}
        docker push ${{ env.DOCKER_IMAGE }}:production
        docker push ${{ env.DOCKER_IMAGE }}:v${{ github.run_number }}
    
    - name: Deploy to Production EKS (Canary)
      run: |
        aws eks update-kubeconfig --region us-east-1 --name prod-cluster
        
        # Deploy canary with 10% traffic
        kubectl apply -f k8s/canary-deployment.yaml
        kubectl set image deployment/myapp-canary \
          app=${{ env.DOCKER_IMAGE }}:${{ github.sha }} \
          -n ${{ env.KUBE_NAMESPACE }}
        
        # Wait for canary to be ready
        kubectl wait --for=condition=available \
          deployment/myapp-canary \
          -n ${{ env.KUBE_NAMESPACE }} \
          --timeout=300s
    
    - name: Monitor Canary Deployment
      run: |
        python scripts/monitor-canary.py \
          --duration 10m \
          --error-threshold 2% \
          --latency-p95-threshold 500ms
    
    - name: Promote Canary to Full Production
      if: success()
      run: |
        # Update main deployment
        kubectl set image deployment/myapp \
          app=${{ env.DOCKER_IMAGE }}:${{ github.sha }} \
          -n ${{ env.KUBE_NAMESPACE }}
        
        kubectl rollout status deployment/myapp -n ${{ env.KUBE_NAMESPACE }}
        
        # Remove canary
        kubectl delete deployment myapp-canary -n ${{ env.KUBE_NAMESPACE }}
    
    - name: Rollback on Failure
      if: failure()
      run: |
        echo "❌ Canary failed, rolling back"
        kubectl delete deployment myapp-canary -n ${{ env.KUBE_NAMESPACE }}
        exit 1
    
    - name: Create GitHub Release
      if: success()
      uses: actions/create-release@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: v${{ github.run_number }}
        release_name: Release v${{ github.run_number }}
        body: |
          Automated release from commit ${{ github.sha }}
          
          Changes: https://github.com/${{ github.repository }}/compare/v${{ github.run_number - 1 }}...v${{ github.run_number }}
    
    - name: Send Success Notification
      if: success()
      uses: slackapi/slack-github-action@v1.24.0
      with:
        webhook-url: ${{ secrets.SLACK_WEBHOOK }}
        payload: |
          {
            "text": "🚀 Deployed to Production: v${{ github.run_number }}",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Production Deployment Successful*\n✅ Version: `v${{ github.run_number }}`\n✅ Commit: `${{ github.sha }}`\n✅ Author: ${{ github.actor }}\n✅ URL: https://example.com"
                }
              }
            ]
          }
    
    - name: Send Failure Notification
      if: failure()
      uses: slackapi/slack-github-action@v1.24.0
      with:
        webhook-url: ${{ secrets.SLACK_WEBHOOK }}
        payload: |
          {
            "text": "❌ Production Deployment Failed",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Production Deployment Failed*\n❌ Commit: `${{ github.sha }}`\n❌ Author: ${{ github.actor }}\n@channel Please investigate!"
                }
              }
            ]
          }
```

---

## Database Migrations in CD

### Challenge
- Can't just deploy new code if database schema changes
- Need backward compatibility during rolling deployments

### Solution: Expand-Contract Pattern

#### Phase 1: Expand (Add new column, keep old)
```sql
-- Deployment 1: Add new column
ALTER TABLE users ADD COLUMN email_address VARCHAR(255);

-- Update code to write to BOTH columns
UPDATE users SET email_address = email WHERE email_address IS NULL;
```

```java
// Code writes to both columns
user.setEmail(email);           // Old column
user.setEmailAddress(email);    // New column
```

#### Phase 2: Migrate Data
```sql
-- Background job migrates old data
UPDATE users 
SET email_address = email 
WHERE email_address IS NULL;
```

#### Phase 3: Contract (Remove old column)
```sql
-- Deployment 2: Remove old column (after all pods updated)
ALTER TABLE users DROP COLUMN email;
```

```java
// Code only uses new column
user.setEmailAddress(email);
```

### Migration Tools

#### Flyway (Java)
```yaml
- name: Run Database Migrations
  run: |
    mvn flyway:migrate \
      -Dflyway.url=jdbc:postgresql://db:5432/myapp \
      -Dflyway.user=${{ secrets.DB_USER }} \
      -Dflyway.password=${{ secrets.DB_PASSWORD }}
```

Migration file: `V1__create_users_table.sql`
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(255) NOT NULL
);
```

#### Liquibase (Multi-language)
```yaml
- name: Run Liquibase Migrations
  run: |
    liquibase update \
      --changelog-file=db/changelog-master.xml \
      --url=jdbc:postgresql://db:5432/myapp \
      --username=${{ secrets.DB_USER }} \
      --password=${{ secrets.DB_PASSWORD }}
```

---

## Rollback Strategies

### 1. **Automatic Rollback on Failed Health Checks**

```yaml
- name: Deploy and Monitor
  run: |
    kubectl set image deployment/myapp app=myapp:v2
    
    # Wait for rollout, timeout after 5 minutes
    if ! kubectl rollout status deployment/myapp --timeout=5m; then
      echo "❌ Deployment failed health checks"
      kubectl rollout undo deployment/myapp
      exit 1
    fi
```

### 2. **Rollback to Previous Version**

```bash
# Kubernetes
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=3  # Specific version

# Docker Compose
docker-compose up -d myapp:previous-version

# AWS ECS
aws ecs update-service --cluster prod --service myapp --task-definition myapp:previous
```

### 3. **Keep Last N Versions**

```yaml
- name: Tag and Keep Last 5 Versions
  run: |
    # Tag new version
    docker tag myapp:latest myapp:v${{ github.run_number }}
    docker push myapp:v${{ github.run_number }}
    
    # Clean up old versions (keep last 5)
    ./scripts/cleanup-old-images.sh --keep 5
```

---

## Secrets Management

### Never Hardcode Secrets! ❌

```yaml
# BAD - Don't do this!
- name: Deploy
  run: |
    export DB_PASSWORD="mypassword123"  # ❌ NEVER!
    ./deploy.sh
```

### Use Secret Management Tools ✅

#### 1. **GitHub Secrets**
```yaml
- name: Deploy with Secrets
  env:
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    API_KEY: ${{ secrets.API_KEY }}
  run: |
    ./deploy.sh
```

#### 2. **AWS Secrets Manager**
```yaml
- name: Fetch Secrets from AWS
  run: |
    SECRET=$(aws secretsmanager get-secret-value \
      --secret-id prod/database/password \
      --query SecretString \
      --output text)
    
    export DB_PASSWORD=$(echo $SECRET | jq -r .password)
    ./deploy.sh
```

#### 3. **HashiCorp Vault**
```yaml
- name: Fetch from Vault
  run: |
    vault login -method=aws
    DB_PASSWORD=$(vault kv get -field=password secret/prod/database)
    export DB_PASSWORD
    ./deploy.sh
```

#### 4. **Kubernetes Secrets**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: dbuser
  password: ${{ secrets.DB_PASSWORD }}
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

---

## Monitoring Post-Deployment

### 1. **Health Checks**

```yaml
# Kubernetes liveness and readiness probes
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

### 2. **Smoke Tests**

```bash
#!/bin/bash
# smoke-tests.sh

BASE_URL=$1

echo "Running smoke tests against $BASE_URL"

# Test 1: Health endpoint
response=$(curl -s -o /dev/null -w "%{http_code}" $BASE_URL/health)
if [ $response -ne 200 ]; then
  echo "❌ Health check failed"
  exit 1
fi

# Test 2: API endpoint
response=$(curl -s $BASE_URL/api/version)
if [[ $response != *"version"* ]]; then
  echo "❌ API test failed"
  exit 1
fi

# Test 3: Database connectivity
response=$(curl -s -o /dev/null -w "%{http_code}" $BASE_URL/api/users)
if [ $response -ne 200 ]; then
  echo "❌ Database connectivity failed"
  exit 1
fi

echo "✅ All smoke tests passed"
```

### 3. **Monitoring Metrics**

```yaml
- name: Monitor Key Metrics
  run: |
    python scripts/check-metrics.py \
      --duration 5m \
      --metrics "error_rate,latency_p95,cpu_usage"
```

```python
# check-metrics.py
import requests
import time

PROMETHEUS_URL = "http://prometheus:9090"

def check_error_rate():
    query = 'rate(http_requests_total{status=~"5.."}[5m])'
    result = requests.get(f"{PROMETHEUS_URL}/api/v1/query", 
                         params={"query": query})
    error_rate = float(result.json()['data']['result'][0]['value'][1])
    
    if error_rate > 0.01:  # 1% threshold
        raise Exception(f"❌ High error rate: {error_rate*100:.2f}%")
    print(f"✅ Error rate OK: {error_rate*100:.2f}%")

def check_latency():
    query = 'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))'
    result = requests.get(f"{PROMETHEUS_URL}/api/v1/query",
                         params={"query": query})
    latency_p95 = float(result.json()['data']['result'][0]['value'][1])
    
    if latency_p95 > 0.5:  # 500ms threshold
        raise Exception(f"❌ High latency: {latency_p95*1000:.0f}ms")
    print(f"✅ Latency OK: {latency_p95*1000:.0f}ms")

# Monitor for 5 minutes
for i in range(5):
    check_error_rate()
    check_latency()
    time.sleep(60)

print("✅ All metrics healthy")
```

---

## CD Best Practices

### 1. **Deploy Frequently**
- Small changes = lower risk
- Easier to identify issues
- Faster feedback

### 2. **Automate Everything**
```yaml
# Automate these steps:
Build → Test → Security Scan → Deploy to Dev → Deploy to Staging → 
Run Smoke Tests → [Manual Approval] → Deploy to Production → 
Monitor → Send Notifications
```

### 3. **Use Feature Flags**

```java
// Feature flag example
if (featureFlags.isEnabled("new_checkout_flow")) {
    return newCheckoutProcess(cart);
} else {
    return oldCheckoutProcess(cart);
}
```

**Benefits**:
- Deploy code before enabling feature
- Enable for % of users (canary)
- Instant rollback (disable flag)

### 4. **Immutable Infrastructure**
- Don't modify running servers
- Deploy new versions, kill old ones
- Containers/VMs are disposable

### 5. **Monitor Everything**

Key metrics to track:
- **Deployment frequency**: How often?
- **Lead time**: Commit to production time
- **MTTR**: Mean time to recovery
- **Change failure rate**: % of failed deployments

### 6. **Have a Rollback Plan**

```yaml
# Document rollback steps
## Rollback Procedure
1. Identify the issue (check logs, metrics)
2. Decide: Fix forward or rollback?
3. Rollback command: `kubectl rollout undo deployment/myapp`
4. Verify rollback: Run smoke tests
5. Post-mortem: What went wrong?
```

---

## Advanced CD Topics

### 1. **GitOps with ArgoCD**

```yaml
# Application definition
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  project: default
  source:
    repoURL: https://github.com/mycompany/myapp-k8s
    targetRevision: main
    path: k8s/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**How it works**:
- Git is the source of truth
- ArgoCD watches Git repo
- Automatically syncs cluster with Git
- Drift detection and auto-correction

### 2. **Progressive Delivery with Flagger**

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  progressDeadlineSeconds: 60
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
```

**What happens**:
- Start with 10% traffic to canary
- Monitor success rate
- Increase by 10% every minute if healthy
- Rollback automatically if metrics fail

---

## Common CD Challenges & Solutions

### Challenge 1: Zero-Downtime Deployments
**Solution**: Use blue-green or rolling deployments + readiness probes

### Challenge 2: Database Schema Changes
**Solution**: Expand-contract pattern + backward-compatible changes

### Challenge 3: Configuration Changes
**Solution**: Store config in Git, use ConfigMaps/environment variables

### Challenge 4: Testing in Production
**Solution**: Feature flags + canary deployments + monitoring

### Challenge 5: Rollback Complexity
**Solution**: Keep previous versions, automated rollback on failure

---

## CD Checklist

Before deploying to production, ensure:

- [ ] All tests pass (unit, integration, E2E)
- [ ] Security scans pass
- [ ] Database migrations tested
- [ ] Secrets properly configured
- [ ] Monitoring/alerting set up
- [ ] Rollback procedure documented
- [ ] Smoke tests defined
- [ ] Stakeholders notified
- [ ] Feature flags configured (if needed)
- [ ] Health checks implemented

---

## Remember

> "The goal of CD is not to deploy more often—it's to deploy with confidence. Frequency is just a side effect of removing fear through automation."

Start simple, automate incrementally, and always prioritize reliability over speed.