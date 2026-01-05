# DevSecOps Essentials - Security in CI/CD

## What is DevSecOps?

**DevSecOps = Development + Security + Operations**

Traditional approach: Security checks at the end (too late, too expensive)  
DevSecOps approach: Security integrated throughout the pipeline (shift-left security)

---

## Core Principle: Shift-Left Security

**Shift-Left** = Catch security issues early in development

```
Traditional:  Code → Build → Test → [Security Check] → Deploy ❌
DevSecOps:    [Security] Code → [Security] Build → [Security] Test → Deploy ✅
```

**Why?**
- Finding bugs in production costs 100x more than in development
- Prevents vulnerabilities from reaching users
- Faster fixes (developers remember context)

---

## The Security Triad

### 1. SAST (Static Application Security Testing)
**Scans source code without running it**

**What it finds:**
- SQL injection vulnerabilities
- Cross-site scripting (XSS)
- Hardcoded secrets (passwords, API keys)
- Insecure cryptography
- Command injection
- Path traversal

**Tools:**
- **SonarQube** - Code quality + security
- **CodeQL** - GitHub's built-in scanner
- **Checkmarx** - Enterprise SAST
- **Semgrep** - Fast, customizable rules

**When to run:** On every commit or pull request

```yaml
# Example: CodeQL scan
- name: Initialize CodeQL
  uses: github/codeql-action/init@v2
  with:
    languages: java

- name: Autobuild
  uses: github/codeql-action/autobuild@v2

- name: Perform CodeQL Analysis
  uses: github/codeql-action/analyze@v2
```

### 2. SCA (Software Composition Analysis)
**Scans third-party dependencies and libraries**

**What it finds:**
- Vulnerable npm packages
- Outdated Maven dependencies
- Known CVEs in libraries
- License violations

**Tools:**
- **OWASP Dependency-Check** - Free, open-source
- **Snyk** - Developer-friendly, great UI
- **Dependabot** - GitHub's automated updates
- **WhiteSource/Mend** - Enterprise SCA

**When to run:** Daily or on every build

```yaml
# Example: OWASP Dependency Check
- name: Dependency Check
  uses: dependency-check/Dependency-Check_Action@main
  with:
    project: 'myapp'
    path: '.'
    format: 'HTML'
    
- name: Upload Results
  uses: actions/upload-artifact@v3
  with:
    name: dependency-check-report
    path: dependency-check-report.html
```

**Real example:** Log4Shell (CVE-2021-44228) - SCA tools caught this before exploitation

### 3. Container Security Scanning
**Scans Docker images for vulnerabilities**

**What it finds:**
- Vulnerable OS packages (Alpine, Ubuntu layers)
- Insecure base images
- Exposed secrets in images
- Misconfigured containers
- Known CVEs in image layers

**Tools:**
- **Trivy** - Fast, accurate, free (recommended)
- **Grype** - Anchore's open-source scanner
- **Clair** - CoreOS project
- **Snyk Container** - Commercial option

**When to run:** After building Docker image, before pushing

```yaml
# Example: Trivy scan
- name: Run Trivy Scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:latest'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: 1  # Fail pipeline on critical vulnerabilities

- name: Upload to GitHub Security
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'
```

---

## Critical Security Practices

### 1. Never Hardcode Secrets

❌ **BAD:**
```java
String apiKey = "sk-1234567890abcdef";
String dbPassword = "MyPassword123!";
```

✅ **GOOD:**
```java
String apiKey = System.getenv("API_KEY");
String dbPassword = System.getenv("DB_PASSWORD");
```

**In CI/CD:**
```yaml
env:
  API_KEY: ${{ secrets.API_KEY }}
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

### 2. Secret Scanning

**Tools:**
- **GitGuardian** - Monitors commits for secrets
- **TruffleHog** - Finds secrets in Git history
- **GitHub Secret Scanning** - Built-in (for public repos)

**Pre-commit hook:**
```bash
# .git/hooks/pre-commit
#!/bin/bash
if git diff --cached | grep -i "password\|api_key\|secret"; then
    echo "⚠️  Possible secret detected!"
    exit 1
fi
```

### 3. Dependency Pinning

❌ **BAD (unpredictable):**
```xml
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>LATEST</version>  <!-- Don't do this! -->
</dependency>
```

✅ **GOOD (pinned version):**
```xml
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.17.1</version>  <!-- Specific, patched version -->
</dependency>
```

### 4. Minimal Docker Images

❌ **BAD (large attack surface):**
```dockerfile
FROM ubuntu:latest
RUN apt-get install -y openjdk-17 curl wget vim git
```

✅ **GOOD (minimal attack surface):**
```dockerfile
FROM openjdk:17-alpine  # Minimal base
# Only install what you need
```

**Size matters:**
- `ubuntu:latest` = ~78 MB (more vulnerabilities)
- `alpine:latest` = ~5 MB (fewer vulnerabilities)

---

## Security Quality Gates

**Quality Gate** = Automated check that must pass before proceeding

### Example Quality Gates:

```yaml
# 1. Block critical vulnerabilities
- name: Trivy Scan
  run: |
    trivy image --severity CRITICAL --exit-code 1 myapp:latest
    # exit-code 1 = fail pipeline if CRITICAL found

