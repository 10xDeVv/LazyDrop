# LazyDrop CI/CD Pipeline Guide

## Overview

LazyDrop uses GitHub Actions for continuous integration and continuous deployment (CI/CD). The pipeline ensures code quality, security, and reliability before deployment.

---

## Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Git Push/Pull Request                    │
└────────────────┬────────────────────────────────┬────────────┘
                 │                                │
          ┌──────▼──────┐              ┌─────────▼────────┐
          │   Backend   │              │    Frontend      │
          │   Tests     │              │    Tests         │
          └──────┬──────┘              └────────┬─────────┘
                 │                              │
          ┌──────▼──────┐              ┌────────▼────────┐
          │   Quality   │              │   Build & Lint  │
          │   Analysis  │              │   & Security    │
          └──────┬──────┘              └────────┬────────┘
                 │                              │
                 └──────────┬───────────────────┘
                            │
                    ┌───────▼────────┐
                    │  Docker Build  │
                    │  & Push        │
                    └───────┬────────┘
                            │
                    ┌───────▼────────┐
                    │  Image Scan    │
                    │  (Trivy)       │
                    └────────────────┘
```

---

## Workflows

### 1. Backend Tests & Code Quality

**File:** `.github/workflows/backend-test.yml`

**Triggers:**
- Push to `main` or `develop` branches
- Pull requests to `main` or `develop`
- Changes to `apps/backend/**` or workflow file

**Jobs:**

#### Test Job
- Sets up PostgreSQL container for integration tests
- Creates test environment variables
- Runs Maven tests: `mvn clean test`
- Generates JaCoCo coverage report
- Uploads coverage to Codecov
- **Duration:** ~3-5 minutes

#### Build Job
- Builds JAR artifact: `mvn clean package -DskipTests`
- Uploads JAR for potential deployment
- **Duration:** ~2-3 minutes
- **Dependency:** Must pass `test` job

#### Quality Job
- Runs SonarQube analysis (if configured)
- Performs OWASP dependency vulnerability check
- **Duration:** ~2-3 minutes
- **Dependency:** Must pass `test` job

#### Notify Job
- Comments on PR with status
- Summarizes all job results
- **Runs:** Always (even if jobs fail)

**Environment Variables:**
```
DATABASE_URL=jdbc:postgresql://localhost:5432/lazydrop_test
DATABASE_USERNAME=test
DATABASE_PASSWORD=test
SUPABASE_URL=https://test.supabase.co
SUPABASE_ANON_KEY=test_key
SUPABASE_JWT_SECRET=test_secret
SPACES_ENDPOINT=https://nyc3.digitaloceanspaces.com
SPACES_REGION=nyc3
SPACES_BUCKET=test-bucket
SPACES_ACCESS_KEY=test_access_key
SPACES_SECRET_KEY=test_secret_key
CORS_ALLOWED_ORIGINS=http://localhost:3000
APP_FRONTEND_URL=http://localhost:3000
APP_COOKIES_SECURE=false
STRIPE_TEST_SECRET_KEY=sk_test_example
STRIPE_WEBHOOK_SECRET=whsec_test
```

---

### 2. Frontend Tests & Build

**File:** `.github/workflows/frontend-test.yml`

**Triggers:**
- Push to `main` or `develop` branches
- Pull requests to `main` or `develop`
- Changes to `apps/frontend/**` or workflow file

**Jobs:**

#### Test & Lint Job
- Installs dependencies: `npm ci`
- Runs ESLint: `npm run lint`
- Runs Vitest: `npm test -- --run`
- **Duration:** ~2-3 minutes
- **Continues on error:** Allows subsequent jobs to run

#### Build Job
- Builds Next.js application: `npm run build`
- Uploads `.next` artifact
- **Duration:** ~3-4 minutes
- **Dependency:** Needs `test` job

#### Security Job
- Runs npm audit: `npm audit --audit-level=moderate`
- Runs Snyk check (if token provided)
- **Duration:** ~1-2 minutes
- **Continues on error:** Non-blocking

#### Notify Job
- Comments on PR with results
- **Runs:** Always

---

### 3. Docker Build & Push

**File:** `.github/workflows/docker-build.yml`

**Triggers:**
- Push to `main` or `develop` branches
- Completion of backend or frontend test workflows
- Manual trigger

**Jobs:**

#### Build Backend Image
- Sets up Docker Buildx for multi-platform builds
- Logs into Docker Hub (if credentials provided)
- Builds backend image with tags
- Pushes to registry
- Uses build cache for faster builds
- **Duration:** ~3-5 minutes

#### Build Frontend Image
- Builds frontend image with environment args
- Includes NEXT_PUBLIC variables
- Pushes to registry
- **Duration:** ~2-3 minutes

#### Scan Images Job
- Scans both images with Trivy
- Generates SARIF reports
- Uploads to GitHub Security tab
- **Duration:** ~2-3 minutes per image
- **Continues on error:** Non-blocking

**Image Tags:**
```
- main-latest                    (for main branch)
- develop-latest                 (for develop branch)
- sha-abc123def                  (commit SHA)
- v1.2.3 (semver if tagged)
```

---

## Setup Instructions

### Prerequisites

1. GitHub repository with GitHub Actions enabled
2. Docker Hub account (optional, for pushing images)
3. SonarQube server (optional, for code quality)
4. Codecov account (optional, for coverage tracking)

### Step 1: Add Secrets to GitHub

Go to **Settings → Secrets and variables → Actions**

**Backend Secrets:**
```
DATABASE_URL
DATABASE_USERNAME
DATABASE_PASSWORD
SUPABASE_URL
SUPABASE_ANON_KEY
SUPABASE_JWT_SECRET
SPACES_ENDPOINT
SPACES_REGION
SPACES_BUCKET
SPACES_ACCESS_KEY
SPACES_SECRET_KEY
CORS_ALLOWED_ORIGINS
APP_FRONTEND_URL
APP_COOKIES_SECURE
STRIPE_TEST_SECRET_KEY
STRIPE_WEBHOOK_SECRET
SONAR_HOST_URL (optional)
SONAR_LOGIN (optional)
```

**Frontend Secrets:**
```
NEXT_PUBLIC_SUPABASE_URL
NEXT_PUBLIC_SUPABASE_ANON_KEY
NEXT_PUBLIC_API_URL
SNYK_TOKEN (optional)
```

**Docker Secrets:**
```
DOCKER_USERNAME
DOCKER_PASSWORD
```

### Step 2: Enable GitHub Pages (Optional)

For hosting coverage reports:
1. Go to **Settings → Pages**
2. Set source to **GitHub Actions**
3. Coverage reports will be published automatically

### Step 3: Configure Branch Protection (Recommended)

1. Go to **Settings → Branches**
2. Add rule for `main` branch:
   - ✅ Require status checks to pass
   - ✅ Require branches to be up to date
   - ✅ Require code reviews
   - ✅ Dismiss stale pull request approvals

---

## Usage

### Running Workflows Manually

Go to **Actions → Select workflow → Run workflow**

### Viewing Workflow Results

1. Go to **Actions** tab
2. Click on workflow run
3. View individual job logs
4. Download artifacts (JAR, build files)

### Checking Test Coverage

Coverage reports are available at:
- Codecov: `https://codecov.io/gh/your-org/lazydrop`
- Local: `apps/backend/target/site/jacoco/index.html`

### Viewing Code Quality

SonarQube dashboard:
- URL: `http://sonarqube:9000/projects`
- Project: `lazydrop-backend`

---

## Workflow Customization

### Modifying Test Commands

**Backend** (`.github/workflows/backend-test.yml`):
```yaml
- name: Run tests with Maven
  run: mvn clean test -B -q
```

**Frontend** (`.github/workflows/frontend-test.yml`):
```yaml
- name: Run tests
  run: npm test -- --run
```

### Adding New Steps

Example: Add Slack notification

```yaml
- name: Notify Slack
  uses: slackapi/slack-github-action@v1
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK }}
    payload: |
      {
        "text": "Tests passed! ✅"
      }
  if: success()
```

### Conditional Execution

```yaml
# Only on main branch
if: github.ref == 'refs/heads/main'

# Only on PR
if: github.event_name == 'pull_request'

# Always, even on failure
if: always()

# On specific file changes
if: |
  contains(github.event.head_commit.modified, 
  'apps/backend/')
```

---

## Troubleshooting

### Test Failures

**Backend tests failing:**
```bash
# Run locally to reproduce
mvn clean test

# Check PostgreSQL connectivity
mvn test -DskipTests=false

# View test output
mvn test -X
```

**Frontend tests failing:**
```bash
# Run locally
npm test -- --run

# Clear cache
rm -rf node_modules package-lock.json
npm ci
npm test
```

### Docker Build Failures

**Common issues:**
1. **Insufficient disk space:** Clean local Docker images
2. **Network timeout:** Check internet connection, increase timeout
3. **Credentials issue:** Verify `DOCKER_USERNAME` and `DOCKER_PASSWORD`

**Solution:**
```bash
# Clean Docker cache
docker system prune -a

# Check build logs
docker build --progress=plain -t test .
```

### Coverage Not Uploading

1. Verify Codecov token in GitHub secrets
2. Check codecov.io account settings
3. View CI logs for upload errors

---

## Performance Optimization

### Caching

Workflows use caching for faster builds:

**Maven:**
```yaml
cache: maven
```

**npm:**
```yaml
cache: 'npm'
cache-dependency-path: 'apps/frontend/package-lock.json'
```

### Parallel Execution

Jobs run in parallel by default. Speed up by:
- Running tests in parallel: `mvn test -T 1C`
- Using Node.js caching
- Using Docker layer caching

### Expected Times

- Backend tests: 3-5 minutes
- Frontend tests: 2-3 minutes
- Docker build: 3-5 minutes per image
- **Total PR check:** ~10-15 minutes

---

## Security Best Practices

### Secrets Management

✅ **Do:**
- Store secrets in GitHub Secrets
- Use specific, scoped secrets
- Rotate secrets regularly
- Use service accounts for CI/CD

❌ **Don't:**
- Commit secrets to repository
- Use personal access tokens
- Share credentials in logs
- Hardcode values

### Code Quality & Security

Workflows automatically:
- Scan dependencies for vulnerabilities
- Check code quality with SonarQube
- Scan Docker images with Trivy
- Enforce code coverage minimums

### Audit Trail

All workflow runs are logged with:
- Timestamp
- Triggered by (user/event)
- Changed files
- Test results
- Artifacts

---

## Integration with Development Workflow

### Before Committing

```bash
# Run local tests
./mvnw clean test          # Backend
cd apps/frontend && npm test -- --run  # Frontend

# Check code style
npm run lint

# Verify build
mvn clean package -DskipTests
npm run build
```

### Creating Pull Request

1. Create feature branch: `git checkout -b feature/my-feature`
2. Make changes and commit: `git commit -m "feat: ..."`
3. Push to remote: `git push origin feature/my-feature`
4. Open PR - workflows run automatically
5. Address any failures or comments
6. Once approved, merge to `develop`

### Merging to Production

Only merge to `main` from `develop`:
1. `git checkout main && git pull`
2. `git merge develop`
3. Tag release: `git tag v1.2.3`
4. Push: `git push origin main --tags`
5. Workflows deploy automatically

---

## Monitoring & Alerts

### Email Notifications

GitHub automatically notifies on:
- Workflow failure (first time per push)
- Successful workflow (if previously failed)

To customize: **Settings → Notifications**

### Slack Integration (Optional)

Add to workflow:
```yaml
- uses: slackapi/slack-github-action@v1.24.0
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK }}
```

### Dashboard Monitoring

Track metrics:
- Build success rate
- Test coverage trends
- Workflow execution time
- Deployment frequency

---

## Examples

### Only Run on Release Tags
```yaml
on:
  push:
    tags:
      - 'v*'
```

### Skip Workflow for Docs Changes
```yaml
on:
  push:
    branches: [main]
    paths-ignore:
      - 'docs/**'
      - '**.md'
```

### Run Tests with Different Java Versions
```yaml
strategy:
  matrix:
    java-version: ['17', '21']
```

---

## Useful Commands

```bash
# View workflow runs locally (using act)
act -l

# Run specific workflow
act -j test

# Debug workflow
act -v

# View all secrets
gh secret list

# Set a secret
gh secret set SECRET_NAME -b "secret_value"
```

---

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Maven Documentation](https://maven.apache.org/)
- [npm Documentation](https://docs.npmjs.com/)
- [Docker Documentation](https://docs.docker.com/)
- [Trivy Vulnerability Scanner](https://github.com/aquasecurity/trivy)
