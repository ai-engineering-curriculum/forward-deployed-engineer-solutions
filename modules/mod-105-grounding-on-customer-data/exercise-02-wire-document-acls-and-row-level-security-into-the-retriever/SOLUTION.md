# exercise-02 — Wire Document ACLs and Row-Level Security into the Retriever · Reference Solution

## 1. Solution overview

The deliverable is a **reference design + runnable artifacts** for a
retrieval layer where **every chunk carries the permissions of its
source document** and **every query is filtered by the caller's
identity at the engine**, not at the application.

Three rules make this exercise defensible:

1. **No chunk goes into context the user could not read in the source
   system.** If HR owns a document, HR must own its embeddings and
   its row in the index. The LLM is a downstream reader; it inherits,
   not overrides, the source ACL.
2. **Filtering happens on the engine.** A post-filter in app code is a
   leak waiting to happen (timing, logging, ranker caches). Push the
   filter into the query DSL — OpenSearch `terms` filter, pgvector
   Postgres row-level security (RLS) policy, or the engine's native
   ACL field.
3. **The identity used to filter is the end user's identity**, not the
   service account. Propagate the user's principal + group tuple into
   the retriever as a short-lived, signed token.

The common FDE mistake is to filter in Python over top-k results. That
pattern works until a reranker, cache, or trace captures the forbidden
chunk, at which point you have both a correctness bug and a disclosure.

This design pins ACLs to the chunk on ingest, enforces them at query
time on the engine, and is testable end-to-end with a per-tenant gold
set of "should allow / must deny" probes.

## 2. Worked implementation

### 2.1 Where the ACL comes from

| Source system | Typical permission model | How it maps |
|---|---|---|
| SharePoint / OneDrive | Users + Azure AD groups, inheritance from site/library | `allow_principals = {users, groups}` (resolved through the site inheritance chain at ingest) |
| Confluence | Users + groups + space permissions + page restrictions | `allow_principals = union(space.read, page.read_restrictions)` |
| Google Drive | Users + Google Groups + domain + "anyone with link" | `allow_principals = effective_readers`; "anyone with link" becomes a labeled domain-wide bucket, never silently wildcarded |
| S3 bucket of PDFs | Bucket/prefix IAM + object tags | `allow_principals = roles_with_get_object` resolved at ingest via IAM simulation |
| Internal CMS | Varies | `allow_principals` resolved via the CMS SDK; **if the CMS has row- or field-level permissions, those project into chunk metadata too** |

The retriever cannot invent permissions. The ingest worker (exercise-03)
resolves effective readers at the source and writes them to the chunk.

### 2.2 Chunk schema (engine-agnostic)

Every chunk in the index carries the following fields at minimum:

| Field | Type | Purpose |
|---|---|---|
| `chunk_id` | string | Globally unique; `{doc_id}:{chunk_ix}` |
| `doc_id` | string | Source-of-truth document identifier |
| `tenant_id` | string | Tenant scope; required on every query |
| `allow_principals` | array\<string\> | Users + groups allowed to read (prefixed, e.g., `user:...`, `group:...`) |
| `deny_principals` | array\<string\> | Explicit denies; evaluated after allow |
| `classification` | string | Public / Internal / Confidential / Restricted (customer scheme) |
| `residency` | string | Region label, e.g., `us-east`, `eu-west` |
| `source_system` | string | `sharepoint`, `confluence`, `drive`, ... |
| `source_url` | string | Deep-link to the source (for citations; see exercise-04) |
| `source_mtime` | timestamp | Last-modified time in the source system |
| `indexed_at` | timestamp | When this chunk entered the index |
| `acl_version` | string | Hash of the resolved principal set; see §2.6 |
| `embedding` | vector | Dense embedding |
| `text` | string | Chunk text (BM25 analyzed) |

Rules:

- `allow_principals` and `deny_principals` are **flattened at ingest**.
  Do not store the raw ACL tree and resolve it at query time — the
  retrieval path must not call the source IAM on every query.
