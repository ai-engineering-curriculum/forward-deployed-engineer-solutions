# exercise-03 — OpenAPI, GraphQL, webhooks, CDC, event bus · Reference Solution

## 1. Solution overview

The deliverable is a **decision brief** that maps a given integration need
to the right transport pattern and justifies the choice. Five patterns
cover most enterprise data movement; the FDE mistake is to reach for the
one they know best instead of the one that fits.

| Pattern            | Delivery        | Semantics         | Push/Pull | Typical latency | Typical ordering              |
|--------------------|-----------------|-------------------|-----------|-----------------|-------------------------------|
| REST (OpenAPI)     | Request/reply   | At-most-once (per call) | Pull      | 50 ms–2 s       | Per-call, no stream ordering  |
| GraphQL            | Request/reply   | At-most-once      | Pull      | 50 ms–2 s       | Per-call                      |
| Webhook            | Fire-and-forget | At-least-once     | Push      | Seconds         | Per-event; retries break order |
| CDC (change data capture) | Stream   | At-least-once (exactly-once with idempotent sink) | Push | Seconds–minutes | Per-key ordered (if partitioned by key) |
| Event bus (Kafka / Kinesis / Pub-Sub) | Stream | At-least-once default | Push | Milliseconds    | Per-partition ordered         |

The brief answers four questions for each integration:

1. **Who owns the schema?** You, the customer, or a third-party SaaS.
2. **What is the delivery semantic the consumer can tolerate?** At-least-once
   is almost always right; "exactly-once" is usually "at-least-once +
   idempotent writes" in disguise.
3. **Can you tolerate the latency floor?** Request/reply can't match
   streaming for < 1 s end-to-end; streaming can't match request/reply for
   < 50 ms single-call lookups.
4. **What is the failure-mode budget?** Webhooks and streams need DLQ,
   retry, and replay; request/reply needs timeouts and circuit breakers.
   Choose the pattern whose failure story the on-call team will actually
   operate.

## 2. Implementation — decision brief + worked integrations

### 2.1 Decision tree

```
Need to move data between systems.
│
├─ Consumer initiates? (I call you when I need something)
│   └─ YES → Request/reply
│       ├─ Resource-oriented, versioned contract across teams/partners? → REST + OpenAPI
│       └─ Client-driven shape, aggregation across many resources?       → GraphQL
│
├─ Producer initiates? (You tell me when something happens)
│   └─ YES → Event-driven
│       ├─ 3rd-party SaaS sends outbound HTTPS calls, low volume?        → Webhook
│       ├─ Mirror a database's row changes into another system?          → CDC
│       └─ Many producers, many consumers, high throughput, replay needed? → Event bus
```

### 2.2 Worked integration A — REST + OpenAPI

Scenario: Lantern exposes a `POST /v1/jobs` endpoint for a customer
orchestrator.

Minimal OpenAPI 3.1 slice (contract first, implementation later):

```yaml
openapi: 3.1.0
info:
  title: Lantern Jobs API
  version: 1.3.0
servers:
  - url: https://api.lantern.example.com
security:
  - bearerAuth: []
paths:
  /v1/jobs:
    post:
      operationId: createJob
      summary: Enqueue a Lantern job
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/CreateJob' }
      responses:
        '202':
          description: Accepted
          headers:
            Location: { schema: { type: string } }
            Idempotency-Key: { schema: { type: string } }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Job' }
        '409': { $ref: '#/components/responses/Conflict' }
        '429': { $ref: '#/components/responses/RateLimited' }
components:
  securitySchemes:
    bearerAuth: { type: http, scheme: bearer, bearerFormat: JWT }
  schemas:
    CreateJob:
      type: object
      required: [tenant, kind, input_uri]
      properties:
        tenant: { type: string }
        kind:   { type: string, enum: [index, embed, classify] }
        input_uri: { type: string, format: uri }
        callback_url: { type: string, format: uri }
    Job:
      type: object
      properties:
        id: { type: string }
        status: { type: string, enum: [queued, running, done, failed] }
```

Non-negotiable rules:

- **Idempotency.** `POST` accepts an `Idempotency-Key` header; the server
  stores the response for 24 h keyed by (tenant, key).
- **Versioning in the URL** for breaking changes (`/v1/`, `/v2/`), plus
  `Accept: application/vnd.lantern.v1+json` for minor variants. Never
  reuse a path with silently changed semantics.
