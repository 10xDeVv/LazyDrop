# Security Policy

## Never commit secrets
- Do not commit `.env` files
- Do not paste real API keys into issues or PRs
- Rotate any exposed keys immediately (Supabase / Stripe)

## Stripe webhooks
- All webhooks must be verified using `STRIPE_WEBHOOK_SECRET`
- Do not disable signature verification

## Auth
- Backend validates Supabase JWTs using Spring Security OAuth2 Resource Server
- Avoid logging sensitive tokens or secrets

## Reporting
If you discover a security issue, please open a private report by contacting the maintainer.
