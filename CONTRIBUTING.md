# Contributing

Thanks for checking out LazyDrop.

This project is open-sourced primarily as a portfolio project, but contributions, bug reports, and suggestions are welcome.

---

## Branch Protection

The `main` branch is protected on all three repositories (umbrella, backend, frontend):

- **No direct pushes to `main`** — all changes go through pull requests
- **Required reviews** — at least 1 approving review from a code owner before merge
- **Status checks must pass** — CI tests, lint, and build must be green
- **No force pushes** — history is immutable on `main`
- **CODEOWNERS enforced** — the maintainer is automatically requested for review

---

## How to Contribute

1. **Fork** the repository
2. **Create a branch** from `main` (e.g. `feat/your-feature` or `fix/your-bugfix`)
3. Make your changes and commit
4. **Open a pull request** against `main`
5. Wait for CI checks to pass and a maintainer review

```bash
# Fork on GitHub first, then:
git clone --recurse-submodules https://github.com/<your-username>/LazyDrop
cd LazyDrop
git checkout -b feat/my-change
# make changes, commit, push
git push origin feat/my-change
# open PR on GitHub
```

---

## Quick Start (Local Dev)

```bash
git clone --recurse-submodules https://github.com/<your-username>/LazyDrop
cd LazyDrop
cp .env.example .env
# fill in required Supabase auth, DO Spaces, + Stripe test keys
docker compose up --build
```

- Frontend: http://localhost:3000
- Backend: http://localhost:8080

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
- The PR template will guide you through the checklist
