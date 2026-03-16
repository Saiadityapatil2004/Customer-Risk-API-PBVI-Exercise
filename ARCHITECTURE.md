# ARCHITECTURE.md
## Customer Risk API — Option B: Nginx Reverse Proxy + FastAPI + Postgres

**Version:** 0.1 — Pre-build  
**Status:** Design phase. Not yet validated against implementation.  
**Stack:** Docker Compose · Nginx · FastAPI (Python 3.11) · Postgres · Vanilla JS  

---

## 1. Problem Framing

### What this system solves

Internal operations staff need to look up a customer's risk tier and the factors behind
it. Today that means ad-hoc SQL queries against a production Postgres database — no
consistent output format, no access controls, no audit trail. This system replaces that
workflow with a controlled read interface: a single HTTP endpoint that accepts a
customer ID, validates the caller, queries the database, and returns a structured JSON
response. A browser UI makes the same lookup available to non-technical staff without
any code.

The system does not compute risk. It surfaces pre-assessed values that already exist in
the database. All business logic that produced those values is outside scope.

### What this system explicitly does not solve

- **Risk computation.** The database is pre-populated. This service is a read layer only.
- **User identity and authentication.** There is no login system. Access is controlled
  by API key, not by user account. The system cannot distinguish between two people
  sharing the same key.
- **Per-user audit trail.** Logs will record what was queried and when, but not who
  initiated the query beyond the key that was used.
- **Write or update operations.** No endpoint modifies the database. Ever.
- **Production hardening.** TLS termination, rate limiting, and secrets management at
  scale are explicitly out of scope per the requirements brief.

---

## 2. Five Key Design Decisions

---

### Decision 1: Nginx as the sole public-facing surface

**What was decided:**  
All browser traffic — both the UI and API requests — enters through a single Nginx
container. Nginx serves the static HTML/JS UI directly and proxies `/api/*` requests
to the FastAPI container. FastAPI is bound to the internal Docker network only and is
not reachable from outside the compose stack.

**Rationale:**  
This closes the attack surface to a single point. If FastAPI were exposed directly to
the browser, any misconfiguration, unhandled exception, or debug endpoint would be
immediately externally reachable. With Nginx in front, the FastAPI container can be
treated as an internal service. Nginx access logs also provide a request record at the
edge, independent of application-level logging — two separate log streams for the same
request, which is better than one.

**Alternatives rejected:**  
- *Expose FastAPI directly (Option A pattern):* Simpler, but puts FastAPI on the public
  surface. A stack trace or debug mode leak becomes immediately visible to anyone with
  network access. Rejected because the marginal complexity of Nginx is worth the
  reduced exposure.
- *Use FastAPI to serve the UI as well as the API:* Conflates two concerns in one
  process. A template rendering error could bring down the API. Rejected in favour of
  clean separation.

---

### Decision 2: API key injected into the UI via Nginx envsubst at container startup

**What was decided:**  
The API key is stored in the `.env` file and passed to the Nginx container as an
environment variable. A container entrypoint script runs `envsubst` against the HTML
template before Nginx starts, writing the key value into the JavaScript that makes API
calls. The key is never written statically into a file that lives in the repository.

**Rationale:**  
The UI must call the API, and the API requires a key. If the key is hardcoded into the
HTML or JS file, any user who opens browser DevTools can extract it. Server-side
injection at startup means the key value is resolved inside the container before the
file is ever served. The HTML template in the repository contains a placeholder variable
(`$API_KEY`), not a real value.

**Alternatives rejected:**  
- *Hardcode the key in the JS file:* Directly violates the constraint that the key must
  not be statically embedded client-side. Rejected unconditionally.
- *Have the UI call a separate `/config` endpoint to fetch the key:* This is circular —
  the config endpoint would itself need to be protected, or would expose the key
  unauthenticated. Adds a round-trip for no gain. Rejected.
- *Use Jinja2 in FastAPI to serve the UI (Option A):* Works, but puts UI serving inside
  the API process. Nginx envsubst achieves the same injection without coupling the two
  concerns. Rejected in favour of separation.

