# exercise-04 — SaaS platform integration: Salesforce, ServiceNow, Snowflake · Reference Solution

## 1. Solution overview

The deliverable is an **integration design** for three SaaS platforms a
forward-deployed engineer meets in almost every enterprise engagement:

- **Salesforce** — the customer's CRM. Lantern reads Accounts/Opportunities
  as context for sales-ops assistants, writes activity records, and
  subscribes to change events.
- **ServiceNow** — the customer's ITSM/ticketing system. Lantern reads
  tickets for triage context, creates tasks, and listens to state
  transitions for workflow automation.
- **Snowflake** — the customer's analytics warehouse. Lantern reads
  curated views for RAG grounding, pushes down summaries, and writes
  back structured outputs under governed schemas.

Each platform has a distinct contract and set of guardrails; the FDE
mistake is to use "we just hit the REST API" as the design. The real
design job is:

1. **Pick the right surface for the job.** Each platform has multiple
   APIs with different costs, limits, and governance stories. Bulk
   vs. REST vs. Streaming vs. pushdown is a load and latency decision,
   not a style preference.
2. **Respect the governance model.** Salesforce profiles/permission
   sets, ServiceNow ACLs, Snowflake RBAC + masking policies — the
   integration runs under a service principal whose grants are the
   customer's security review.
3. **Design for the platform's rate limits.** Every one has quotas that
   silently cap your throughput. The integration spec names the limit
   and the budget you'll use.
4. **Keep a replayable, auditable trail.** All three platforms are
   systems-of-record; auditors will ask what Lantern read and wrote.

## 2. Implementation — three SaaS integrations

### 2.1 Salesforce

**Surface selection:**

| Need                                       | Right surface                                   |
|--------------------------------------------|-------------------------------------------------|
| Read 20 records to answer a chat query     | REST API `/services/data/vXX.X/query`           |
| Read 10M records for an eval set           | **Bulk API 2.0** (`/services/data/vXX.X/jobs/query`) |
| Subscribe to record changes                | **Change Data Capture (CDC) via Pub/Sub API** (gRPC) or Platform Events |
| Write ~1M activity records                 | Bulk API 2.0 (`jobs/ingest`), not REST          |
| Call out from Salesforce to Lantern        | Flow + Named Credential → HTTPS callout         |

**Authentication:** OAuth 2.0 Connected App with the **JWT Bearer Flow**
for M2M (RFC 7523 + Salesforce's extension). No username/password; no
security tokens in config.

- Connected App registered by the customer's Salesforce admin.
- Lantern holds the signing key in KMS; signs a JWT assertion with
  `iss=<consumer_key>`, `aud=https://login.salesforce.com`, `sub=<service user>`.
- Token exchange at `/services/oauth2/token`, short-lived access token
  (default ~2 hours), re-minted on expiry.
- Minimum scopes: `api refresh_token` only if you truly need refresh;
  pure M2M needs neither.

**Rate-limit model:**

- Per-org daily API request allotment (sometimes called API Request Limit).
- Per-user concurrent long-running request cap.
- Bulk API has its own job and batch quotas.
- Platform Events have daily delivery volumes.

Lantern must:

- Read `Sforce-Limit-Info` headers on every response; emit a
  `salesforce.api.remaining` gauge; alert at ≤20%.
- Cap concurrent connections per org at a documented budget (e.g. 20
  at steady state; burst to 50 only under on-call guidance).
- Prefer Bulk 2.0 for anything larger than a few thousand records.
- Partition long-running polling across a schedule so no single window
  monopolizes the org's quota.

**Governance:**

- Lantern's service user has a **Permission Set** with field-level
  grants documented in the integration spec. Profile-based all-access
  service users are a review blocker.
- Field-level audit: Lantern records the exact SOQL and the field list
  per call; retention ≥90 days.
- Masking: fields flagged PII in the customer's data catalog are either
  not selected or passed through a redaction step before reaching the
  model. See mod-105 for the grounding-layer rules.

**Example queries and writes:**

```sql
-- SOQL for context fetch (read-side)
SELECT Id, Name, StageName, Amount, CloseDate, Owner.Email
FROM Opportunity
WHERE AccountId = :accountId
  AND LastModifiedDate >= LAST_N_DAYS:30
ORDER BY LastModifiedDate DESC
LIMIT 25
```

```http
# Bulk 2.0 ingest (write-side) — pattern
POST /services/data/v62.0/jobs/ingest
Content-Type: application/json
{ "object": "Task", "contentType": "CSV", "operation": "insert" }

PUT  /services/data/v62.0/jobs/ingest/{jobId}/batches   (CSV body)
PATCH /services/data/v62.0/jobs/ingest/{jobId}         { "state": "UploadComplete" }
```

**Change capture (Pub/Sub API):**

- gRPC stream on `api.pubsub.salesforce.com:7443`, Avro-encoded events.
- Replay with a monotonic Replay ID — Lantern persists the Replay ID
  after each successful downstream commit; on reconnect, resumes from
  `replay_id - 1` to guarantee at-least-once.
- Lantern's consumer must be idempotent (keyed by Salesforce record Id
  + ChangeEventHeader.commitNumber).

