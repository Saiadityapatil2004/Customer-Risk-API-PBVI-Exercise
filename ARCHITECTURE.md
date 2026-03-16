# ARCHITECTURE.md
## Customer Risk API — Option A: FastAPI + Postgres (No Proxy Layer)

**Version:** 0.1 — Pre-build  
**Status:** Design phase. Not yet validated against implementation.  
**Stack:** Docker Compose · FastAPI (Python 3.11) · Postgres · Vanilla JS  

---

## 1. Problem Framing

### What this system solves

Internal operations staff need to look up a customer's risk tier and the contributing
factors behind it. Today that means ad-hoc SQL queries against a production Postgres
database — no consistent output format, no access controls, no audit trail, and answer
quality that varies with whoever wrote the query. This system replaces that workflow
with a controlled read interface: a single HTTP endpoint that accepts a customer ID,
validates the caller, queries the database, and returns a structured JSON response. A
browser UI makes the same lookup available to non-technical staff without any code.

The system does not compute risk. It surfaces pre-assessed values that already exist in
the database. All business logic that produced those values is outside scope.

### What this system explicitly does not solve

- **Risk computation.** The database is pre-populated. This service is a read layer
  only. No endpoint derives, updates, or recalculates risk values.
- **User identity and authentication.** There is no login system. Access is controlled
  by a single shared API key. The system cannot distinguish between two people using
  the same key.
- **Per-user audit trail.** Logs will record what was queried and when, but not which
  individual initiated the query.
- **Write or update operations.** No endpoint modifies the database under any
  circumstances.
- **Production hardening.** TLS termination, rate limiting, and secrets management at
  scale are explicitly out of scope per the requirements brief.
- **Reverse proxy or network isolation.** FastAPI is the public-facing surface directly.
  There is no intermediary layer between the browser and the application process.

---

## 2. Five Key Design Decisions

---

### Decision 1: FastAPI serves both the UI and the API from a single process

**What was decided:**  
One FastAPI container handles all concerns: the `/api/risk` endpoint that returns JSON,
and a server-rendered HTML route (`/`) that serves the browser UI. Jinja2 templating
(bundled with Starlette, which FastAPI is built on) renders the HTML page server-side
at request time. The API key is written into the page during rendering — it is resolved
inside the container and never exists as a static file in the repository. Postgres runs
as a second container. The compose file has exactly two services.

```
[Browser]
    │
    ├── GET /          → [FastAPI: render UI via Jinja2, inject API_KEY]
    └── GET /api/risk  → [FastAPI: validate key, query DB, return JSON]
                                    │
                               [Postgres]
```

**Rationale:**  
The fixed stack specifies FastAPI and nothing else for application serving. Adding Nginx
introduces a third container, a non-obvious `envsubst` entrypoint pattern, and a
second startup dependency chain to manage correctly — none of which is required by the
brief. Jinja2 ships with Starlette and requires zero additional dependencies. The
two-container compose file is the simplest possible topology that satisfies all stated
requirements. For a training demo system with a fixed, constrained stack, simplicity is
the correct optimisation target.

**Alternatives rejected:**  
- *Nginx reverse proxy (Option B):* Adds a third container and `envsubst` complexity
  for network isolation benefit that may be irrelevant on an internal network. Rejected
  on grounds of unnecessary complexity given the fixed stack and stated scope.
- *Serve the UI as a static file with the API key hardcoded:* Violates the implied
  constraint that credentials must not be statically embedded client-side. Any user
  opening browser DevTools extracts the key trivially. Rejected unconditionally.
- *Serve the UI from a separate static file server container:* Adds a third container
  with no meaningful benefit over Jinja2 rendering, which is already available in the
  FastAPI process. Rejected.

---

### Decision 2: API key validated as a FastAPI dependency, injected into the UI via Jinja2

**What was decided:**  
The API key is stored in `.env` and loaded into the FastAPI application at startup via
`os.environ`. Two things happen with it:

1. **API protection:** A FastAPI dependency function reads the `X-API-Key` header on
   every request to `/api/*` routes. Requests with a missing or non-matching key are
   rejected with `401` before any route handler — and therefore before any database
   call — executes.
