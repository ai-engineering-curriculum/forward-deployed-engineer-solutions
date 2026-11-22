# exercise-04 — Citation Freshness and Evidence-Missing UX · Reference Solution

## 1. Solution overview

The deliverable is a **UX + enforcement design** for how a grounded LLM
app treats evidence: how citations render, how freshness surfaces, and
what the app does when there is no evidence to cite. A good retrieval
pipeline is not enough; if the UI hides the citation state, users
cannot tell when the model is grounded, when it is guessing, or when
the source is stale.

Four invariants:

1. **Every claim with a numeric, name, or dated fact is either
   cited or refused.** The model's text alone is never authoritative;
   the citation is the authority.
2. **Citations are deep-links to the source**, not opaque chunk ids.
   The user can open the source, in the source system, with their
   own identity.
3. **Freshness is visible per citation**, not aggregated across the
   answer. One stale source in a four-source answer is a stale
   answer — the UI must say so and the generator must not hide it.
4. **"No evidence" is a first-class answer.** The app renders a
   designed empty state, not a hallucinated best guess.

The common FDE mistake is to build the retrieval layer first and the
UX last, then discover that the model freely cites chunks the user
cannot open, that staleness is invisible, or that empty-retrieval
cases silently fall through to an ungrounded answer. This exercise
closes those gaps on the way in.

## 2. Worked implementation

### 2.1 What "a citation" means on the wire

The prompt-to-UI contract is a structured answer object:

```jsonc
{
  "answer": "Shipment SHPMT-421 is delayed by 3 days due to a port strike.",
  "claims": [
    {
      "text_span": [0, 24],                      // "Shipment SHPMT-421 is delayed"
      "citations": ["c1"],
      "confidence": "high"
    },
    {
      "text_span": [25, 60],                     // "by 3 days due to a port strike"
      "citations": ["c1", "c2"],
      "confidence": "medium"
    }
  ],
  "sources": [
    {
      "id": "c1",
      "doc_id": "sharepoint://ops/shipments/SHPMT-421.docx",
      "title": "SHPMT-421 Status Log",
      "source_url": "https://acme.sharepoint.com/sites/ops/_layouts/15/Doc.aspx?...",
      "snippet": "Shipment SHPMT-421 delayed by 3 days.",
      "source_system": "sharepoint",
      "source_mtime": "2026-11-18T09:04:12Z",
      "indexed_at":   "2026-11-18T09:07:03Z",
      "classification": "Internal",
      "residency": "us-east"
    },
    {
      "id": "c2",
      "doc_id": "confluence://ops/port-ops/strike-updates",
      "title": "Port Ops: Strike Updates",
      "source_url": "https://acme.atlassian.net/wiki/spaces/OPS/pages/9881",
      "snippet": "Port strike projected to continue 2–4 days.",
      "source_system": "confluence",
      "source_mtime": "2026-10-04T18:40:00Z",
      "indexed_at":   "2026-11-18T08:50:17Z",
      "classification": "Internal",
      "residency": "us-east"
    }
  ],
  "evidence_state": "grounded",       // grounded | partial | missing
  "freshness": {
    "max_age_days":  48,              // across all cited sources
    "stale_sources": ["c2"],
    "policy_days":   30               // per-tenant threshold
  },
  "generation_trace_id": "0196af...",
  "retriever_trace_id":  "b07e3a..."
}
```

- The LLM generates `answer` and `claims`; the application appends
  `sources`, `freshness`, and `evidence_state` from the retrieval
  pipeline. The LLM never invents the metadata.
- The prompt instructs the model to emit `claims` as structured
  references to the retrieved sources; a validator rejects the
  response if any citation id does not match a retrieved source.
- `text_span` lets the UI underline the exact phrase the citation
  supports, so hovering a citation highlights only the supported
  words, not the whole paragraph.
- `evidence_state` is derived server-side:
  - `grounded` — every claim has at least one citation.
  - `partial` — at least one claim is uncited (model opinion, no
    evidence).
  - `missing` — no sources retrieved; see §2.4.

