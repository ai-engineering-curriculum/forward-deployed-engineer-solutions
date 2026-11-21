# exercise-05 — Legacy surface integration: SOAP, SFTP, mainframe · Reference Solution

## 1. Solution overview

The deliverable is an **integration design** for three legacy surfaces a
forward-deployed engineer still meets in regulated industries:

- **SOAP / WS-* (XML over HTTP)** — insurance, banking back-office,
  telco provisioning APIs. WSDL contract, strict XML Schema, often
  WS-Security for signatures.
- **SFTP batch file drop** — payers, clearinghouses, transfer agents,
  anything with a "nightly file." CSV/positional/fixed-width, PGP
  signed/encrypted, SLA'd by arrival window.
- **Mainframe (CICS / IMS / zOS)** — core banking, claims, inventory
  of record. Reached through a gateway (IBM z/OS Connect, MQ, or a
  3270-screen-scrape façade when nothing else exists).

The common FDE mistake is to treat these as modern REST and get
surprised by brittle contracts, slow release cycles, and failure modes
that live in file-system races and CICS transaction timeouts, not in
HTTP status codes.

The design principles:

1. **Put a modern facade in front, never consume raw from the agent.**
   Lantern's inner services talk to an **adapter** that normalizes to a
   small internal API. The adapter is where the legacy warts live.
2. **Idempotency through keys you own.** Legacy writes almost never
   support native idempotency; the adapter adds a request-id,
   persists it, and lets the system of record detect replays.
3. **Treat time as a first-class concern.** SFTP windows, batch
   cutovers, mainframe daily maintenance windows — every integration
   has blackout hours; the scheduler respects them.
4. **Governance.** Legacy systems often sit on data classified at a
   higher tier than any of the modern SaaS. Lantern's minimum-access
   path is stricter, not looser.

## 2. Implementation — three legacy integrations

### 2.1 SOAP service (claims-status check)

Scenario: Lantern's agent calls a national payer's **claims status**
service to answer eligibility questions. WSDL 1.1, document/literal,
WS-Security UsernameToken + X.509 message signature over mTLS.

**Contract first.** Pull the WSDL into the repo; generate the client
with a stable toolchain:

```bash
# Example only — pick the generator your team runs in CI
wsdl2code --wsdl claims-status-v3.wsdl \
          --language python \
          --out gen/claims_status/
```

Every change to the WSDL is a tracked artifact; regenerate in CI; code
review diffs the generated client to catch silent breaking changes.

**Request (SOAP 1.1):**

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:cs="http://payer.example.com/claims/v3">
  <soapenv:Header>
    <wsse:Security xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/..."
                   soapenv:mustUnderstand="1">
      <wsse:UsernameToken>
        <wsse:Username>lantern_svc</wsse:Username>
        <wsse:Password Type="...#PasswordDigest">...</wsse:Password>
        <wsse:Nonce EncodingType="...#Base64Binary">...</wsse:Nonce>
        <wsu:Created>2026-11-20T15:00:00Z</wsu:Created>
      </wsse:UsernameToken>
      <!-- XML-Signature over soapenv:Body with Lantern's X.509 cert -->
      <ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#"> ... </ds:Signature>
    </wsse:Security>
  </soapenv:Header>
  <soapenv:Body>
    <cs:StatusRequest>
      <cs:RequestId>5f3a0f16-0b15-4d1a-9da3-24d1a61e6201</cs:RequestId>
      <cs:MemberId>MBR-9981234</cs:MemberId>
      <cs:ClaimId>CLM-20261118-00231</cs:ClaimId>
    </cs:StatusRequest>
  </soapenv:Body>
