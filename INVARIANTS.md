# INVARIANTS.md
## Customer Risk API — Option A: FastAPI + Postgres

**Version:** 0.2  
**Status:** Updated against ARCHITECTURE.md v0.1. All invariants derived from architecture sections 1–7 and the original invariant set.  
**Coverage note:** Where an invariant closes a gap identified during architecture review, the source section is cited inline.

---

## How to read this document

Each invariant states a condition that must hold at all times for this system to be
considered correct. Invariants are grouped by concern. Within each group, the invariant
text is the normative statement — rationale and references are non-normative commentary
to explain why the invariant exists and where to look if it is violated or disputed.

An implementation that satisfies the requirements brief but violates any invariant here
is incorrectly built.

---

## Group 1: Data Integrity

### INV-1.1 — Passthrough fidelity

The API must return the risk tier and risk factors stored in the database for the
requested `customer_id`, without alteration. No route handler, middleware component, or
response serialiser may modify, reorder, reinterpret, or supplement these values before
they reach the caller.

*Source: original invariant set (Data Correctness); §1 Problem Framing.*

---

### INV-1.2 — No risk computation

No endpoint, route handler, background task, or middleware component may derive,
calculate, or recalculate a risk tier or risk factor value. The service is a read layer
over pre-assessed data. Any code path that produces a risk value rather than retrieves
one is a violation of this invariant, regardless of whether the result matches what the
database contains.

*Source: §1 — "The system does not compute risk." This is distinct from INV-1.1, which
governs fidelity of retrieval; INV-1.2 governs the absence of computation entirely.*

---

### INV-1.3 — Valid tier values only

The API must only ever return one of the three enumerated risk tier values: `LOW`,
`MEDIUM`, or `HIGH`. A `CHECK` constraint enforcing this must exist at the database
level. Application-level validation of the tier value is a secondary control, not a
substitute for the database constraint.

*Source: §6 Data Model — "A `CHECK` constraint must enforce this at the database level,
not only in application logic."*

---

### INV-1.4 — Empty risk factors list is a valid response

A `risk_factors` value of an empty list (`[]`) is a valid and expected API response.
It must not be treated as an error, a missing field, or a data anomaly. This is the
defined state for LOW-tier customers with no flagged factors.

*Source: §5 Assumption A2; §6 Data Model.*

---

### INV-1.5 — No write operations

No endpoint, route handler, or database call may issue an INSERT, UPDATE, DELETE, or
DDL statement against the database. All database access must be read-only SELECT
queries. There are no exceptions.

*Source: original invariant set (Mutability); §1 — "Write or update operations: no
endpoint modifies the database under any circumstances."*

---

## Group 2: Response Contract

### INV-2.1 — Required response fields

Every successful (HTTP 200) response must contain exactly the following fields at the
top level of the JSON body: `customer_id`, `risk_tier`, `risk_factors`. No field in
this set may be absent or null in a 200 response.

*Source: original invariant set (Response Shape).*

---

### INV-2.2 — No additional fields beyond scope

A successful response must not include fields beyond `customer_id`, `risk_tier`, and
`risk_factors`. In particular, the `created_at` timestamp column and any other
database column not specified in the response contract must never appear in a response
body. Adding PII fields is a scope change requiring explicit review before
implementation, not a configuration decision.

*Source: §6 Data Model — "Not exposed by the API and not relevant to consumers."*

---

### INV-2.3 — Existence mapping

If a `customer_id` exists in the database and the request is otherwise valid, the API
must return HTTP 200 with the corresponding data. If the `customer_id` does not exist
in the database, the API must return HTTP 404. These two conditions must never be
conflated.

*Source: original invariant set (Existence Mapping).*

---

### INV-2.4 — Malformed input returns 422 before any database call

A `customer_id` that does not conform to the agreed format constraint (maximum 64
characters; alphanumeric characters, hyphens, and underscores only, per Assumption A1)
must be rejected with HTTP 422 by the FastAPI input validator. The 422 rejection must
occur before any database call is made. A malformed `customer_id` must never reach the
query layer.

*Source: §2 Decision 4 — "a Pydantic validator or `Query(pattern=...)` constraint must
reject non-conforming values with `422` before any database call is made." Gap flagged
in architecture review against §2 Decision 4.*

---

### INV-2.5 — Database unavailability returns 503, not 404 or 200

If the database is unavailable or returns an unhandled connection error, the API must
return HTTP 503 with a generic, sanitised error message. It must never return HTTP 404
(which means the customer does not exist) or HTTP 200 (which means the lookup
succeeded) under a database failure condition.