- Flattening must re-resolve on change. Group membership drift is the
  single most common cause of "why did the LLM answer with HR's doc"
  — see §2.6.

### 2.3 Identity at query time

The request to the retriever carries the **end user's** identity, not
the gateway's service account. A short-lived, signed token (JWT or
equivalent) bound to the end-user principal and their resolved groups
arrives from the identity hop (see mod-103) and is passed through the
model gateway (mod-104 ex-02) to the retriever.

The retriever derives, server-side:

- `caller.principal` — e.g., `user:alice@acme.com`
- `caller.groups` — e.g., `["group:finance-us","group:all-employees"]`
- `caller.tenant_id`
- `caller.residency`
- `caller.classification_ceiling` — the highest classification this
  caller is cleared for (for archetype C-class deployments)

Never trust `caller.*` fields passed in the request body; derive them
from the signed token.

### 2.4 Engine enforcement — three implementations

The matrix in exercise-01 picks an engine per archetype; this section
shows the ACL wiring for the three most common ones.

#### 2.4.1 OpenSearch / Elasticsearch (hybrid)

Store ACLs as keyword fields, filter with a `bool` query, never at the
app.

```jsonc
// Build one list, bind it to both branches. Never hand-write two arrays —
// any drift silently converts deny-overrides-allow into allow-wins for
// the dropped principal.
//
//   <caller_principals> = [caller.principal] + caller.groups
//                       = ["user:alice@acme.com",
//                          "group:finance-us",
//                          "group:all-employees"]
//
// Query
{
  "query": {
    "bool": {
      "must": [
        { "match": { "text": "<user_question>" } }     // BM25 branch
      ],
      "should": [
        { "knn": { "embedding": {
            "vector": "<query_embedding>", "k": 50
        }}}                                             // dense branch
      ],
      "filter": [
        { "term":  { "tenant_id": "<tenant>" } },
        { "term":  { "residency": "<caller_residency>" } },
        { "terms": { "allow_principals": "<caller_principals>" } }
      ],
      "must_not": [
        // Same `<caller_principals>` binding as the allow branch above.
        // If a group the caller belongs to is dropped here, a chunk that
        // denies that group can still match and leak downstream.
        { "terms": { "deny_principals":  "<caller_principals>" } }
      ]
    }
  }
}
```

Combine BM25 and dense via reciprocal-rank fusion (`rank` feature or
client-side RRF) **after** the filter has executed. The filter is
non-negotiable; fusion is a correctness/quality tune.

For engines with native field-level security (Elasticsearch document-
and field-level security, OpenSearch security plugin), pin the filter
to a security role rather than to the client query; the client-provided
query cannot bypass a role-scoped index.

#### 2.4.2 pgvector on Postgres (RLS)

Row-level security lets the database enforce the filter even if the
app forgets. Prefer this when the customer already runs Postgres with
RLS.

```sql
-- schema
create table chunks (
    chunk_id          text primary key,
    doc_id            text not null,
    tenant_id         text not null,
    allow_principals  text[] not null,
    deny_principals   text[] not null default '{}',
    classification    text not null,
    residency         text not null,
    source_system     text not null,
    source_url        text not null,
    source_mtime      timestamptz not null,
    indexed_at        timestamptz not null default now(),
    acl_version       text not null,
    text              text not null,
    embedding         vector(768) not null
);

create index on chunks using hnsw (embedding vector_cosine_ops);
create index on chunks using gin (allow_principals);
create index on chunks (tenant_id, residency);

alter table chunks enable row level security;

-- Policy: you may read a chunk iff tenant+residency match
-- AND your principal set intersects allow_principals
-- AND does not intersect deny_principals.
create policy chunks_read on chunks for select
using (
    tenant_id = current_setting('lantern.tenant', true)
    and residency = current_setting('lantern.residency', true)
    and allow_principals && string_to_array(
            current_setting('lantern.principals', true), ',')
    and not (deny_principals && string_to_array(
            current_setting('lantern.principals', true), ','))
);

-- Session setup at connection time:
-- set_config('lantern.tenant',     '<tenant_id>',          true);
-- set_config('lantern.residency',  '<caller_residency>',   true);
-- set_config('lantern.principals',
--     'user:alice@acme.com,group:finance-us,group:all-employees', true);
```

