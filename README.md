# Stockwise — Inventory Intelligence

Stockwise is a responsive inventory-operations demonstration for retail teams. It includes a React + TypeScript dashboard, a FastAPI REST API, SQLAlchemy persistence, PostgreSQL, reproducible inventory audits, baseline stockout forecasts, and simulated replenishment decisions.

> **Demo data:** The seeded catalog is synthetic. The built-in WMS, POS, and commerce examples are mock adapters. No external API or supplier order is contacted.

## What works in the demo

- Ten navigation areas: overview, inventory, integrations, stock audit, forecasting, replenishment, alerts, locations, reports, and settings.
- Searchable/filterable inventory with pagination, product creation, deactivation, CSV export, browser persistence, and input validation.
- Dashboard KPI cards and charts derived from the same 100-product synthetic dataset.
- Reproducible source-snapshot reconciliation with matched, mismatched, missing, and stale records; audit history and review notes are persisted by the API.
- Explainable average-demand stockout baseline for 7, 14, or 30 days, without presenting it as an advanced ML model.
- Replenishment recommendations with approval/rejection recording; decisions do not create real supplier orders.
- In-app alert acknowledgement and resolution, mock source connection tests and sync history, location rollups, and CSV reports.
- JWT authentication, password hashing, role checks for write operations, and a first-user administrator bootstrap flow.
- SQLAlchemy tables and an Alembic initial migration for users, products, locations, inventory, snapshots, demand, integrations, sync jobs, audits, forecasts, recommendations, alerts, and activity history.

The browser demo stores its working inventory changes in local storage. API writes require an administrator token. Use the API setup below to exercise persistent workflows and the built-in Swagger UI.

## Run locally with Docker Compose

1. Copy `.env.example` to `.env`.
2. Replace `POSTGRES_PASSWORD` and `JWT_SECRET` with long random values. Do not commit `.env`.
3. From the project root, run:

   ```powershell
   docker compose up --build
   ```

4. Open:
   - Dashboard: <http://localhost:5173>
   - API docs: <http://localhost:8000/docs>
   - Health check: <http://localhost:8000/health>

The API applies the Alembic migration and seeds demo data when the database is empty. The PostgreSQL volume persists across restarts.

## Run without Docker

### Frontend

```powershell
cd frontend
npm ci
npm run dev
```

### Backend