*Source: §2 Decision 5; Challenge 4 verdict. Gap flagged in architecture review against
§2 Decision 3.*

---

### INV-2.6 — Complete and non-overlapping status code mapping

The API must return HTTP status codes according to this exhaustive mapping, with no
collapsing of distinct conditions into a shared code:

| Code | Condition |
|------|-----------|
| `200` | Customer found; data returned |
| `401` | Missing or invalid API key |
| `404` | Customer ID not found in the database |
| `422` | Malformed customer ID — fails input validation |
| `503` | Database unavailable or unhandled connection error |

Any deviation from this mapping — including returning `500` for any condition listed
above — is a violation of this invariant.

*Source: §2 Decision 5. Gap flagged in architecture review against §2 Decision 5.*

---

### INV-2.7 — All error responses use sanitised messages

All error response bodies must be structured JSON containing a `detail` field with a
human-readable message safe for client consumption. Postgres error text, table names,
column names, stack traces, connection strings, and any other internal system
information must never appear in any response body. FastAPI's default unhandled
exception behaviour must be overridden by explicit exception handlers.

*Source: original invariant set (Error Surfaces); §2 Decision 5.*

---

## Group 3: Authentication and Credential Handling

### INV-3.1 — API key required on all data endpoints

No caller may access customer risk data without a valid API key. All routes under
`/api/*` must enforce API key validation via a FastAPI dependency function that runs
before any route handler logic and before any database call. A request with a missing
or non-matching `X-API-Key` header must be rejected with HTTP 401 without executing
any downstream logic.

*Source: original invariant set (Authentication); §2 Decision 2.*

---

### INV-3.2 — API key must not appear in responses, logs, or error messages

The API key value must never appear in any API response body, HTTP header, log line, or
error message, in any context.

*Source: original invariant set (Credential Handling).*

---

### INV-3.3 — API key must not exist as a literal value in repository files or container images

The API key must not be hardcoded in any file committed to the repository, including
HTML templates, JavaScript files, configuration files, Dockerfiles, or compose
definitions. The Jinja2 template file must contain the placeholder `{{ api_key }}`, not
a resolved value. The key must only exist as a runtime environment variable resolved
from `.env` at container startup.

*Source: §2 Decision 1 and Decision 2 — "The template file in the repository contains a
Jinja2 placeholder, not a real value." Gap flagged in architecture review against §2
Decision 2.*

---

### INV-3.4 — UI route must set Cache-Control: no-store

The `GET /` route that renders and serves the browser UI must set the response header
`Cache-Control: no-store` on every response. Because the rendered page contains the API
key injected by Jinja2, caching any response from this route constitutes a credential
leak. This header is a required security control, not optional polish.

*Source: §3 Challenge 2 verdict; §4 Risk R5. Gap flagged in architecture review against
§2 Decision 2.*

---

## Group 4: Input Handling and Query Safety

### INV-4.1 — Parameterised queries only

All database queries must use psycopg2 parameterised syntax, with user-supplied values
passed as a separate parameter tuple. No part of any user-supplied value may be
concatenated or interpolated into a query string at any point in the codebase. This is
a project-wide rule applied uniformly to all query sites — it is not a per-query
judgement call, and it applies to all future additions to the query layer.

*Source: §2 Decision 4. Gap flagged in architecture review against §2 Decision 4.*

---

### INV-4.2 — Validator and schema must be consistent with the agreed customer_id format

The FastAPI input validator (the `Query(pattern=...)` constraint or Pydantic model),
the database column definition (`VARCHAR(64)`), and the UI text field `maxlength`
attribute must all reflect the same agreed `customer_id` format. If the format
assumption changes (per OQ1), all three must be updated in the same change. An
implementation where any of these three artefacts accepts a value the others reject
is a violation of this invariant.

*Source: §7 OQ1; §5 Assumption A1. Gap flagged in architecture review against §7.*

---

## Group 5: Topology and Isolation

### INV-5.1 — No external network calls

The system must not make any external network calls at any point during normal
operation. All data access must occur within the Docker Compose environment. The
FastAPI container may communicate only with the Postgres container on the internal
compose network.

*Source: original invariant set (Operational Isolation).*

---

### INV-5.2 — Exactly two network-accessible services

The compose stack must contain exactly two services: the FastAPI application container
and the Postgres database container. No additional container — including reverse
proxies, cache layers, or sidecar processes — may be introduced as a network-accessible
service. A third container creates an uncontrolled network path that may bypass
authentication controls.

*Source: §2 Decision 1. Gap flagged in architecture review against §2 Decision 1.*

---

### INV-5.3 — UI failure must not degrade API availability

