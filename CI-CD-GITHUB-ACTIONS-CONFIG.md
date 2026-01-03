# GitHub Actions CI/CD Configuration
## VCHASNO EDO System - Complete Workflow Setup

## .github/workflows/main.yml

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  REGISTRY: gcr.io
  PROJECT_ID: vchasno-edo
  IMAGE_NAME: vchasno-api

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run lint
      - run: npm run format:check

  test:
    runs-on: ubuntu-latest
    needs: lint
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - run: npm run test:integration
      - uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json

  security:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - run: npm install -g snyk
      - run: snyk auth ${{ secrets.SNYK_TOKEN }}
      - run: snyk test --severity-threshold=high
      - run: snyk code test --severity-threshold=high

  build:
    runs-on: ubuntu-latest
    needs: [test, security]
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      - uses: google-github-actions/setup-gcloud@v2
      - run: gcloud auth configure-docker ${{ env.REGISTRY }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.vchasno.dev
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      - uses: google-github-actions/setup-gcloud@v2
      - run: |
          gcloud container clusters get-credentials vchasno-staging --zone us-central1-a
          kubectl set image deployment/vchasno-api app=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.IMAGE_NAME }}:${{ github.sha }} -n staging
          kubectl rollout status deployment/vchasno-api -n staging

  e2e-staging:
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.event_name == 'push' && github.ref == 'refs/heads/develop'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run test:e2e:staging
      - uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: cypress-screenshots
          path: cypress/screenshots/

  deploy-production:
    runs-on: ubuntu-latest
    needs: [build, e2e-staging]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://vchasno.com
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      - uses: google-github-actions/setup-gcloud@v2
      - run: |
          gcloud container clusters get-credentials vchasno-prod --zone us-central1-a
          kubectl set image deployment/vchasno-api app=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.IMAGE_NAME }}:${{ github.sha }} -n production
          kubectl rollout status deployment/vchasno-api -n production

  smoke-test-production:
    runs-on: ubuntu-latest
    needs: deploy-production
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run test:smoke:production
```

## Branch Protection Rules

### main (Production)
- Require pull request reviews: 2 approvals
- Require status checks to pass: All must pass
- Dismiss stale pull request approvals
- Require branches to be up to date
- Require conversation resolution

### develop (Staging)
- Require pull request reviews: 1 approval
- Require status checks to pass: Lint, Test, Security
- Auto-delete head branches

## Required Secrets

```
SNYK_TOKEN              # Snyk security scanning
GCP_SA_KEY              # Google Cloud credentials
CODECOV_TOKEN           # Code coverage tracking
```

## Deployment Checklist

- [ ] Configure GitHub branch protection rules
- [ ] Add GCP service account credentials
- [ ] Configure Snyk token
- [ ] Set up Codecov integration
- [ ] Test workflow on develop branch
- [ ] Verify staging deployment
- [ ] Perform smoke tests
- [ ] Document deployment runbook