Important:

- Use a **non-superuser** role for query traffic; RLS is bypassed by
  `BYPASSRLS` roles.
- Set the session GUCs from the **server side** of the retriever based
  on the signed token, not from client-provided SQL parameters.
- Add an explicit index on `allow_principals` (GIN) and on the filter
  prefix (tenant, residency) — otherwise the planner will not prune
  before the KNN step and recall plus latency both suffer.

#### 2.4.3 Qdrant / Weaviate / Milvus (native payload filters)

All three support metadata filters that compose with the vector
search. Push the ACL into the payload at write time and filter at
read time.

```python
# Qdrant (example; API shape varies between engines)
from qdrant_client import QdrantClient
from qdrant_client.http.models import Filter, FieldCondition, MatchAny, MatchValue

client = QdrantClient(url=os.environ["QDRANT_URL"])

q_filter = Filter(
    must=[
        FieldCondition(key="tenant_id",  match=MatchValue(value=tenant)),
        FieldCondition(key="residency",  match=MatchValue(value=residency)),
        FieldCondition(key="allow_principals",
                       match=MatchAny(any=principals)),
    ],
    must_not=[
        FieldCondition(key="deny_principals",
                       match=MatchAny(any=principals)),
    ],
)

hits = client.search(
    collection_name="lantern_chunks",
    query_vector=q_embedding,
    query_filter=q_filter,
    limit=top_k,
)
```

The same pattern applies to Weaviate (where filter) and Milvus
(`expr=`). In every case, the filter is **pre-filter** (runs before /
during the ANN search), not a post-filter over top-k.

### 2.5 Retrieval service — the surface the app calls

```python
# retriever.py (sketch; engine-agnostic surface)
from dataclasses import dataclass
from typing import List

@dataclass(frozen=True)
class Caller:
    tenant_id: str
    principal: str                # from signed token
    groups: tuple[str, ...]       # from signed token
    residency: str
    classification_ceiling: str

@dataclass(frozen=True)
class Retrieved:
    chunk_id: str
    doc_id: str
    text: str
    score: float
    source_url: str
    classification: str
    source_mtime: str

def retrieve(question: str, caller: Caller, k: int = 8) -> List[Retrieved]:
    principals = [f"user:{caller.principal}"] + list(caller.groups)

    # Reject queries that cannot be filtered.
    if not caller.tenant_id or not caller.residency:
        raise PermissionError("caller missing tenant or residency")

    q_vec   = embed(question)                # calls embedding behind gateway
    engine  = get_engine()                   # engine selected per ex-01
    hits    = engine.hybrid_search(
        text=question,
        vector=q_vec,
        k=k,
        filters={
            "tenant_id":        caller.tenant_id,
            "residency":        caller.residency,
            "allow_principals": principals,
            "deny_principals":  principals,  # engine applies as must_not
            "classification_le": caller.classification_ceiling,
        },
        audit={
            "principal": caller.principal,
            "trace_id":  current_trace_id(),
        },
    )
    return [to_retrieved(h) for h in hits]
```

Non-negotiables in the surface:

- `Caller` is derived server-side from the signed token; the retrieval
  service never trusts a client-supplied principal.
- `audit` is a required parameter; every search writes a row to an
  audit log (customer-owned, see mod-110) with
  `{trace_id, principal, filter_hash, chunk_ids_returned}`.
- `raise PermissionError` is the default on missing tenant / residency
  / token. **Never** fall back to an unfiltered search.

### 2.6 Keeping ACLs fresh — the subtle part

The most common failure mode is not the query filter but **stale
permissions in the index**:

1. **Group-membership drift.** Alice leaves Finance; her removal from
   `group:finance-us` must be reflected in her token (identity provider)
   **and** in any materialized principal lists.
