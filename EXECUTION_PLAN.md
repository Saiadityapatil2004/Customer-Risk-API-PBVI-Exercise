# Customer Risk API — Execution Plan

> Based on ARCHITECTURE.md v0.1 · INVARIANTS.md v0.2 · Requirements Brief v1.0  
> Stack: Docker Compose · FastAPI (Python 3.11) · Postgres · Vanilla JS · psycopg2

---

## Overview

| | |
|---|---|
| **Sessions** | 5 sessions · 24 tasks |
| **Est. Total** | ≈ 855 – 1,190 min (14 – 20 hrs across sessions) |
| **Timing basis** | 28–40 min per standard task · 33–50 min per invariant-touching task (excludes manual verification time) |

Tasks marked **⚠ invariant-touching** carry a dedicated code review checklist and use the longer time estimate.

---

## Session Overview

| Session | Title | Est. (min) | Tasks | Delivers |
|---|---|---|---|---|
| S1 | Repository Scaffold & Compose Topology | 130–180 | 4 | `docker compose up` succeeds; both containers healthy |
| S2 | Database Schema, Seed Data & DB Layer | 185–270 | 5 | All 12 seed records queryable; parameterised query layer validated |
| S3 | FastAPI Core — Auth, Risk Endpoint & Error Handling | 225–315 | 6 | Full status-code matrix verified via curl; no credential leaks |
| S4 | Health Endpoint, Startup Checks & Observability | 175–250 | 5 | Health endpoint reflects live DB state; logs contain customer_id |
| S5 | Browser UI, Integration & Documentation | 140–175 | 4 | Full end-to-end flow from browser; README complete |

---

## Invariant Coverage Map

> Red = invariant-touching tasks. Every invariant in INVARIANTS.md v0.2 is covered by at least one task.