### 2.2 Rendering rules (UI)

- Each citation chip shows `[n]` where `n` is a stable, per-answer
  index. Hover reveals title, source system, last-modified date,
  and a "stale" chip if the source is past the policy threshold.
- Click opens the `source_url` in a new tab in the **source system**,
  under the user's identity, so the source system (SharePoint,
  Confluence, Drive) enforces its own ACL at open time. This is
  defense-in-depth against drift between the index ACL and the
  source ACL.
- Uncited spans are visually distinct (dimmed or underlined with a
  dotted line) and tagged "ungrounded" on hover.
- When `evidence_state = partial`, show an inline banner: "N of M
  claims are ungrounded. Expand to see which."
- When `evidence_state = missing`, see §2.4 — this is its own
  render, not a dimmed answer.

### 2.3 Freshness policy

Three timestamps matter:

- `source_mtime` — when the source document was last modified in the
  source system.
- `indexed_at` — when the ingest pipeline wrote the current chunk.
- `now` — query time.

Definitions:

- **Source age** = `now - source_mtime`. The user-visible freshness.
- **Index lag** = `indexed_at - source_mtime`. The pipeline's own
  lag (should match the ingest SLO from exercise-01 / ex-03).
- A source is **stale** when `source_age > policy_days` for its
  document class. The policy is per tenant and per class.

Reasonable default `policy_days` (confirm with the customer):

| Content class | Default policy | Rationale |
|---|---|---|
| Policy / SOP / compliance | 90 | Low churn; stale values corrupt operational answers |
| Product spec / engineering doc | 30 | Moderate churn; teams expect weekly edits |
| Operational status / tickets | 7 | High churn; a week-old status is almost never current |
| Pricing / contractual | 1 | Must be as-of-today or refused |
| Public-facing FAQ | 60 | Low churn; customer-facing |

Rules:

- Thresholds are **per tenant**, overridable per content class. The
  values above are starting points, not defaults to ship.
- Classes come from the chunk's `source_system` + `doc_id` prefix +
  a per-tenant rules file maintained by the customer's content
  owners. This is not an AI classifier; it is a predictable
  mapping.
- Mixed-freshness answers render *per citation*. Do not aggregate
  freshness into one chip — the user needs to know *which* source
  is stale, not that *some* source is.

### 2.4 The "no evidence" case — a first-class answer

When the retriever returns zero chunks that clear the ACL and
classification filters, the app does not generate a free-form
answer. It emits:

- A short, templated response: "I did not find a source you can
  read that answers this question."
- The retriever filter trace (not the chunks it would have returned
  for a more-privileged user — see exercise-02): tenant, residency,
  freshness window, whether the filter denied hits at all.
- A "Request access" affordance that invokes the customer's
  existing access-request flow for the likely sources (identified
  by title match on a higher-privileged shadow query that returns
  only titles, never content — the title-only path requires explicit
  customer opt-in and is out of scope if their access policy
  forbids it).
- A "Refine" affordance that the UI uses to suggest query rewrites.

This design rejects three tempting shortcuts:

- **Returning a general LLM answer "from training."** The whole
  product promise is grounded-on-customer-data. An ungrounded
  answer in a grounded app is a bug class, not a feature.
- **Falling back to a broader corpus (web, other tenants).** Never,
  not even for "harmless" questions. The retrieval scope is the
  customer's corpus; broadening it silently is a disclosure risk.
- **Hallucinating a citation to appear grounded.** The validator
  in §2.5 rejects any response whose citation id is not in the
  retrieved set.

### 2.5 Server-side validation of the LLM output

Before the answer is shown to the user, the app validates:

1. `answer` and `claims` parse per schema.
2. Every `citations` entry in `claims` is a known `sources[].id`.
3. If `evidence_state = grounded`, no claim may have empty
   `citations` and non-trivial content (regex for a digit, proper
   noun, or dated phrase forces a citation).
4. For each cited `sources[]` entry, `source_mtime` and `source_url`
   are present and the URL matches the configured host for that
   `source_system`.