2. **Document re-permissioning.** HR moves a doc from "HR managers" to
   "All employees." The chunk's `allow_principals` must be rewritten.
3. **Document deletion.** A deleted doc leaves tombstone chunks; those
   must be deleted *before* anyone queries again.

Design:

- **ACL changes propagate as index writes, not as query-time lookups.**
  A `document.permissions.changed` event from the source system (or a
  nightly reconcile for systems with no change feed) triggers a
  partial update of all chunks for that document.
- **`acl_version`** is a hash of `(sorted(allow_principals),
  sorted(deny_principals))`. Store the same hash on the source-of-
  truth record; reconcile jobs compare and re-ingest on mismatch.
- **Tombstones.** On document delete, write a tombstone row with
  `allow_principals = []`, let the retriever treat `[]` as "no one,"
  and garbage-collect on a schedule. Hard-deleting on the write path
  risks race conditions with in-flight queries.
- **Target propagation SLO.** For streaming sources, aim for p95
  under 60 seconds from source change to index visibility; for batch
  sources, name the cadence in the exercise-01 matrix and expose the
  "last ACL sync" timestamp in the citation UX (see exercise-04).

### 2.7 Safe-by-default behaviors

- **Empty-principal set denies, not allows.** If principal resolution
  fails, return zero results. Do not silently fall back to a service-
  account view.
- **Classification ceiling is enforced server-side.** A classification
  label on the chunk greater than the caller's ceiling filters the
  chunk out, even if the principal would otherwise match.
- **Denylists dominate allowlists.** A chunk with a principal in both
  `allow_principals` and `deny_principals` is denied.
- **Deny filter uses the caller's complete principal set.** Both the
  allow branch and the deny branch of the query evaluate against the
  same list — the caller's user principal plus every resolved group.
  Dropping a group from the deny branch (even "obviously broad" ones
  like `group:all-employees`) silently converts a deny-overrides-allow
  policy into an allow-wins policy for that group.
- **ACL filter is logged, not embedded in the message to the LLM.**
  Never put raw principal lists or ACL JSON into the prompt.

## 3. Validation steps

1. **Unit tests on the retrieval service.**
   - `test_missing_tenant_denies` — call without tenant → raises.
   - `test_missing_residency_denies` — call without residency → raises.
   - `test_unknown_principal_returns_empty`.
   - `test_group_drift_removes_access` — remove principal, re-ingest,
     assert the previously-allowed chunk no longer appears.
   - `test_tombstone_returns_zero` — deleted doc returns nothing.
   - `test_classification_ceiling_filters_out_restricted`.
   - `test_filter_runs_on_engine` — mock the engine, assert the filter
     clause is present; fail if any top-k is retained by app-side code.

2. **Integration tests per engine (OpenSearch / pgvector / Qdrant).**
   Seed a small fixture corpus with mixed principals and residencies,
   run three test callers (owner, outsider, restricted), confirm the
   retrieved set is a subset of the expected-visible set.

3. **ACL drift drill.** Change a user's group membership in the IdP
   fixture, let the ACL-sync job run, re-query, assert the previously
   accessible chunks are now denied, with p95 propagation under the
   SLO named in exercise-01.

4. **Red-team probe set.** A per-tenant "must-deny" set of questions
   whose correct answer is "no citation available" because the user
   cannot read any source. Any citation leakage is a failed test and
   a release-blocking defect (classification per mod-108).

5. **Audit log review.** Verify every retrieval produced an audit row
   with `{trace_id, principal, filter_hash, chunk_ids}` in the
   customer-owned log sink. No retrieval must be audit-silent.

6. **Explain-query sanity check.** For OpenSearch, run `_explain`; for
   pgvector, `EXPLAIN (ANALYZE, BUFFERS)`; for vector DBs, inspect
   the query plan. The plan must show the ACL filter pruning before
   the vector step, not after.

## 4. Rubric / review checklist

Score 0–2 per row. 14/22 passes before promoting to production.

