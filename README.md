# XVORTEX

Personal financial intelligence platform: loan management, market monitoring, paper trading
(simulated only, never real orders), analytics and an optional AI assistant.

**Status: Phase 1 of 10 (architecture and database).** The frontend arrives in Phase 2.

## Repository layout

```
backend/            FastAPI app, SQLAlchemy models, Alembic migrations, tests
  app/core/         settings, structured JSON logging
  app/models/       one module per domain (user, loan, market, paper, system)
  app/services/     business logic (loan amortization and risk rules, pure and tested)
  alembic/          migrations (0001_initial creates every table)
render.yaml         Render Blueprint
.github/workflows/  CI (runs the backend tests)
```

## Local development

```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
cp ../.env.example .env        # then fill in DATABASE_URL and SECRET_KEY
alembic upgrade head
uvicorn app.main:app --reload  # http://localhost:8000/docs
pytest -q
```

## Deploy: GitHub, Neon, Render

1. Push this repository to a **private** GitHub repo. Never commit `.env`.
2. In Neon, create a project and copy the connection string (keep `sslmode=require`).
3. In Render choose New > Blueprint, select the repo; `render.yaml` creates the `xvortex-api` service.
   Paste the Neon string into the `DATABASE_URL` variable when prompted. `SECRET_KEY` is generated for you.
   The service runs `alembic upgrade head` on every start, then serves the API.
4. Check `https://<service>.onrender.com/healthz` returns `{"status":"ok"}` and `/readyz` returns `ready`.

### Custom domains (one.com DNS)

1. In Render > the service > Settings > Custom Domains, add `api.xvortex.se` (the frontend gets `app.xvortex.se` in Phase 2).
2. At one.com, in the DNS settings for xvortex.se, add a CNAME record for `api` pointing to the `*.onrender.com`
   hostname Render shows. Leave the existing `www` and root records unchanged.
3. Wait for Render to verify; it then issues the HTTPS certificate automatically.
   If you have CAA records, they must allow Let's Encrypt.

## Design notes

- Money, prices and quantities are `NUMERIC`/`Decimal`, never floats.
- Paper-trading rows carry `is_simulated`, enforced `TRUE` by a database check constraint.
- Loan calculation rules are documented at the top of `app/services/loans/amortization.py`.
- Risk indicators are transparent rule flags, not predictions.
- Sessions are stored server-side; only a hash of the cookie token is kept in the database.

## Roadmap

1 Architecture and database (this commit) · 2 Auth and design system · 3 Command Center · 4 Loans ·
5 Market intelligence · 6 Paper trading · 7 Analytics · 8 AI assistant · 9 Security and tests · 10 Final deploy