**What is vague and needs resolution before build:**  
The `.env` file approach means anyone with access to the running container can read the
key via `docker exec`. This is acceptable for an internal training system; it would not
be acceptable in production. This assumption must be stated explicitly in the project
README.

---

### Decision 3: FastAPI startup retry loop with explicit health check dependency

**What was decided:**  
The FastAPI container does not attempt a database connection on import. Instead, it
uses a startup event handler that retries the Postgres connection with exponential
backoff (up to a defined maximum). The Docker Compose `depends_on` configuration uses
the `service_healthy` condition against a Postgres health check, not just
`service_started`. Nginx similarly depends on FastAPI being healthy before it begins
proxying.

**Rationale:**  
`docker compose up` starts containers in dependency order but does not wait for
services inside those containers to be ready. Postgres takes several seconds to
initialise its data directory on first run. Without a retry loop, FastAPI will attempt
its first connection, fail, and crash — and `docker compose up` will show all services
as running while the API is actually dead. This is the single most common failure mode
for compose-based stacks and must be explicitly designed against.

**Alternatives rejected:**  
- *Use `depends_on: db` without a health check:* This only waits for the container to
  start, not for Postgres to accept connections. Rejected — this is the exact failure
  mode we are designing against.
- *Use a `sleep` in the entrypoint script:* Fragile. Sleep duration is a guess, not a
  guarantee. Works on fast machines, fails on slow ones. Rejected in favour of proper
  health-check polling.

---

### Decision 4: Parameterised queries only — no string interpolation in SQL

**What was decided:**  
All database queries use psycopg2 parameterised query syntax (`%s` placeholders) with
parameters passed separately. No part of the `customer_id` value is ever interpolated
into a query string using Python string formatting or concatenation.

**Rationale:**  
`customer_id` is user-supplied input. It enters the system through an HTTP GET
parameter, passes through Nginx unchanged, and arrives at the FastAPI route handler.
Raw psycopg2 with string interpolation (`f"SELECT ... WHERE id = '{customer_id}'"`)
is a direct SQL injection vector. Parameterised queries are not optional hardening —
they are the baseline for any system that touches a database with user input. This
decision also applies to any future query additions; the pattern must be treated as a
project-wide rule, not a per-query choice.

Additionally, `customer_id` must be validated for format and length before it reaches
the database layer. The brief does not specify the ID format (see Open Questions), but
whatever format is defined, a FastAPI path or query parameter validator must reject
non-conforming values with a 422 before any database call is made.

**Alternatives rejected:**  
- *String interpolation with manual escaping:* Manual escaping is error-prone and has
  been the source of injection vulnerabilities in systems that believed they had
  sanitised their inputs. Rejected unconditionally.
- *ORM for query safety:* The brief explicitly prohibits ORM use. Rejected by
  constraint, though parameterised psycopg2 achieves the same safety property.

---

### Decision 5: Differentiated error responses with sanitised messages

**What was decided:**  
The API returns distinct HTTP status codes for distinct failure conditions:
- `200` — customer found, data returned
- `404` — customer ID not found in the database
- `401` — missing or invalid API key (rejected before any database call)
- `422` — malformed customer ID (fails input validation)
- `503` — database unavailable or connection error

Error response bodies are structured JSON with a human-readable `detail` field.
They never contain Postgres error messages, table names, stack traces, or connection
strings. FastAPI exception handlers are explicitly configured to intercept all
unhandled exceptions and return a generic `503` with a safe message.

**Rationale:**  
Undifferentiated errors create two problems. First, operations staff cannot tell whether
a result means "this customer does not exist" or "the system is broken" — both become
operationally meaningless if they look the same. Second, unhandled Postgres exceptions
that bubble up to the default FastAPI error handler will include database internals in
the response body, violating the constraint that schema and infrastructure details must
never reach the client.