### 2.2 ServiceNow

**Surface selection:**

| Need                                   | Right surface                                                 |
|----------------------------------------|---------------------------------------------------------------|
| Read a ticket or a list of tickets     | Table API (`/api/now/table/{table}`)                          |
| Read with fewer admin privileges       | Scripted REST API (customer-exposed, scoped read)             |
| Bulk read                              | Table API with pagination + ETag                              |
| Create or update a ticket              | Table API POST/PATCH                                          |
| Attachments                            | Attachment API                                                |
| Subscribe to state changes             | **Business Rules → outbound REST message** to Lantern webhook, or MID-Server for private tenants |
| Long-running workflow from ServiceNow  | Flow Designer → Spoke action → Lantern API                    |

**Authentication:** OAuth 2.0 with the customer's internal identity
provider is the default. ServiceNow also supports Basic (don't), API
tokens (avoid), or inbound OAuth with JWT bearer flow (good).

- Lantern registers an OAuth Application Registry entry in the
  customer's instance.
- `grant_type=client_credentials` for M2M; or `jwt-bearer` where supported.
- Scope the application to a dedicated **sys_user** with a specific
  **role** and ACLs; never give it `admin`.

**Rate-limit model:**

- Inbound REST API rate limit rules (per user / per instance / per
  endpoint) — ServiceNow enforces and returns 429 with `X-RateLimit-*`
  headers.
- Transaction quota — long-running queries get killed by the transaction
  quota rule even if the HTTP limit is fine.
- Table API `sysparm_limit` default is low (10); must be set explicitly.

Lantern must:

- Honor `X-RateLimit-Remaining` and backoff on 429 with `Retry-After`.
- Use display-value and reference-link controls (`sysparm_display_value`,
  `sysparm_exclude_reference_link`) to avoid N+1-style lookups.
- Request only the fields needed (`sysparm_fields`) — the default
  response is huge and costs quota.
- Paginate using `sysparm_offset`/`sysparm_limit` or the `Link` header;
  checkpoint the offset so retries resume.

**Governance:**

- Lantern's service user has a **scoped role** with ACLs that read
  only the tables/fields in scope. Document the role grants in the
  integration spec.
- Script sandboxing: any customer-side Scripted REST that proxies
  Lantern reads must go through the customer's security review (see
  OWASP ML Top 10 — input validation if Lantern output lands back in
  ServiceNow).

**Example queries:**

```http
GET /api/now/table/incident
    ?sysparm_query=assignment_group=javascript:gs.getUser().getMyGroups()^active=true
    &sysparm_fields=number,short_description,state,priority,caller_id.display_name
    &sysparm_display_value=true
    &sysparm_limit=100
    &sysparm_offset=0
Accept: application/json
Authorization: Bearer <token>
```

```http
POST /api/now/table/incident
Content-Type: application/json
{
  "short_description": "Lantern flagged anomaly in run <id>",
  "assignment_group": "<sys_id>",
  "caller_id": "<sys_id>",
  "u_lantern_run_id": "<id>"
}
```