5. Freshness fields are computed, not model-emitted.

Validation failures:

- Soft failure (missing citation, dangling id): re-prompt the LLM
  one time with the validator errors in the system message. On
  second failure, demote to `evidence_state: partial` with an
  explanatory banner. Do not silently ship.
- Hard failure (malformed JSON, unknown source id after retry):
  show the "evidence-missing" treatment and page oncall (mod-110).

### 2.6 Minimal implementation sketch

```python
# answer_shaper.py
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from typing import List

@dataclass(frozen=True)
class Source:
    id: str; doc_id: str; title: str; source_url: str
    snippet: str; source_system: str
    source_mtime: str; indexed_at: str
    classification: str; residency: str

@dataclass
class Freshness:
    max_age_days: int
    stale_sources: List[str]
    policy_days: int

def freshness(sources: List[Source], policy_days: int) -> Freshness:
    if not sources:
        return Freshness(0, [], policy_days)
    now = datetime.now(timezone.utc)
    ages = []
    stale = []
    for s in sources:
        mtime = datetime.fromisoformat(s.source_mtime.replace("Z", "+00:00"))
        age = (now - mtime).days
        ages.append(age)
        if age > policy_days:
            stale.append(s.id)
    return Freshness(max(ages), stale, policy_days)

def shape(llm_structured, retrieved: List[Source], policy_days: int):
    # llm_structured is the validated model output (see §2.5).
    # The server fills sources/freshness/evidence_state; the LLM did not.
    ids_known = {s.id for s in retrieved}
    for claim in llm_structured["claims"]:
        if not set(claim["citations"]).issubset(ids_known):
            raise ValueError("citation id not in retrieved set")
    state = "grounded"
    if not retrieved:
        state = "missing"
    elif any(not c["citations"] for c in llm_structured["claims"]):
        state = "partial"
    return {
        **llm_structured,
        "sources":        [asdict(s) for s in retrieved],
        "evidence_state": state,
        "freshness":      asdict(freshness(retrieved, policy_days)),
    }
```

### 2.7 Edge cases worth designing for

- **Mixed-residency retrieval.** If a query somehow returns a mix of
  regions, drop the out-of-region hits and log; never show them,
  even as citations.
- **ACL drift during an open session.** The chunk cleared the index
  filter but the user has since lost access in the source system.
  The deep-link in `source_url` lets the source system refuse to
  open the document; the citation UI handles that gracefully
  ("source no longer accessible").
- **Pseudonymised entities.** If exercise-03 pseudonyms appear in
  cited snippets, the UI resolves them per the re-hydration
  policy; the citation itself never shows cleartext the caller
  cannot see.
- **PDF / non-URL sources.** `source_url` can be a
  `signed-download://` URL resolved by a dedicated service that
  checks ACLs at click time. Opaque chunk ids are not citations.
- **Multi-chunk claims.** A claim backed by three contiguous
  chunks from the same document uses the doc-level citation and
  the span, not three separate citations.
- **Non-English sources.** The citation title is in the source
  language; the snippet is in the source language; the answer
  can be translated, but the citation stays source-faithful so
  the user can verify.

## 3. Validation steps

1. **Schema conformance.** Golden set of "good" and "bad" LLM
   outputs; the validator passes the former and rejects the latter
   with specific errors.
2. **Citation-click test.** For each citation, open the deep-link
   and confirm the source system renders the source for a user
   who should see it, and refuses for a user who should not.
3. **Freshness render test.** Seed sources at known mtimes; confirm
   per-citation staleness chips render correctly across the
   per-class thresholds, and that mixed-freshness answers show
   per-citation state rather than a single aggregate.
4. **No-evidence render test.** Queries with zero retrievable
   chunks render the designed empty state, not a free-form LLM
   answer; the Refine/Request-access affordances are present and
   wired.
5. **Hallucinated-citation test.** Inject a model response with a
   `citation` id not in the retrieved set; the validator must
   reject before render.
