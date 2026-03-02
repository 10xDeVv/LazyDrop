# LazyDrop

LazyDrop is a real-time, session-based file sharing platform. Users create temporary “drop sessions”, invite others via a short code (or QR), then upload files using secure signed URLs while everyone in the session receives live updates.

Live demo: https://lazydrop.app

This repository is an **umbrella repo** that provides documentation, diagrams, and a one-command local development setup. The app runs in production with:
- Frontend on Vercel
- Backend on DigitalOcean
- Supabase for Auth + Storage
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
- Supabase Storage signed URLs
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
# fill in required Supabase + Stripe test keys
```

Start everything:

```bash
docker compose up --build
```

- Frontend: http://localhost:3000  
- Backend: http://localhost:8080  
- API base: http://localhost:8080/api/v1  

---

## Documentation

- **SYSTEM_DESIGN.md** — architecture + flows (uploads, sessions, webhooks, scaling notes)
- **ARCHITECTURE.md** — module breakdown + data model diagram
- **SECURITY.md** — security model and operational rules
- **CONTRIBUTING.md** — PR workflow + local dev + submodules

---

## Key Engineering Highlights

- **Two-phase file upload** (signed URL generation + confirmation) to prevent orphaned records
- **STOMP WebSockets** for real-time session updates (joins/leaves, file events, notes)
- **Stripe webhook idempotency** backed by persistent event log (`stripe_webhook_events`)
- **Plan enforcement** server-side (limits can’t be bypassed by calling APIs directly)
- **Scheduled cleanup jobs** for expired sessions + unconfirmed uploads

---

## License

MIT
