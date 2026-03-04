# Contributing

Thanks for checking out LazyDrop.

This project is open-sourced primarily as a portfolio project, but contributions, bug reports, and suggestions are welcome.

---

## Quick Start (Local Dev)

```bash
git clone --recurse-submodules https://github.com/<your-username>/lazydrop
cd lazydrop
cp .env.example .env
# fill in required Supabase auth, DO Spaces, + Stripe test keys
docker compose up --build
```

---

## Repo Structure

This is an umbrella repository with submodules:

- `apps/frontend` (separate repo)
- `apps/backend` (separate repo)

Please open PRs against the appropriate submodule repository unless the change is docs-only.

---

## Submodule Notes (important)

If you change code inside `apps/frontend` or `apps/backend`:

1) Commit + push inside the submodule repo
2) Update the umbrella repo pointer

Example:

```bash
cd apps/frontend
git checkout main
git pull
# make changes
git add .
git commit -m "feat: ..."
git push

cd ../..
git add apps/frontend
git commit -m "chore: bump frontend submodule"
git push
```

---

## PR Guidelines

- Keep PRs focused and small
- Include a clear description and screenshots if UI changes
- Avoid committing secrets (`.env` is ignored)
- Use **test** keys for Stripe and a personal Supabase dev project