**Alternatives rejected:**  
- *Return 500 for all server-side errors:* Technically correct but operationally useless.
  Staff will escalate everything as a system failure. Rejected.
- *Return 404 for database unavailability:* Actively dangerous. Staff would conclude
  customers do not exist when the system is actually down. Rejected.
- *Pass Postgres error messages through to the client:* Violates the implied constraint
  on information leakage. Rejected unconditionally.

---

## 3. Challenge My Decisions

---

### Challenge 1: Is Nginx actually worth it for an internal tool?

**Strongest argument against:**  
This is an internal system accessed by operations staff, likely behind a corporate
network or VPN. The security benefit of putting Nginx in front of FastAPI — preventing
external exposure of the API container — may be irrelevant if the system is never
internet-facing. The Nginx layer adds a third container, a third startup dependency to
manage, a second log stream to interpret, and an `envsubst` entrypoint pattern that is
genuinely non-obvious to engineers who haven't seen it before. For a training demo
system, this complexity may exceed the actual risk being mitigated.

**Verdict: Valid challenge. Partially accepted.**  
The concern about operational complexity is legitimate — particularly for the `envsubst`
pattern, which will need clear documentation. However, the decision is retained for two
reasons: (1) the network isolation benefit holds regardless of whether the system is
internet-facing, because it also protects against lateral movement within the internal
network; (2) this architecture is described as a demo and learning artefact, and
establishing the correct pattern for API-behind-proxy is a legitimate teaching goal even
if the immediate threat model is low. The complexity cost must be paid in documentation,
not in corners cut.

---

### Challenge 2: Is envsubst the right injection mechanism for the API key?

**Strongest argument against:**  
`envsubst` with a shell entrypoint script is a fragile, easy-to-misconfigure pattern.
If the template file uses a variable name not in the environment, `envsubst` silently
replaces it with an empty string — the UI then makes API calls with an empty key, gets
401 errors, and the failure is not immediately obvious. The entrypoint script must also
be carefully scoped (only substitute `$API_KEY`, not every shell variable in the file,
or Nginx config variables like `$uri` will be corrupted). This is a known pitfall of
the pattern.

**Verdict: Valid challenge. Accepted with mitigation.**  
The risk is real. Mitigation: the entrypoint script must use `envsubst '$API_KEY'` with
an explicit variable list, not bare `envsubst`. The compose file must fail fast if
`API_KEY` is not set (use `${API_KEY:?API_KEY is required}` syntax). A startup
validation step that checks the substituted file for the expected value should be
considered. This does not change the decision — envsubst remains the most appropriate
tool for this pattern — but it must be implemented carefully and documented.

---

### Challenge 3: Is a shared API key sufficient given the stated problem?

**Strongest argument against:**  
The core problem identified in the architectural analysis (Move 1) is ungoverned access
to sensitive data. A shared API key governs access in the sense that requests without
it are rejected — but it provides no attribution. If two teams and one downstream tool
all use the same key, and that key is compromised, there is no way to revoke access for
one consumer without revoking it for all. The audit log records that the key was used,
not who used it. This partially reproduces the problem the system was supposed to solve:
access is now mediated by an API, but it is still not individually accountable.

**Verdict: Valid challenge. Accepted as a known limitation, not a design failure.**  
Option C (per-consumer key table) would address this. Option B with a shared key does
not. This is a deliberate tradeoff accepted when Option B was selected. The limitation
must be documented explicitly — both in the README and in any handoff to the client —
so it is not mistaken for a fully solved access-control problem. If the client considers
per-consumer accountability a hard requirement, this architecture is the wrong choice
and the decision should be revisited before build begins.

---

### Challenge 4: Does the 503 vs 404 distinction matter in practice?

**Strongest argument against:**  
Operations staff querying the UI will see either a result or an error message. Whether
that error says "customer not found" (404) or "service unavailable" (503) may not
change their immediate action — in both cases they will call someone. Adding explicit
`503` handling with a custom exception handler is code complexity that might be
overengineered for a system that will serve a handful of internal users.