**Change capture:** no native stream equivalent. The customer defines
a Business Rule on state change that calls Lantern's webhook with the
record sys_id; Lantern re-reads the record to get a consistent view.
Protect the webhook with HMAC + shared secret per ex-03 §2.4.

### 2.3 Snowflake

**Surface selection:**

| Need                                     | Right surface                                                         |
|------------------------------------------|-----------------------------------------------------------------------|
| Interactive query from Lantern           | Snowflake SQL API (`/api/v2/statements`) or driver (JDBC/Python)      |
| Bulk ingest from Lantern into Snowflake  | Snowpipe (continuous) or `COPY INTO` from internal stage              |
| Bulk read for eval sets                  | `COPY INTO <external stage>` → object storage → Lantern reader        |
| Change capture out of Snowflake          | **Streams** + **Tasks**                                               |
| Run model-flavored logic in-warehouse    | **Snowpark** (Python) with network-bound restrictions                 |
| Govern access                            | Role hierarchy + Masking Policies + Row Access Policies               |

**Authentication:** key-pair authentication (RSA) for M2M; OAuth 2.0 via
the customer's IdP for user-authenticated sessions. **Never** username +
password for service principals.

- Service user `LANTERN_SVC` with public key registered in Snowflake.
- Private key in KMS; Lantern signs JWTs per Snowflake's key-pair flow.
- For OAuth, use the customer's external OAuth integration (Okta /
  Azure AD / Ping) with token-exchange where Alice's identity must
  reach Snowflake (see exercise-02 §2.4).

**Rate-limit model:**

- Warehouse concurrency (statements per warehouse size).
- Query queueing with `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS`.
- Account-level API rate limits (SQL API, REST APIs).

Lantern must:

- Use a dedicated, auto-suspending warehouse (`LANTERN_WH`) sized to
  the workload; separate from BI warehouses to avoid noisy-neighbor.
- Set `STATEMENT_TIMEOUT_IN_SECONDS` on every session to a documented
  cap; never run unbounded queries.
- Prefer async execution via the SQL API's `asyncExec=true` for long
  queries; poll by `statementHandle`.
- Push filters down; never `SELECT *` from a fact table.

**Governance:**

- `LANTERN_SVC` lives in a dedicated role (`LANTERN_READER`) with
  grants on exactly the schemas and views Lantern needs.
- All reads go through **governed views** owned by the customer's
  data team — not base tables. Views enforce:
  - Row Access Policies (tenant isolation)
  - Masking Policies on PII columns
  - Column selection down to only what the use case needs
- Writes land in a schema `LANTERN_OUT.<run_id>` with retention
  policies; the customer's data team owns promotion.
- Query tagging (`ALTER SESSION SET QUERY_TAG = 'lantern/<run_id>'`)
  so audit links the query to the Lantern run.

**Example — governed view + query:**

```sql
-- Customer-side: governed view with masking + row access
CREATE MASKING POLICY mask_email AS (val string) RETURNS string ->
  CASE
    WHEN CURRENT_ROLE() IN ('LANTERN_READER') THEN REGEXP_REPLACE(val,'(.{2}).*@','\\1***@')
    ELSE val
  END;

CREATE OR REPLACE VIEW ANALYTICS.V_CUSTOMER_FOR_LANTERN AS
SELECT customer_id, email, segment, lifetime_value
FROM ANALYTICS.CUSTOMER
WHERE tenant_id = CURRENT_TENANT();   -- row access via policy

-- Lantern-side: query the view only
SELECT customer_id, email, segment, lifetime_value
FROM ANALYTICS.V_CUSTOMER_FOR_LANTERN
WHERE segment = :segment
LIMIT 500;
```

**Change capture (Streams + Tasks):**

- `CREATE STREAM lantern_feed ON TABLE orders;` captures CDC.
- A `TASK` consumes the stream on a cron and either writes to an
  export stage Lantern reads, or sends a Snowpipe notification.
- Lantern's reader processes deltas with idempotent upserts keyed by
  primary key; retries no-op if the delta was already applied.

