# exercise-03 — Ingest Pipeline with PII Redaction and Reversible Pseudonymisation · Reference Solution

## 1. Solution overview

The deliverable is a **reference design + runnable artifacts** for an
ingest pipeline that:

1. Pulls documents from the source-of-truth system,
2. Detects PII and other sensitive tokens with a pluggable classifier
   (customer-tunable),
3. Replaces them with **reversible pseudonyms** keyed by a KMS-backed
   secret, so downstream retrieval and generation see realistic
   placeholders, and
4. **Re-hydrates the originals only at the trust boundary** (the
   caller's UI) when the caller is authorized.

The pipeline writes into the retrieval index from exercise-02; the
per-chunk ACL, residency, and classification fields it emits are the
same schema. The ingest worker is the **first and last hop that sees
cleartext PII** — everything downstream (embeddings, index, prompts,
LLM calls, traces, caches) works on pseudonymised text.

The common FDE mistake is to redact one way: strip PII to `[REDACTED]`
and move on. That breaks referential integrity ("the second person
mentioned above") and makes the LLM's answers useless for operational
tasks. Reversible pseudonymisation preserves utility while keeping
the raw value off the wire.

> This is not a replacement for a formal data-loss-prevention program.
> It is the retrieval-layer slice of it. The customer's privacy officer
> owns the policy; this exercise owns the implementation shape.

## 2. Worked implementation

### 2.1 Pipeline shape

```
+------------- customer VPC -------------+       +--- optional ---+
|                                        |       |                |
|  source  --->  fetch  --->  normalize  |       |  reverse-map   |
|  system         |            |         |       |  resolver (UI) |
|                 v            v         |       |                |
|              virus-scan   PII detect   |       +----------------+
|                              |         |               ^
|                              v         |               |
|                          pseudonymise  |               |
|                              |         |               |
|                              v         |               |
|                           chunk +      |               |
|                           ACL flatten  |               |
|                              |         |               |
|                              v         |               |
|                           embed via    |               |
|                           gateway      |               |
|                              |         |               |
|                              v         |               |
|                           upsert ---->  retrieval index (ex-02)  |
|                                        |               |
|                           token-vault  |---------------+
|                           (KMS-wrapped |
|                            per-tenant) |
+----------------------------------------+
```

- The ingest workers live in the customer VPC; no PII crosses an
  untrusted boundary.
- The **token vault** holds `(tenant, token_id) → ciphertext(original)`
  rows with per-tenant envelope keys. It is the only component that
  can de-pseudonymise.
- The **reverse-map resolver** sits with the UI (optional — see §2.6).
  It fetches cleartext only when the caller is authorized and only
  for the tokens that appear in the answer.

### 2.2 Normalisation before detection

Before the classifier runs, the raw document is normalised:

- Unicode NFC, CRLF → LF, strip zero-width characters that would
  break token matching (ZWJ, ZWNJ, ZWSP, soft-hyphen).
- OCR output (for PDFs / images) is run through a confidence filter;
  pages below threshold are quarantined and not indexed.
- Boilerplate (headers, footers, running titles) is stripped before
  detection so detectors are not fighting repeated noise.
- Language is detected per section; multilingual documents are split
  before classification so a language-specific detector is used.

This matters because PII detectors have very different quality across
languages, and detection on un-normalised text misses variants.

### 2.3 Detection — pluggable, not a single monolith

Use a **layered** detector so the customer can tune without rewriting
the pipeline. A workable layering:

| Layer | Examples | Use for |
|---|---|---|
| **Deterministic** | Luhn (credit cards), ISO country IBANs, structured ID formats (SSN, NINO, SIRET), email, phone, URLs | High precision; cheap; run first |
| **Dictionary / list** | Customer-provided lists of employee names, internal product codes, cost-center IDs | Names, internal identifiers; customer-maintained |
| **NER model** | Open-source NER (e.g., spaCy / Stanza) or a vendor DLP API in the customer's cloud (e.g., AWS Comprehend PII, Azure AI Language PII detection, GCP DLP) | Person, org, location, non-trivial spans |
| **Classifier-of-classifiers** | Weighted vote + a confidence threshold per entity type | Resolve conflicts; drop low-confidence spans |

Rules:

- Never send raw text to a classifier **outside** the customer's
  trust boundary. Vendor DLP APIs are acceptable only if they live in
  the customer's cloud account (same-region, same-tenant), with
  zero retention and BYOK.
- Detector output is a list of `{span, entity_type, confidence}`
  tuples. Overlapping spans are merged by entity type; ties go to
  the most specific type (`SSN` beats `ID_NUMBER`).
- A customer-tunable allowlist exists for false positives that are
  not sensitive (e.g., sample numbers in the product documentation).

### 2.4 Pseudonymisation — reversible by design

Every detected span is replaced with a **stable, per-tenant pseudonym**:

```
Original: "Contact Jane Doe at jane.doe@acme.com or 415-555-0188."
Redacted: "Contact [[PERSON:t_7f3a]] at [[EMAIL:t_8c21]] or [[PHONE:t_a9b0]]."
```

The token grammar (`[[TYPE:ID]]`) must:

- Be recognisable to downstream code (retrieval, reranker, LLM)
  without being mistaken for legitimate content.
- Survive tokenizer normalisation — pick delimiters that most
  tokenizers keep intact (double brackets with an ASCII type, short
  alphanumeric id).
- Carry the entity type so the LLM can generate natural answers
  ("call PERSON_1 at PHONE_1") that the resolver can expand.

Stability rules:

- The **same original** in the same tenant produces the **same
  token id** across documents. Enables referential integrity:
  the LLM can correctly say "Jane and Jane are the same person"
  across two retrieved chunks.
- The token id is derived as an HMAC of the normalised cleartext
  with a per-tenant key, truncated. **Never** use a non-keyed hash
  — that is a dictionary-attack vector against common names.
- The token id is **not** reversible on its own. The vault row is
  the only way back.

Schema:

```sql
create table token_vault (
    tenant_id      text not null,
    token_id       text not null,
    entity_type    text not null,
    ciphertext     bytea not null,    -- envelope-encrypted original
    created_at     timestamptz not null default now(),
    source_system  text not null,
    doc_id         text not null,
    primary key (tenant_id, token_id)
);

create index on token_vault (tenant_id, source_system, doc_id);
```

Encryption:

- Envelope encryption: a per-tenant Data Encryption Key (DEK) wrapped
  by a customer-managed KMS CMK. Rotate CMKs per the customer's
  rotation policy; rewrap DEKs on rotation; cleartext is never
  rewritten.
- Vault is **in the customer VPC**. The LLM provider, embedding
  provider, and the vector index never see cleartext.

### 2.5 A minimal, runnable sketch

```python
# ingest/pseudonymise.py
# Intent: sketch, not a full DLP. See §2.3 — real deployments layer
# deterministic regex + dictionary + NER + a confidence threshold.
import os, re, hmac, hashlib, base64
from dataclasses import dataclass
from typing import List, Tuple

TENANT_KEY = os.environ["LANTERN_TENANT_HMAC_KEY"].encode()  # 32 bytes

EMAIL_RE = re.compile(r"\b[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}\b", re.I)
PHONE_RE = re.compile(r"\b(?:\+?\d{1,3}[-. ]?)?(?:\(\d{2,4}\)|\d{2,4})"
                      r"[-. ]?\d{3,4}[-. ]?\d{3,4}\b")
# SSN-shaped (do not assume this detects every SSN variant; see §2.3)
SSN_RE   = re.compile(r"\b\d{3}-\d{2}-\d{4}\b")

@dataclass(frozen=True)
class Span:
    start: int
    end: int
    entity_type: str

def _token_id(tenant: str, cleartext: str) -> str:
    mac = hmac.new(TENANT_KEY, f"{tenant}:{cleartext}".encode(),
                   hashlib.sha256).digest()
    return base64.b32encode(mac[:5]).decode().rstrip("=").lower()

def _detect(text: str) -> List[Span]:
    spans: List[Span] = []
    for m in EMAIL_RE.finditer(text):
        spans.append(Span(m.start(), m.end(), "EMAIL"))
    for m in PHONE_RE.finditer(text):
        spans.append(Span(m.start(), m.end(), "PHONE"))
    for m in SSN_RE.finditer(text):
        spans.append(Span(m.start(), m.end(), "SSN"))
    # NER / dictionary layers would append more here.
    # Merge overlaps; most-specific type wins.
    spans.sort(key=lambda s: (s.start, -s.end))
    merged: List[Span] = []
    for s in spans:
        if merged and s.start < merged[-1].end:
            keep = s if _specificity(s.entity_type) > \
                _specificity(merged[-1].entity_type) else merged[-1]
            merged[-1] = keep
        else:
            merged.append(s)
    return merged

_PRIORITY = {"SSN": 3, "EMAIL": 2, "PHONE": 2, "PERSON": 1, "ID": 1}
def _specificity(t: str) -> int:
    return _PRIORITY.get(t, 0)

def pseudonymise(text: str, tenant: str, doc_id: str, source: str
                 ) -> Tuple[str, List[dict]]:
    spans = _detect(text)
    out, vault_rows, cursor = [], [], 0
    for s in spans:
        out.append(text[cursor:s.start])
        cleartext = text[s.start:s.end]
        tid = _token_id(tenant, cleartext.lower())
        out.append(f"[[{s.entity_type}:{tid}]]")
        vault_rows.append({
            "tenant_id":     tenant,
            "token_id":      tid,
            "entity_type":   s.entity_type,
            "cleartext":     cleartext,  # vault writer envelope-encrypts
            "source_system": source,
            "doc_id":        doc_id,
        })
        cursor = s.end
    out.append(text[cursor:])
    return "".join(out), vault_rows
```

Notes:

- Treat the regexes above as *illustration*. Production deployments
  layer dictionary lists and NER on top; see §2.3.
- `vault_rows` is handed to a writer that envelope-encrypts
  `cleartext` under the tenant DEK and discards the plaintext from
  process memory.
- Token ids are derived from **normalised** cleartext (lowercased,
  trimmed) so "Jane Doe" and " jane doe " collapse.

### 2.6 Re-hydration at the trust boundary

The retrieval and generation path works on pseudonymised text. When
the LLM emits an answer like:

```
Call [[PERSON:a7q3]] at [[EMAIL:x9k2]] about the shipment delay.
```

the UI (or an answer-shaping proxy owned by the application) decides
whether to resolve tokens:

- **If the caller is authorized to see the cleartext** (per ACL +
  classification + purpose-of-use rules), the resolver queries the
  token vault and returns a version of the answer with the originals
  spliced back in.
- **If not**, the pseudonyms stay in the rendered answer; the UI
  shows a hover ("Contact information hidden; request access") that
  invokes the customer's existing access-request flow.

The resolver:

- Runs inside the customer VPC.
- Takes a signed caller identity (same shape as exercise-02).
- Returns a map `{token_id → cleartext | null}` scoped to the tokens
  **present in this answer**. Never return the full vault.
- Logs every resolution to the customer-owned audit sink with
  `{trace_id, principal, tokens, doc_ids}`.
- Rejects bulk resolution requests — one call per rendered answer.

The LLM provider, the model gateway, and the vector index must never
have the vault credentials. This separation is what keeps "pseudonym
leaked into a log" from becoming "cleartext leaked into a log."

### 2.7 Deletes, erasure requests, and re-ingestion

Three operations the pipeline must support cleanly:

1. **Document delete (source).** Delete the chunks (exercise-02
   tombstone), delete the vault rows for `(tenant, source, doc_id)`.
   Order matters: tombstone first (new queries stop seeing), then
   purge vault.
2. **GDPR / data-subject erasure (per person).** Find all
   `token_vault` rows where the cleartext maps to the subject.
   Because the token id is HMAC-keyed with the tenant key, the
   token id for a subject is reproducible: recompute the ids from
   the subject's known identifiers, delete matching vault rows,
   and rewrite the affected chunks' pseudonyms to a non-reversible
   `[[REDACTED:TYPE]]` form. Do **not** keep an "erased but
   recoverable" state.
3. **Key rotation.** CMK rotation re-wraps the DEK without touching
   ciphertext. DEK rotation (rare, e.g., on compromise) requires
   re-encrypting all vault rows under a new DEK — plan the
   maintenance window with the customer; the pipeline has an
   offline tool for this, not an online path.

### 2.8 What never leaves the VPC

- Cleartext PII.
- Raw documents (except when the source is already outside the VPC
  and the ingest worker pulls from it).
- Vault contents.
- KMS keys.
- Audit logs (the customer owns the sink).

The LLM gateway sees only pseudonymised chunks; the vector index
stores only pseudonymised text + its embedding.

## 3. Validation steps

1. **Round-trip test.** Ingest a corpus with known PII, assert that
   `pseudonymise → resolve` returns the original cleartext exactly,
   and that `pseudonymise → index → retrieve → render` yields the
   same final text when the caller is authorized.
2. **Cleartext-leak test.** Run a canary document through the whole
   pipeline (ingest → embed → index → retrieve → prompt → LLM → log)
   and grep every emitted artifact (chunks, embeddings-adjacent
   metadata, prompt bytes, traces, audit logs, cache entries) for
   each cleartext canary. Zero hits required.
3. **Referential integrity.** Confirm that the same cleartext appearing
   in two chunks produces the same token id, and that an LLM asked
   "is PERSON_A the same as PERSON_B?" (when they are) gets to a
   correct answer without seeing cleartext.
4. **Detector coverage.** A labeled regression set, per tenant, of
   known PII examples (customer-provided). Precision ≥ customer's
   target; recall ≥ customer's target. Thresholds are customer-
   owned decisions.
5. **False-positive regression set.** Non-sensitive strings that
   superficially resemble PII (sample SSN placeholders in training
   docs, test emails on `example.com`). Confirm the allowlist
   suppresses them.
6. **Erasure drill.** Subject-erasure request on a known subject;
   assert vault rows gone, chunks rewritten to irreversible redactions,
   search for the subject's identifiers returns no matches.
7. **Key rotation drill.** Simulate CMK rotation; the pipeline must
   keep running and reads must keep working.
8. **Boundary audit.** Log review confirming no cleartext left the
   VPC; a sampling script picks N random chunks and asserts no
   detected entity type appears in the stored text.

## 4. Rubric / review checklist

Score 0–2 per row. 16/24 passes before pilot.

| Dimension | 0 — Missing | 1 — Partial | 2 — Strong |
|---|---|---|---|
| Layered detection | Regex only | Regex + dictionary | Regex + dict + NER + confidence vote |
| Normalisation before detection | None | Partial | NFC + CRLF + ZW strip + per-language split |
| Pseudonym grammar | `[REDACTED]` | Typed, unstable | Typed, stable, tokenizer-safe |
| Token derivation | Non-keyed hash | HMAC with global key | HMAC with per-tenant KMS key |
| Vault storage | Plain DB | Encrypted at rest | Envelope + per-tenant DEK + CMK |
| Residency of vault and detectors | External vendor API | Managed in customer cloud | In-VPC only, no egress |
| Re-hydration policy | Everyone sees cleartext | ACL at the UI, ad hoc | Server-side authorization + audit per resolve |
| Erasure support | None | Delete chunk only | Chunk + vault + irreversible rewrite |
| Key rotation | None | Manual | CMK rotation is a no-op; DEK rotation documented |
| Audit | None | Service log | Per-resolve audit to customer sink |
| Downstream contract | Cleartext in prompts | Sometimes clean | LLM/index/caches see only pseudonyms |
| Regression + canary tests | None | One per class | Per-tenant regression + canary pipeline |

## 5. Common mistakes

- **Irreversible `[REDACTED]`.** Loses referential integrity; answers
  become unusable ("call the person at the email"). Reversible
  pseudonyms are the whole point.
- **Non-keyed hash token ids.** Known-plaintext attacks recover
  common values from the token id directly. Always HMAC with a
  per-tenant key.
- **Vault in the app database.** Keep the vault separate, with its
  own IAM, so a database compromise on the app tier is not a
  cleartext dump.
- **Detector outside the trust boundary.** Sending raw text to a
  vendor DLP API in a different region to "classify PII" is itself
  the leak you were trying to prevent.
- **Forgetting language and OCR.** English-only regexes miss most
  non-US PII; low-confidence OCR produces garbled strings that
  escape both regex and NER. Normalise and language-split first.
- **No audit on resolution.** Resolving pseudonyms without an
  audit row is a backdoor. Every resolve is a security event.
- **Bulk resolution endpoint.** A "resolve this entire doc" endpoint
  is a vault-dump by API. Resolve only tokens present in a
  specific rendered answer.
- **Ignoring key rotation.** Vault rows accumulate for years; a
  rotation plan on day 1 is cheaper than a scramble on day 730.
- **No erasure drill.** Subject-erasure requests arrive with legal
  SLAs. Dry-run the operation before the first request.
- **Letting the LLM see cleartext "just once" for one use case.**
  Every exception becomes a permanent exception; enforce the
  pseudonymised-everywhere invariant.
- **No referential-integrity test.** Without stable tokens, the LLM
  loses coreference across chunks and the UX degrades silently.

## 6. References

- **NIST AI Risk Management Framework** — use Map for data lineage
  (what's in the corpus, what's redacted), Govern for the privacy
  policy owner, Measure for detector precision/recall, Manage for
  erasure SLOs and key rotation.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — map training-data /
  retrieval leakage and model-output-based exfiltration to the
  current project list; the pseudonymisation layer is the retrieval
  analogue of training-data sanitisation.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — covers data-exfiltration-by-prompt-injection and
  model-output manipulation tactics where an attacker coaxes the
  model into echoing stored secrets. <https://atlas.mitre.org/>
- Companion exercises: exercise-01 (engine selection), exercise-02
  (ACLs/RLS), exercise-04 (citation UX including how pseudonyms
  render), exercise-05 (gold set with "must not leak" probes);
  mod-104 ex-02 (gateway) and mod-108 (security + compliance gate).
- Customer-specific privacy references (GDPR Art. 4 / Art. 17,
  HIPAA Privacy Rule, state-level privacy acts, sectoral guidance)
  — cite the specific obligation the customer's privacy officer has
  asserted applies; this exercise does not substitute for that
  determination.