| Invariant | Statement (summary) | Covered by |
|---|---|---|
| INV-1.1 | Passthrough fidelity — no alteration of retrieved values | T2.3, T3.3 |
| INV-1.2 | No risk computation anywhere in the codebase | T3.3 (code review) |
| INV-1.3 | Valid tier values enforced at DB level via CHECK constraint | T2.1 |
| INV-1.4 | Empty risk_factors list is a valid 200 response | T2.2, T3.3 |
| INV-1.5 | No write operations — SELECT only | T2.3 (code review) |
| INV-2.1 | Required fields: customer_id, risk_tier, risk_factors | T3.3 |
| INV-2.2 | No extra fields (created_at, PII) in response | T3.3 |
| INV-2.3 | Existence mapping — 200 vs 404 never conflated | T3.3, T3.4 |
| INV-2.4 | Malformed input → 422, no DB call | T3.2 |
| INV-2.5 | DB unavailable → 503, not 404 or 200 | T3.4 |
| INV-2.6 | Complete non-overlapping status code mapping | T3.2, T3.3, T3.4 |
| INV-2.7 | Sanitised error messages — no internal details in response | T3.4 |
| INV-3.1 | API key required on all /api/* routes | T3.1 |
| INV-3.2 | API key never in responses, logs, or error messages | T3.1, T4.3 |
| INV-3.3 | API key not literal in repo — Jinja2 placeholder only | T3.5 |
| INV-3.4 | Cache-Control: no-store on GET / | T5.1 |
| INV-4.1 | Parameterised queries only — no string interpolation | T2.3 |
| INV-4.2 | Validator, schema, and UI field width all consistent | T2.1, T3.2, T5.1 |
| INV-5.1 | No external network calls | T4.5 (code review) |
| INV-5.2 | Exactly two services in compose | T1.3 |
| INV-5.3 | UI failure does not degrade API | T4.2 |
| INV-6.1 | No traffic before DB connection established | T1.4, T4.1 |
| INV-6.2 | Seed data present before service accepts traffic | T2.2, T4.1 |
| INV-7.1 | Health endpoint reflects live DB state | T4.1 |
| INV-7.2 | Logs record customer_id and timestamp per request | T4.3 |
| INV-8.1 | UI displays API data without modification | T5.1 |
| INV-8.2 | UI displays errors visibly and distinctly | T5.1 |

---

---

# SESSION 1: Repository Scaffold & Compose Topology

**Est: 130–180 min**

**Integration check:** `docker compose up` completes with both containers healthy; `curl http://localhost:8000/` returns a non-500 response.

---

## T1.1 — Initialise repository structure

**28–35 min**

**Description:** Create the project directory tree with all placeholder files. No code is written yet — only the skeleton that Claude Code will fill in subsequent tasks.

- **Inputs:** Empty working directory
- **Outputs:** Directory tree: `app/`, `db/`, `templates/`, `.env.example`, `docker-compose.yml` (stub), `Dockerfile` (stub), `requirements.txt` (stub), `.gitignore`

**Prompt for Claude Code:**

```
Create the following project directory structure for the Customer Risk API.
No implementation code yet — only placeholder files with correct names.

customer-risk-api/
  app/
    __init__.py          (empty)
    main.py              (# TODO: FastAPI app)
    db.py                (# TODO: psycopg2 connection pool)
    dependencies.py      (# TODO: API key dependency)
  db/
    init.sql             (-- TODO: schema + seed)
  templates/
    index.html           (<!-- TODO: UI -->)
  .env.example           (API_KEY=changeme\nDATABASE_URL=...)
  .gitignore             (include .env, __pycache__, *.pyc)
  docker-compose.yml     (# TODO)
  Dockerfile             (# TODO)
  requirements.txt       (# TODO)
  README.md              (# placeholder)

After creating the files, run: find . -type f | sort
and confirm all paths are present.
```

**Test cases:**

1. `find . -type f | sort` lists all 12 expected paths → All paths present, no extras
2. `.gitignore` contains `.env` entry → `.env` is ignored
3. `.env.example` does not contain a real key value → Shows placeholder string only

**Verification command:**

```bash
find customer-risk-api -type f | sort
```

---

## T1.2 — Write requirements.txt and Dockerfile

**28–35 min**

**Description:** Define Python dependencies and the application Dockerfile. The image must use Python 3.11-slim, install dependencies, and set the correct CMD.

- **Inputs:** Stub `Dockerfile` and `requirements.txt` from T1.1
- **Outputs:** `requirements.txt` with pinned versions; `Dockerfile` that builds and runs the FastAPI app on port 8000

**Prompt for Claude Code:**

```
Write requirements.txt and Dockerfile for the FastAPI application.

requirements.txt must include (pin to specific versions):
- fastapi==0.111.0
- uvicorn[standard]==0.30.1
- psycopg2-binary==2.9.9
- jinja2==3.1.4
- python-dotenv==1.0.1

Dockerfile requirements:
- Base image: python:3.11-slim
- WORKDIR /app
- Copy and install requirements first (for layer caching)
- Copy app code
- Copy templates/ directory
- Expose port 8000
- CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

After writing, run: docker build -t risk-api-test . && echo "BUILD OK"
Fix any build errors before finishing.
```

**Test cases:**

1. `docker build` succeeds → Exit 0, BUILD OK printed
2. `requirements.txt` has no floating versions → All versions pinned
3. Dockerfile COPY order puts requirements before app code → Layer cache optimised

**Verification command:**

```bash
docker build -t risk-api-test . 2>&1 | tail -5
```

---

## T1.3 — Write docker-compose.yml with health check ⚠

**40–50 min · Invariants: INV-5.2, INV-6.1**

**Description:** Define the two-service compose file. The Postgres service must include a `pg_isready` health check. The FastAPI service must use `depends_on` with `service_healthy` condition. No third service.

- **Inputs:** `docker-compose.yml` stub from T1.1; `.env.example`
- **Outputs:** `docker-compose.yml` with exactly two services, health check on `db`, `depends_on` with `service_healthy` on `api`

**Prompt for Claude Code:**

```
Write docker-compose.yml for the Customer Risk API.

Requirements:
- Exactly two services: 'db' (Postgres 15) and 'api' (FastAPI)
- db service:
    image: postgres:15
    environment loaded from .env: POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD
    volume mount: ./db/init.sql to /docker-entrypoint-initdb.d/init.sql
    healthcheck: test pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
    healthcheck interval: 5s, timeout: 5s, retries: 10
    restart: unless-stopped
- api service:
    build: .
    ports: "8000:8000"
    env_file: .env
    depends_on: db with condition: service_healthy
    restart: unless-stopped

Also update .env.example to include:
  API_KEY=changeme
  DATABASE_URL=postgresql://riskuser:riskpass@db:5432/riskdb
  POSTGRES_DB=riskdb
  POSTGRES_USER=riskuser
  POSTGRES_PASSWORD=riskpass

Do NOT add any third service (no nginx, no redis, no proxy).
After writing, run: docker compose config --quiet && echo "COMPOSE VALID"
```

**Test cases:**

1. `docker compose config --quiet` succeeds → COMPOSE VALID printed
2. Service count is exactly 2 → Only `api` and `db` defined
3. Healthcheck uses `pg_isready` → Grep confirms in compose output
4. `depends_on` condition is `service_healthy` → Not `service_started`

**Verification command:**

```bash
docker compose config --quiet && echo "COMPOSE VALID"
docker compose config | grep -E "services:|condition:"
```

**Code review — what to look for:**

- Confirm exactly two top-level services — any third service violates INV-5.2
- Confirm `depends_on` uses `condition: service_healthy` not `service_started` — INV-6.1 requires this
- Confirm no external port bindings on the `db` service beyond what Postgres needs internally
- Confirm `restart: unless-stopped` is present on both services

---

## T1.4 — Write stub FastAPI main.py with lifespan handler ⚠

**33–45 min · Invariants: INV-6.1**

**Description:** Write a minimal FastAPI app that boots, attempts a DB connection in the lifespan handler with exponential backoff retry, and exposes a stub `GET /` that returns 200. The app must not serve traffic until the DB connection succeeds.

- **Inputs:** `app/main.py` stub from T1.1; `docker-compose.yml` from T1.3
- **Outputs:** Bootable FastAPI app; `docker compose up` brings both containers healthy; `GET /` returns 200

**Prompt for Claude Code:**

```
Write app/main.py as a minimal FastAPI application with a lifespan startup handler.

Requirements:
1. Use FastAPI's lifespan context manager (not deprecated on_event).
2. In the lifespan startup phase, attempt to connect to Postgres using the
   DATABASE_URL environment variable. Retry up to 10 times with exponential
   backoff starting at 1 second, doubling each attempt, capped at 16 seconds.
   Log each retry attempt. If all retries fail, raise an exception so the
   container exits with a non-zero code.
3. Store the psycopg2 connection or connection details in app state
   (app.state.db_conn or similar) for use by route handlers.
4. Add a stub route: GET / that returns {"status": "ok"} as JSON for now
   (the real UI will replace this in Session 5).
5. Do NOT import db.py or dependencies.py yet — keep this minimal.

Also create a .env file by copying .env.example:
  cp .env.example .env

Then run: docker compose up -d && sleep 15 && curl -sf http://localhost:8000/ && echo "BOOT OK"
If the boot fails, check logs: docker compose logs api
```

**Test cases:**

1. `docker compose up -d` exits 0 → Stack starts without error
2. `docker compose ps` shows both containers healthy/running → Health checks pass
3. `curl http://localhost:8000/` returns 200 → App accepts traffic after DB is ready
4. `docker compose logs api` shows retry log messages on first boot → Startup retry loop executed

**Verification command:**

```bash
docker compose up -d && sleep 20
docker compose ps
curl -sf http://localhost:8000/ && echo "ROUTE OK"
docker compose logs api | grep -i "retry\|connect\|ready"
```

**Code review — what to look for:**

- Confirm lifespan handler raises (not just logs) on final retry failure — INV-6.1 requires the container to not accept traffic if DB never connects
- Confirm no database call happens at module import time — only inside lifespan
- Confirm the retry loop has a finite ceiling (not infinite) to avoid masking real errors

---

---

# SESSION 2: Database Schema, Seed Data & DB Layer

**Est: 185–270 min**

**Integration check:** `SELECT * FROM customers` returns all 12 seed rows; calling `get_customer('CUST-LOW-001')` returns a correct dict; calling `get_customer` with `'bad id!'` is rejected before reaching the DB.

---

## T2.1 — Write init.sql — schema with CHECK constraint and seed data ⚠

**40–50 min · Invariants: INV-1.3, INV-4.2, INV-6.2**

**Description:** Write the Postgres initialisation SQL: `CREATE TABLE` with the correct types and a `CHECK` constraint on `risk_tier`, then `INSERT` 12 representative seed records covering all three tiers including one customer with an empty `risk_factors` array.

- **Inputs:** `db/init.sql` stub from T1.1; ARCHITECTURE.md §6 data model
- **Outputs:** `init.sql` that creates the `customers` table with `CHECK` constraint and seeds 12 rows (3 LOW, 4 MEDIUM, 4 HIGH, 1 LOW with empty factors array)

**Prompt for Claude Code:**

```
Write db/init.sql to initialise the Postgres database for the Customer Risk API.

Schema requirements:
CREATE TABLE customers (
  customer_id  VARCHAR(64) PRIMARY KEY,
  risk_tier    VARCHAR(10) NOT NULL CHECK (risk_tier IN ('LOW','MEDIUM','HIGH')),
  risk_factors TEXT[]      NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

Seed data requirements (minimum 12 rows):
- 3 LOW rows: one must have risk_factors = '{}' (empty array), one with '{"verified_identity"}'
- 4 MEDIUM rows: use factors like 'new_account', 'moderate_transaction_volume', 'recent_address_change'
- 4 HIGH rows: use factors like 'high_transaction_volume', 'geographic_flag', 'unverified_identity'
- 1 additional edge-case LOW row with a long customer_id (exactly 64 chars of alphanumeric)

Customer ID format: alphanumeric, hyphens, underscores only; max 64 chars.
Use IDs like: CUST-LOW-001, CUST-MED-001, CUST-HIGH-001, etc.

After writing, tear down and recreate the stack to load the new schema:
docker compose down -v && docker compose up -d && sleep 20
Then verify:
docker compose exec db psql -U riskuser -d riskdb \
  -c "SELECT customer_id, risk_tier, array_length(risk_factors,1) FROM customers ORDER BY risk_tier;"
```

**Test cases:**

1. Table exists with correct columns → `psql \d customers` shows all 4 columns
2. CHECK constraint rejects invalid tier → `INSERT` with `tier='INVALID'` raises error
3. 12 seed rows present → `SELECT COUNT(*) FROM customers` returns 12
4. All three tiers present → `SELECT DISTINCT risk_tier` returns LOW, MEDIUM, HIGH
5. Empty `risk_factors` row exists → `WHERE array_length(risk_factors,1) IS NULL` returns 1 row

**Verification command:**

```bash
docker compose exec db psql -U riskuser -d riskdb -c "SELECT COUNT(*) FROM customers;"
docker compose exec db psql -U riskuser -d riskdb -c "SELECT DISTINCT risk_tier FROM customers ORDER BY risk_tier;"
docker compose exec db psql -U riskuser -d riskdb -c "SELECT customer_id FROM customers WHERE risk_factors = '{}';"
docker compose exec db psql -U riskuser -d riskdb \
  -c "INSERT INTO customers(customer_id,risk_tier,risk_factors) VALUES('test','INVALID','{}');" \
  2>&1 | grep -i "check\|violat"
```

**Code review — what to look for:**

- Confirm `CHECK` constraint is in the `CREATE TABLE` DDL, not enforced only in application code — INV-1.3 requires database-level enforcement
- Confirm `VARCHAR(64)` on `customer_id` — this must match the validator in T3.2 and the UI field `maxlength` in T5.1 per INV-4.2
- Confirm empty array `{}` is explicitly seeded — INV-6.2 requires seed data to cover edge cases including INV-1.4
- Confirm no INSERT/UPDATE triggers that compute or transform risk values — INV-1.2 must hold at the DB layer too

---

## T2.2 — Write db.py — connection pool initialisation ⚠

**33–45 min · Invariants: INV-6.2**

**Description:** Extract the database connection logic into `db.py`. Expose a function that creates a psycopg2 connection with retry logic, a context-manager cursor helper, and a `seed_check()` that warns if the table is empty.

- **Inputs:** `app/db.py` stub; `DATABASE_URL` env var from `.env`
- **Outputs:** `db.py` with `get_connection()`, cursor helper, `init_db()`, and `seed_check()`; connection established at startup

**Prompt for Claude Code:**

```
Write app/db.py to manage the psycopg2 database connection.

Requirements:
1. Function: create_connection(database_url: str, max_retries=10) -> psycopg2 connection
   - Retry with exponential backoff (1s, 2s, 4s... capped at 16s)
   - Log each attempt
   - Raise RuntimeError after all retries exhausted
2. A module-level _conn variable set to None initially.
3. Function: init_db(database_url: str) -> None — calls create_connection and stores in _conn
4. Function: get_cursor() — returns a psycopg2 cursor from _conn; raises RuntimeError with
   message "Database unavailable" if _conn is None or closed.
5. A simple seed_check() function that runs:
     SELECT COUNT(*) FROM customers
   and returns the count as an integer. If count is 0, log a WARNING:
   "Database seeded with 0 rows — all lookups will return 404."

Update app/main.py lifespan handler to:
  - call db.init_db(os.environ['DATABASE_URL']) instead of inline connection code
  - call db.seed_check() and log the result

Test by restarting the stack and checking logs:
docker compose restart api && sleep 10 && docker compose logs api | tail -20
```

**Test cases:**

1. `docker compose logs api` shows seed count > 0 → `seed_check()` runs and logs row count
2. If DB is down, `get_cursor()` raises `RuntimeError` not psycopg2 error → Error is application-level, not DB internals
3. Connection not attempted at module import → No DB call on import

**Verification command:**

```bash
docker compose restart api && sleep 10
docker compose logs api | tail -20
docker compose logs api | grep -i "seed\|rows\|connect"
```

**Code review — what to look for:**

- Confirm `seed_check()` result is logged at WARNING if count is 0 — INV-6.2 requires detectable failure if seed script didn't run
- Confirm `get_cursor()` raises a generic `RuntimeError` (not `psycopg2.OperationalError`) — the exception handler in T3.4 will catch `RuntimeError` and return 503; raw psycopg2 errors must never reach the response body per INV-2.7

---

## T2.3 — Write the customer query function ⚠

**40–50 min · Invariants: INV-1.1, INV-1.2, INV-1.5, INV-4.1**

**Description:** Implement `get_customer(customer_id)` in `db.py` using a parameterised psycopg2 query. The function returns a dict with exactly the three response fields or `None` if not found. No string interpolation in the query string.

- **Inputs:** `app/db.py` from T2.2; seed data from T2.1
- **Outputs:** `get_customer()` function returning `{'customer_id': ..., 'risk_tier': ..., 'risk_factors': [...]}` or `None`; confirmed with direct Python test

**Prompt for Claude Code:**

```
Add a get_customer function to app/db.py.

Function signature: get_customer(customer_id: str) -> dict | None

Requirements:
1. Use ONLY parameterised psycopg2 syntax:
   cursor.execute(
     "SELECT customer_id, risk_tier, risk_factors FROM customers WHERE customer_id = %s",
     (customer_id,)
   )
   NEVER concatenate or f-string the customer_id into the query string.
2. Return a dict with exactly these keys: customer_id, risk_tier, risk_factors
   risk_factors must be a Python list (psycopg2 returns TEXT[] as a list automatically).
3. Return None if no row is found.
4. SELECT only the three columns needed for the response. Do NOT select created_at or *.
5. Do NOT perform any INSERT, UPDATE, DELETE, or DDL. SELECT only.

After writing, test directly from within the api container:
docker compose exec api python3 -c "
from app.db import init_db, get_customer
import os
init_db(os.environ['DATABASE_URL'])
print(get_customer('CUST-LOW-001'))
print(get_customer('NONEXISTENT-999'))
print(type(get_customer('CUST-LOW-001')['risk_factors']))
"
```

**Test cases:**

1. `get_customer('CUST-LOW-001')` returns dict with 3 keys → `{'customer_id': ..., 'risk_tier': 'LOW', 'risk_factors': [...]}`
2. `get_customer('NONEXISTENT-999')` returns `None` → `None` printed
3. `risk_factors` field is a Python list → `type()` shows `<class 'list'>`
4. Customer with empty factors returns empty list `[]` → `risk_factors = []`
5. No `%s` substitution appears in query string at runtime → Code review — no f-string or `%` format

**Verification command:**

```bash
docker compose exec api python3 -c "
from app.db import init_db, get_customer
import os
init_db(os.environ['DATABASE_URL'])
result = get_customer('CUST-LOW-001')
print('Keys:', list(result.keys()))
print('Tier:', result['risk_tier'])
print('Factors type:', type(result['risk_factors']))
print('Missing:', get_customer('NONEXISTENT-999'))
"
```

**Code review — what to look for:**

- Read the query string literally — confirm `%s` placeholder with tuple parameter, never an f-string or `.format()` — INV-4.1 is violated by any interpolation regardless of how "safe" it seems
- Confirm `SELECT` lists only `customer_id, risk_tier, risk_factors` — not `SELECT *` and not `created_at` — INV-2.2 requires no extra fields
- Confirm no `INSERT`/`UPDATE`/`DELETE` anywhere in `db.py` — INV-1.5
- Confirm the function returns exactly the database values with no transformation — INV-1.1 and INV-1.2 require pure passthrough

---

## T2.4 — Write the input validator ⚠

**33–45 min · Invariants: INV-2.4, INV-4.2**

**Description:** Define the FastAPI `Query` parameter constraint for `customer_id`: alphanumeric, hyphens, underscores, max 64 characters. The validator must reject non-conforming values with 422 before any database call.

- **Inputs:** `app/dependencies.py` stub; INVARIANTS.md INV-4.2 (format consistency)
- **Outputs:** A reusable `CustomerIdQuery` type; tested that SQL-injection strings return 422

**Prompt for Claude Code:**

```
Write the customer_id input validator in app/dependencies.py.

Requirements:
1. Define a FastAPI Query parameter with:
   - pattern: "^[a-zA-Z0-9_-]{1,64}$"
   - max_length: 64
   - description: "Customer ID: alphanumeric, hyphens, underscores, max 64 chars"
2. Export it as: CustomerIdQuery = Annotated[str, Query(pattern=..., max_length=64)]
3. Do NOT perform any database call in this validator — pure input validation only.

Also update app/main.py to import CustomerIdQuery and add a temporary test route:
  GET /api/test-validate?customer_id=<value>
  returns {"customer_id": customer_id} if valid, lets FastAPI's 422 fire if invalid.
  (This test route will be removed in T3.3.)

Test:
curl -s "http://localhost:8000/api/test-validate?customer_id=CUST-001" | python3 -m json.tool
curl -s -o /dev/null -w "%{http_code}" \
  "http://localhost:8000/api/test-validate?customer_id=%27%3B+DROP+TABLE+customers%3B--"
curl -s -o /dev/null -w "%{http_code}" \
  "http://localhost:8000/api/test-validate?customer_id=$(python3 -c 'print(\"A\"*65)')"
```

**Test cases:**

1. Valid ID `CUST-001` returns 200 with `customer_id` echoed → 200 + JSON
2. SQL injection string returns 422 → 422 status code
3. 65-character string returns 422 → 422 status code
4. String with spaces returns 422 → 422 status code
5. UUID-style `CUST-0001-ABCD` returns 200 → Hyphens are valid

**Verification command:**

```bash
curl -s -o /dev/null -w "%{http_code}" "http://localhost:8000/api/test-validate?customer_id=CUST-001"
curl -s -o /dev/null -w "%{http_code}" \
  "http://localhost:8000/api/test-validate?customer_id=%27%3B+DROP+TABLE+customers%3B--"
curl -s -o /dev/null -w "%{http_code}" \
  "http://localhost:8000/api/test-validate?customer_id=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
```

**Code review — what to look for:**

- Confirm the pattern regex anchors both start `^` and end `$` — unanchored patterns allow prefix/suffix injection
- Confirm `max_length=64` matches the `VARCHAR(64)` column definition in `init.sql` — INV-4.2 requires all three artefacts to agree
- Confirm no database call happens inside the validator — INV-2.4 requires 422 to fire before the DB layer is reached
- The UI `maxlength` attribute in T5.1 must also be 64 — note this for the T5.1 reviewer

---

## T2.5 — Write the API key dependency ⚠

**33–45 min · Invariants: INV-3.1, INV-3.2**

**Description:** Implement the FastAPI dependency that reads the `X-API-Key` header and compares it to `API_KEY` from the environment. Requests with missing or wrong key must be rejected with 401 before any downstream logic runs.

- **Inputs:** `app/dependencies.py` from T2.4; `API_KEY` env var from `.env`
- **Outputs:** `verify_api_key` dependency function; tested that missing and wrong keys return 401 with no DB activity

**Prompt for Claude Code:**

```
Add the API key authentication dependency to app/dependencies.py.

Requirements:
1. Read API_KEY from os.environ at module load time (not per-request).
   If API_KEY is not set, raise a RuntimeError at startup.
2. Function: verify_api_key(x_api_key: str = Header(None)) -> None
   - If x_api_key is None or x_api_key != API_KEY:
       raise HTTPException(status_code=401, detail="Invalid or missing API key")
   - If valid: return (no body needed — it's a dependency, not a route handler)
3. The function must NOT log the api key value under any circumstances.
4. The function must NOT make any database call.

Update the test route from T2.4 to also depend on verify_api_key:
  GET /api/test-validate — Depends(verify_api_key)

Test:
curl -s -o /dev/null -w "%{http_code}" \
  "http://localhost:8000/api/test-validate?customer_id=CUST-001"
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: wrongkey" \
  "http://localhost:8000/api/test-validate?customer_id=CUST-001"
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: changeme" \
  "http://localhost:8000/api/test-validate?customer_id=CUST-001"
```

**Test cases:**

1. No `X-API-Key` header → 401 → 401 status code
2. Wrong key → 401 → 401 status code
3. Correct key (`changeme`) → 200 → 200 status code
4. 401 response body contains `detail` field → Structured JSON error
5. API key value does not appear in any log line → `grep logs for 'changeme'`

**Verification command:**

```bash
curl -s -o /dev/null -w "%{http_code}" \
  "http://localhost:8000/api/test-validate?customer_id=CUST-001"
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: wrongkey" \
  "http://localhost:8000/api/test-validate?customer_id=CUST-001"
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: changeme" \
  "http://localhost:8000/api/test-validate?customer_id=CUST-001"
docker compose logs api | grep -c "changeme" && echo "KEY LEAKED - FAIL" || echo "Key not in logs - OK"
```

**Code review — what to look for:**

- Confirm the dependency is applied via `Depends()` not a manual check inside the route handler — manual checks are easy to forget on new routes
- Confirm the API_KEY is compared using `==` not `in` or `startswith`
- Confirm the error detail message does not include the received key value — INV-3.2
- Confirm no database import or call is reachable from `verify_api_key` — INV-3.1 requires rejection before DB activity

---

---

# SESSION 3: FastAPI Core — Auth, Risk Endpoint & Error Handling

**Est: 225–315 min**

**Integration check:** Full status-code matrix passes: 200 on valid lookup, 401 on missing/bad key, 404 on missing customer, 422 on malformed ID, 503 when DB is unreachable. Response body contains exactly the three required fields and no extras.

---

## T3.1 — Remove test route and wire dependencies to /api/risk stub ⚠

**28–35 min · Invariants: INV-3.1**

**Description:** Remove the T2.4/T2.5 test route. Register `verify_api_key` globally on all `/api/*` routes using an `APIRouter` with a dependency. Confirm 401 fires before any other handler logic.

- **Inputs:** `app/main.py` with test route; `dependencies.py` with `verify_api_key`
- **Outputs:** `APIRouter` for `/api` prefix with `verify_api_key` dependency; test route removed; stub `GET /api/risk` returns 501 but requires valid key

**Prompt for Claude Code:**

```
Refactor app/main.py to use a dedicated APIRouter for all /api routes.

Requirements:
1. Create an APIRouter with prefix="/api" and dependencies=[Depends(verify_api_key)].
   This means ALL routes on this router require a valid API key automatically.
2. Remove the /api/test-validate route entirely.
3. Add a stub route on the router:
   GET /risk returns {"status": "not implemented"} with status_code=501.
   This will be replaced in T3.3.
4. Include the router in the main app: app.include_router(api_router)
5. Keep the GET / stub route on the main app (not the router) — it does not require auth.

Verify the routing:
- GET /api/risk with no key → 401 (dependency fires before the stub handler)
- GET /api/risk with correct key → 501 (stub handler reached)
- GET / with no key → 200 (main route, no auth required)
```

**Test cases:**

1. `GET /api/risk` with no key → 401 → Auth dependency fires
2. `GET /api/risk` with correct key → 501 → Route reached but not implemented
3. `GET /` with no key → 200 → Main route not protected
4. No `/api/test-validate` route exists → 404 on that path

**Verification command:**

```bash
curl -s -o /dev/null -w "%{http_code}" "http://localhost:8000/api/risk"
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: changeme" "http://localhost:8000/api/risk"
curl -s -o /dev/null -w "%{http_code}" "http://localhost:8000/"
curl -s -o /dev/null -w "%{http_code}" "http://localhost:8000/api/test-validate"
```

**Code review — what to look for:**

- Confirm the dependency is on the router, not copy-pasted into each route — adding a new `/api/*` route must inherit auth automatically per INV-3.1
- Confirm `GET /` is on the main app not the router — it must never require an API key

---

## T3.2 — Implement input validation on /api/risk ⚠

**33–45 min · Invariants: INV-2.4, INV-4.2**

**Description:** Wire the `CustomerIdQuery` validator from T2.4 into the `/api/risk` route. Confirm 422 is returned for malformed IDs before any database call, and that the response body is structured JSON with a `detail` field.

- **Inputs:** `CustomerIdQuery` from `dependencies.py`; `/api/risk` stub from T3.1
- **Outputs:** `/api/risk` returns 422 with structured JSON `detail` for any `customer_id` failing the pattern

**Prompt for Claude Code:**

```
Update the GET /api/risk route in app/main.py to use the CustomerIdQuery validator.

Route signature:
  @api_router.get("/risk")
  async def get_risk(customer_id: CustomerIdQuery) -> dict:
      return {"status": "not implemented", "received_id": customer_id}

The CustomerIdQuery type (Annotated[str, Query(pattern=..., max_length=64)])
will cause FastAPI to return 422 automatically for invalid inputs.

Test all of these (all require X-API-Key: changeme header):
1. customer_id=CUST-001 → 200 (stub response)
2. customer_id='; DROP TABLE customers;-- → 422
3. customer_id=<script>alert(1)</script> → 422
4. customer_id=(65 A chars) → 422
5. customer_id= (empty string) → 422
6. No customer_id param at all → 422

For each 422 response, confirm:
  curl -s ... | python3 -m json.tool  shows a "detail" key in the JSON body
```

**Test cases:**

1. SQL injection string → 422 with JSON detail → 422 + structured body
2. XSS string → 422 → 422 status code
3. 65-char ID → 422 → 422 status code
4. Empty string → 422 → 422 status code
5. Missing param → 422 → 422 status code
6. Valid `CUST-001` → 200 → 200 status code

**Verification command:**

```bash
KEY=changeme
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: $KEY" \
  "http://localhost:8000/api/risk?customer_id=CUST-001"
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: $KEY" \
  "http://localhost:8000/api/risk?customer_id=%27%3BDROP+TABLE+customers%3B--"
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: $KEY" \
  "http://localhost:8000/api/risk?customer_id=%3Cscript%3Ealert%281%29%3C%2Fscript%3E"
curl -s -w "\n" -H "X-API-Key: $KEY" \
  "http://localhost:8000/api/risk?customer_id=%27bad%27" | python3 -m json.tool
```

**Code review — what to look for:**

- Confirm the 422 response body has a `detail` key — FastAPI's default validation error format satisfies INV-2.7 without extra work, but confirm it is not suppressed
- Confirm no log entry appears for rejected `customer_id` values — the DB was never called
- Confirm the pattern rejects empty string — the `{1,64}` quantifier in the regex requires at least one character

---

## T3.3 — Implement the full /api/risk route handler ⚠

**40–50 min · Invariants: INV-1.1, INV-1.2, INV-2.1, INV-2.2, INV-2.3**

**Description:** Replace the stub with the real route: call `get_customer()`, return 200 with the three required fields on hit, return 404 on miss. The response must contain exactly `customer_id`, `risk_tier`, `risk_factors` and nothing else.

- **Inputs:** `get_customer()` from T2.3; `/api/risk` stub with validation from T3.2
- **Outputs:** Fully functional `GET /api/risk`; 200 on hit with exact response shape; 404 on miss; verified against all 12 seed records

**Prompt for Claude Code:**

```
Replace the stub handler in GET /api/risk with the real implementation.

Requirements:
1. Call db.get_customer(customer_id).
2. If result is None: raise HTTPException(status_code=404, detail="Customer not found")
3. If result is found: return a dict with EXACTLY these three keys:
   {"customer_id": result["customer_id"],
    "risk_tier": result["risk_tier"],
    "risk_factors": result["risk_factors"]}
   Do NOT return result directly if it might contain extra keys.
   Do NOT add any computed fields, timestamps, or metadata.
4. Add a log.info call:
   f"risk_lookup customer_id={customer_id} tier={result['risk_tier']}"
5. Remove the "received_id" field from the response — it was only for debugging.

Test the full response shape:
curl -s -H "X-API-Key: changeme" \
  "http://localhost:8000/api/risk?customer_id=CUST-LOW-001" | python3 -m json.tool
curl -s -H "X-API-Key: changeme" \
  "http://localhost:8000/api/risk?customer_id=CUST-HIGH-001" | python3 -m json.tool
curl -s -H "X-API-Key: changeme" \
  "http://localhost:8000/api/risk?customer_id=NONEXISTENT-999" | python3 -m json.tool
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: changeme" \
  "http://localhost:8000/api/risk?customer_id=NONEXISTENT-999"
```

**Test cases:**

1. Known LOW customer → 200 with `risk_tier=LOW` → Correct data returned
2. Known HIGH customer → 200 with `risk_tier=HIGH` and factors list → Correct data returned
3. Customer with empty factors → 200 with `risk_factors=[]` → Empty list not treated as error
4. Unknown ID → 404 with JSON detail → `{"detail": "Customer not found"}`
5. 200 response has exactly 3 keys → No extra fields like `created_at`
6. Log contains `customer_id` and `tier` → Log line visible in `docker compose logs`

**Verification command:**

```bash
KEY=changeme
curl -s -H "X-API-Key: $KEY" \
  "http://localhost:8000/api/risk?customer_id=CUST-LOW-001" | python3 -m json.tool
curl -s -H "X-API-Key: $KEY" \
  "http://localhost:8000/api/risk?customer_id=CUST-HIGH-001" | python3 -m json.tool
EMPTY_ID=$(docker compose exec db psql -U riskuser -d riskdb -t \
  -c "SELECT customer_id FROM customers WHERE risk_factors = '{}';" | xargs)
curl -s -H "X-API-Key: $KEY" \
  "http://localhost:8000/api/risk?customer_id=$EMPTY_ID" | python3 -m json.tool
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: $KEY" \
  "http://localhost:8000/api/risk?customer_id=NONEXISTENT-999"
docker compose logs api | grep "risk_lookup"
```

**Code review — what to look for:**

- Count the keys in the 200 response — must be exactly 3. Any 4th key violates INV-2.2
- Confirm no risk value is derived or computed in the handler — tier and factors must come directly from `get_customer()` return value per INV-1.1 and INV-1.2
- Confirm empty `risk_factors` returns `[]` not `null` and not omitted — INV-1.4
- Confirm 404 is raised not returned — `raise HTTPException(404)` not `return JSONResponse(404)`

---

## T3.4 — Implement exception handlers — 503 and catch-all ⚠

**40–50 min · Invariants: INV-2.5, INV-2.6, INV-2.7**

**Description:** Add FastAPI exception handlers that intercept `psycopg2.OperationalError` and unhandled `RuntimeError`, returning 503 with a sanitised message. Override the default 500 handler to prevent internal details leaking.

- **Inputs:** `app/main.py` from T3.3; `db.py` `get_cursor()` raising `RuntimeError`
- **Outputs:** 503 returned when DB is unreachable; no Postgres error text in any response body; custom 500 handler in place

**Prompt for Claude Code:**

```
Add exception handlers to app/main.py to handle database errors gracefully.

Requirements:
1. Handler for psycopg2.OperationalError:
   @app.exception_handler(psycopg2.OperationalError)
   async def db_op_error_handler(request, exc):
       log.error("Database operational error (details suppressed from response)")
       return JSONResponse(status_code=503,
                           content={"detail": "Service temporarily unavailable"})

2. Handler for RuntimeError (which db.get_cursor() raises when conn is None):
   Same: return 503 with {"detail": "Service temporarily unavailable"}

3. Generic catch-all to prevent FastAPI from leaking internal errors:
   @app.exception_handler(Exception)
   async def generic_error_handler(request, exc):
       log.error(f"Unhandled exception type={type(exc).__name__}")
       return JSONResponse(status_code=503,
                           content={"detail": "Service temporarily unavailable"})
   CRITICAL: HTTPException must NOT be caught here — add at the top:
   if isinstance(exc, HTTPException): raise exc

Test by simulating DB failure:
docker compose stop db
sleep 3
curl -s -H "X-API-Key: changeme" "http://localhost:8000/api/risk?customer_id=CUST-LOW-001"
# Expect: 503 with {"detail": "Service temporarily unavailable"}
# Confirm: no "postgresql" or "psycopg2" text in response body
docker compose start db
sleep 10
```

**Test cases:**

1. DB stopped → 503 with sanitised detail → `{"detail": "Service temporarily unavailable"}`
2. 503 body does not contain `postgresql` or `psycopg2` → No internal info
3. 503 body does not contain connection string → No DSN in response
4. After DB restart, 200 returns correctly → Connection recovers
5. 404 still works after adding catch-all → `HTTPException` passes through

**Verification command:**

```bash
docker compose stop db && sleep 5
curl -s -H "X-API-Key: changeme" "http://localhost:8000/api/risk?customer_id=CUST-LOW-001"
RESP=$(curl -s -H "X-API-Key: changeme" "http://localhost:8000/api/risk?customer_id=CUST-LOW-001")
echo $RESP | python3 -m json.tool
echo $RESP | grep -i "psycopg\|postgresql\|connection\|host\|password" \
  && echo "LEAK DETECTED" || echo "No leak"
docker compose start db && sleep 15
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: changeme" \
  "http://localhost:8000/api/risk?customer_id=CUST-LOW-001"
curl -s -o /dev/null -w "%{http_code}" -H "X-API-Key: changeme" \
  "http://localhost:8000/api/risk?customer_id=NONEXISTENT-999"
```

**Code review — what to look for:**

- Confirm the catch-all handler explicitly re-raises `HTTPException` before handling — without this, 401 and 404 will be swallowed and returned as 503, violating INV-2.6
- Confirm the `log.error` call does NOT include `str(exc)` if `exc` is a psycopg2 error — `OperationalError` includes the connection string in its message per INV-2.7
- Confirm status code is exactly 503 not 500 — INV-2.6 prohibits returning 500 for database unavailability

---

## T3.5 — Jinja2 template setup and GET / with API key injection ⚠

**40–50 min · Invariants: INV-3.3, INV-3.4**

**Description:** Replace the `GET /` stub with a real Jinja2-rendered route that injects the API key into the HTML template. The template file in the repo must contain `{{ api_key }}` placeholder only. The route must set `Cache-Control: no-store`.

- **Inputs:** `templates/index.html` stub from T1.1; `app/main.py`; Jinja2 (bundled with Starlette)
- **Outputs:** `GET /` serves rendered HTML with API key injected; `Cache-Control: no-store` header present; no literal key in the template file; UI structural elements in place (JS wired in T5.1)

**Prompt for Claude Code:**

```
Set up Jinja2 templating in app/main.py and update the GET / route.

Requirements:
1. Configure Jinja2Templates:
   from fastapi.templating import Jinja2Templates
   templates = Jinja2Templates(directory="templates")

2. Update GET / to render index.html with the API key injected:
   @app.get("/")
   async def serve_ui(request: Request):
       response = templates.TemplateResponse(
           "index.html", {"request": request, "api_key": os.environ["API_KEY"]}
       )
       response.headers["Cache-Control"] = "no-store"
       return response

3. Write templates/index.html as a minimal placeholder that:
   - Contains the Jinja2 placeholder: const API_KEY = "{{ api_key }}";
   - Has <title>Customer Risk Lookup</title>
   - Has a visible <h1>Customer Risk Lookup</h1>
   - Has a text input: id="customer-id-input", maxlength="64"
   - Has a button: id="lookup-btn"
   - Has a div: id="result-display"
   - The template file must contain {{ api_key }} literally — not a real key value.

After implementation:
curl -s http://localhost:8000/ | grep 'const API_KEY'
curl -I http://localhost:8000/ | grep -i cache
```

**Test cases:**

1. `GET /` returns 200 → 200 status
2. Response HTML contains `const API_KEY = "changeme"` → Key injected at render time
3. Response header includes `Cache-Control: no-store` → Header present
4. `templates/index.html` contains `{{ api_key }}` literally → Placeholder, not real value
5. `GET /` does not require `X-API-Key` header → No auth on UI route

**Verification command:**

```bash
curl -s http://localhost:8000/ | grep -i "api_key\|API_KEY"
curl -I http://localhost:8000/ | grep -i "cache-control"
grep "api_key" templates/index.html
grep "changeme" templates/index.html && echo "KEY HARDCODED - FAIL" || echo "No hardcoded key - OK"
```

**Code review — what to look for:**

- Open `templates/index.html` in the repo and confirm it contains `{{ api_key }}` not a real key value — INV-3.3
- Confirm `Cache-Control: no-store` is set on the response object, not as a default header — must be present on every `GET /` response per INV-3.4
- Confirm the `GET /` route is NOT on the APIRouter (which has the auth dependency)
- Confirm `maxlength="64"` is on the customer ID input — INV-4.2

---

---

# SESSION 4: Health Endpoint, Startup Checks & Observability

**Est: 175–250 min**

**Integration check:** `GET /health` returns 200 with DB up and 503 with DB down. Startup refuses to accept traffic if template is missing. `docker compose logs` shows `customer_id` and timestamp for every `/api/risk` call. API key is absent from all log lines.

---

## T4.1 — Implement GET /health endpoint ⚠

**40–50 min · Invariants: INV-6.1, INV-7.1**

**Description:** Add a `/health` route that performs a live database ping and returns 200 when the DB is reachable and 503 when it is not. The result must not be cached or hardcoded.

- **Inputs:** `app/main.py`; `app/db.py` with `get_cursor()`
- **Outputs:** `GET /health` returns 200 when DB is up; 503 when DB is down; compose healthcheck updated to use this endpoint

**Prompt for Claude Code:**

```
Add a GET /health endpoint to app/main.py and update docker-compose.yml.

Health endpoint requirements:
1. Route: GET /health (on main app, not the APIRouter — no auth required)
2. On each request, perform a live DB check:
   try:
     with db.get_cursor() as cursor:
       cursor.execute("SELECT 1")
     return {"status": "ok", "database": "connected"}
   except Exception:
     return JSONResponse(status_code=503,
                         content={"status": "degraded", "database": "unavailable"})
3. Do NOT cache the result. Each request must check the DB.
4. The 503 response must NOT include any postgres error details.

Update docker-compose.yml api service healthcheck:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
    interval: 10s
    timeout: 5s
    retries: 5

Test:
curl -s http://localhost:8000/health | python3 -m json.tool
docker compose stop db && sleep 5
curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/health
docker compose start db && sleep 15
```

**Test cases:**

1. DB up → `GET /health` returns 200 with `status=ok` → 200 + JSON
2. DB stopped → `GET /health` returns 503 with `status=degraded` → 503 + JSON
3. 503 response body has no postgres error text → No internal details
4. `GET /health` does not require `X-API-Key` → 200 without auth header

**Verification command:**

```bash
curl -s http://localhost:8000/health | python3 -m json.tool
docker compose stop db && sleep 5
STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/health)
echo "Health status with DB down: $STATUS"
BODY=$(curl -s http://localhost:8000/health)
echo $BODY | grep -i "psycopg\|postgresql\|connection" && echo "LEAK" || echo "Clean"
docker compose start db && sleep 15
curl -s http://localhost:8000/health
```

**Code review — what to look for:**

- Confirm the health check performs a real DB query on each call — hardcoding `"ok"` violates INV-7.1
- Confirm the health check route is NOT on the `api_router` — health must be accessible to orchestrators without credentials
- Confirm the compose healthcheck for the `api` service points to `GET /health` — INV-6.1 requires the compose health system to reflect actual DB connectivity

---

## T4.2 — Add startup template validation check ⚠

**33–45 min · Invariants: INV-5.3**

**Description:** Extend the lifespan startup handler to verify the Jinja2 template file exists and renders without error. A missing or broken template must log a warning but must not crash the app — the `/api/risk` endpoint must remain available.

- **Inputs:** `app/main.py` lifespan handler; `templates/index.html` from T3.5
- **Outputs:** Startup check verifies template renders; if template is missing the app logs an error but the API routes remain available

**Prompt for Claude Code:**

```
Add a startup template validation check to the lifespan handler in app/main.py.

Requirements:
In the lifespan startup handler, after the DB connection succeeds, add:
  try:
    from jinja2 import Environment, FileSystemLoader, TemplateNotFound
    env = Environment(loader=FileSystemLoader("templates"))
    tmpl = env.get_template("index.html")
    tmpl.render(api_key="__startup_test__", request=None)
    log.info("Template validation: OK")
  except TemplateNotFound:
    log.error("STARTUP WARNING: templates/index.html not found. UI unavailable. API routes unaffected.")
  except Exception as e:
    log.error(f"STARTUP WARNING: Template render error: {type(e).__name__}. UI may be unavailable.")

CRITICAL: The template check must NOT raise / crash the application.
The API must remain available even if the template is broken.

Test:
# Normal startup
docker compose logs api | grep "Template"

# Test broken template
mv templates/index.html templates/index.html.bak
docker compose restart api && sleep 10
docker compose logs api | grep -i "template\|warning"
curl -s -o /dev/null -w "%{http_code}" \
  -H "X-API-Key: changeme" "http://localhost:8000/api/risk?customer_id=CUST-LOW-001"
# Must return 200 even with missing template
mv templates/index.html.bak templates/index.html && docker compose restart api
```

**Test cases:**

1. Normal startup logs `Template validation: OK` → Validation runs at startup
2. With missing template, API still returns 200 → INV-5.3: failure surfaces are independent
3. With missing template, startup log contains warning → Error is visible but non-fatal
4. After restoring template and restarting, validation passes again → Recovery works

**Verification command:**

```bash
docker compose logs api | grep -i "template"
mv templates/index.html templates/index.html.bak
docker compose restart api && sleep 12
docker compose logs api | grep -i "template\|warning\|error"
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -H "X-API-Key: changeme" "http://localhost:8000/api/risk?customer_id=CUST-LOW-001")
echo "API with broken template: $STATUS"
mv templates/index.html.bak templates/index.html
docker compose restart api
```

**Code review — what to look for:**

- Confirm the template check exception is caught and logged, not re-raised — raising here would violate INV-5.3 by crashing the app on UI failure
- Confirm the API route test after removing the template returns 200, not 503 — this is the invariant: failure surfaces must be independent

---

## T4.3 — Implement structured logging for audit trail ⚠

**40–50 min · Invariants: INV-3.2, INV-7.2**

**Description:** Configure Python logging in `app/main.py` to emit JSON-formatted log lines for `/api/risk` requests. Each log entry must include `customer_id` and timestamp. The API key value must never appear in any log line.

- **Inputs:** `app/main.py` with `log.info` call from T3.3; `.env` with `API_KEY`
- **Outputs:** Structured log lines for every `/api/risk` call; timestamp and `customer_id` present; API key absent; verified via `docker compose logs`

**Prompt for Claude Code:**

```
Configure structured logging in app/main.py for the Customer Risk API.

Requirements:
1. Configure Python's logging module at app startup to emit JSON log lines to stdout:
   import json, logging
   class JsonFormatter(logging.Formatter):
     def format(self, record):
       return json.dumps({
         "ts": self.formatTime(record),
         "level": record.levelname,
         "msg": record.getMessage()
       })
   handler = logging.StreamHandler()
   handler.setFormatter(JsonFormatter())
   logging.basicConfig(handlers=[handler], level=logging.INFO, force=True)

2. Ensure the log.info call in T3.3 includes customer_id and risk_tier:
   log.info(f"risk_lookup customer_id={customer_id} tier={result['risk_tier']}")

3. For 404 lookups, also log:
   log.info(f"risk_lookup customer_id={customer_id} result=not_found")

4. Verify the API key never appears in any log line.

Test:
curl -s -H "X-API-Key: changeme" "http://localhost:8000/api/risk?customer_id=CUST-LOW-001"
curl -s -H "X-API-Key: changeme" "http://localhost:8000/api/risk?customer_id=NONEXISTENT-999"
docker compose logs api | grep "risk_lookup" | python3 -m json.tool
docker compose logs api | grep "changeme" && echo "KEY IN LOGS - FAIL" || echo "Key not in logs - OK"
```

**Test cases:**

1. Every `/api/risk` call produces a JSON log line → `grep risk_lookup` shows entries
2. Log lines are valid JSON → `python3 -m json.tool` parses without error
3. Log line for hit contains `customer_id` and `tier` → Fields present in JSON
4. Log line for miss contains `customer_id` and `result=not_found` → Miss is logged
5. API key `changeme` does not appear in any log line → `grep` returns no match

**Verification command:**

```bash
curl -s -H "X-API-Key: changeme" "http://localhost:8000/api/risk?customer_id=CUST-LOW-001"
curl -s -H "X-API-Key: changeme" "http://localhost:8000/api/risk?customer_id=NONEXISTENT-999"
docker compose logs api | grep "risk_lookup"
docker compose logs api | grep "risk_lookup" | head -1 | python3 -m json.tool
docker compose logs api | grep "changeme" && echo "KEY LEAKED - FAIL" || echo "No key in logs - OK"
```

**Code review — what to look for:**

- Search the entire log output for the `API_KEY` value from `.env` — INV-3.2 prohibits the key from appearing in any log line in any context
- Confirm the log call uses `customer_id` from the validated route parameter not from raw headers or request object
- Confirm 401 rejections do NOT log a `customer_id` — there is none, and logging a fabricated value would be misleading

---

## T4.4 — Full status-code matrix integration test

**28–35 min**

**Description:** Run a complete curl-based integration test covering all five status codes defined in INV-2.6. This is a verification-only task — no new code if all pass.

- **Inputs:** Running stack from Sessions 1–4
- **Outputs:** Test script that exercises all 5 status codes and reports PASS/FAIL for each; all 5 must pass

**Prompt for Claude Code:**

```
Write and run a shell script that tests the complete status code matrix for the API.

Create test_matrix.sh:
#!/bin/bash
KEY=changeme
BASE=http://localhost:8000
PASS=0; FAIL=0

check() {
  local desc=$1 expected=$2 actual=$3
  if [ "$actual" = "$expected" ]; then
    echo "PASS [$actual] $desc"; ((PASS++))
  else
    echo "FAIL [got $actual, expected $expected] $desc"; ((FAIL++))
  fi
}

check "200: valid customer" 200 \
  $(curl -s -o/dev/null -w "%{http_code}" -H "X-API-Key: $KEY" "$BASE/api/risk?customer_id=CUST-LOW-001")
check "401: no key" 401 \
  $(curl -s -o/dev/null -w "%{http_code}" "$BASE/api/risk?customer_id=CUST-LOW-001")
check "401: wrong key" 401 \
  $(curl -s -o/dev/null -w "%{http_code}" -H "X-API-Key: wrong" "$BASE/api/risk?customer_id=CUST-LOW-001")
check "404: unknown customer" 404 \
  $(curl -s -o/dev/null -w "%{http_code}" -H "X-API-Key: $KEY" "$BASE/api/risk?customer_id=NONEXISTENT-999")
check "422: malformed id" 422 \
  $(curl -s -o/dev/null -w "%{http_code}" -H "X-API-Key: $KEY" "$BASE/api/risk?customer_id=%27%3BDROP+TABLE%3B--")
check "200: health up" 200 \
  $(curl -s -o/dev/null -w "%{http_code}" "$BASE/health")
docker compose stop db && sleep 5
check "503: db down" 503 \
  $(curl -s -o/dev/null -w "%{http_code}" -H "X-API-Key: $KEY" "$BASE/api/risk?customer_id=CUST-LOW-001")
docker compose start db && sleep 15

echo ""
echo "Results: $PASS passed, $FAIL failed"
[ $FAIL -eq 0 ] && echo "ALL PASS" || exit 1

chmod +x test_matrix.sh && ./test_matrix.sh
```

**Test cases:**

1. 200 on valid customer → PASS
2. 401 on no key → PASS
3. 401 on wrong key → PASS
4. 404 on unknown customer → PASS
5. 422 on malformed ID → PASS
6. 503 on DB down → PASS
7. ALL PASS printed at end → Exit 0

**Verification command:**

```bash
chmod +x test_matrix.sh && ./test_matrix.sh
```

---

## T4.5 — Verify no external network calls and compose topology ⚠

**28–35 min · Invariants: INV-5.1, INV-5.2**

**Description:** Confirm the compose stack has exactly two services and that the FastAPI container makes no outbound network calls to addresses outside the compose network. No code changes — verification only.

- **Inputs:** `docker-compose.yml` from T1.3; running stack
- **Outputs:** Verified service count; network audit showing no external calls in application code

**Prompt for Claude Code:**

```
Verify the compose topology and network isolation. No code changes — verification task.

Run these checks and document the results:

1. Service count:
   docker compose config | grep -E "^  [a-z]" | grep -v "^    "

2. Confirm no extra networks or external links:
   docker compose config | grep -E "external:|driver: bridge"

3. From inside the api container, confirm the db host is reachable:
   docker compose exec api python3 -c "
   import socket
   try:
     socket.getaddrinfo('db', 5432)
     print('db: reachable OK')
   except: print('db: unreachable FAIL')
   "

4. Confirm no requests, httpx, or urllib calls exist in app/ code:
   grep -r "requests\|httpx\|urllib.request" app/ \
     && echo "EXTERNAL CALL FOUND" || echo "No external calls in code"
```

**Test cases:**

1. Exactly two services: `api` and `db` → `docker compose config` confirms
2. No `requests`, `httpx`, or `urllib.request` in `app/` code → grep returns no match
3. `db` hostname resolves inside `api` container → `socket.getaddrinfo` succeeds

**Verification command:**

```bash
docker compose config | grep "services:" -A 20 | grep -E "^  [a-z][a-z]"
grep -r "requests\|httpx\|urllib.request" app/ \
  && echo "EXTERNAL CALL FOUND" || echo "No external calls in code"
docker compose exec api python3 -c \
  "import socket; print(socket.getaddrinfo('db', 5432)[0])"
```

**Code review — what to look for:**

- Count top-level keys under `services:` in the compose config — must be exactly 2 per INV-5.2
- If external DNS resolves from the api container, note it as a limitation — Docker does not provide host-level network isolation by default; the invariant is satisfied by the absence of external calls in application code per INV-5.1

---

---

# SESSION 5: Browser UI, Integration & Documentation

**Est: 140–175 min**

**Integration check:** Complete end-to-end flow in browser: enter `CUST-HIGH-001`, see `tier=HIGH` and factors displayed. Enter unknown ID, see error message. Open DevTools — confirm no 500 errors, `Cache-Control: no-store` present on `GET /`.

---

## T5.1 — Implement the browser UI with full JS logic ⚠

**40–50 min · Invariants: INV-3.4, INV-4.2, INV-8.1, INV-8.2**

**Description:** Write the complete `templates/index.html` with vanilla JavaScript fetch logic. The UI must call `GET /api/risk` with the injected API key, display the result without modification, and display errors visibly for all error codes.

- **Inputs:** `templates/index.html` stub from T3.5 (with API_KEY injected and structural elements in place)
- **Outputs:** Fully functional browser UI; all error codes displayed distinctly; result displayed verbatim; `Cache-Control: no-store` confirmed; `maxlength=64` on input

**Prompt for Claude Code:**

```
Write the complete templates/index.html for the Customer Risk API browser UI.

Requirements:
1. Keep the Jinja2 placeholder: const API_KEY = "{{ api_key }}";
2. Text input: id="customer-id-input", maxlength="64", placeholder="e.g. CUST-LOW-001"
3. Button: id="lookup-btn", text="Look up customer"
4. Result div: id="result-display"
5. On button click (and on Enter key in the input):
   a. Read customer_id from the input
   b. Call: fetch('/api/risk?customer_id=' + encodeURIComponent(customer_id),
             {headers: {'X-API-Key': API_KEY}})
   c. On 200: display in result-display:
        Customer ID: <value>
        Risk Tier: <value>  (style HIGH in red, MEDIUM in amber, LOW in green)
        Risk Factors: <comma-separated list, or "None" if empty>
      Display verbatim from API response — no transformation of values.
   d. On 4xx/5xx: display the HTTP status and the detail message from the JSON
      response in a visually distinct error style (e.g. red border or red text).
      Never show blank on error.
   e. Show a loading indicator while the fetch is in flight.

6. No external CDN or script tag — vanilla JS only, no frameworks.

After writing:
docker compose restart api && sleep 10
curl -I http://localhost:8000/ | grep -i cache
curl -s http://localhost:8000/ | grep 'maxlength'
```

**Test cases:**

1. `CUST-HIGH-001` shows `tier=HIGH` in red and factors list → Visual display correct
2. Customer with empty factors shows `None` for factors → Empty list handled
3. `NONEXISTENT-999` shows 404 error with detail message → Error displayed visibly
4. SQL injection string shows 422 error message → Validation error displayed
5. `Cache-Control: no-store` in response headers → Header present
6. `maxlength="64"` on input element → INV-4.2 satisfied

**Verification command:**

```bash
curl -I http://localhost:8000/ | grep -i "cache-control"
curl -s http://localhost:8000/ | grep 'maxlength'
curl -s http://localhost:8000/ | grep '{{ api_key }}' \
  && echo "PLACEHOLDER NOT RENDERED - FAIL" || echo "Template rendered OK"
curl -s http://localhost:8000/ | grep 'const API_KEY'
```

**Code review — what to look for:**

- Confirm `maxlength="64"` is in the rendered HTML — this must match the validator pattern and `VARCHAR(64)` per INV-4.2
- Confirm the JS displays `risk_tier` and `risk_factors` verbatim without remapping — INV-8.1 prohibits any transformation
- Confirm all four error codes (401, 404, 422, 503) produce a visible, non-blank error state — test each in browser DevTools — INV-8.2
- Confirm `Cache-Control: no-store` appears in `curl -I` output — INV-3.4
- Confirm `{{ api_key }}` does not appear in the rendered page (it must be replaced by the actual key) — INV-3.3

---

## T5.2 — Write README.md

**33–40 min**

**Description:** Write the complete README documenting prerequisites, quickstart, `.env` setup, known limitations, and the invariants that are explicitly not satisfied in production. This is a required deliverable, not optional polish.

- **Inputs:** All implementation from Sessions 1–5; ARCHITECTURE.md §3 and §4 known limitations
- **Outputs:** `README.md` with: prerequisites, quickstart, `.env` variables, architecture diagram (ASCII), known limitations, OQ resolutions

**Prompt for Claude Code:**

```
Write README.md for the Customer Risk API.

Required sections:
1. Overview (2-3 sentences)
2. Prerequisites: Docker, Docker Compose
3. Quickstart:
     git clone ...
     cp .env.example .env
     # Edit .env: set API_KEY to a secure value
     docker compose up
     # Open http://localhost:8000 in browser
4. Environment variables table:
   API_KEY, DATABASE_URL, POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD
5. API reference:
   GET /api/risk?customer_id=<id>  Header: X-API-Key: <key>
   Response shape and status codes table (200/401/404/422/503)
6. Known limitations — MUST include:
   a. "The API_KEY is a shared key. Anyone with docker exec access can read it from the
      container environment. This is acceptable for a training demo and is not acceptable
      in production."
   b. "There is no TLS. Do not deploy on a public network."
   c. "The shared key provides no per-consumer attribution. All requests appear identical
      in logs."
   d. "Key rotation requires: update .env, then docker compose up --force-recreate"
   e. "The deployment environment must not have a caching proxy that ignores
      Cache-Control headers."
7. OQ resolutions: document that customer_id format is alphanumeric/hyphens/underscores
   max 64 chars, risk_factors are flat string labels.
8. Short ASCII architecture diagram: Browser → FastAPI → Postgres
```

**Test cases:**

1. README contains Quickstart section with `cp .env.example .env` → Onboarding complete
2. README contains known limitations section → Limitations documented
3. README mentions `Cache-Control` and why it matters → Security note present
4. README documents all 5 env vars → Config complete
5. README contains status code table → API reference present

**Verification command:**

```bash
grep -i "quickstart\|quick start" README.md
grep -i "limitation\|known" README.md
grep -i "cache.control\|Cache-Control" README.md
grep -i "API_KEY\|api_key" README.md | head -5
```

---

## T5.3 — End-to-end smoke test from fresh stack

**33–40 min**

**Description:** Tear down the stack completely (including volumes), rebuild from scratch, and run the full status-code matrix test. Confirms the zero-manual-setup requirement from the brief.

- **Inputs:** Running stack; `test_matrix.sh` from T4.4
- **Outputs:** Clean `docker compose up` with no prior state; all matrix tests pass; confirms `init.sql` seeds correctly on first boot

**Prompt for Claude Code:**

```
Perform a full clean stack rebuild and run the integration test suite.

Steps:
1. Stop and remove everything including volumes:
   docker compose down -v --remove-orphans
   docker image rm customer-risk-api-api 2>/dev/null || true

2. Confirm .env is present (the only manual step):
   ls -la .env

3. Rebuild and start:
   docker compose up --build -d

4. Wait for both containers to be healthy (poll up to 120 seconds):
   for i in $(seq 1 12); do
     docker compose ps | grep "healthy" | wc -l
     sleep 10
   done

5. Run the full test matrix:
   ./test_matrix.sh

6. Manually open http://localhost:8000 and perform one lookup via the UI.
   Record: did the result display correctly? Yes/No.
```

**Test cases:**

1. `docker compose up --build` succeeds with no errors → Exit 0
2. Both containers healthy within 120 seconds → No timeout
3. `test_matrix.sh` reports ALL PASS → All 7 status code checks pass
4. `GET /` serves rendered UI with injected API key → 200 + HTML with API key
5. Fresh seed data present — `CUST-LOW-001` returns 200 → Seed ran on first boot

**Verification command:**

```bash
docker compose down -v --remove-orphans
docker compose up --build -d
sleep 30
docker compose ps
./test_matrix.sh
```

---

## T5.4 — Final invariant audit

**33–45 min**

**Description:** Walk through every invariant in INVARIANTS.md and confirm each is satisfied in the implementation. Document any gaps. This is a human review task assisted by targeted grep and curl checks. Any FAIL is a blocking issue before handoff.

- **Inputs:** Complete implementation from all sessions; INVARIANTS.md
- **Outputs:** Audit checklist with PASS/FAIL per invariant

**Prompt for Claude Code:**

```
Perform a final invariant audit against INVARIANTS.md v0.2.
For each check below, run the command and record PASS or FAIL.
Any FAIL is a blocking issue.

INV-1.1  Passthrough fidelity
  grep -n "risk_tier\|risk_factors" app/main.py | grep -v "import"
  Confirm no transformation between get_customer() result and response dict.

INV-1.3  DB-level CHECK constraint
  docker compose exec db psql -U riskuser -d riskdb -c "\d customers" | grep -i check

INV-2.1/2.2  Response shape — exactly 3 keys
  curl -s -H "X-API-Key: changeme" \
    "http://localhost:8000/api/risk?customer_id=CUST-LOW-001" | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print(sorted(d.keys()))"
  Expect: ['customer_id', 'risk_factors', 'risk_tier']

INV-2.6  Status code matrix
  ./test_matrix.sh

INV-3.3  No literal key in repo
  grep -r "changeme" . --include="*.html" --include="*.js" --include="*.py" \
    --exclude-dir=".git"
  Expect: no matches outside .env and README

INV-3.4  Cache-Control header
  curl -I http://localhost:8000/ | grep -i "cache-control: no-store"

INV-4.1  Parameterised queries — no f-strings in SQL
  grep -n "f\"SELECT\|f'SELECT\|\.format.*SELECT" app/db.py
  Expect: no matches

INV-4.2  Consistent ID format across all 3 artefacts
  grep -n "pattern\|max_length\|VARCHAR\|maxlength" \
    app/dependencies.py db/init.sql templates/index.html

INV-5.2  Two services only
  docker compose config | grep -c "image:\|build:"

INV-7.2  Logs contain customer_id
  curl -s -H "X-API-Key: changeme" \
    "http://localhost:8000/api/risk?customer_id=CUST-LOW-001" > /dev/null
  docker compose logs api | grep "risk_lookup" | head -3

Document each result. Raise a blocking issue for any FAIL.
```

**Test cases:**

1. All 27 invariants audited → Checklist complete
2. Zero FAILs → All invariants satisfied
3. Any FAIL documented with specific code location → Actionable issue

**Verification command:**

```bash
# Run the most critical checks
grep -n "f\"SELECT\|f'SELECT" app/db.py && echo "SQL INJECTION RISK - INV-4.1 FAIL" || echo "INV-4.1 OK"
curl -I http://localhost:8000/ | grep -i "cache-control"
curl -s -H "X-API-Key: changeme" \
  "http://localhost:8000/api/risk?customer_id=CUST-LOW-001" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print('Keys:', sorted(d.keys())); print('Count:', len(d))"
./test_matrix.sh
```