**Ingest:** Lantern pushes model-structured outputs via `COPY INTO`
from an internal stage, in Parquet, partitioned by `run_date`. For
near-real-time, Snowpipe with an event-driven loader off object
storage notifications.

### 2.4 Cross-platform concerns

- **Credential rotation runbook.** Each platform's service principal
  has keys with a documented rotation cadence (90 days default). The
  runbook lives in the integration folder, with a last-rotated date.
- **Sandbox-first.** Every change deploys first to the customer's
  sandbox (SF sandbox, ServiceNow sub-prod instance, Snowflake
  non-prod account) with an identical shape; promotion is gated by
  tests in the sandbox.
- **Platform change management.** Salesforce releases three times a
  year; ServiceNow family upgrades roughly yearly; Snowflake
  release versions behavior-change by default. Subscribe the
  integration owner to each platform's release notes.
- **Observability.** Per-platform SLIs:
  - API quota remaining (gauge)
  - Request p95 latency
  - Error rate per endpoint
  - Rate-limit hit count (429)
  - Change-event lag (CDC/Streams)
  - Bulk job duration & success rate

## 3. Validation steps

1. **Auth smoke test, each platform.** Lantern obtains a token with the
   service-principal path; stores no long-lived secret except the KMS
   key reference; token observed to expire and auto-renew.
2. **Scope negative test.** Lantern attempts an out-of-scope read
   (e.g., a field or table not in the permission set / role). Platform
   returns 403; Lantern logs it as `expected_denied`, not an error.
3. **Rate-limit behavior test.** Burst traffic until 429 (or Snowflake
   queueing); verify Lantern backs off, does not exhaust retries, and
   emits the quota-remaining metric.
4. **Pagination resume test.** Kill the Lantern worker mid-pagination;
   restart; verify it resumes at the correct cursor/offset without
   duplication or gap.
5. **CDC replay test.** Replay the last hour of Salesforce CDC events;
   verify the Lantern sink is idempotent (no duplicated side-effects).
6. **Masking policy test (Snowflake).** Query the governed view as
   `LANTERN_READER`; confirm masked values; query as another role to
   confirm unmasked (customer-run); evidence saved.
7. **Row-access test (Snowflake).** Query with a tenant switch;
   confirm no cross-tenant leakage.
8. **Sandbox-to-prod diff.** Produce an artifact listing every object
   (SF permission set, SN role, SF/SN scripts, Snowflake roles and
   grants) created on each side; verify sandbox and prod match.
9. **Rollback drill.** Disable Lantern's service user on each
   platform; verify Lantern surfaces a clear "access revoked" error,
   no retries hammer the platform, and the on-call runbook names the
   mitigation.
10. **PII redaction test (ServiceNow / Salesforce).** Pass a record
    with PII through the integration; verify the model input stripped
    or masked per the integration spec.

## 4. Rubric / review checklist

Score 0–2 per row. 14/22 passes.

| Dimension                                     | 0 — Missing                                  | 1 — Partial                                        | 2 — Strong                                                                 |
|-----------------------------------------------|----------------------------------------------|----------------------------------------------------|----------------------------------------------------------------------------|
| Surface selection                             | REST for everything                          | Right surface for most                             | Right surface per workload, rationale documented                            |
| Authentication                                | Shared user/pass                             | OAuth but overly broad                             | Dedicated service principal, scoped grants, KMS-held key, documented rotation |
| Rate-limit awareness                          | Not mentioned                                | Generic backoff                                    | Per-platform quota tracked, metric emitted, alert threshold documented      |
| Governance (SF permission set / SN role / Snowflake RBAC) | admin / unrestricted               | Scoped but ad-hoc                                  | Role defined in spec, minimum grants, masking/RLS in use (Snowflake)        |
| Pagination / bulk choice                      | REST for 10M records                         | Bulk considered                                    | Right API (Bulk/Snowpipe/Streams) for volume; resume safely                 |
| Change capture                                | Polling only                                 | Native event listed                                | SF Pub/Sub or ServiceNow business rule or Snowflake Stream + replay / dedup |
| Write safety                                  | Blind writes                                 | Mostly idempotent                                  | Idempotent on retry, schema-validated, dry-run flag supported               |
| Observability                                 | None                                         | Error logging                                      | Per-platform SLIs, quota gauge, 429/queue counters, CDC lag                 |
| Sandbox / release plan                        | Deploy to prod                               | Sandbox exists                                     | Sandbox-first, platform release notes subscribed, deploy artifact versioned |
| Audit trail                                   | None                                         | Partial                                            | Every read/write logged with query tag or sys_id; retained ≥90 days         |
| Governance mapping                            | None                                         | Mentions NIST                                      | Explicit NIST AI RMF Measure/Manage mapping per platform                    |