</soapenv:Envelope>
```

**Transport & security:**

- TLS 1.3 to the payer endpoint.
- Lantern's X.509 cert pinned per environment; rotation playbook.
- Message signed with XML-Signature over `soapenv:Body`.
- Timestamp in `wsu:Created`; replay rejection if skew > 5 minutes.

**Idempotency:**

- `RequestId` is Lantern-generated, persisted in the adapter DB with
  the response. Retries reuse the same `RequestId`; the payer dedups
  or Lantern returns the cached response if the payer does not.

**Error model:** SOAP Faults are strictly typed in the WSDL; map every
fault code to a Lantern error class. Do not swallow `<soapenv:Fault>`
into a generic 500.

**Rate and schedule:**

- Payer publishes a QPS and a daily cap; adapter rate-limits per
  environment.
- Provider's maintenance window (e.g., Sunday 02:00–04:00 ET) is a
  no-call window; scheduler suppresses non-urgent calls.

**Governance:**

- PHI in both request and response; Lantern's adapter encrypts at
  rest (customer-managed key) and never logs full bodies — only
  `RequestId` + status code.
- BAA (Business Associate Agreement) acknowledged in the integration
  spec where HIPAA applies.

### 2.2 SFTP batch file drop

Scenario: Each night, a clearinghouse writes an `835` (EDI remittance
advice) file to a shared SFTP server. Lantern picks it up, parses, and
ships normalized events into the pilot workflow.

**Transport:** SFTP (SSH File Transfer Protocol) — not FTPS.

- Key-based authentication; Lantern's public key provisioned by the
  partner; private key in KMS.
- Known-hosts file pinned; host key rotation choreographed.
- Dedicated chroot on the partner side; Lantern cannot list outside
  the inbound directory.

**File contract:**

- Naming: `835_<payer>_<YYYYMMDD>_<seq>.pgp` (encrypted + signed) and
  `835_<payer>_<YYYYMMDD>_<seq>.pgp.sig` (optional detached sig).
- Arrival window: 23:00–02:00 ET; SLA says 95% of files by 00:30 ET.
- Encoding: PGP-encrypted to Lantern's public key; signed with the
  partner's PGP key.
- Delivery semantics: at-least-once. Partner MAY resend a file after
  the window; Lantern deduplicates by `<payer, date, seq>` and by
  checksum.

**Atomic-swap convention:** files land first as `.part` and are
renamed to the final name only after the full transfer completes.
Lantern's watcher only acts on the final name — polling on
`.part` causes torn reads.

**Pipeline:**

```
SFTP watcher (every 60 s)
   │
   ├── enumerate inbound/*.pgp (exclude *.part)
   │
   └── for each new file:
         1. compute SHA-256; dedup by (filename, sha256)
         2. verify PGP signature
         3. decrypt → /staging/<payer>/<YYYYMMDD>/
         4. parse (EDI parser, strict ISA/GS/ST segment validation)
         5. produce CloudEvents to Lantern event bus
         6. move source file to /archive/<payer>/<YYYYMMDD>/
         7. commit watermark (last-seen filename, lexicographic)
```

**Checkpoint & resume:**

- The watermark (last successful filename or ISO timestamp) persists in
  Lantern's control plane. On restart, resume from the watermark.
- A file in `/staging/` with no corresponding `/archive/` move is
  incomplete; re-process on restart.

**Observability:**

- Metrics: `sftp.files.pending`, `sftp.files.failed`, `sftp.oldest.age_s`,
  `sftp.last_success_ts`.
- Alert: no file received in the arrival window for a partner that
  publishes daily → page the integration owner.

**Error handling:**

- Decryption failure → quarantine, do not retry (likely key rotation
  mismatch); alert partner.
- Parse failure → quarantine individual file; move malformed file to
  `quarantine/` with the parser error.
- Downstream event-bus failure → retry with backoff; file stays in
  `staging/` until commit succeeds. Never move to archive before
  commit.

**Security:**

- `umask 077` on all transfer directories.
- File-system permissions deny read to other service accounts.
- Lantern logs file metadata only, never PHI/PII content.

### 2.3 Mainframe (z/OS CICS via z/OS Connect)

Scenario: The customer's core banking is on CICS/COBOL. Lantern needs
to look up the current balance on an account and record a note. The
customer has **IBM z/OS Connect** in front of CICS, exposing REST +
JSON contracts over a bounded set of transactions.

**Preferred path:** z/OS Connect REST. The alternatives, in decreasing
order of preference:

1. z/OS Connect REST API (HTTPS, JSON, OAuth or mTLS).
2. IBM MQ with request/reply queues (JMS / AMQP).
3. Direct CICS Transaction Gateway (ECI), only via a Lantern-hosted
   adapter.
4. 3270 screen scrape — only when nothing else exists, and with the
   customer's explicit sign-off that this is a transitional bridge.

**z/OS Connect request:**

```http
POST /services/AccountInquiry HTTPS/1.1
Host: zcon.customer.bank.example.com
Authorization: Bearer <access_token>
X-Correlation-Id: 5f3a0f16-0b15-4d1a-9da3-24d1a61e6201
Content-Type: application/json

{
  "accountNumber": "0000123456789",
  "effectiveDate": "2026-11-20"
}
```

Response (strict JSON schema published by z/OS Connect):

```json
{
  "status": "OK",
  "balance": { "amount": "1245.67", "currency": "USD" },
  "asOf": "2026-11-20T15:03:07Z",
  "mainframeTxId": "AB12-000089341"
}
```

**Security:**

- mTLS between Lantern adapter and z/OS Connect.
- Service principal registered in RACF (or ACF2/Top Secret) with
  **exactly** the CICS transactions in scope — `AccountInquiry`,
  `AccountNoteCreate`, no wildcards.
- RACF audit trail retained ≥180 days per customer policy.

**Throughput and scheduling:**

- CICS transaction concurrency is capped at the region's MAXTASKS; the
  customer's mainframe team publishes a budget (e.g., 50 concurrent
  Lantern transactions). Lantern enforces client-side.
- **Batch window.** Nightly from 23:30–03:30 local, online transactions
  can be unavailable. The scheduler suppresses non-urgent calls.
- EOD (end-of-day) process windows published monthly; integrate with
  the customer's change calendar.

**Idempotency:**

- `X-Correlation-Id` is Lantern-generated UUID; the adapter stores
  it with the mainframe `TxId` returned. Retries reuse the same
  correlation id; the adapter checks for existing `TxId` before
  re-submitting a write.

**Error semantics:**

- Mainframe timeouts are different from HTTP timeouts. A 504 from
  z/OS Connect does not mean the CICS transaction didn't execute.
  **Write operations** must use idempotency + a reconciliation
  query; do not blind-retry.
- Specific SQLCODE / ABEND codes from DB2 or CICS map to retryable
  vs. non-retryable in the adapter.

**3270 screen-scrape path (last resort):**

- Terminal emulator library (e.g., `s3270`) driven by a Lantern-hosted
  adapter; HLLAPI or equivalent for structured field access.
- Each session is isolated (process per session); timeouts enforced;
  screen captures of failures retained for debugging.
- Explicit customer sign-off that this is a bridge until z/OS Connect
  or MQ is available; deprecation date in the spec.

### 2.4 Cross-surface concerns

- **Dedicated adapter per surface.** Lantern services never speak SOAP,
  SFTP, or MQ directly. The adapter owns the protocol quirks.
- **Change and release calendar.** Legacy platforms release quarterly
  at best, after lengthy UAT. Treat every change as a mini-project:
  WSDL / schema diff, UAT on a pre-prod, promotion window agreed.
- **Audit and retention.** Regulated systems demand longer retention
  (often 7 years for financial, 6+ for health). Lantern's adapter
  stores only correlation metadata; the system of record keeps the
  payload.
- **Data residency.** Legacy data often lives in-country; Lantern's
  adapter must deploy within the same jurisdiction. Call out in the
  deployment architecture.

## 3. Validation steps

1. **Contract diff in CI.** For SOAP, the generated client is in the
   repo; any WSDL change produces a visible diff and requires review.
2. **Negative TLS test.** Attempt to connect without mTLS; verify
   rejection. Attempt with an expired cert; verify rejection.
3. **Replay test (SOAP).** Re-send a signed message with a stale
   timestamp (>5 min); verify payer rejects, adapter logs the rejection.
4. **Idempotency test (writes).** Retry a `AccountNoteCreate` with the
   same `X-Correlation-Id`; verify the mainframe records one note.
5. **SFTP atomic-swap test.** Drop a `.part` file; verify Lantern does
   not process it. Rename to final; verify it does.
6. **SFTP replay test.** Re-upload yesterday's file with the same
   checksum; verify dedup — no duplicate downstream events.
7. **PGP failure path.** Encrypt with an invalid key; verify the file
   is quarantined, not retried, and the partner alert fires.
8. **Watermark resume test.** Kill the SFTP watcher after it decrypts
   a file but before archive; restart; verify re-processing is
   idempotent at the event-bus consumer.
9. **Batch-window suppressor test.** Attempt a non-urgent mainframe
   call during the published batch window; verify Lantern queues it
   until the window closes.
10. **Write-timeout reconciliation.** Simulate a 504 from z/OS Connect
    on a write; verify the adapter runs the reconciliation query
    before deciding to retry.
11. **Audit record spot-check.** Pull a Lantern log and a system-of-
    record log for the same correlation id; both present, both
    time-aligned within 1 s.
12. **Data-residency check.** Confirm all compute and storage for
    the adapter live in the required jurisdiction.

## 4. Rubric / review checklist

Score 0–2 per row. 14/22 passes.

| Dimension                                       | 0 — Missing                              | 1 — Partial                                        | 2 — Strong                                                                         |
|-------------------------------------------------|------------------------------------------|----------------------------------------------------|------------------------------------------------------------------------------------|
| Adapter boundary                                | Services call protocol directly          | Shared adapter library                             | Separate adapter service per surface, protocol quirks contained                    |
| SOAP contract management                       | WSDL not in repo                         | WSDL in repo, hand-edited client                   | WSDL + generated client + CI diff                                                   |
| WS-Security / mTLS                             | Basic auth over TLS                      | UsernameToken only                                 | UsernameToken + XML-Sig + mTLS + replay protection                                   |
| SFTP transport                                 | Password auth                            | Key auth, no known-hosts pinning                   | Key auth + known-hosts pinning + chroot + atomic swap                                |
| Batch dedup / checkpointing                    | None                                     | Watermark only                                     | Watermark + checksum + archive-after-commit                                          |
| Encryption at rest and in transit              | Transit only                             | Transit + KMS for keys                             | Transit + KMS + PGP on batch + CMK on adapter store                                  |
| Mainframe path choice                          | 3270 screen-scrape by default            | ECI without adapter                                | z/OS Connect + mTLS + scoped RACF entitlements                                       |
| Idempotency on writes                          | Blind retry                              | Correlation id, no reconciliation                  | Correlation id + reconciliation query + dedup against system of record                |
| Schedule awareness                             | 24/7 assumed                             | Known windows documented                           | Windows enforced in scheduler, alerts tuned to window                                |
| Error taxonomy                                 | All errors → 500                         | Partial mapping                                    | Faults mapped per WSDL / SQLCODE / ABEND; retry policy per class                     |
| Governance mapping                             | None                                     | Mentions NIST                                      | NIST AI RMF Map/Manage mapping + jurisdiction/BAA call-outs                           |

## 5. Common mistakes

- **Letting Lantern services speak SOAP directly.** The protocol code
  leaks into every call site; a WSDL change becomes a mass refactor.
  Adapter from day one.
- **Password auth on SFTP.** You will rotate it, lose it, or leak it.
  Keys only; rotate via the partner's documented process.
- **Treating file arrival as the trigger for `.part` files.** You read a
  half-written file, parse garbage, and propagate bad data downstream.
- **Blind retry on a mainframe write.** The customer's reconciliation
  team finds duplicate journal entries three days later.
- **Logging full SOAP bodies.** You just put PHI/PII into your
  application log. Log the correlation id and the fault code only.
- **No batch window awareness.** You call the core at 01:00 during EOD
  and fail hard. Customer's mainframe team now distrusts the team.
- **WSDL hand-edited.** The next partner release silently breaks the
  hand edits; nobody notices until an obscure field returns empty.
- **Trusting SFTP file naming conventions without checksums.** A partner
  retransmits the same file with a new sequence number; Lantern
  processes it twice.
- **Skipping XML-Signature verification.** You've authenticated the
  transport but not the sender; a man-in-the-middle at the partner's
  gateway could inject content.
- **Running the adapter outside the required jurisdiction.** The data
  crossed a border it wasn't allowed to; the customer's privacy officer
  now wants a full audit.

## 6. References

- **SOAP 1.1 / 1.2** (W3C) — envelope, encoding rules, fault model.
  <https://www.w3.org/TR/soap/>
- **WSDL 1.1** (W3C) — Web Services Description Language.
  <https://www.w3.org/TR/wsdl/>
- **WS-Security 1.1** (OASIS) — SOAP message security, UsernameToken,
  X.509 token, XML-Signature.
  <https://docs.oasis-open.org/wss/v1.1/>
- **XML-Signature Syntax and Processing** (W3C).
  <https://www.w3.org/TR/xmldsig-core/>
- **IETF SFTP** — Secure Shell File Transfer Protocol (draft; widely
  implemented via OpenSSH).
  <https://datatracker.ietf.org/doc/html/draft-ietf-secsh-filexfer>
- **OpenPGP (RFC 9580)** — message encryption and signature format.
  <https://www.rfc-editor.org/rfc/rfc9580>
- **RFC 8446** — TLS 1.3. <https://www.rfc-editor.org/rfc/rfc8446>
- **IBM z/OS Connect** documentation — REST façade for CICS/IMS.
  <https://www.ibm.com/docs/en/zosconnect>
- **IBM CICS Transaction Server** documentation.
  <https://www.ibm.com/docs/en/cics-ts>
- **IBM MQ** documentation — message-queued integration with mainframe.
  <https://www.ibm.com/docs/en/ibm-mq>
- **ASC X12 EDI** standards — 835/837 structure (through the X12
  organization; many implementation guides are customer-specific).
- **NIST AI Risk Management Framework** — Map, Measure, and Manage
  functions cited for legacy-surface governance, audit, and
  jurisdictional controls.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — input and output
  validation when legacy records feed the model context.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — reference for adversarial techniques on data
  pipelines sourced from legacy systems. <https://atlas.mitre.org/>