Python 3.11+ is recommended. SQLite is the local default; set `DATABASE_URL` to use PostgreSQL.

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:DATABASE_URL = "sqlite:///./stockwise.db"
$env:JWT_SECRET = "replace-with-a-long-random-secret"
alembic upgrade head
uvicorn app.main:app --reload
```

The frontend dashboard runs independently on synthetic data and persists edits in the current browser. The backend API is separately available at port 8000; API documentation is generated at `/docs`.

## Authentication and roles

Create the first account with `POST /api/auth/register`. The first account is assigned the `admin` role. Later registrations receive `analyst`; only administrators can create or modify inventory and approve decisions. Obtain a bearer token with `POST /api/auth/token`, then use **Authorize** in Swagger UI.

Passwords must be at least 12 characters. Tokens expire after `TOKEN_MINUTES`. Always set a high-entropy `JWT_SECRET` outside local development. The application generates a process-local random key if the environment variable is omitted, which invalidates tokens on restart.

## REST API overview

| Area | Endpoint | Purpose |
|---|---|---|
| Auth | `POST /api/auth/register`, `POST /api/auth/token`, `GET /api/auth/me` | Bootstrap and authenticate accounts |
| Dashboard | `GET /api/dashboard` | KPI summary and risk counts |
| Inventory | `GET/POST /api/inventory`, `PATCH/DELETE /api/inventory/{id}` | Query and manage inventory |
| Locations | `GET /api/locations` | Physical/digital location rollups |
| Integrations | `GET/POST /api/integrations`, `POST /api/integrations/{id}/test`, `POST /api/integrations/{id}/sync` | Mock adapter management and sync records |
| Audit | `POST /api/audits`, `GET /api/audits`, `GET /api/audits/{id}/discrepancies`, `PATCH /api/discrepancies/{id}/review` | Reconciliation and review notes |
| Forecast | `GET /api/forecasts?horizon_days=7\|14\|30`, `POST /api/forecasts/run?horizon_days=...` | Explainable baseline forecast and persisted run |
| Replenishment | `POST /api/recommendations/generate`, `GET /api/recommendations`, `PATCH /api/recommendations/{id}` | Generate and decide simulated recommendations |
| Alerts | `GET /api/alerts`, `PATCH /api/alerts/{id}/acknowledge`, `PATCH /api/alerts/{id}/resolve` | Review generated alerts |
| Reports | `GET /api/reports/inventory.csv` | Download inventory CSV |
| Demo | `POST /api/demo/reset` | Restore synthetic data (administrator only) |

Audit source snapshots are retained; each run writes its own result rows. Discrepancy percentages are omitted when the comparison source quantity is zero. Records older than six hours are classified as stale. Tolerance is expressed in units.

## Forecasting and recommendations

The initial forecast is a transparent baseline, not an ML claim:

- Available stock = `max(0, on-hand − reserved)`.
- Days until stockout = `available stock ÷ average daily demand`.
- Forecast demand = `average daily demand × selected horizon`.
- Reorder point = `expected demand during supplier lead time + configured safety stock + a variability buffer when enough history exists`.
- Recommendation quantity = `max(0, target stock − available stock − confirmed incoming stock)`.

Synthetic demand history spans 60 days. The API calculates demand standard deviation and a planning interval only when at least 14 historical observations are available. The frontend demo forecast intentionally shows no uncertainty band. Results include model assumptions and do not place orders.

## Business impact

Stockwise demonstrates how a shared inventory view can help retail teams:

- **Reduce stockout exposure:** highlight products with limited days of stock so planners can prioritize review.
- **Improve inventory accuracy:** compare warehouse and point-of-sale snapshots and surface mismatches, missing records, and stale data for investigation.
- **Focus replenishment work:** suggest quantities based on available stock, demand, lead time, and safety stock; approvals remain under team control.
- **Support faster decisions:** summarize inventory, risk, and audit results in one dashboard, with CSV exports for follow-up analysis.
- **Create an audit trail:** record audit runs, review notes, forecast runs, and recommendation decisions through the API.

These are intended operational benefits, not measured results. The project is a demonstration with synthetic data and mock integrations; it has not established realized cost savings, forecast accuracy, or reductions in stockouts. A production rollout should measure outcomes against a baseline, for example stockout rate, inventory-record accuracy, excess stock, replenishment lead time, and time spent resolving discrepancies.

## Future scope

Potential next steps for evolving the demonstration into a production-ready inventory platform include:

1. **Connect real systems:** implement authenticated WMS, POS, and commerce adapters, with scheduled synchronization, retry handling, and source-health monitoring.
2. **Improve forecasting:** use validated historical sales and seasonality, account for promotions and supply constraints, compare candidate forecasting methods against a holdout period, and expose accuracy metrics and uncertainty.
3. **Strengthen replenishment planning:** include confirmed purchase orders, minimum order quantities, pack sizes, supplier constraints, and review/approval workflows before any order is submitted.
4. **Expand operational workflows:** add configurable thresholds, role-based dashboards, notifications, and richer discrepancy assignment and resolution.
5. **Prepare for scale and production:** add integration and end-to-end testing, observability, rate limiting, backup and recovery procedures, and deployment-specific security controls.

Each step should be validated with representative business data and measured against agreed operational KPIs before relying on recommendations for purchasing decisions.

## Tests

```powershell
cd backend
pip install -r requirements.txt
pytest
```

```powershell
cd frontend
npm ci
npm run build
```

## Deployment notes

- Build the frontend as a static Vite bundle for Vercel; set environment-specific API configuration and allow only the deployed origin in `CORS_ORIGINS`.
- Host the FastAPI container on a Python container platform and use a managed PostgreSQL database.
- Provide `DATABASE_URL`, `JWT_SECRET`, and `CORS_ORIGINS` through the host's secret/configuration service. Never put API credentials in frontend environment variables.
- Implement each real source as a backend adapter that retrieves credentials by secret reference, normalizes into the inventory schema, and persists source snapshots. The included integrations are mock-only and explicitly report that no external system was contacted.
- Enable TLS, restrict network access to PostgreSQL, add a production rate limiter, configure centralized logs/monitoring, and use managed backups before handling real operational data.

## Demo walkthrough

1. Open the dashboard and review the KPI cards and products-to-watch table.
2. Search `SKU-41001` from Inventory, filter by category/location, and export a CSV.
3. Add a product using a new SKU and non-negative on-hand/reorder values. Refresh; the browser retains the edit.
4. Run a Stock audit. The synthetic sources include matching records, deliberate count variances, missing POS rows, and stale snapshots.
5. In Forecasting, switch among 7, 14, and 30 days and inspect the baseline assumptions.
6. Approve or reject a Replenishment recommendation and observe the recorded simulated decision.
7. Acknowledge an alert and review the mock integration status. Reset demo data in Settings when finished.
