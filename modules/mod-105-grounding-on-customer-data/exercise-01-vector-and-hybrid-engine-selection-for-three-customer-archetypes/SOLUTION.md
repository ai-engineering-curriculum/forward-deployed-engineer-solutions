# exercise-01 — Vector and Hybrid Engine Selection for Three Customer Archetypes · Reference Solution

## 1. Solution overview

The deliverable is a **retrieval-engine decision matrix** that picks a
vector + keyword stack for three distinct customer archetypes and defends
each choice on the three axes a customer's security and data-platform
owners will push on: **where the index lives**, **what talks to it**, and
**how it stays correct over time**.

A retrieval stack for grounded LLM work is not just "which vector DB" —
it is the tuple:

- **Index topology** — dense-vector only, lexical (BM25-class) only, or
  hybrid (dense + lexical with a fusion step).
- **Hosting surface** — managed cloud service (vendor SaaS), customer-
  cloud managed (first-party cloud offering in the customer's account),
  or self-hosted inside the customer VPC / on-prem.
- **Data posture** — where embeddings and chunks physically sit, which
  KMS keys protect them, who can read them, what residency region the
  bytes are pinned to.
- **ACL model** — how per-document permissions are represented in the
  index and enforced at query time (hand off to exercise-02 for the
  wiring).
- **Refresh shape** — streaming ingest, batch, or hybrid; how deletions
  and re-permissionings propagate.
- **Operational ownership** — who patches the engine, who scales shards,
  who owns schema migrations.

A defensible answer names **three stacks**, one per archetype, each with
trade-offs written down. The common FDE mistake is to fall in love with
a single vector DB vendor and shoehorn every archetype onto it. The
exercise forces you to start from the customer's constraints, mirroring
the discipline from mod-104 ex-01 (provider stack).

> This exercise selects the **retrieval engine**. It does not design the
> chunking strategy, the embedding model, or the reranker — those are
> composition decisions on top of the engine. Name them in each row;
> do not resolve them here.

## 2. Implementation — the decision matrix

### 2.1 Three archetypes

Reuse the archetype shapes from mod-104 ex-01 so the same customer maps
through both matrices.

| Archetype | Example industry | Key constraints |
|---|---|---|
| **A — Regulated enterprise, US-centric** | Large US bank or health-insurance payer | SOC 2 + HIPAA or FFIEC guidance applies; vendor-risk committee gates every new SaaS; retrieval corpus mixes SharePoint, Confluence, and a document system with document-level ACLs; "no vendor can read the corpus" is a contract clause. |
| **B — EU-regulated / data-residency-locked** | EU manufacturer, EU public-sector adjacent, or non-US financial institution | GDPR applies; corpus must stay in-region; parts of the corpus are classified as high-risk under EU AI Act Annex III per customer counsel; needs a sovereign-friendly hosting option. |
| **C — Air-gapped / sovereign / offline-tolerant** | Defense, intelligence, utilities, classified-adjacent research | No egress to vendor cloud; sometimes no internet at all; index must run on customer-owned compute; weights, binaries, and schemas ship through a review gate. |

### 2.2 The matrix

Rows are the archetypes; columns are the dimensions a defensible stack
has to answer. Every cell carries a concrete constraint, not a
preference.

| Dimension | Archetype A (Regulated US) | Archetype B (EU residency) | Archetype C (Air-gapped) |
|---|---|---|---|
| **Index topology** | **Hybrid** (BM25 + dense) with reciprocal-rank fusion or a reranker on top. Lexical recall covers regulated jargon, dense covers paraphrase. | **Hybrid**. Same justification; EU regulatory text is heavy on exact tokens (law/clause IDs) that pure dense retrieval misses. | **Hybrid** with a smaller dense model (open-weights embeddings). BM25 carries more load; dense quality is capped by the embedding model that fits on-prem. |
| **Hosting surface** | Customer-cloud managed engine in the customer's existing cloud (e.g., OpenSearch / Elasticsearch managed service, pgvector in a managed Postgres, or an in-cloud managed vector DB the vendor-risk team has already cleared). Keeps the data plane inside the customer's SOC 2 scope. | In-region managed engine; residency pinned at account/region level; cross-region replication disabled. Prefer the same cloud the customer's data lives in. | Self-hosted inside the customer VPC / on-prem. Candidates: OpenSearch (BM25 + dense), pgvector on managed Postgres mirror, Qdrant / Weaviate / Milvus deployed into the enclave — whichever the customer's platform team can patch and run. |
| **Data posture** | Embeddings and chunks encrypted at rest with customer-managed keys (CMK/BYOK); index resides in a project/account the vendor-risk team scopes under SOC 2; audit logs retained per customer policy. | Same, plus residency pinned at the resource level; a Data Processing Agreement and Transfer Impact Assessment on file; cross-border replication disabled. | Keys live in an on-prem HSM; no off-site telemetry; index artifacts shipped on removable media with signed manifests. |
| **ACL model** | Per-document ACL fields on every chunk (exercise-02). ACLs enforced at query time via post-filter or ACL-aware query DSL. | Same; plus any residency labels copied into the chunk metadata so a query from the wrong region is denied at the engine, not at the app. | Same; plus classification labels on every chunk; queries carry the user's clearance tuple and are filtered server-side. |
| **Refresh shape** | Streaming ingest from source-of-truth systems (e.g., SharePoint change feed, Confluence webhooks). Deletions and re-permissionings propagate within minutes; tombstone pattern for hard-deletes; see exercise-04 for the user-visible "stale" treatment. | Streaming where source APIs support it; batch fallback for systems that don't expose change feeds. Residency enforced on ingest workers. | Batch-first, air-gapped import pipeline; manifests signed before import; no outbound calls from ingest workers. |
| **Embedding model placement** | Managed embedding API behind the gateway (see mod-104 ex-02), zero-retention on, BYOK. | Same, in-region endpoint only. | Open-weights embedding model served on customer GPUs; versioned alongside the index. |
| **Reranker** | Optional cross-encoder behind the gateway for top-k. Treat as a cost/latency choice, not a correctness fix. | Same, in-region. | Smaller open-weights cross-encoder on customer GPUs; adds latency; justify per use case. |
| **Observability** | Query-level metrics (recall@k on gold set, filter hit rate, ACL-denied count, latency, cost) exported to the customer's own observability stack (see mod-110). | Same; log sink in-region. | Same; log sink on-prem. |
| **Compliance hooks** | NIST AI RMF Map/Measure for the retrieval surface (data lineage, measurement plan); SOC 2 change-management and access controls on the index tier. | Same spine; GDPR Art. 28 processor terms in the DPA; EU AI Act transparency obligations if the use case is Annex III. | NIST AI RMF spine plus the customer-specific regime (e.g., FedRAMP High); customer counsel owns the regulatory call. |
| **Risk mitigations (OWASP ML Top 10, MITRE ATLAS)** | Named mitigations for **training-data / retrieval poisoning** (poisoned corpus pages inject attacker content into context) and **prompt injection via retrieved context** (ATLAS LLM-prompt-injection sub-techniques). Chunk sanitisation, source allow-listing, and signed ingest manifests are the cell contents. | Same menu, plus explicit treatment of user-facing transparency when retrieved text is shown. | Same menu; adversarial robustness on the embedding model before promotion; signed-weights + signed-schema pipeline. |
| **Operational ownership** | Platform team owns the engine lifecycle; app team owns schema and query surface. Clear RACI, not shared ownership. | Same. | Same; ops team typically has a hard change-window constraint — reflect that in the refresh cadence. |
| **Rollback** | Blue/green indexes with alias flip. No in-place reindexing that can't be reverted. | Same. | Pin to prior index artifact; retain N-2 versions; promote behind a gate. |
| **Commercial posture** | Ride existing cloud EA where possible; new paper is the slowest step. | Same; sovereign/in-region only if existing EA cannot cover residency. | License + support + GPU/CPU capex; no per-query SaaS billing. |

### 2.3 Rationale — why these three stacks

- **A (Regulated US)**: the binding constraint is paperwork, not recall.
  The fastest path to a defensible retrieval tier is a hybrid engine
  hosted inside the customer's existing cloud and already covered by
  their enterprise agreement. Start with a managed OpenSearch /
  Elasticsearch or pgvector surface because both give you lexical **and**
  dense in one place, which simplifies ACL propagation (one engine, one
  ACL field). Pure dense-only stacks lose exact-match recall on
  regulatory language; pure lexical stacks lose semantic paraphrase.
  Hybrid is the defensible middle, and the fusion method (RRF vs
  weighted) is a measurement decision (see exercise-05).
- **B (EU residency)**: the binding constraint is where the bytes live
  and under which legal clauses. Use the same hybrid engine as A **if
  and only if** it has an in-region offering with an acceptable DPA.
  Otherwise defer to a sovereign-cloud option legal has already blessed.
  Confirm Annex III classification with the customer's counsel before
  layering AI Act obligations onto the retrieval tier — the retrieval
  engine is not where you decide whether the use case is high-risk, but
  it is where transparency, record-keeping, and human-oversight hooks
  get wired.
- **C (Air-gapped)**: the binding constraint is no egress ever. Managed
  SaaS is off the table. Self-host a hybrid engine the customer's
  platform team can run — OpenSearch is often the path of least
  resistance because the customer already runs it for logs. Accept
  weaker dense retrieval (smaller open-weights embedding model) and
  compensate with BM25 weight and chunking discipline.

### 2.4 Negative choices — what was explicitly rejected

Writing down what you rejected is as important as the matrix.

- **Rejected for A:** vendor-hosted vector SaaS that stores the corpus
  outside the customer's SOC 2 scope, even with "zero retention."
  The corpus is the regulated artifact, not just the queries; the
  index is more sensitive than the LLM call, not less.
- **Rejected for A/B:** pure dense-only vector DB where ACLs must be
  layered on in the app. Pushing ACLs to the app means every new
  retriever gets its own leak path. Keep ACLs on the engine (see
  exercise-02).
- **Rejected for B:** the US region of a frontier provider's vector
  service for POC. Region leakage during POC poisons the audit trail.
- **Rejected for C:** anything that calls home for license checks or
  telemetry, including "offline capable" engines that batch call
  outbound weekly.
- **Rejected for all:** rolling your own engine in Python on top of
  FAISS and a sidecar for BM25. The storage, replication, shard-
  balancing, ACL, and backup story is a years-long investment that
  turns the FDE into an infra team. Prefer boring engines.

### 2.5 Fusion and reranking (one paragraph per archetype)

Hybrid is "lexical + dense + fusion." Name the fusion method per row:

- **A / B:** start with reciprocal-rank fusion (RRF) for simplicity; it
  needs no training and is robust to score-scale drift between BM25 and
  dense. Promote to a learned combiner only after exercise-05 has a
  gold set to measure against. A cross-encoder reranker (bge-reranker
  class or vendor equivalent) on the top-k candidates is optional and
  pays off most on long, ambiguous questions.
- **C:** RRF only, unless you have budget for a local cross-encoder; in
  air-gapped deployments the serving stack is already load-bearing,
  and every new component is a new patch cadence.

## 3. Validation steps

Before the matrix is customer-ready:

1. **Three real customers behind the archetypes.** Each archetype maps
   to an actual customer in your pipeline (or a close proxy from mod-102
   discovery). An archetype with no customer attached is a demo.
2. **Every cell cites a concrete constraint.** "Zero retention on" must
   cite the provider's documented behavior for that endpoint. "EU AI
   Act applies" must cite which Annex per customer counsel. "FedRAMP
   High applies" must cite who in the customer's compliance org asserted
   that.
3. **Data-flow diagram per archetype matches the matrix.** For each
   archetype, draw where chunks, embeddings, logs, and evals
   physically live, and which network boundaries they cross.
4. **ACL model is engine-enforced, not app-enforced.** The row that
   says "ACLs enforced at query time" is only true if a filter runs on
   the engine; exercise-02 implements the wiring, but the matrix must
   commit to the pattern.
5. **Refresh cadence matches the freshness SLO.** The row that says
   "streaming" must match the user-visible freshness story from
   exercise-04. If the SLO says "within 5 minutes" and the pipeline
   is nightly batch, fix one or the other now.
6. **Rollback is a single toggle.** Blue/green aliasing or pinning to
   prior index artifact must be named and dry-run-able.
7. **The matrix survives a security review tabletop.** Walk a customer
   security lead through the matrix archetype by archetype; if they
   ask for a control you don't have, add it or pick a different stack.

## 4. Rubric / review checklist

Score 0–2 per dimension. 16/24 passes before sending to a customer.

| Dimension | 0 — Missing | 1 — Partial | 2 — Strong |
|---|---|---|---|
| Three archetypes, not three vendors | One archetype repeated | Three but one is a thin variant | Three distinct constraint shapes |
| Binding constraint named per archetype | Not stated | Vague ("security") | Specific — paperwork, residency, no egress |
| Index topology defended | Dense-only by default | Hybrid mentioned, no fusion method | Hybrid with named fusion/rerank per row |
| Hosting surface matches constraint | Public SaaS for regulated row | In-region but not scoped | Managed-in-customer-cloud, in-region, or on-prem, each defended |
| Data posture explicit | "Enterprise plan" | Encryption yes, keys vague | CMK/BYOK, residency pinned, DPA/SCC filed |
| ACL model engine-side | App-side filter | Mentioned, not wired | Engine filter + schema named; hands off to ex-02 |
| Refresh cadence tied to freshness SLO | Not named | Named generically | Matches the SLO in ex-04 |
| Embedding model placement | Not named | Managed only, no zero-retention | Zero-retention/BYOK or on-prem, per archetype |
| Reranker argued | "Add a reranker" | Named but not justified | Cost/latency/correctness trade-off stated |
| Risk mitigations | None | Named generically | Specific OWASP ML Top 10 / ATLAS risks per row |
| Rollback and blue/green | None | Named only | Rollback dry-run-able, N-2 retention |
| Negative choices | No rejection section | One rejection | Each archetype lists ≥1 rejected stack with reasoning |

## 5. Common mistakes

- **Dense-only by default.** Pure vector retrieval loses on exact-match
  queries — product IDs, statute numbers, drug codes. Hybrid is the
  default for enterprise text; pure dense is a specific optimization
  you justify, not a starting point.
- **Picking the engine the FDE likes.** The engine must match the
  customer's platform team's operating competence. A customer that
  already runs OpenSearch for logs will support OpenSearch for
  retrieval cheaply; forcing a new engine on them creates a shadow
  ops burden.
- **Letting the corpus leave the SOC 2 scope.** Sending the corpus to a
  managed vector SaaS "because the queries have zero retention" ignores
  that the corpus itself is the regulated asset.
- **Confusing residency with hosting.** "In the EU region" is not the
  same as "logs stay in the EU region." Separate query residency,
  log residency, and backup residency on the matrix.
- **Hand-wavy chunking.** Chunk size, overlap, and metadata schema are
  not this exercise, but they must at least be *named* so exercise-02
  knows what to filter on and exercise-04 knows what to cite.
- **Forgetting deletes.** "Streaming ingest" is a half-answer unless
  deletions and re-permissionings propagate as fast as inserts. A
  stale ACL in the index is an exfiltration path.
- **Skipping evaluation.** The matrix picks the engine; exercise-05
  measures it. Don't let "we'll measure recall later" quietly become
  "we never measure recall."
- **Overfitting to pgvector or to one managed vendor.** pgvector is
  often a correct answer, but not universally; if the customer's
  Postgres team has no appetite for a mixed OLTP + retrieval workload,
  pay the second-engine tax instead of overloading theirs.
- **Ignoring operational ownership.** An engine with no named owner
  will decay. Name the team on day 1.
- **No rollback story.** A matrix without a tested rollback is a plan
  to live with one engine forever.
- **Citing standards generically.** "NIST AI RMF applies" without
  naming the function (Govern/Map/Measure/Manage) is a smell.

## 6. References

- **NIST AI Risk Management Framework** — use Map for data lineage and
  the retrieval surface, Measure for recall / ACL-denied / staleness
  metrics, Manage for rollback and change management.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — map retrieval-specific
  threats (poisoned corpus, data leakage through the index, supply-
  chain risk in the embedding model) to the current project list.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — matrix of adversarial tactics against ML and LLM
  systems. Cite the specific sub-tactics (prompt injection via
  retrieved content, poisoning, model evasion) in the risk-mitigation
  row per archetype. <https://atlas.mitre.org/>
- Customer-specific regulatory references (GDPR, EU AI Act Annex III,
  HIPAA, FFIEC guidance, FedRAMP) — cite the specific obligation the
  customer's compliance owner has asserted applies; do not paraphrase
  from memory.
- Companion exercises in this module: exercise-02 (ACLs/RLS wired into
  the retriever), exercise-03 (PII handling on ingest), exercise-04
  (freshness / evidence UX), exercise-05 (customer gold set).