2. **UI injection:** The `GET /` route passes the key value to the Jinja2 template as
   a context variable. The rendered HTML contains the key embedded in the JavaScript
   that makes fetch calls. The template file in the repository contains a Jinja2
   placeholder (`{{ api_key }}`), not a real value.

**Rationale:**  
This solves the UI authentication problem — which the requirements brief leaves entirely
unaddressed — without any additional infrastructure. The key is resolved server-side at
render time; the browser receives a complete page with the key already present. No
round-trip to a config endpoint, no static embedding, no proxy injection script. The
FastAPI dependency pattern also ensures the key check runs before route logic on every
protected request, making it structurally impossible to accidentally write a route that
skips authentication.

**Alternatives rejected:**  
- *Hardcode the key in the HTML/JS template:* Embeds credentials in the repository.
  Rejected unconditionally.
- *Expose a `/config` endpoint that returns the key unauthenticated:* Circular — the
  key is what grants access, so serving it without protection defeats the purpose.
  Rejected.
- *Nginx envsubst injection:* Achieves the same result but requires a third container
  and an entrypoint shell script. Rejected in favour of Jinja2, which is already
  available in the FastAPI process.

**What is vague and needs resolution before build:**  
The `.env` file means anyone with `docker exec` access to the running container can
read the key from the environment. For an internal training system this is acceptable.
It would not be acceptable in production. This boundary must be stated explicitly in
the README.

---

### Decision 3: FastAPI startup retry loop with explicit Postgres health check

**What was decided:**  
The FastAPI container does not attempt a database connection on module import. A
FastAPI `lifespan` startup handler retries the Postgres connection with exponential
backoff, up to a defined ceiling (e.g., 10 attempts over ~30 seconds). The Docker
Compose `depends_on` configuration uses the `service_healthy` condition against a
Postgres `pg_isready` health check — not just `service_started`. FastAPI only reports
itself as ready after the connection pool is established.

**Rationale:**  
`docker compose up` respects container start order but does not wait for the process
inside the container to be ready. Postgres takes several seconds to initialise its data
directory on first run. Without a retry loop, FastAPI attempts its first connection,
fails immediately, and crashes — while compose reports all containers as running.
This is the most common silent failure mode in compose stacks and must be explicitly
designed against. `pg_isready` is a lightweight built-in Postgres utility that checks
whether the server is accepting connections; it does not require application-level
credentials and introduces no external dependency.

**Alternatives rejected:**  
- *`depends_on: db` without a health check condition:* Waits for the container process
  to start, not for Postgres to accept connections. This is the failure mode being
  designed against. Rejected.
- *Fixed `sleep` in the entrypoint:* Duration is a guess. Passes on fast hardware,
  fails on slow hardware. Rejected in favour of deterministic health-check polling.
- *Fail on first connection error and rely on Docker restart policy:* Introduces an
  undefined delay between restart attempts and produces confusing compose output.
  Rejected.

---

### Decision 4: Parameterised queries only — no string interpolation in SQL

**What was decided:**  
All database queries use psycopg2 parameterised syntax — `%s` placeholders with
parameters passed as a separate tuple. No part of any user-supplied value is
concatenated or interpolated into a query string at any point in the codebase. This is
treated as a project-wide rule, not a per-query judgement call.

Additionally, `customer_id` is validated by a FastAPI query parameter validator before
it reaches the database layer. The brief does not specify the ID format (see Open
Questions), but whatever format is agreed, a Pydantic validator or `Query(pattern=...)`
constraint must reject non-conforming values with `422` before any database call is
made.

**Rationale:**  
`customer_id` is user-supplied input arriving via HTTP GET. It enters the application
through the network, passes through the route handler, and is used in a database query.
Raw psycopg2 with string interpolation (`f"SELECT ... WHERE id = '{customer_id}'"`) is
a direct and well-documented SQL injection vector. Parameterised queries eliminate this
class of vulnerability categorically — the database driver handles escaping, and the
query structure is fixed regardless of input content. This is not optional hardening;
it is the baseline for any code that constructs queries from external input.

