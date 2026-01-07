# Complete DevSecOps Pipeline - Implementation Guide

## Production-Ready Security Pipeline

This is a real-world DevSecOps pipeline that integrates security at every stage.

---

## Complete Pipeline Architecture

```
Code Push → GitHub Actions VM
    ↓
1. Checkout Code
    ↓
2. Setup Environment (Java/Node/Python)
    ↓
3. Code Quality (Linting)
    ↓
4. SAST Scan (CodeQL) ────────┐
    ↓                          │
5. SCA Scan (OWASP) ──────────┤→ Security Gate
    ↓                          │
6. Unit Tests                  │
    ↓                          │
7. Build Application           │
    ↓                          │
8. Build Docker Image          │
    ↓                          │
9. Container Scan (Trivy) ────┘
    ↓
10. Runtime Test (Smoke Test)
    ↓
11. Push to Registry (if all pass)
    ↓
12. Notify Team
```

---

## Full GitHub Actions Pipeline

### File: `.github/workflows/devsecops-pipeline.yml`

```yaml
name: DevSecOps Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:  # Manual trigger

env:
  JAVA_VERSION: '17'
  DOCKER_IMAGE: ${{ secrets.DOCKERHUB_USERNAME }}/myapp
  
permissions:
  contents: read
  security-events: write  # Required for security alerts

jobs:
  security-pipeline:
    runs-on: ubuntu-latest
    
    steps:
    #=========================================
    # STAGE 1: CODE CHECKOUT
    #=========================================
    - name: Checkout Code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0  # Full history for better analysis
    
    #=========================================
    # STAGE 2: SETUP ENVIRONMENT
    #=========================================
    - name: Set up JDK ${{ env.JAVA_VERSION }}
      uses: actions/setup-java@v4
      with:
        java-version: ${{ env.JAVA_VERSION }}
        distribution: 'temurin'
        cache: 'maven'
    
    #=========================================
    # STAGE 3: CODE QUALITY CHECK
    #=========================================
    - name: Run Checkstyle (Linting)
      run: mvn checkstyle:check
      continue-on-error: true  # Don't fail, but report
    
    - name: Run SpotBugs (Static Analysis)
      run: mvn spotbugs:check
      continue-on-error: true
    
    #=========================================
    # STAGE 4: SAST - SOURCE CODE SECURITY
    #=========================================
    - name: Initialize CodeQL
      uses: github/codeql-action/init@v2
      with:
        languages: java
        queries: security-extended  # More thorough security checks
    
    - name: Build for CodeQL
      run: mvn clean compile -DskipTests
    
    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v2
    
    #=========================================
    # STAGE 5: SCA - DEPENDENCY SECURITY
    #=========================================
    - name: OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'myapp'
        path: '.'
        format: 'HTML'
        args: >
          --failOnCVSS 7
          --enableRetired
    
    - name: Upload Dependency Check Report
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: dependency-check-report
        path: dependency-check-report.html
    
    #=========================================
    # STAGE 6: UNIT TESTS
    #=========================================
    - name: Run Unit Tests
      run: mvn test
    
    - name: Generate Test Coverage
      run: mvn jacoco:report
    
    - name: Check Coverage Threshold
      run: |
        COVERAGE=$(mvn jacoco:check -Dcoverage.threshold=70 | grep -o '[0-9]*%')
        echo "Code Coverage: $COVERAGE"
    
    - name: Upload Coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        files: ./target/site/jacoco/jacoco.xml
        fail_ci_if_error: false
    
    #=========================================
    # STAGE 7: BUILD APPLICATION
    #=========================================
    - name: Build Application
      run: mvn clean package -DskipTests
    
    - name: Upload JAR Artifact
      uses: actions/upload-artifact@v3
      with:
        name: application-jar
        path: target/*.jar
    
    #=========================================
    # STAGE 8: BUILD DOCKER IMAGE
    #=========================================
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Build Docker Image
      run: |
        docker build -t ${{ env.DOCKER_IMAGE }}:${{ github.sha }} .
        docker tag ${{ env.DOCKER_IMAGE }}:${{ github.sha }} ${{ env.DOCKER_IMAGE }}:latest
    
    #=========================================
    # STAGE 9: CONTAINER SECURITY SCAN
    #=========================================
    - name: Run Trivy Vulnerability Scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: 1  # Fail on critical/high vulnerabilities
    
    - name: Upload Trivy Results to GitHub Security
      if: always()
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Run Trivy HTML Report
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
        format: 'template'
        template: '@/contrib/html.tpl'
        output: 'trivy-report.html'
    
    - name: Upload Trivy HTML Report
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: trivy-report
        path: trivy-report.html
    
    #=========================================
    # STAGE 10: CONTAINER RUNTIME TEST
    #=========================================
    - name: Test Container Runtime
      run: |
        echo "Starting container..."
        docker run -d --name test-container -p 8080:8080 ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
        
        echo "Waiting for application to start..."
        timeout 60 bash -c 'until curl -f http://localhost:8080/actuator/health; do sleep 2; done'
        
        echo "Running smoke tests..."
        curl -f http://localhost:8080/actuator/health || exit 1
        
        echo "Stopping container..."
        docker stop test-container
        docker rm test-container
    
    #=========================================
    # STAGE 11: GENERATE SBOM
    #=========================================
    - name: Generate SBOM (Software Bill of Materials)
      uses: anchore/sbom-action@v0
      with:
        image: ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
        format: spdx-json
        output-file: sbom.spdx.json
    
    - name: Upload SBOM
      uses: actions/upload-artifact@v3
      with:
        name: sbom
        path: sbom.spdx.json
    
    #=========================================
    # STAGE 12: PUSH TO REGISTRY (Only on main)
    #=========================================
    - name: Login to DockerHub
      if: github.ref == 'refs/heads/main'
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    
    - name: Push Docker Image
      if: github.ref == 'refs/heads/main'
      run: |
        docker push ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
        docker push ${{ env.DOCKER_IMAGE }}:latest
    
    #=========================================
    # STAGE 13: NOTIFICATIONS
    #=========================================
    - name: Notify Success
      if: success() && github.ref == 'refs/heads/main'
      uses: slackapi/slack-github-action@v1.24.0
      with:
        webhook-url: ${{ secrets.SLACK_WEBHOOK }}
        payload: |
          {
            "text": "✅ DevSecOps Pipeline Passed",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Pipeline Successful*\n✅ All security scans passed\n✅ Image pushed: `${{ env.DOCKER_IMAGE }}:${{ github.sha }}`\n✅ Commit: `${{ github.sha }}`"
                }
              }
            ]
          }
    
    - name: Notify Failure
      if: failure()
      uses: slackapi/slack-github-action@v1.24.0
      with:
        webhook-url: ${{ secrets.SLACK_WEBHOOK }}
        payload: |
          {
            "text": "❌ DevSecOps Pipeline Failed",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Pipeline Failed*\n❌ Check security scans and logs\n❌ Commit: `${{ github.sha }}`\n@channel"
                }
              }
            ]
          }
```

