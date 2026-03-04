# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| main    | ✅        |

## Reporting a Vulnerability

If you discover a security vulnerability, **please do NOT open a public issue**.

Instead, please report it privately:
- Email: [maintainer's email or GitHub Security Advisory]
- Or use GitHub's **private vulnerability reporting** feature on this repository.

We will acknowledge receipt within 48 hours and aim to release a fix within 7 days for critical issues.

## Security Practices

### Secrets Management
- **Never commit `.env` files** — they are listed in `.gitignore`.
- All secrets (Supabase, Stripe, DO Spaces, DB) are injected via environment variables.
- `.env.example` contains placeholder values only — no real keys.
- If you suspect any key was committed to git history, **rotate it immediately**.

### Authentication
- Backend validates Supabase JWTs using Spring Security OAuth2 Resource Server with HMAC signature verification.
- Guest users receive HttpOnly, Secure, SameSite=Lax cookies — no client-readable tokens.
- No session state on the server — fully stateless JWT validation.

### Stripe Webhooks
- All incoming Stripe webhooks are verified using `Webhook.constructEvent()` with the `STRIPE_WEBHOOK_SECRET`.
- Webhook events are stored in `stripe_webhook_events` with idempotency checks to prevent replay attacks.
- Signature verification must never be disabled.

### File Storage
- Files are uploaded directly to object storage via S3 pre-signed URLs with short expiry (5 minutes for upload, 1 hour for download). The backend never handles file bytes.
- All uploaded files are automatically deleted when a session ends.
- File size limits are enforced server-side per subscription plan.

### CORS
- CORS is restricted to the configured `CORS_ALLOWED_ORIGINS` value.
- WebSocket endpoints use the same origin restriction.

### Dependencies
- Dependencies are monitored for known CVEs.
- The project uses Dependabot / manual review for updates.

## For Contributors

Before submitting a PR, please ensure:
1. No secrets, API keys, or credentials are included in the diff.
2. No `.env` files are staged for commit.
3. Test data uses dummy/placeholder values only (e.g., `sk_test_example`, `whsec_test`).