**Alternatives rejected:**  
- *String interpolation with manual escaping:* Relies on the correctness of every
  future developer who touches the query layer. A single missed escape is a
  vulnerability. Rejected unconditionally.
- *ORM:* The requirements brief explicitly prohibits ORM use. Rejected by constraint.
  Note that parameterised psycopg2 provides equivalent injection safety.

---

### Decision 5: Differentiated error responses with sanitised messages

**What was decided:**  
The API returns distinct HTTP status codes for distinct failure conditions:

| Code | Condition |
|------|-----------|
| `200` | Customer found; data returned |
| `401` | Missing or invalid API key — rejected before any DB call |
| `404` | Customer ID not found in the database |
| `422` | Malformed customer ID — fails input validation |
| `503` | Database unavailable or unhandled connection error |

All error response bodies are structured JSON with a `detail` field containing a
human-readable message safe for client consumption. A FastAPI exception handler
intercepts all unhandled exceptions — including psycopg2 `OperationalError` — and
returns `503` with a generic message. Postgres error text, table names, column names,
stack traces, and connection strings never appear in any response body.

**Rationale:**  
Two distinct problems are solved by this decision. First, operational clarity: a `503`
in the logs means the database is down; a `404` means the customer does not exist.
Collapsing these into a single status code makes diagnosis materially harder. Second,
information hygiene: FastAPI's default unhandled exception behaviour surfaces internal
error details in the response body. An uncaught `psycopg2.OperationalError` will
include the connection string in its message. Explicit exception handlers prevent this.

**Alternatives rejected:**  
- *Return `500` for all server-side errors:* Operationally useless — staff and
  operators cannot distinguish application errors from infrastructure failures.
  Rejected.
- *Return `404` for database unavailability:* Actively misleading. Staff would conclude
  customers do not exist when the system is down. Rejected.
- *Pass Postgres error messages to the client:* Leaks schema and infrastructure
  details. Rejected unconditionally.

---

## 3. Challenge My Decisions

---

### Challenge 1: Does serving the UI and API from the same FastAPI process create unacceptable coupling?

**Strongest argument against:**  
A Jinja2 template rendering failure — a missing template file, a context key error, a
bad deployment — crashes or degrades the same process that serves the API. If an
operations tool has integrated against the `/api/risk` endpoint and the UI template
breaks, the integration goes down with it. In Option B, Nginx serves the UI
independently; a broken HTML template does not touch the API process. Coupling two
concerns in one process violates a basic separation principle and creates a blast radius
larger than either concern individually.

**Verdict: Valid challenge. Accepted as a known tradeoff.**  
The coupling is real. A production system with downstream integrations should separate
UI serving from API serving. However, the requirements brief describes a training demo
system with a fixed two-technology stack (FastAPI + Postgres) and no downstream
integrations specified. The Jinja2 route is one additional route handler — the failure
surface is narrow and the coupling is shallow. The tradeoff is documented here so that
if the system's operational lifetime or integration requirements change, the separation
rationale is already written and the architectural path to Option B is clear.

---

### Challenge 2: Is server-side key injection via Jinja2 actually secure if the page is cached?

**Strongest argument against:**  
If any caching layer — a CDN, a corporate proxy, a browser cache with aggressive
headers — caches the rendered `GET /` response, the injected API key will be served
to subsequent users from cache, potentially after the key has been rotated. Worse, if
the cached response is stored somewhere outside the container (a proxy, a shared
cache), the key may persist beyond the intended recipients. The requirements brief
prohibits external service calls but says nothing about intermediate caching
infrastructure that may exist on the client's internal network.

**Verdict: Valid challenge. Accepted with mitigation.**  
The risk is real in environments with aggressive caching. Mitigation: the `GET /`
route must set `Cache-Control: no-store` response headers explicitly. This is a
single header on one route and must be treated as a required implementation detail,
not optional polish. The README must document this and explain why. If the deployment
environment includes a caching proxy that ignores `Cache-Control`, that is an
infrastructure constraint outside this system's scope — but it must be flagged at
handoff.

