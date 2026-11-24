# XVORTEX

Personal financial intelligence platform: loan management, market monitoring, paper trading
(simulated only, never real orders), analytics and an optional AI assistant.

**Status: phase 2 of 10 (authentication and design system).** Modules arrive in phases 3 to 8; until then
their pages say so and show no sample data.

## Repository layout

```
backend/            FastAPI app, SQLAlchemy models, Alembic migrations, tests
  app/api/          routers (auth, admin), dependencies (session, CSRF, admin checks)
  app/core/         settings, Argon2 helpers, rate limiter, JSON logging
  app/models/       one module per domain
  app/services/     business logic (loan calculations, audit, admin bootstrap)
frontend/           React + TypeScript + Vite + Tailwind + Framer Motion
  src/components/   layout shell (sidebar, top bar) and UI pieces
  src/pages/        sign-in, users admin, module placeholders
render.yaml         Render Blueprint (API + static frontend)
.github/workflows/  CI
```

## Security model (phase 2)

- No public registration: an administrator creates accounts (Users page, or `ADMIN_EMAIL`/`ADMIN_PASSWORD` for the first one).
- Passwords are hashed with Argon2id; minimum 12 characters.
- Login creates a server-side session. The browser only holds a random token in an `HttpOnly`, `Secure`
  (production), `SameSite=Lax` cookie; the database stores only its SHA-256 hash. Sessions expire after
  `SESSION_TTL_MINUTES` (default 12 hours).
- State-changing requests need the `X-CSRF-Token` header that matches the session.
- Login is rate limited per IP (20 attempts / 5 min) and per account (5 failures / 15 min). The limiter is
  in memory, which is fine for one Render instance; use a shared store before scaling out.
- Unknown email and wrong password return the same message and take similar time.
- Logins, failures, logouts and user changes are written to `audit_logs`.
- Every private route is attached via `PRIVATE` in `app/api/deps.py`; admin routes add `require_admin`.

## Local development

```bash
# backend
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
cp ../.env.example .env      # set DATABASE_URL, SECRET_KEY, ADMIN_EMAIL, ADMIN_PASSWORD
alembic upgrade head
uvicorn app.main:app --reload
pytest -q

# frontend (second terminal)
cd frontend
npm install
npm run dev                  # http://localhost:5173, /api is proxied to localhost:8000
```

Commit `frontend/package-lock.json` after the first `npm install`.

## Deploy: GitHub, Neon, Render

1. Push to a **private** GitHub repo. Never commit `.env`.
2. In Neon, create a project and copy the connection string (keep `sslmode=require`).
3. In Render choose New > Blueprint and select the repo. It creates `xvortex-api` and `xvortex-app`.
   When prompted, paste `DATABASE_URL`, `ADMIN_EMAIL` and `ADMIN_PASSWORD` (12+ characters).
4. After the first successful sign-in, **delete `ADMIN_PASSWORD`** from the API service's environment.
5. Check `https://<api-service>.onrender.com/healthz` returns `{"status":"ok"}`.

### Custom domains (one.com DNS)

| Service | Custom domain | one.com record |
|---|---|---|
| `xvortex-app` | `app.xvortex.se` | CNAME `app` to the `*.onrender.com` host Render shows |
| `xvortex-api` | `api.xvortex.se` | CNAME `api` to the `*.onrender.com` host Render shows |

Add each domain in Render (Settings > Custom Domains), then create the matching CNAME at one.com and leave
`www` and the root record alone. Render issues HTTPS certificates after verification. If you use CAA
records, they must allow Let's Encrypt. Both hosts share `xvortex.se`, which is what lets the session
cookie work between the app and the API.

## Design notes

- Money, prices and quantities are `NUMERIC`/`Decimal`, never floats.
- Paper-trading rows carry `is_simulated`, enforced `TRUE` by a database check constraint.
- Loan rules are documented in `backend/app/services/loans/amortization.py`; risk flags are rule-based, not predictions.
- Theme (dark or light) is stored in the browser for now; per-user settings sync arrives with the settings API.

## Roadmap

1 Architecture and database · **2 Auth and design system (this commit)** · 3 Command Center · 4 Loans ·
5 Market intelligence · 6 Paper trading · 7 Analytics · 8 AI assistant · 9 Security and tests · 10 Final deploy