---

## Required Secrets Setup

**GitHub Repository → Settings → Secrets and Variables → Actions**

| Secret Name | Value | Purpose |
|------------|-------|---------|
| `DOCKERHUB_USERNAME` | your_username | Docker registry auth |
| `DOCKERHUB_TOKEN` | access_token | Docker registry auth |
| `SLACK_WEBHOOK` | webhook_url | Notifications |
| `SONAR_TOKEN` | sonar_token | SonarQube (optional) |

---

## Dockerfile Best Practices

### Secure Dockerfile Example

```dockerfile
# Multi-stage build for security and size
FROM maven:3.9-eclipse-temurin-17-alpine AS builder

WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn package -DskipTests

# Runtime image - minimal and secure
FROM eclipse-temurin:17-jre-alpine

# Create non-root user
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

WORKDIR /app

# Copy only the JAR, not source code
COPY --from=builder /app/target/*.jar app.jar

# Set ownership
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# Run application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Why this is secure:**
- ✅ Multi-stage build (smaller final image)
- ✅ Alpine Linux (minimal attack surface)
- ✅ Non-root user
- ✅ No source code in final image
- ✅ Health check included
- ✅ Only production dependencies

---

## Security Scan Examples & Interpretation

### 1. Trivy Output Example

```
Total: 45 (CRITICAL: 2, HIGH: 8, MEDIUM: 20, LOW: 15)