---

### Challenge 3: Is a shared API key sufficient given the stated problem?

**Strongest argument against:**  
The core problem identified in the architectural analysis is ungoverned access to
sensitive financial data. A shared API key governs access in the sense that requests
without it are rejected — but it provides no attribution. If the key is compromised,
there is no way to revoke access for one consumer without revoking it for all. The
audit log records that the key was used, not who used it. This partially reproduces the
problem the system was intended to solve: access is mediated by an API, but it remains
unaccountable at the individual level.

**Verdict: Valid challenge. Accepted as a known limitation, not a design failure.**  
Option C's per-consumer key table would address this. Option A does not. This is a
deliberate tradeoff. The limitation must be documented explicitly — in the README and
in any client handoff — so it is not mistaken for a fully solved access-control
problem. If the client considers per-consumer accountability a hard requirement, this
architecture is the wrong choice and the decision must be revisited before build begins.

---

### Challenge 4: Does the 503 / 404 distinction matter enough to justify the implementation complexity?

**Strongest argument against:**  
Operations staff querying the UI see either a result or an error message. Whether the
error reads "customer not found" or "service unavailable" may not change their
immediate action — in both cases they escalate. The custom exception handler for 503 is
extra code that must be written, tested, and maintained. For a system serving a handful
of internal users, this may be overengineered.

**Verdict: Rejected.**  
The audience for the status code distinction is not operations staff reading the UI
message — it is whoever is operating and diagnosing the system. A 503 in the logs or
in a downstream integration's error handling means "infrastructure problem." A 404
means "data problem." Conflating them produces operational noise that makes diagnosis
slower. The implementation cost is one exception handler and a handful of lines. The
diagnostic benefit is concrete and recurring. The challenge is useful as a reminder to
keep the implementation minimal, but does not change the decision.

---

### Challenge 5: Is the two-container topology resilient enough for even a demo deployment?

**Strongest argument against:**  
With only two containers and no process supervisor, any crash in the FastAPI container
takes down both the UI and the API simultaneously with no graceful degradation. There
is no health endpoint for external monitoring, no restart policy specified in the
compose file, and no way for a consumer to distinguish "FastAPI is down" from "the
compose stack isn't running." In Option B, Nginx can serve a maintenance page when
FastAPI is unavailable. In Option A, the user gets a connection refused.

**Verdict: Valid challenge. Accepted with mitigation.**  
For a demo system, full redundancy is out of scope. Mitigations within scope: (1) the
compose file should specify `restart: unless-stopped` on both containers so crashes
trigger automatic restarts; (2) FastAPI should expose a `GET /health` endpoint that
returns `200` when the DB connection is live and `503` when it is not — this costs
almost nothing to implement and gives both the compose health check and any future
monitoring a stable target. The absence of graceful degradation on FastAPI crash is a
documented limitation of the two-container topology, not a fixable design flaw within
the stated constraints.

---

## 4. Key Risks

**R1 — Jinja2 template misconfiguration silently breaks the UI while the API continues.**  
If the template file is missing, misnamed, or contains a rendering error, `GET /`
returns a 500 while `GET /api/risk` continues to function. This is a silent partial
failure — the API works but the UI is inaccessible, and there is no alert. Mitigation:
include a startup check that confirms the template file is present and renderable before
the application accepts traffic.

**R2 — Startup race condition if the Postgres health check is misconfigured.**  
The entire `docker compose up` reliability guarantee depends on the `pg_isready` health
check being configured with the correct database name and user. If these are wrong, the
health check never passes, the FastAPI container never starts, and the stack appears
permanently unhealthy with no actionable error message. This is a high-likelihood
failure for first-time setup if the health check parameters are copied without
verification.

**R3 — customer_id format is undefined, leaving input validation underspecified.**  
The brief does not specify whether customer IDs are integers, UUIDs, or alphanumeric
strings. The FastAPI input validator cannot be meaningfully constrained until the format
is defined. The current design can apply length limits and a conservative character
class as a default, but if the real format differs, the validation layer must be
revisited. Queries using IDs that pass the default validator but fail silently against
real data will return 404s that are indistinguishable from genuine misses.

