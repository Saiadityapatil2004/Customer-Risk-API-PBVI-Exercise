# CLAUDE.md — Customer Risk API

## What this system does

Read-only HTTP API that returns a customer's pre-assessed risk tier (`LOW` / `MEDIUM` / `HIGH`) and contributing risk factors from a Postgres database. Exposes a browser UI for operations staff. **Does not compute risk** — it surfaces values that already exist in the database.

---

## Architecture

Two containers only (`docker compose up`, no manual setup beyond `.env`):
```
Browser → GET /          → FastAPI (Jinja2 renders UI, injects API_KEY at render time)
          GET /api/risk  → FastAPI (validates key + input, queries Postgres, returns JSON)
          GET /health    → FastAPI (live DB ping, no auth)
                                  ↕
                             Postgres 15
                          (init.sql seeds on first boot)
```

- **Auth:** `X-API-Key` header checked by a FastAPI dependency on all `/api/*` routes — fires before any DB call.
- **API key in UI:** Injected server-side via Jinja2 (`{{ api_key }}`); `Cache-Control: no-store` required on every `GET /` response.
- **DB access:** psycopg2, parameterised queries only (`%s` + tuple). No ORM. No string interpolation in SQL — ever.
- **Startup:** `lifespan` handler retries DB connection (exponential backoff, 10 attempts). `depends_on: service_healthy` using `pg_isready`. No traffic accepted until connection pool is live.
- **Error handling:** Explicit exception handlers intercept `psycopg2.OperationalError` and `RuntimeError`; return 503 with sanitised message. `HTTPException` must be re-raised inside the catch-all.

---

## Non-negotiable invariants

| # | Rule |
|---|------|
| INV-1.2 | No code path may derive, calculate, or recalculate a risk value. Retrieval only. |
| INV-1.3 | `CHECK (risk_tier IN ('LOW','MEDIUM','HIGH'))` must exist at DB level — not only in app code. |
| INV-1.5 | All DB calls are `SELECT` only. No `INSERT`, `UPDATE`, `DELETE`, or DDL. |
| INV-2.6 | Status codes are exhaustive and non-overlapping: 200 / 401 / 404 / 422 / 503. Returning 500 for any listed condition is a violation. |
| INV-2.7 | Postgres error text, table names, connection strings, and stack traces must never appear in any response body. |
| INV-3.3 | `templates/index.html` must contain `{{ api_key }}` — never a resolved key value. |
| INV-3.4 | `Cache-Control: no-store` is a required security control on `GET /`, not optional polish. |
| INV-4.1 | Parameterised queries everywhere. One f-string in a SQL call is a SQL injection vulnerability. |
| INV-4.2 | `Query(pattern="^[a-zA-Z0-9_-]{1,64}$")`, `VARCHAR(64)`, and `maxlength="64"` must all agree. Change one, change all three. |
| INV-5.2 | Exactly two services in compose. A third container may bypass auth controls. |
| INV-5.3 | A broken Jinja2 template must log an error but must not crash the app or degrade `/api/risk`. |

---

## Known limitations (do not mistake for bugs)

- **Shared API key provides no per-consumer attribution.** All requests are indistinguishable in logs. Key rotation requires updating `.env` and running `docker compose up --force-recreate`.
- **No TLS.** Do not deploy on a public network.
- **`Cache-Control: no-store` can be defeated by a caching proxy that ignores it.** Confirm the deployment environment has no such proxy before handing off.
- **Empty `risk_factors = []` is valid and expected** for LOW-tier customers — treat it as a normal 200, not a data error.