6. **Partial-evidence banner test.** Inject a response with one
   uncited claim; the banner shows and the span is visually
   tagged ungrounded.
7. **ACL-drift test.** Revoke a user's access in the source system
   after retrieval; the deep-link click shows the source-system
   denial, not stale cached content.
8. **Trace alignment.** Every rendered answer has both retriever and
   generator trace ids wired into the customer's observability
   (mod-110); oncall can follow a user-reported "wrong citation"
   back to the exact chunks and the exact LLM call.

## 4. Rubric / review checklist

Score 0–2 per row. 14/22 passes before user-facing release.

| Dimension | 0 — Missing | 1 — Partial | 2 — Strong |
|---|---|---|---|
| Structured answer schema | Free text | Citations as footnotes | Structured claims + spans + sources |
| Validator before render | None | Format only | Format + citation-id + freshness computed server-side |
| Deep-link to source | Opaque id | Internal viewer | Source-system URL under user identity |
| Per-citation freshness | Aggregate only | Per-citation, no policy | Per-citation + per-class policy |
| Stale rendering | Invisible | Tooltip-only | Chip + per-class threshold + policy visible |
| Evidence-missing UX | Falls through to LLM | Static "no answer" | Designed empty state + refine + request-access |
| Hallucination guard | None | Soft re-prompt | Soft retry + hard reject + oncall page |
| ACL drift handling | Cached content shown | Error page | Source-system refusal + graceful UI |
| Pseudonym resolution at render | Cleartext leaked | All hidden | Authorized-only resolve per §2.6 of ex-03 |
| Residency enforcement in UI | Not considered | Dropped silently | Dropped + logged + per-answer residency visible if mixed |
| Trace ids in UI / logs | Absent | Present | Present and wired to mod-110 oncall path |

## 5. Common mistakes

- **Free-text answer with a "Sources:" list at the bottom.** The
  user cannot tell which sentence is backed by which source.
  Structure the claims.
- **Aggregated freshness ("most recent source N days old").** The
  user needs to know which source is stale, not the average.
- **Opaque chunk ids as "citations."** A citation the user cannot
  open is not a citation. Deep-link or do not call it a citation.
- **"No evidence → best-effort answer."** Breaks the grounded
  contract and trains users to trust the model's guesses.
- **Trusting the model to emit timestamps.** Freshness is
  retrieved and computed server-side. Never let the LLM mint
  `source_mtime`.
- **Citing a chunk the user cannot open in the source system.**
  The index ACL and the source-system ACL can drift; make the
  deep-link the last word on access.
- **Hiding staleness in a tooltip.** Staleness is a visible chip,
  not a hover-only.
- **One click = bulk cleartext resolution.** Pseudonymised entities
  must resolve only per authorized caller, per rendered answer
  (see exercise-03 §2.6).
- **No retry on citation validation failure.** One re-prompt with
  the validator errors recovers most failures; not retrying
  fails the user needlessly.
- **No trace correlation.** User reports "this answer is wrong";
  oncall cannot find the retrieval or the generation. Trace ids
  belong in the rendered answer (small, copyable).
- **Treating "partial evidence" as acceptable by default.** Some
  tenants want `partial` blocked entirely for compliance reasons;
  the policy is per tenant.

## 6. References

- **NIST AI Risk Management Framework** — Govern for freshness
  policy ownership, Map for source lineage surfaced in the UI,
  Measure for evidence-state distribution over time, Manage for
  oncall paths when validators fail.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — the "ungrounded
  / hallucinated output" and "insecure output handling" classes
  map directly to the hallucinated-citation guard and the
  deep-link policy.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — prompt-injection via retrieved content and
  model-output manipulation tactics motivate the server-side
  citation validator.
  <https://atlas.mitre.org/>
- Companion exercises: exercise-02 (ACLs/RLS enforced on the
  retriever), exercise-03 (pseudonymisation and re-hydration at
  the trust boundary), exercise-05 (gold set includes
  "must-not-answer" and "must-cite" probes); mod-110
  (observability / oncall) for trace plumbing.