**R4 — The shared API key is the single point of credential failure.**  
If the key in `.env` is committed to version control, shared insecurely, or appears in
a log line, the entire authentication model is compromised. There is no revocation
mechanism short of changing the key and restarting the stack. For a training system
this is an accepted limitation. For any production use it is not. The risk must be
documented as an explicit boundary in the README.

**R5 — No `Cache-Control` header on the UI route leaks the API key via caching.**  
If `GET /` does not set `Cache-Control: no-store`, the rendered page — which contains
the API key — may be stored by a browser, a corporate proxy, or any intermediate cache.
A cached page served to a different user after key rotation is a credential leak. This
is a single-line mitigation that becomes a vulnerability if omitted.

---

## 5. Key Assumptions

**A1 — customer_id is a short, bounded, alphanumeric string.**  
The system assumes IDs have a maximum length of 64 characters and consist of
alphanumeric characters, hyphens, or underscores. If IDs are free-form strings of
arbitrary length or contain special characters, the input validation layer and the
database column definition must be revisited.

**A2 — risk_factors is a flat list of string labels.**  
The data model assumes risk factors are stored as a Postgres `TEXT[]` array of
human-readable labels (e.g., `["high_transaction_volume", "geographic_flag"]`). If
factors carry associated metadata — severity scores, categories, timestamps — the
schema must change to `JSONB` or a separate table, and the API response shape changes
accordingly.

**A3 — A single shared API key is acceptable for the initial deployment.**  
The system is built around one key in `.env`. This satisfies the brief's authentication
requirement. It does not satisfy per-consumer accountability. This assumption holds for
the stated scope and must be revisited explicitly if the system is adopted for ongoing
use or if individual attribution becomes a requirement.

**A4 — The system runs on a host with Docker and Docker Compose installed.**  
Docker and Docker Compose are treated as given prerequisites. No assumption is made
about the operating system. No install scripts are provided beyond `docker compose up`.

**A5 — The database seed script runs successfully on first container start.**  
The init SQL script populates the database when the Postgres container is first
created. The system has no mechanism to detect or report an empty database. If the seed
script fails silently, all lookups return 404 with no indication that the database is
empty rather than the queried customers being absent.

**A6 — The deployment environment does not have an intermediate caching proxy.**  
The `Cache-Control: no-store` header on `GET /` is assumed to be respected. If the
deployment environment includes a caching layer that ignores this header, the API key
embedded in the rendered UI page may be cached and served to unintended recipients.
This assumption must be confirmed with the client before deployment.

---

## 6. Data Model

### Table: `customers`

The primary and only first-class entity. One row per customer, containing a
pre-assessed risk profile. The values in this table are authoritative and treated as
read-only facts by this service — they were produced by an upstream risk assessment
process that is entirely outside this system's scope.

| Column | Type | Notes |
|--------|------|-------|
| `customer_id` | `VARCHAR(64)` PRIMARY KEY | The external identifier used in API requests. Format is TBD — see Open Questions. Used as the lookup key; never modified. |
| `risk_tier` | `VARCHAR(10)` NOT NULL | Constrained to the values `LOW`, `MEDIUM`, or `HIGH`. A `CHECK` constraint must enforce this at the database level, not only in application logic. |
| `risk_factors` | `TEXT[]` NOT NULL | Array of string labels that contributed to the tier assessment. May be an empty array `{}` for LOW tier customers with no flagged factors. |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT NOW() | Record creation timestamp. Used for seed data verification only. Not exposed by the API and not relevant to consumers. |

**Intentional omissions:**  
No `updated_at` — the system is read-only with no update path, making this field
misleading. No customer PII (name, address, account number) — the brief specifies risk
tier and factors only. If PII is required by downstream consumers, that is a scope
change that must be reviewed before implementation.

---

### Seed data (minimum representative set)

The Postgres init script must seed at least one customer per risk tier before the
service accepts traffic. Recommended minimum of 9–12 records to make the UI
meaningfully testable across all conditions.