- **Pagination.** Cursor-based, not offset-based. `?cursor=opaque` in; next
  cursor in response envelope.
- **Rate limits.** Return `429` with `Retry-After`; publish per-token quota
  in the docs.
- **Problem details.** RFC 9457 `application/problem+json` for errors.

### 2.3 Worked integration B — GraphQL

Scenario: Lantern's web console hydrates a dashboard that pulls across
jobs, tenants, and recent runs in one page. REST would be N+1; GraphQL
collapses it.

Schema slice:

```graphql
type Query {
  tenant(id: ID!): Tenant
  jobs(tenant: ID!, first: Int = 20, after: String, status: [JobStatus!]):
    JobConnection!
}
type Tenant { id: ID!, name: String!, quota: Quota! }
type JobConnection { edges: [JobEdge!]!, pageInfo: PageInfo! }
type JobEdge { cursor: String!, node: Job! }
type Job { id: ID!, status: JobStatus!, createdAt: DateTime!, runs: [Run!]! }
enum JobStatus { QUEUED RUNNING DONE FAILED }
```

Guardrails without which GraphQL kills your backend:

- **Depth limit** (≤10) and **complexity scoring** on every query; reject
  above threshold.
- **Persisted queries.** Clients register queries at build time; production
  accepts only query hashes, not free-form query strings. This also solves
  the request-logging and security story.
- **No introspection in production** on tenant-facing endpoints.
- **DataLoader / batch** in resolvers; every `runs` resolver must coalesce,
  not N+1.
- **Caching.** Per-field cache hints (`@cacheControl`) and an edge cache; or
  go operation-level with persisted queries + ETag.

When **not** to use GraphQL:

- Public B2B API where many teams produce clients without your SDK —
  OpenAPI's codegen ecosystem beats GraphQL here.
- High-fanout write patterns; mutations are fine but there's no bulk-write
  story that's better than REST.

### 2.4 Worked integration C — webhooks (producer-initiated HTTPS)

Scenario: Lantern tells the customer's CRM when a job completes.

Specification Lantern publishes to the customer:

- **Delivery:** `POST` to the customer's configured URL. JSON body. HTTPS
  only. TLS 1.3.
- **Headers:**
  - `X-Lantern-Event: job.completed`
  - `X-Lantern-Delivery: <uuid>` (dedup key)
  - `X-Lantern-Timestamp: <unix seconds>`
  - `X-Lantern-Signature: v1=<hex HMAC-SHA256 of timestamp + "." + body>`
- **Signing:** HMAC-SHA256, shared signing secret per customer, rotated
  via UI with 24-hour overlap. Customer rejects requests with timestamp
  more than 5 minutes old (replay window).
- **Retry:** Exponential backoff — 30 s, 2 m, 10 m, 1 h, 6 h, 24 h — then
  dead-letter and page the integration owner.
- **Delivery semantics:** at-least-once. Customer MUST dedup on
  `X-Lantern-Delivery`.
- **Ordering:** no guarantee. Each event carries a monotonic per-tenant
  `sequence` field; consumers that need order sort client-side.

Example payload:

```json
{
  "event": "job.completed",
  "sequence": 1821,
  "id": "job_01HZ...",
  "tenant": "acme",
  "completed_at": "2026-11-20T15:03:12Z",
  "result": { "pages": 412, "tokens": 2_118_443 }
}
```

Operator surface:

- **Delivery log UI** per tenant showing attempt count, last status,
  next retry ETA, HTTP response.
- **Manual replay** by event id or time window — a must-have for
  customer incidents.
- **DLQ dashboard** with p99 age and oldest-stuck pager alert.

### 2.5 Worked integration D — CDC (change data capture)

Scenario: Mirror the customer's `orders` Postgres table into Lantern's
feature store. Use **logical replication** on the source side; a
Debezium-style connector ships row-level events to Kafka; a sink writes
into Lantern's warehouse.

Design:

```
Postgres (orders) --[logical decoding]--> connector --> Kafka topic (orders.cdc)
                                                              |
                                                              v
                                               Lantern sink (upsert by PK)
```

Event shape (Debezium-compatible):

```json
{
  "op": "u",
  "ts_ms": 1731000090000,
  "source": { "connector": "postgres", "db": "crm", "schema": "public",
              "table": "orders", "lsn": "0/16B37A0", "snapshot": "false" },
  "before": { "id": 42, "status": "pending",  "amount_cents": 10000 },
  "after":  { "id": 42, "status": "paid",     "amount_cents": 10000 }
}
```

