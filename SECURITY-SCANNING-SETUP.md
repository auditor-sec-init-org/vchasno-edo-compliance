# Security Scanning Integration Setup
## VCHASNO EDO - Week 5 Implementation

## 1. SNYK CONFIGURATION

### 1.1 Installation
```bash
npm install -g snyk
snyk auth  # Interactive login with GitHub account
npm test   # First test to establish baselines
snyk test  # Run Snyk test
snyk code test  # Run code vulnerability scan
```

### 1.2 Snyk CLI Commands
```bash
# Test for dependency vulnerabilities
snyk test --severity-threshold=high

# Code analysis
snyk code test --severity-threshold=high

# Monitor for continuous tracking
snyk monitor

# Fix vulnerabilities automatically
snyk fix --severity-threshold=high
```

### 1.3 GitHub Actions Integration
Add to `.github/workflows/main.yml` (already configured)

## 2. SONARQUBE SETUP

### 2.1 Docker Deployment
```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -e SONAR_JDBC_URL=jdbc:postgresql://postgres:5432/sonarqube \
  -e SONAR_JDBC_USERNAME=sonar \
  -e SONAR_JDBC_PASSWORD=secure_password \
  sonarqube:latest
```

### 2.2 Project Configuration (sonar-project.properties)
```properties
sonar.projectKey=vchasno-edo
sonar.projectName=VCHASNO EDO
sonar.sources=src
sonar.tests=tests
sonar.exclusions=node_modules/**,dist/**
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.coverage.exclusions=**/*.test.js,**/node_modules/**
sonar.qualitygate.wait=true
```

### 2.3 CI/CD Integration
```yaml
- name: SonarQube Analysis
  run: |
    npx sonar-scanner \
      -Dsonar.projectKey=vchasno-edo \
      -Dsonar.sources=. \
      -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
      -Dsonar.login=${{ secrets.SONAR_TOKEN }}
```

## 3. DEPENDENCY SCANNING

### 3.1 Enable GitHub Dependabot
- Go to Repository → Security → Code security and analysis
- Enable "Dependabot alerts"
- Enable "Dependabot security updates"

### 3.2 Dependabot Configuration (.github/dependabot.yml)
```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "03:00"
    open-pull-requests-limit: 5
    reviewers: ["security-team"]
    allow:
      - dependency-type: "direct"
```

## 4. CONTAINER SCANNING

### 4.1 Trivy Image Scanning
```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image vchasno-api:latest
```

### 4.2 GitHub Container Scanning
Enable in: Settings → Code security and analysis → Container image scanning

## 5. INFRASTRUCTURE SCANNING

### 5.1 Bridgecrewâ€™s Checkov (IaC Scanning)
```bash
pip install checkov
checkov -d . --check CKV_K8S_1,CKV_K8S_8  # Scan Kubernetes configs
```

### 5.2 Terraform/CloudFormation Scanning
```bash
checkov -f infrastructure/main.tf  # Scan Terraform
checkov --framework cloudformation   # Scan CloudFormation
```

## 6. SECRETS SCANNING

### 6.1 GitHub Secret Scanning
- Enabled by default for public repos
- Monitors commits for exposed secrets (API keys, tokens)
- Alerts on detection

### 6.2 TruffleHog (Local Scanning)
```bash
pip install truffleHog
trufflehog filesystem . --json
```

## 7. SECURITY DASHBOARD

### Consolidated Monitoring
- Snyk: https://app.snyk.io/[organization]
- SonarQube: http://sonarqube:9000
- GitHub Alerts: repo/security/overview
- Dependabot: repo/security/dependabot

## 8. QUALITY GATES

### SonarQube Gate Configuration
- Bugs: 0 (Blocker)
- Vulnerabilities: 0 (Blocker)
- Security Hotspots: 100% reviewed
- Coverage: >80%
- Duplications: <3%

### Required Secrets for CI/CD
```
SNYK_TOKEN          # Snyk authentication
SONAR_HOST_URL      # SonarQube server URL
SONAR_TOKEN         # SonarQube authentication
```

## 9. DEPLOYMENT CHECKLIST
- [ ] Install Snyk CLI
- [ ] Run initial Snyk test
- [ ] Deploy SonarQube instance
- [ ] Configure GitHub secret scanning
- [ ] Enable Dependabot
- [ ] Set up security dashboard
- [ ] Configure quality gates
- [ ] Add secrets to GitHub Actions
- [ ] Test scanning in develop branch
- [ ] Document security procedures