# 2. Enforce code coverage
- name: Check Coverage
  run: |
    mvn jacoco:check -Dcoverage.threshold=80
    # Fail if coverage < 80%

# 3. Enforce linting
- name: Lint Code
  run: |
    mvn checkstyle:check
    # Fail on style violations

# 4. Block vulnerable dependencies
- name: Dependency Check
  run: |
    mvn dependency-check:check -DfailBuildOnCVSS=7
    # Fail if CVSS score >= 7 (high severity)
```

---

## SBOM (Software Bill of Materials)

**SBOM** = List of all components in your software (like ingredient list on food)

**Why it matters:**
- Quickly identify if you're affected by new CVE
- Compliance requirements (executive order 14028)
- Supply chain transparency

**Generate SBOM:**

```yaml
# Using Syft
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: myapp:latest
    format: spdx-json
    output-file: sbom.spdx.json

- name: Upload SBOM
  uses: actions/upload-artifact@v3
  with:
    name: sbom
    path: sbom.spdx.json
```

---

## Runtime Security

### 1. Container Runtime Protection
- **Falco** - Detects anomalous behavior
- **AppArmor/SELinux** - Mandatory access control
- **Seccomp** - Syscall filtering

### 2. Read-Only Root Filesystem
```yaml
# Kubernetes Pod Security
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
```

### 3. Network Policies
```yaml
# Kubernetes Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

---

## Security Metrics to Track

| Metric | Target | Why |
|--------|--------|-----|
| **Mean Time to Remediate (MTTR)** | < 7 days | Speed of fixing vulnerabilities |
| **Critical Vulnerabilities** | 0 | No critical issues in production |
| **Dependency Freshness** | < 6 months old | Reduce outdated libraries |
| **Secret Exposure Incidents** | 0 | No leaked credentials |
| **Failed Security Scans** | Trend down | Improving security posture |

---

## Compliance & Standards

### OWASP Top 10 (2021)
1. Broken Access Control
2. Cryptographic Failures
3. Injection
4. Insecure Design
5. Security Misconfiguration
6. Vulnerable Components
7. Authentication Failures
8. Software/Data Integrity Failures
9. Security Logging Failures
10. Server-Side Request Forgery

**SAST tools catch most of these automatically.**

### CIS Benchmarks
- Docker CIS Benchmark
- Kubernetes CIS Benchmark
- Linux CIS Benchmark

**Use:** `docker-bench-security` and `kube-bench` to audit

---

## Quick Security Checklist

**Before Committing:**
- [ ] No hardcoded secrets
- [ ] Dependencies up to date
- [ ] Code follows secure coding standards

**Before Building:**
- [ ] SAST scan passes
- [ ] SCA scan passes
- [ ] Unit tests pass

**Before Pushing Image:**
- [ ] Container scan passes
- [ ] Image size optimized
- [ ] Running as non-root user

**Before Deploying:**
- [ ] Smoke tests pass
- [ ] Security headers configured
- [ ] Monitoring/logging enabled

---

## Common Vulnerabilities & Fixes

### 1. SQL Injection
❌ **Vulnerable:**
```java
String query = "SELECT * FROM users WHERE id = " + userId;
```

✅ **Fixed:**
```java
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
stmt.setInt(1, userId);
```

### 2. Path Traversal
❌ **Vulnerable:**
```java
String filename = request.getParameter("file");
File file = new File("/uploads/" + filename);  // Can access ../../../etc/passwd
```

✅ **Fixed:**
```java
String filename = Paths.get(request.getParameter("file")).getFileName().toString();
File file = new File("/uploads/" + filename);
```

### 3. XSS (Cross-Site Scripting)
❌ **Vulnerable:**
```html
<div>${userInput}</div>  <!-- Executes <script> tags -->
```

✅ **Fixed:**
```html
<div th:text="${userInput}"></div>  <!-- Escaped by Thymeleaf -->
```

---

## DevSecOps Culture

### The Three Ways:
1. **Automate everything** - No manual security checks
2. **Fast feedback** - Fail fast on security issues
3. **Continuous improvement** - Learn from incidents

### Developer Responsibilities:
- Write secure code (training matters)
- Fix vulnerabilities promptly
- Don't disable security checks

### Security Team Responsibilities:
- Provide tools (don't be blockers)
- Enable developers
- Define security policies


**Remember:** Security is a continuous process, not a one-time checkbox.