┌────────────────┬──────────────┬──────────┬───────────────────┬───────────────┐
│    Library     │ Vulnerability│ Severity │  Installed Version│ Fixed Version │
├────────────────┼──────────────┼──────────┼───────────────────┼───────────────┤
│ log4j-core     │ CVE-2021-44228│ CRITICAL│     2.14.1        │    2.17.1     │
│ spring-web     │ CVE-2022-22965│   HIGH   │     5.3.15        │    5.3.18     │
└────────────────┴──────────────┴──────────┴───────────────────┴───────────────┘
```

**Action Required:**
1. Update log4j to 2.17.1 immediately (CRITICAL)
2. Update spring-web to 5.3.18 (HIGH)
3. Pipeline will fail until fixed

### 2. OWASP Dependency Check Output

```
Dependency-Check Report
Project: myapp
Generated: 2024-01-27

Summary:
  Total Dependencies: 42
  Vulnerable Dependencies: 3
  
Critical Vulnerabilities: 1
  - log4j-core 2.14.1 (CVE-2021-44228)
    CVSS Score: 10.0
    Fix: Upgrade to 2.17.1+
```

### 3. CodeQL Findings Example

```
SQL Injection vulnerability detected:
  File: UserController.java
  Line: 45
  Code: String query = "SELECT * FROM users WHERE id = " + userId;
  
  Fix: Use PreparedStatement
  PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
  stmt.setInt(1, userId);
```

---

## Handling Security Failures

### Strategy 1: Fail Fast (Recommended)
```yaml
- name: Trivy Scan
  run: trivy image --severity CRITICAL,HIGH --exit-code 1 myapp:latest
  # Pipeline stops here if vulnerabilities found
```

### Strategy 2: Continue with Warning
```yaml
- name: Trivy Scan
  run: trivy image --severity CRITICAL,HIGH myapp:latest
  continue-on-error: true
  # Scan runs, reports, but doesn't block
```

### Strategy 3: Conditional Blocking
```yaml
- name: Trivy Scan
  run: trivy image --severity CRITICAL --exit-code 1 myapp:latest
  # Only block on CRITICAL, allow HIGH
```

---

## Pre-commit Hooks for Developers

### Install pre-commit framework
```bash
pip install pre-commit
```

### `.pre-commit-config.yaml`
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ['--maxkb=500']
  
  - repo: https://github.com/trufflesecurity/trufflehog
    rev: v3.63.2
    hooks:
      - id: trufflehog
        name: TruffleHog
        entry: bash -c 'trufflehog git file://. --since-commit HEAD --only-verified --fail'
  
  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black
```

**Install hooks:**
```bash
pre-commit install
```

Now every commit is automatically checked for:
- Secrets
- Large files
- Code formatting
- YAML syntax

---

## Monitoring & Dashboards

### GitHub Security Dashboard
**Repository → Security tab shows:**
- CodeQL alerts
- Dependabot alerts
- Secret scanning alerts
- Trivy findings

### Metrics to Track

```yaml
# Create metrics dashboard
metrics:
  - vulnerabilities_found: sum(critical + high)
  - vulnerabilities_fixed: count(closed_issues)
  - mean_time_to_fix: avg(close_time - open_time)
  - dependency_age: max(dependency_last_updated)
  - scan_frequency: count(scans_per_week)
```

**Target KPIs:**
- Critical vulnerabilities: **0**
- High vulnerabilities: **< 5**
- MTTR (Mean Time to Remediate): **< 7 days**
- Dependency age: **< 6 months**