Rules:

- **Primary key is mandatory** on every replicated table. CDC without a
  stable PK is a straight line to data corruption.
- **Partition by PK** on the Kafka topic to keep per-row ordering.
- **Sink idempotency.** The sink uses `before.id` to key the upsert;
  retried events are no-ops if the LSN is not newer than the sink's
  committed LSN per key.
- **Schema evolution.** The source may add columns; the connector's
  schema registry entries are versioned; sinks tolerate unknown fields.
- **Lag monitoring.** `max(source_now - event_ts_ms)` per topic; page
  above the agreed SLA (e.g. 2 minutes for operational data, 15 minutes
  for analytics).
- **Snapshots.** Initial snapshot is bounded and chunked; it must be
  resumable; the customer's DBA signs off on load timing.
- **PII and tombstones.** Deletes produce a tombstone event
  (`op: "d"`, `before`, `after: null`); the sink honors it with a hard
  delete or a tombstone marker depending on the policy.

### 2.6 Worked integration E — event bus

Scenario: Lantern's internal services publish `JobEvent`s to a Kafka
cluster. Many consumers (billing, audit, telemetry, downstream agents).

Topic design:

- `job.events.v1` — the fact topic. Keyed by `job_id`.
- `job.events.v1.dlq` — dead-letter topic for consumers.
- Retention: 7 days active topic; compacted "latest state" topic
  `job.state.v1` for consumers that only want current state.
- Producers use **exactly-once semantics** (idempotent producer +
  transactional writes when bundling with a sink).
- Schemas managed via a schema registry (Avro or Protobuf). Backwards-
  compatible changes allowed; incompatible changes require a `v2` topic
  and a dual-write window.

Consumer contract:

- Consumer groups per subsystem.
- Commit offsets **after** side-effects are durable; never before.
- Poisoned messages after N retries go to DLQ with a reason header.
- Lag > T triggers autoscale; sustained lag > T' pages on-call.

When to pick event bus over webhook:

- Multiple internal consumers with different cadences.
- Replay needed (regenerate downstream after a bug fix).
- Fan-in from many producers.

When to pick webhook over event bus:

- Single external third-party; they don't have a bus; HTTPS is their
  only accessible surface.

### 2.7 Decision brief template (use this for each new integration)

```markdown
Integration: <name>
Direction: <producer> → <consumer>
Pattern: <REST/GraphQL/Webhook/CDC/Event bus>
Why this pattern (one paragraph): …
Delivery semantics: <at-least-once / exactly-once via idempotent sink>
Ordering: <none / per-key>
Latency target: <p50=X, p95=Y>
Schema owner: <who>
Versioning policy: <URL versioning / schema registry>
Failure handling: <retry policy, DLQ, replay>
Auth: <OAuth scope / mTLS / signed webhook>
Observability: <metrics names, logs, alerts>
Governance: <NIST AI RMF function alignment — Measure for throughput, Manage for rollback>
```

## 3. Validation steps

For each chosen pattern, the integration is "ready to pilot" when:

1. **Contract exists and is machine-readable.** OpenAPI 3.x / GraphQL
   SDL / Avro or Protobuf schema / signed webhook spec. Review it in the
   repo, not on a slide.
2. **Idempotency is testable.** Send the same event/request twice;
   observe one effect. Record the test case in the integration folder.
3. **Backpressure is testable.** Overrun the consumer; verify `429` /
   consumer lag / DLQ behavior matches the spec.
4. **Replay is testable.** Replay the last hour of events; verify the
   consumer handles it safely.
5. **Delivery log / lag dashboard exists.** Operator can see "when was
   the last successful delivery to tenant X."
6. **Auth and signature paths are negative-tested.** Expired tokens;
   wrong HMAC; stale timestamp — all rejected with distinct errors.
7. **Schema evolution plan documented.** Who owns each field, how
   additions are rolled out, how deletes happen.
8. **Runbook linked in the repo.** On-call reads this during an incident;
   if it isn't written you don't have a production integration.

## 4. Rubric / review checklist

Score 0–2 per row. 14/22 passes.