| Dimension | 0 — Missing | 1 — Partial | 2 — Strong |
|---|---|---|---|
| ACL on the chunk, not the app | App-side filter | Mixed | `allow`/`deny` stored on the chunk; engine filters |
| Engine enforcement | Client-side filter | Native filter without role scoping | Role-scoped and client-query-bypass-proof |
| Identity source | Service-account view | End-user token in body (untrusted) | Signed token verified server-side |
| Principal flattening on ingest | Resolved at query | Flattened but not versioned | Flattened + `acl_version` hash |
| Freshness on permission changes | Nightly only | Within hours | Within minutes, with propagation SLO |
| Deletes / tombstones | Hard delete only | Tombstone pattern | Tombstone + GC + safety fence on `[]` |
| Classification ceiling | Not implemented | Label on chunk only | Server-side enforcement |
| Residency enforcement | App-side | Metadata only | Engine filter + rejected cross-region |
| Audit log | Absent | Service log | Per-query audit with `filter_hash` |
| Query plan / EXPLAIN | Not checked | Checked ad hoc | Automated check that filter is pre-pruned |
| Red-team / must-deny probes | None | Smoke-test only | Per-tenant probe set in CI |

## 5. Common mistakes

- **App-side filtering on top-k.** The retriever returns 50 hits and
  the app filters to 8. The forbidden chunks already touched the
  reranker, cache, trace, and sometimes the prompt. Push the filter
  to the engine.
- **Resolving ACLs at query time.** Calling SharePoint / Drive / IAM
  on every query is slow, brittle, and ties retrieval to source-
  system availability. Flatten on ingest; reconcile on change.
- **Trusting principals in the request body.** Any field the client
  can set is not an authentication claim. Derive from the signed
  token server-side.
- **Forgetting group drift.** Users keep access to chunks they could
  no longer read in the source, because their principals were never
  re-flattened after a membership change. Add the drift drill to CI.
- **Hard-deleting on the write path.** Query A is in flight, delete
  arrives, query A fails or (worse) succeeds with stale data. Use
  tombstones and GC.
- **Wildcarding "anyone with link" into public.** A Google Drive
  "anyone with link" becomes a label (`link_shared`), not an implicit
  allow for every principal.
- **Empty principal set falling back to unfiltered.** The safe default
  for "I could not resolve the user" is zero results, not everything.
- **Mixing tenant and residency.** A cross-tenant leak and a cross-
  region leak both show up as "weird citation" and both are
  release-blocking; they are separate filters and must be audited
  separately.
- **ACL in the prompt.** Putting the principal list or ACL JSON into
  the LLM's prompt is a disclosure (prompt injection can echo it
  back) and adds no retrieval value.
- **Index-wide reindex instead of per-doc partial update.** On a
  permission change, rewrite only the affected document's chunks.
  Full reindex on every ACL twitch is unaffordable and creates long
  windows of inconsistent state.
- **Skipping the red-team set.** Correctness tests pass because the
  retriever is permissive by construction. The must-deny set is the
  only test that *fails* when the filter is wrong.

## 6. References

- **NIST AI Risk Management Framework** — use Govern for the access-
  policy decision, Map for data lineage, Measure for ACL-denied /
  drift metrics, Manage for change propagation SLOs.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — map "Model Data
  Disclosure" / "Training Data Leakage" / "Access to retrieval
  context" exposures to the current project list; retrieval-time
  ACL leaks are the retrieval analogue of training-data leakage.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — covers exfiltration, access-control bypass, and
  prompt-injection tactics that attempt to coax the retriever into
  surfacing forbidden chunks. <https://atlas.mitre.org/>
- Companion exercises: exercise-01 (engine selection), exercise-03
  (ingest and PII), exercise-04 (citation UX including "no evidence"),
  exercise-05 (gold set and must-deny probes); mod-103 ex-01 for the
  identity hop; mod-104 ex-02 for the gateway that forwards the signed
  token; mod-108 for the release gate.