---

## Incident Response Plan

### When Security Issue Found:

**1. Assess Severity**
```
CRITICAL (CVSS 9.0-10.0) → Fix within 24 hours
HIGH (CVSS 7.0-8.9) → Fix within 1 week
MEDIUM (CVSS 4.0-6.9) → Fix within 1 month
LOW (CVSS 0.1-3.9) → Fix in next sprint
```

**2. Create Ticket**
```markdown
Title: [SECURITY] CVE-2021-44228 in log4j-core
Priority: P0 (Critical)
Assigned: Security Team + Dev Lead

Description:
- CVE: CVE-2021-44228
- Component: log4j-core 2.14.1
- CVSS Score: 10.0 (Critical)
- Impact: Remote code execution
- Fix: Upgrade to 2.17.1+
- Affected: All environments
```

**3. Fix and Verify**
```bash
# Update dependency
mvn versions:use-latest-releases -Dincludes=org.apache.logging.log4j

# Re-scan
mvn dependency-check:check

# Rebuild and deploy
git commit -am "fix: update log4j to 2.17.1 (CVE-2021-44228)"
git push
```

**4. Notify Stakeholders**
```
Subject: RESOLVED - Critical Security Vulnerability Fixed

The critical Log4Shell vulnerability (CVE-2021-44228) has been patched.
- Fixed in: commit abc123
- Deployed to: all environments
- Verified by: security scans passing
```

---

## Advanced Topics

### 1. Container Image Signing (Cosign)

```yaml
- name: Install Cosign
  uses: sigstore/cosign-installer@main

- name: Sign Image
  run: |
    cosign sign --key cosign.key ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
```

### 2. Policy Enforcement (OPA)

```yaml
- name: Run OPA Policy Check
  run: |
    opa test policies/ --verbose
    docker run --rm -v $(pwd)/policies:/policies openpolicyagent/opa:latest \
      eval -d /policies -i image.json "data.image.deny"
```

### 3. Runtime Monitoring (Falco)

```yaml
# falco-rules.yaml
- rule: Unexpected Network Connection
  condition: spawned_process and network_connection
  output: "Unexpected network connection (user=%user.name command=%proc.cmdline)"
  priority: WARNING
```

---

## Complete Checklist

**Pipeline Setup:**
- [ ] SAST scan configured
- [ ] SCA scan configured
- [ ] Container scan configured
- [ ] Secrets management in place
- [ ] Quality gates defined
- [ ] Notifications configured

**Code Security:**
- [ ] No hardcoded secrets
- [ ] Input validation everywhere
- [ ] Parameterized queries (no SQL injection)
- [ ] Output encoding (no XSS)
- [ ] Dependencies pinned to versions

**Container Security:**
- [ ] Minimal base image (Alpine/Distroless)
- [ ] Non-root user
- [ ] Multi-stage builds
- [ ] No secrets in images
- [ ] Health checks defined

**Operations:**
- [ ] Security dashboard monitored
- [ ] Automated dependency updates
- [ ] Incident response plan documented
- [ ] Team trained on security practices

---

## Quick Reference Commands

```bash
# Local security scanning before push
trivy image myapp:latest
docker scan myapp:latest
mvn dependency-check:check

# Check for secrets
trufflehog git file://. --since-commit HEAD

# Generate SBOM
syft myapp:latest -o spdx-json > sbom.json

# Sign image
cosign sign --key cosign.key myapp:latest

# Verify signature
cosign verify --key cosign.pub myapp:latest
```

---

## Key Takeaways

1. **Automate all security checks** - No manual reviews
2. **Fail fast on critical issues** - Block deployment
3. **Three-layer scanning** - Code, dependencies, containers
4. **Generate SBOMs** - Track all components
5. **Monitor continuously** - Security doesn't stop at deployment
6. **Train the team** - Security is everyone's job

**This pipeline ensures only secure, tested, and verified software reaches production.**