| Risk Tier | Min. records | Example `risk_factors` values |
|-----------|--------------|-------------------------------|
| `LOW` | 3 | `{}` or `{"verified_identity"}` |
| `MEDIUM` | 3 | `{"new_account", "moderate_transaction_volume"}` |
| `HIGH` | 3 | `{"high_transaction_volume", "geographic_flag", "unverified_identity"}` |

The seed data must include at least one customer ID that matches the assumed format
(see OQ1) and at least one that does not, to validate that input rejection is working
correctly during development.

---

### Not a first-class entity: the API key

The API key is a single scalar value read from the environment at startup. It has no
table, no schema, and no database representation. It is a configuration value, not a
data entity. If the system evolves to require per-consumer keys with revocation (Option
C), `api_keys` becomes a first-class table with `consumer_name`, `key_hash`, and
`active` columns. Under Option A it does not exist as a database object.

---

## 7. Open Questions

These are unresolved items that directly affect Phase 3 (implementation). They are not
deferred polish. Each one has a decision or assumption baked into the current design
that must be confirmed or corrected before the build is considered complete.

---

**OQ1 — What is the canonical format of customer_id?**  
*Why it matters:* The FastAPI query parameter validator, the Pydantic model or `Query`
pattern constraint, the UI text field's `maxlength` and client-side hint, and the
`VARCHAR(64)` column definition all depend on this. A UUID format needs a different
regex than an integer or an alphanumeric code. The current design defaults to
`VARCHAR(64)` alphanumeric with hyphens and underscores — this assumption will be wrong
if the real format differs and must be corrected before the validator is written.  
*Suggested resolution:* Define the format before writing the route handler. If the
client cannot specify it, document the default assumption explicitly in the README.

---

**OQ2 — What does a risk factor entry contain — a label only, or structured data?**  
*Why it matters:* The schema uses `TEXT[]` — a flat array of string labels. If risk
factors have associated metadata (a severity level, a category, a detection timestamp),
`TEXT[]` is the wrong type and the API response shape must change. Changing the column
type after seed data is loaded requires a migration.  
*Suggested resolution:* Request a sample of real or representative risk factor data
from the client before writing the init SQL. If factors are labels only, `TEXT[]` is
correct. If they carry structure, use `JSONB`.

---

**OQ3 — Should the API key be rotatable without a full stack restart?**  
*Why it matters:* Currently, rotating the key means updating `.env` and running
`docker compose up` again, which restarts both containers. If the client expects to
revoke or rotate credentials without service interruption, the single `.env` scalar
model is insufficient. This is a scope question that points toward Option C if the
answer is yes.  
*Suggested resolution:* Confirm with the client before handoff. If zero-downtime
rotation is a requirement, document it as a scope change and revisit the architecture.

---

**OQ4 — Is there a logging retention or format requirement?**  
*Why it matters:* The problem statement implies auditability is a goal — replacing
ungoverned SQL queries with a controlled interface suggests someone cares about what
was looked up and when. The brief specifies no logging requirements. If the client
expects to answer "what was queried on date X", structured JSON logs to stdout are
needed from day one. Human-readable logs are sufficient for informal observation but
are not machine-parseable.  
*Suggested resolution:* Confirm whether logs need to be machine-parseable. If yes,
configure FastAPI's logging to emit structured JSON. This is a configuration change,
not an architectural one, but it must be decided before deployment.

---

**OQ5 — What is the expected operational lifetime of this system?**  
*Why it matters:* The brief describes this as a training demo system. If it is a
genuine one-time demo, the shared key, lack of TLS, and single-process topology are
acceptable. If it is likely to be adopted as an ongoing internal tool, several decisions
in this architecture become technical debt: the shared key provides no attribution, TLS
is absent, there is no rate limiting, and the UI/API coupling in one process limits
independent scaling. The build decisions made here will need to be explicitly unwound if
the operational context changes.  
*Suggested resolution:* Confirm with the client before handoff. If the answer is
"ongoing use", document which architectural constraints from the brief would need to be
revisited first, and in what order of priority.