## 5. Common mistakes

- **Username + password against any SaaS.** Compromises on every list.
  OAuth + KMS-held keys, or key-pair auth — anything else is a review
  stop.
- **REST-only with no bulk path.** The integration works for 100 records,
  then dies at 100k. Pick Bulk/Snowpipe/Streams before the pilot.
- **Admin service principals.** Convenient in dev, becomes a scope
  creep nightmare. Permission set / role from day one, documented and
  grep-able.
- **Ignoring platform release notes.** Salesforce flips a behavior flag
  in a seasonal release and your integration falls over on the Sunday
  before a steering meeting. Subscribe. Diff.
- **Reading base tables in Snowflake.** You bypass masking and row
  access. Always go through governed views; let the customer's data
  team own the layer.
- **Shared service user across pilots.** One bad query blocks every
  tenant. One principal per tenant / per integration; separate quotas.
- **Blind polling.** Change event APIs exist for a reason; polling
  Salesforce every 5 minutes burns the API quota and still misses
  in-between updates.
- **Writing audit data to a mutable log file.** Snowflake query tagging
  and Salesforce event log file / ServiceNow sys_audit are the system-
  of-record audit sources; mirror into Lantern's own audit store.
- **No sandbox test for a schema change.** Your new field didn't
  propagate; Flows or Business Rules in prod break at the worst time.
- **Logging full records with PII.** Salesforce / ServiceNow records
  hold personal data. Redact before you log; cite the OWASP ML Top 10
  input-handling items in the integration spec.

## 6. References

- **Salesforce REST API** developer guide.
  <https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/>
- **Salesforce Bulk API 2.0** developer guide.
  <https://developer.salesforce.com/docs/atlas.en-us.api_asynch.meta/api_asynch/>
- **Salesforce Pub/Sub API** (gRPC change event stream).
  <https://developer.salesforce.com/docs/platform/pub-sub-api/overview>
- **Salesforce OAuth 2.0 JWT Bearer Flow** developer guide.
  <https://help.salesforce.com/s/articleView?id=sf.remoteaccess_oauth_jwt_flow.htm>
- **ServiceNow REST API / Table API** product docs.
  <https://developer.servicenow.com/dev.do#!/reference/api/latest/rest/>
- **ServiceNow OAuth 2.0** administration docs.
  <https://docs.servicenow.com/csh?topicname=c_OAuthApplications>
- **Snowflake SQL API** documentation.
  <https://docs.snowflake.com/en/developer-guide/sql-api/intro>
- **Snowflake Streams and Tasks** documentation.
  <https://docs.snowflake.com/en/user-guide/streams-intro>
- **Snowflake Snowpipe** documentation.
  <https://docs.snowflake.com/en/user-guide/data-load-snowpipe-intro>
- **Snowflake Masking Policies and Row Access Policies** documentation.
  <https://docs.snowflake.com/en/user-guide/security-column-ddm-intro>
- **Snowflake key-pair authentication** documentation.
  <https://docs.snowflake.com/en/user-guide/key-pair-auth>
- **NIST AI Risk Management Framework** — Measure and Manage functions
  cited for SaaS-platform governance and change management.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — input and output
  handling for SaaS records feeding model context.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — adversary techniques to consider when SaaS
  records are the model's input surface. <https://atlas.mitre.org/>
