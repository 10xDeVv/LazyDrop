# LazyDrop

[![Backend Tests](https://github.com/10xDeVv/LazyDrop/actions/workflows/backend-test.yml/badge.svg)](https://github.com/10xDeVv/LazyDrop/actions/workflows/backend-test.yml)
[![Frontend Tests](https://github.com/10xDeVv/LazyDrop/actions/workflows/frontend-test.yml/badge.svg)](https://github.com/10xDeVv/LazyDrop/actions/workflows/frontend-test.yml)
[![Docker Build](https://github.com/10xDeVv/LazyDrop/actions/workflows/docker-build.yml/badge.svg)](https://github.com/10xDeVv/LazyDrop/actions/workflows/docker-build.yml)
[![Qodana](https://github.com/10xDeVv/LazyDrop/actions/workflows/qodana_code_quality.yml/badge.svg)](https://github.com/10xDeVv/LazyDrop/actions/workflows/qodana_code_quality.yml)

LazyDrop is a real-time, session-based file sharing platform. Users create temporary "drop sessions", invite others via a short code (or QR), then upload files using secure signed URLs while everyone in the session receives live updates.

Live demo: https://lazydrop.app

This repository is an **umbrella repo** that provides documentation, diagrams, and a one-command local development setup. The app runs in production with:
- Frontend on Vercel
- Backend on DigitalOcean
- Supabase for Auth
- DigitalOcean Spaces for file storage (S3-compatible CDN)
- Stripe for subscriptions/payments

---

## Repositories

- `apps/frontend` → Next.js frontend (git submodule)
- `apps/backend` → Spring Boot backend (git submodule)

---

## Tech Stack

**Frontend**
- Next.js (App Router)
- React
- Supabase Auth (JWT)
- WebSockets client

**Backend**
- Java 21, Spring Boot 4
- Spring Security (OAuth2 Resource Server)
- Spring WebSocket (STOMP)
- PostgreSQL + Flyway
- Stripe Checkout + Webhooks (idempotent processing)

**Infra**
- Docker Compose local dev
- DigitalOcean Spaces CDN (S3 pre-signed URLs)
- Vercel + DigitalOcean production

---

## Run Locally (Docker)

Clone with submodules:

```bash
git clone --recurse-submodules https://github.com/<your-username>/lazydrop
cd lazydrop
```

Create your local env file:

```bash
cp .env.example .env
# fill in required Supabase auth, DO Spaces, + Stripe test keys
```

Start everything:

```bash
docker compose up --build
```

- Frontend: http://localhost:3000  
- Backend: http://localhost:8080  

---

## Documentation

- **[SYSTEM_DESIGN.md](./SYSTEM_DESIGN.md)** — architecture + flows (uploads, sessions, webhooks, scaling notes)
- **[ARCHITECTURE.md](./ARCHITECTURE.md)** — module breakdown + data model diagram
- **[SCALING.md](./SCALING.md)** — phased scaling roadmap (single node → Kubernetes)
- **[SECURITY.md](./SECURITY.md)** — security model and operational rules
- **[CONTRIBUTING.md](./CONTRIBUTING.md)** — PR workflow + local dev + submodules

---

## Key Engineering Highlights

- **Two-phase file upload** (signed URL generation + confirmation) to prevent orphaned records
- **STOMP WebSockets** for real-time session updates (joins/leaves, file events, notes)
- **Stripe webhook idempotency** backed by persistent event log (`stripe_webhook_events`)
- **Plan enforcement** server-side (limits can’t be bypassed by calling APIs directly)
- **Scheduled cleanup jobs** for expired sessions + unconfirmed uploads

---

## Testing & CI/CD

LazyDrop has comprehensive test coverage and automated CI/CD pipelines:

### Running Tests Locally

**Backend (Maven):**
```bash
cd apps/backend
mvn clean test                    # Run all tests
mvn test jacoco:report           # Generate coverage report
```

**Frontend (npm):**
```bash
cd apps/frontend
npm test -- --run                # Run all tests
npm run test:coverage            # Generate coverage report
npm run test:ui                  # Interactive test UI
```

### GitHub Actions Pipelines

Three automated workflows ensure code quality:

1. **Backend Tests & Code Quality** — Runs Maven tests, generates coverage, performs SonarQube analysis
2. **Frontend Tests & Build** — Runs Vitest, ESLint, and builds Next.js app
3. **Docker Build & Push** — Builds images, scans with Trivy, pushes to registry

**Status badges & reports:** Available in GitHub Actions tab

### Documentation

- **[TESTING.md](./TESTING.md)** — Comprehensive testing guide (unit, integration, coverage)
- **[CI_CD.md](./CI_CD.md)** — Pipeline setup, workflows, and troubleshooting

**Test Coverage:**
- Backend: 43 tests (6 test classes with mocks + TestContainers)
- Frontend: 24+ tests (components + utilities)
- Target: 80%+ coverage

---  