| Dimension                                      | 0 — Missing                                   | 1 — Partial                                      | 2 — Strong                                                                 |
|------------------------------------------------|-----------------------------------------------|--------------------------------------------------|----------------------------------------------------------------------------|
| Pattern selection with rationale               | "REST because we always do REST"              | Pattern listed                                   | Pattern + rationale tied to the four questions in §1                       |
| Machine-readable contract                      | Prose doc only                                | Partial schema                                   | OpenAPI/SDL/Avro committed, versioned, generated client in repo            |
| Delivery semantics stated and enforced         | "Should be fine"                              | Stated                                           | Stated + test case demonstrating dedup/idempotency                         |
| Ordering guarantees stated and enforced        | None                                          | Per-topic                                        | Per-key partitioning, consumer honors `sequence` / LSN                     |
| Retry, DLQ, replay                             | None                                          | Retry only                                       | Retry + DLQ + manual replay + lag alert with threshold                     |
| Authn + payload integrity                      | Shared secret in URL                          | Bearer only                                      | OAuth or mTLS plus HMAC-signed body (webhooks) with replay window          |
| Pagination / query cost                        | Offset or unbounded                           | Cursor                                           | Cursor + max page + GraphQL depth/complexity limits                        |
| Observability                                  | None                                          | Per-service counters                             | Per-integration: throughput, p95 latency, lag, DLQ depth, oldest-stuck     |
| Schema evolution                               | "Break and fix"                               | Backwards-compat statement                       | Registry + dual-write plan + consumer tolerance test                       |
| Governance mapping                             | None                                          | Mentions NIST                                    | Explicit NIST AI RMF Measure/Manage mapping for throughput and rollback    |
| Runbook                                        | None                                          | Draft                                            | Linked, dated, tested (last drill within 90 days)                          |

## 5. Common mistakes

- **Picking REST for a producer-driven event stream.** You end up writing
  a polling client with worse latency and higher cost than a webhook.
- **Picking an event bus for a 1→1, low-volume integration.** Kafka
  infrastructure dwarfs the value delivered.
- **GraphQL without depth limits or persisted queries.** The first
  curious customer brings the service down with a nested query.
- **Webhooks without replay dedup.** The customer processes the same
  event twice and double-books an order. HMAC alone isn't enough; include
  a delivery id and require consumer-side dedup.
- **CDC without a primary key.** The sink can't dedup, order is
  guaranteed only globally, and bulk edits produce duplicates.
- **Treating "exactly-once" as a transport guarantee.** No general-purpose
  network protocol gives you exactly-once end-to-end. "At-least-once +
  idempotent consumer" is the real engineering answer.
- **Logging webhook bodies in plaintext.** PII leaks into application
  logs. Redact before log, or log the delivery id and header only.
- **Versioning by "we deploy and announce in Slack."** External consumers
  never see the Slack. Version in the URL or in the schema registry; keep
  the old version live for a documented period.
- **No DLQ, no lag alert.** The integration stops silently; the first
  signal is the customer complaining three days later.
- **Treating OpenAPI as documentation after the fact.** If your OpenAPI is
  hand-written after you shipped, it will drift. Generate it from code or
  code from it, but keep them locked together.

## 6. References

- **OpenAPI Specification** (OpenAPI Initiative) — 3.1 schema and tooling.
  <https://spec.openapis.org/oas/latest.html>
- **RFC 9457** — Problem Details for HTTP APIs.
  <https://www.rfc-editor.org/rfc/rfc9457>
- **RFC 5789** — PATCH method (for JSON Merge / JSON Patch).
  <https://www.rfc-editor.org/rfc/rfc5789>
- **GraphQL Specification** (GraphQL Foundation).
  <https://spec.graphql.org/>
- **CloudEvents 1.0** (CNCF) — common event envelope across webhooks and
  event buses. <https://cloudevents.io/>
- **Apache Kafka** documentation — exactly-once semantics, transactional
  producer, compacted topics. <https://kafka.apache.org/documentation/>
- **Debezium** documentation — connector contract for Postgres / MySQL /
  SQL Server CDC. <https://debezium.io/documentation/>
- **PostgreSQL logical decoding** — source for CDC.
  <https://www.postgresql.org/docs/current/logicaldecoding.html>
- **NIST AI Risk Management Framework** — Measure and Manage functions
  used in §2.7 for throughput, rollback, and incident handling.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — relevant when event
  payloads feed training or feature pipelines.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — reference for data-poisoning and feature-store
  attacks on event pipelines. <https://atlas.mitre.org/>