A failure in the `GET /` Jinja2 rendering route — including a missing template file, a
context key error, or a rendering exception — must not affect the availability or
correctness of the `GET /api/risk` endpoint. The two routes share a process, but their
failure surfaces must be independent. A startup check must confirm that the template
file is present and renderable before the application begins accepting traffic.

*Source: §3 Challenge 1 verdict; §4 Risk R1. Gap flagged in architecture review against
§3 Challenge 1.*

---

## Group 6: Startup and Readiness

### INV-6.1 — No traffic accepted before database connection is established

The FastAPI container must not accept or respond to any request before the connection
pool to Postgres has been successfully established. The `lifespan` startup handler must
complete its retry loop and confirm a live connection before the application is marked
ready. The compose `depends_on` configuration must use the `service_healthy` condition
against a `pg_isready` health check — not `service_started`.

*Source: §2 Decision 3. Gap flagged in architecture review against §2 Decision 3.*

---

### INV-6.2 — Seed data must be present before the service accepts traffic

The service must not accept traffic against a database that has not been seeded. The
Postgres init script must populate at least one customer record for each risk tier
(`LOW`, `MEDIUM`, `HIGH`) before the database container is considered healthy. If the
seed script fails, the system must surface a detectable failure condition rather than
silently returning 404 for all lookups.

*Source: §5 Assumption A5; §6 Seed data. Gap flagged in architecture review against §4
Risk R5 and §5 Assumption A5.*

---

## Group 7: Health and Observability

### INV-7.1 — Health endpoint must reflect live database connectivity

The `GET /health` endpoint must return HTTP 200 when the database connection is live and
HTTP 503 when it is not. It must be kept current with the actual state of the database
connection — it must not cache or hardcode its result. This endpoint is the stable
target for the compose health check and for any future monitoring integration.

*Source: §3 Challenge 5 verdict.*

---

### INV-7.2 — Logs must record what was queried and when

Every request to `GET /api/risk` must produce a log entry recording at minimum the
`customer_id` that was queried and the timestamp of the request. The API key value must
not appear in any log entry (see INV-3.2). Whether logs are structured JSON or
human-readable is subject to OQ4, but the minimum content requirement holds regardless
of format.

*Source: §1 — "Logs will record what was queried and when"; §7 OQ4.*

---

## Group 8: UI Behaviour

### INV-8.1 — UI must display API response data without modification

The browser UI must display the data returned by the API exactly as received. It must
not modify, reinterpret, reformat, or recalculate the risk tier or risk factors before
presenting them to the user.

*Source: original invariant set (UI Fidelity).*

---

### INV-8.2 — UI must display errors returned by the API

When the API returns an error response (401, 404, 422, 503), the UI must surface the
error to the user in a visible, distinguishable way. It must not silently suppress
errors, display a blank result, or present an error state as a successful empty response.

*Source: §2 Decision 5; original invariant set (UI Fidelity) by extension.*

---

## Appendix A: Open Questions that affect invariants

The following open questions from §7 of ARCHITECTURE.md have direct bearing on
invariants in this document. Resolution of each question may require an invariant to
be updated.

| OQ | Invariant affected | Impact if unresolved |
|----|-------------------|----------------------|
| OQ1 — `customer_id` format | INV-4.2, INV-2.4 | Validator pattern, column definition, and UI field cannot be correctly specified |
| OQ2 — `risk_factors` structure | INV-2.1, INV-1.4 | Schema type (`TEXT[]` vs `JSONB`) and response shape depend on this |
| OQ4 — Logging format | INV-7.2 | Structured vs human-readable log format is unresolved; minimum content requirement holds either way |
| OQ5 — Operational lifetime | All invariants | If the system moves to production use, several invariants (INV-3.1, INV-5.2, INV-7.2) become insufficient and must be strengthened |

---

## Appendix B: Known limitations that invariants do not resolve

The following are accepted limitations of the current architecture. They are documented
here to prevent them from being mistaken for invariant violations or design oversights.

**Shared API key provides no per-consumer attribution.** INV-3.1 enforces that a valid
key is required. It does not and cannot enforce that the key identifies an individual.
If per-consumer accountability is required, the architecture must change to Option C
before build. This is not a fixable constraint within Option A.

**Key rotation requires a full stack restart.** There is no mechanism to revoke or
rotate the API key without restarting both containers. For a training demo system this
is accepted. It is not acceptable for production use. See OQ3.

**The deployment environment is assumed not to have a caching proxy that ignores
`Cache-Control`.** INV-3.4 requires the `no-store` header to be set. If intermediate
infrastructure ignores this header, the invariant cannot be satisfied by the application
alone. This must be confirmed with the client before deployment.