**Verdict: Rejected.**  
The distinction is not about the UI message presented to staff — it is about the
signal available to whoever is operating the system. A 503 in the logs means the
database is down. A 404 means a lookup miss. Conflating these two conditions makes
operational diagnosis materially harder. The code cost is low (one exception handler,
one additional status code). The operational benefit is concrete. The challenge is
retained as a useful prompt to keep the error handling implementation simple, but does
not change the decision.

---

## 4. Key Risks

**R1 — envsubst misconfiguration silently breaks authentication.**  
If the `envsubst` entrypoint is scoped incorrectly or the `API_KEY` variable is empty,
the UI will embed an empty string as the API key. Every request from the UI will receive
a 401. The failure looks identical to a broken API, not a misconfigured key. This risk
is high-likelihood if the entrypoint is implemented without explicit variable scoping
and startup validation.

**R2 — Startup race condition if health checks are misconfigured.**  
The entire `docker compose up` reliability guarantee depends on the Postgres health
check being configured correctly. If the health check command is wrong (e.g., checks
the wrong database name, or uses a user that doesn't exist yet), the health check will
never pass and the stack will never become ready. This is a silent failure — compose
will report the Postgres container as unhealthy indefinitely with no clear error.

**R3 — customer_id format is undefined, leaving input validation without a spec.**  
The brief does not specify whether customer IDs are integers, UUIDs, or alphanumeric
strings. Without a defined format, the FastAPI input validator cannot be meaningfully
constrained. The current design can implement length limits and character class
restrictions as a reasonable default, but the correct constraint cannot be confirmed
until the ID format is agreed. If the format assumption turns out to be wrong, the
validation layer must be revisited.

**R4 — The shared API key becomes the single point of credential failure.**  
If the key in `.env` is committed to version control, shared insecurely, or leaked via
a log message, the entire authentication model is compromised. There is no revocation
mechanism short of changing the key and redeploying. For a training system this is
acceptable; for any production use it is not. The risk must be documented as a known
boundary of this design.

---

## 5. Key Assumptions

**A1 — customer_id is a short, bounded string.**  
The system assumes customer IDs have a reasonable maximum length (e.g., ≤ 64
characters) and consist of alphanumeric characters, hyphens, or underscores. If IDs
are free-form strings of arbitrary length or contain special characters, the input
validation layer must be redesigned.

**A2 — The risk_factors field is a list of strings.**  
The data model assumes risk factors are stored as an array of human-readable strings
(e.g., `["high_transaction_volume", "new_account", "geographic_flag"]`). If factors
have structure beyond a label (e.g., a severity score, a timestamp, a category), the
schema and API response shape must change.

**A3 — A single shared API key is acceptable for the initial deployment.**  
The system is designed around one key in `.env`. This satisfies the brief's requirement
for API key authentication. It does not satisfy per-consumer accountability. This
assumption holds for the stated scope and must be explicitly revisited if the system
is used in a context where individual attribution matters.

**A4 — The system runs on a host with Docker and Docker Compose installed.**  
No assumption is made about the operating system. Docker and Docker Compose are treated
as given prerequisites. No install scripts or setup steps beyond `docker compose up`
are provided.

**A5 — The database is pre-populated before the service is first used.**  
The init SQL script seeds the database at first container start. The system has no
mechanism to detect or report an empty database. If the seed script fails silently, all
lookups will return 404 with no indication that the database is empty rather than the
customers being absent.

---

## 6. Data Model

### Table: `customers`

The primary entity. One row per customer with a pre-assessed risk profile.

| Column | Type | Notes |
|--------|------|-------|
| `customer_id` | `VARCHAR(64)` PRIMARY KEY | External identifier used in API requests. Format TBD — see Open Questions. |
| `risk_tier` | `VARCHAR(10)` NOT NULL | Constrained to `LOW`, `MEDIUM`, or `HIGH`. |
| `risk_factors` | `TEXT[]` NOT NULL | Array of factor labels that contributed to the tier assessment. May be empty for LOW tier customers. |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT NOW() | Record creation timestamp. Used for seed data verification, not exposed by the API. |

**What this entity represents:** A snapshot of a customer's assessed risk status. The
values are authoritative — they were placed here by an upstream risk assessment process
that is outside this system's scope. This service treats them as read-only facts.

**Intentional omissions:** No `updated_at` field — the system is read-only and has no
update path. No customer PII (name, address, etc.) — the brief specifies risk tier and
factors only. If PII is required by downstream consumers, that is a scope change.

---

### Seed data (minimum representative set)

The init script must include at least one customer for each risk tier. Recommended
minimum of 9–12 records (3–4 per tier) to make the UI meaningfully testable.

| Risk Tier | Minimum seed count | Example factors |
|-----------|--------------------|-----------------|
| LOW | 3 | `[]` or `["verified_identity"]` |
| MEDIUM | 3 | `["new_account", "moderate_transaction_volume"]` |
| HIGH | 3 | `["high_transaction_volume", "geographic_flag", "unverified_identity"]` |

---

### Not a first-class entity: API keys

In this architecture, the API key is a single value in the `.env` file — it is not
stored in the database and has no table. It is a configuration value, not a data
entity. If this changes (i.e., if the system moves to Option C's per-consumer key
model), `api_keys` becomes a first-class table. Under Option B it does not.

---

## 7. Open Questions

These are unresolved items that directly affect Phase 3 (implementation). They are not
deferred polish — they must be answered before the build can be considered complete and
correct.

---

**OQ1 — What is the format of customer_id?**  
*Why it matters:* The FastAPI input validator, the UI text field constraints, and the
database column type all depend on this. A UUID needs a different regex than an integer
or an alphanumeric code. Until this is defined, the validation layer is built on an
assumption that may need to be changed.  
*Suggested resolution:* Define the format before writing the route handler. If the
client cannot specify it, default to `VARCHAR(64)` alphanumeric with hyphens and
document the assumption explicitly.

---

**OQ2 — Should the API key be rotatable without a full redeploy?**  
*Why it matters:* Currently, rotating the key means changing `.env` and running
`docker compose up` again. For a training system this is probably fine. If the client
expects to be able to revoke or rotate credentials without downtime, the single `.env`
key model is insufficient and the design must change.  
*Suggested resolution:* Confirm with the client whether zero-downtime key rotation is
a requirement. If yes, this is a scope change that points toward Option C.

---

**OQ3 — What does a risk factor look like — label only, or structured?**  
*Why it matters:* The current data model stores risk factors as `TEXT[]` — an array of
string labels. If factors have associated metadata (severity, category, timestamp), the
schema must change to `JSONB` or a separate `risk_factor_details` table, and the API
response shape changes accordingly.  
*Suggested resolution:* Request a sample of real or representative risk factor data
from the client before finalising the schema.

---

**OQ4 — Is there a logging retention or format requirement?**  
*Why it matters:* The problem statement implies that auditability matters — replacing
ad-hoc SQL queries with a governed interface suggests someone cares about what was
looked up. But the brief specifies no logging requirements. If the client expects to be
able to answer "who looked up customer X on date Y", structured logs with a consistent
format are needed from day one.  
*Suggested resolution:* Confirm whether logs need to be machine-parseable (e.g., JSON
structured logging to stdout) or whether human-readable logs are sufficient. This
affects the FastAPI logging configuration but not the architecture.

---

**OQ5 — What is the expected operational lifetime of this system?**  
*Why it matters:* The brief calls this a training demo system. If it is genuinely a
one-time demo, the shared-key limitation and lack of TLS are acceptable. If it is
likely to be adopted as an ongoing internal tool, several decisions in this architecture
(shared key, no TLS, no rate limiting) become technical debt that will need to be
addressed. The build decisions made here will feel different depending on the answer.  
*Suggested resolution:* Confirm with the client before handoff. If the answer is
"ongoing use", document explicitly which constraints from the brief would need to be
revisited and in what priority order.
