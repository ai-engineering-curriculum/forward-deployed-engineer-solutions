# exercise-03 — Design the Human-in-the-Loop Checkpoint Model · Reference Solution

## 1. Solution overview

The deliverable is a **checkpoint model** that names, for each agent
surface in the pilot, exactly *when* a human reviews, *what* they see,
*what* they can do, and *what happens if they do nothing*. The model has
to survive the customer's change-management and security review without
hand-waving the word "oversight."

Three design principles separate a defensible checkpoint model from a
"please confirm" dialog box:

1. **Tier by reversibility and blast radius, not by vibe.** A read-only
   query, a dry-run plan, and a blind irreversible write are three
   different checkpoints. Mapping each tool to a tier is the whole
   exercise.
2. **Approvals are typed, scoped, signed, and short-lived.** "User
   clicked Approve" is not an approval. An approval is a token bound to
   a specific payload, issued by an authenticated principal, with a TTL.
3. **Timeouts fail *closed*.** If the approver walks away, the write
   must *not* execute. Fail-open is the single most common HITL
   anti-pattern and the fastest way to lose an audit.

The model lands on **four checkpoint tiers** (L0–L3) plus two
orthogonal concerns (session-level and asynchronous). The tiers mirror
the NIST AI RMF "Manage" function: maintain the ability to oversee,
override, and recover from AI output in production.

> This exercise designs the checkpoint model. Implementation of the
> approval-token flow lives in exercise-02; the tool inventory it
> gates lives in exercise-04; the adversarial failure modes it must
> survive live in exercise-05.

## 2. Implementation — the checkpoint model

### 2.1 The four tiers

| Tier | Name | What the agent can do | Human role | Default |
|---|---|---|---|---|
| **L0 — Autonomous (read-only)** | "Observe freely" | Call read-only tools unbounded within the budget. | None at per-call level. Post-hoc review. | Default for `log_search`, `metric_query`, `runbook_lookup`, `docs_search`. |
| **L1 — Interactive (per-action)** | "Ask before each action" | Propose a write via `dry_run=true`; must receive an approval token before issuing non-dry call. | Approve/edit/reject each proposed write. | Default for all write tools against production. |
| **L2 — Batch (pre-approved set)** | "Approve a bundle" | Execute each item in a pre-approved batch within a declared scope. | Approve the batch envelope once; audit the receipts after. | Example: approve "restart deployments in `ns=billing` where `cpu>90%`," capped at N targets. |
| **L3 — Advisory (draft-and-send)** | "Draft; a human signs" | Produce artifacts (tickets, emails, PRs, evidence packets); never actuates. | Review the artifact, edit, publish. | Default for SOC 2 evidence packets, memo drafts, triage summaries. |

Every tool in the inventory (exercise-04) is tagged with exactly one
default tier; a per-session override can **tighten** the tier (L1 → L2
is tightening only if it reduces the number of approvals at a given
blast radius; otherwise it loosens) but must never silently loosen.

### 2.2 How to pick a tier per tool

A tool lands at the **lowest** tier (most autonomous) that satisfies all
of these:

- **Reversibility**: can the effect be undone within 15 minutes by the
  customer's on-call without paging more than one person?
- **Blast radius**: how many customers, tenants, or production units
  are touched if the call is wrong?
- **Auditability**: is the resulting record uniquely attributable to a
  human principal, or only to "the agent"?
- **Regulatory sensitivity**: does the call produce an artifact that
  shows up in an auditor's file, a legal filing, or a customer-facing
  surface?

| Tool | Reversible? | Blast radius | Audit need | Tier |
|---|---|---|---|---|
| `log_search` | n/a — read | 0 | agent-attributed OK | **L0** |
| `metric_query` | n/a — read | 0 | agent-attributed OK | **L0** |
| `runbook_lookup` | n/a — read | 0 | agent-attributed OK | **L0** |
| `jira.create_ticket` | Yes (close/delete) | 1 ticket | medium — ticket shows up in audit | **L3** (draft) or **L1** (if auto-file approved) |
| `kubectl rollout-restart` | Yes within minutes | 1 service | high | **L1** |
| `kubectl scale` | Yes | 1 deployment | high | **L1** |
| `kubectl cordon/drain` | Yes but slow | 1 node, ≤N pods | high | **L1** |
| `terraform apply` | Partial; rollback is a new apply | state-wide | critical | **L1** + two-person review |
| `email.send_customer` | **No** | 1+ customers | critical | **L3** only |
| `payments.issue_refund` | Partial | 1 customer | critical | **L1** + two-person review |

A tool with "no" on reversibility is **never** L2 and **almost always**
L3. Irreversible actuations do not get a batch approval tier.

### 2.3 Session-level checkpoints (orthogonal to per-tool tier)

The per-tool tier controls each action. Three session-level checkpoints
sit on top:

- **Session start.** Session is created by an authenticated principal
  (SSO, mod-103 ex-01). Session carries the principal's scopes; tools
  intersect their own scope with the session's.
- **Scope escalation.** If the agent wants to use a tool outside the
  session scope, the session asks for a one-shot scope grant with the
  **same approval token semantics** as an L1 write.
- **Session end.** Session exceeds the stopping envelope (step/
  token/wall-clock from exercise-02) → hard stop + summary emitted
  for post-hoc review. A stuck session is auto-closed.

### 2.4 Asynchronous checkpoints

Some workflows run outside a human's live attention — e.g., the U-02
SOC 2 evidence collector runs overnight. The model has two
asynchronous shapes:

- **Deferred review queue.** L3 drafts land in a reviewer's queue with
  a SLA. If the queue is unattended beyond SLA, the drafts expire; the
  system does **not** auto-publish.
- **Pre-committed guardrail.** For L2 batches, the batch envelope
  includes a hard cap (N targets, Σ dollars, scope prefix). Execution
  past the cap is a new approval, not a silent retry.

### 2.5 The approval token — what "approved" means

An approval token is **not** a boolean on a UI button. It is a signed
envelope:

```
APPROVAL_V1 = {
  "payload_hash": "<sha256 over the canonical JSON of the tool payload>",
  "tool": "<fully qualified tool name>",
  "principal": "<human user ID, from SSO>",
  "principal_scopes": ["cluster:ops", ...],
  "two_person": false | "<second principal ID>",
  "issued_at": <unix seconds>,
  "expires_at": <unix seconds>,          # max 5 minutes default
  "nonce": "<random 128-bit>",
  "approval_text": "<human-readable summary the approver saw>"
}
signature = ed25519(approver_private_key, canonical_json(APPROVAL_V1))
```

Verification rules (also enforced in exercise-02 `hitl.py`):

1. Signature verifies against a key the agent trusts (from the
   customer's SSO/PKI).
2. `payload_hash` matches the hash of the exact write payload being
   executed. **Reject on any mismatch** — the model can regenerate
   the payload after approval and the token must break.
3. `expires_at` is in the future.
4. `nonce` has not been seen before (replay protection; a small LRU or
   DB table per tenant).
5. `principal_scopes` ⊇ the scopes required by the tool (mod-103
   ex-01).
6. For two-person actions, `two_person` is a second signed envelope
   from a distinct principal; both must verify.
7. The approval is for *this* session (`session_id` is in the
   canonical payload hash preimage).

### 2.6 The approver UX

The human approver sees a **diff view**, not a chat bubble.

```
Proposed action — kubectl scale                          (session s-01HMQ3…)
Cluster         : prod-eu-1
Namespace       : billing
Target          : deployment/checkout-api
Current replicas: 8
Proposed replicas: 4
Reason (agent)  : "CPU at 23%, rightsizing per runbook §3.2"
Runbook         : https://runbook.internal/billing/rightsizing (verified)
Blast radius    : 1 service, 1 region
Reversal plan   : `kubectl scale --replicas=8` (one-click revert)

[ Approve  ]  [ Edit…  ]  [ Reject ]              approval expires in 4:52
```

Non-negotiables for the approver UX:

- **Diff is from current state, not from the agent's assertion.** The
  `current_replicas` field is fetched live, not echoed from the
  prompt. Otherwise a prompt-injected observation can lie to the
  approver.
- **"Edit…" regenerates the approval token with the new payload hash.**
  No silent payload mutation between approval and execution.
- **Reversal plan is pre-computed and shown.** If no reversal plan is
  possible, the UI says so explicitly ("this action is irreversible")
  and the tier should probably be L3, not L1.
- **Timer visible.** If approval expires, the token stops verifying;
  the agent must re-request.

### 2.7 Tier assignment per the three mod-106 use cases

Reuse the use cases from exercise-01.

#### U-01 — Contract-intake triage

| Call | Tier | Why |
|---|---|---|
| Classifier + extractor LLM calls | n/a (internal LLM calls, not tools) | Guarded by output schema + rubric gate |
| `queue.enqueue` | **L2** (batch, auto-flow) | Reversible by one-click move; blast radius is a single queue; low-confidence rows diverted to **L3** (human-reviewed inbox) by the rubric step |
| `docs.get` | **L0** | Read-only |

The rubric step is a *gate*, not a human — it is code. The human gate
sits on the low-confidence slice (typically ~5–15% of volume).

#### U-02 — SOC 2 evidence collection

| Call | Tier | Why |
|---|---|---|
| Source reads (Confluence, Jira, SharePoint) | **L0** | Read-only, scoped to the audit evidence project |
| `audit.assemble_packet` (draft) | internal, not actuating | Packet is produced in a staging space |
| `audit.file_evidence_packet` | **L3** | Auditor-visible artifact; wrong evidence is a finding; must be signed by the control owner |

L3 drafts expire after 14 days if not published. No auto-file.

#### U-03 — On-call triage copilot

| Call | Tier | Why |
|---|---|---|
| `log_search`, `metric_query`, `runbook_lookup` | **L0** | Read-only |
| `kubectl rollout-restart` | **L1** | Reversible; 1 service blast; approval per call |
| `kubectl scale` | **L1** | Same; approval payload includes delta |
| `kubectl cordon`/`drain` | **L1 + two-person** when target is a prod-primary node | Slow to reverse; coordination sensitive |
| `kubectl delete` | **not exposed** | Irreversible in this surface; downgrade to a documented manual step |
| `chatops.page_secondary` | **L1** | Side effect on another human; approval records who paged whom |

Notice the **not exposed** entry. The checkpoint model also decides what
*doesn't* get a tool at all — exercise-04 enforces this in the
permission matrix.

### 2.8 What happens when the human disappears

Three failure modes, three behaviors:

- **Approver away from keyboard.** Token expires. Non-dry call refuses.
  Agent either retries after a human returns or stops the session.
  **Fail closed.**
- **Approver clicks approve but the payload mutated.** Signature
  verification fails on `payload_hash`. Call refuses. **Fail closed.**
- **Approver's SSO session is stale.** Approval token signature fails
  because key rotation invalidated it. Call refuses. Human
  re-authenticates and re-approves. **Fail closed.**

There is no fail-open path.

### 2.9 Transparency — what the end user sees

For live-driver surfaces (U-03), the UI shows a running transcript with
every tool call, every observation summary, every pending approval,
every decision. This is the audit trail the customer presents to their
compliance team; it is also the artifact the exercise-05 threat model
assumes exists.

For deferred surfaces (U-01, U-02), the reviewer sees the artifact
(draft or batch envelope), the agent's chain of tool calls that
produced it, and the attribution: which model version, which system
prompt version, which tool registry version. See exercise-02 `v1.3.0`
tagging.

### 2.10 Change-management interactions

The checkpoint model is part of the customer's change-management
contract. Document three explicit links:

- **Tier changes are ticketed.** Moving a tool from L3 → L1 or
  broadening L2 scope is a change request, not a config PR.
- **Scope grants have an expiration policy.** Session-level scope
  escalations do not persist across sessions; one-shot grants
  expire with the approval token.
- **Approver pool is maintained.** The set of principals authorized
  to sign approvals is reviewed per release; departed staff lose
  signing authority within the customer's deprovisioning SLA
  (mod-103 ex-01 SCIM).

## 3. Validation steps

Before the checkpoint model is customer-ready:

1. **Every tool in exercise-04 has exactly one default tier.** No
   tool is "L1 or L3, depends." Defaults are a single row.
2. **Every L1/L2 write has a signed-approval test** that proves
   missing/stale/wrong-payload approvals fail closed (exercise-02
   `test_hitl.py`).
3. **Every L3 artifact has an expiration.** Drafts that are never
   published roll off; no shadow queue of forever-pending drafts.
4. **Approver UX has been walked by a real approver.** A customer
   SRE / control owner has used the diff UI in staging; their
   feedback has been incorporated. No design-by-mockup.
5. **Two-person flow has been exercised.** For at least one tool
   (e.g., `terraform apply`, `payments.issue_refund`), two
   principals have signed in sequence, the agent has executed, and
   the audit shows both signatures.
6. **"Current state" in the approver UI is live-fetched.** Deliberately
   tamper the agent's claimed state in a test; confirm the UI shows
   the real state and the human sees the discrepancy.
7. **Timeout drill.** Trigger an L1 call, wait past the TTL, assert
   the call refuses. No manual intervention.
8. **Session-end cleanup.** Pending approvals at session end are
   voided. A re-opened session does not re-use old approvals.
9. **Audit end-to-end for one call.** From the approver's click to the
   tool execution, every step is traceable: UI event, signed
   envelope, agent verify, tool invocation, provider response,
   result emit. One missing link invalidates the audit.
10. **Fire drill.** With the kill switch (exercise-02) engaged mid-
    approval, confirm the agent cannot execute even if an approval
    token is valid. Ops override > signed approval.

## 4. Rubric / review checklist

Score 0–2 per dimension. 16/24 passes before security review.

| Dimension | 0 — Missing | 1 — Partial | 2 — Strong |
|---|---|---|---|
| Tier definitions | Vague "ask before action" | L0 and L1 only | Full L0/L1/L2/L3 with criteria |
| Tier-per-tool matrix | Missing | Written but inconsistent | Every tool has one default tier with reasoning |
| Approval token spec | Boolean | Signed, no replay protection | Signed + nonce + TTL + payload_hash + scopes |
| Payload binding | Approval is session-scoped only | Token binds to tool name | Token binds to canonical payload hash |
| Timeout behavior | Fails open | Fails closed but no test | Fails closed + drilled in staging |
| Live current-state in UI | Echoes agent assertion | Fetched but not shown clearly | Live-fetched, diff shown, drift surfaced |
| Reversal plan in UI | None | Named | Pre-computed per tool; irreversibles called out |
| Two-person review | Absent | Named | Enforced on at least one tool; both sigs audited |
| L2 batch scoping | Open-ended | Cap on N targets | Cap on N + scope prefix + Σ dollar/damage |
| L3 draft expiration | Never expires | Expires on manual review | Expires on wall-clock SLA, non-auto-publish |
| Change-management links | None | Named | Tier change = ticket; approver pool reviewed |
| Session-level checkpoints | Only per-tool | Session start only | Start + scope escalation + end + kill switch |

## 5. Common mistakes

- **Treating "click Approve" as the control.** A UI click without a
  signed envelope is a UX placeholder, not an approval. It cannot be
  audited, replayed for proof, or verified after key rotation.
- **Approving the prompt, not the payload.** Tokens bound to a prompt
  or a session can be reused after the agent regenerates a different
  payload. Bind to `canonical_json(payload)` + session ID.
- **Fail-open on timeout.** Writing "if the approver doesn't respond in
  60 seconds, proceed" is the single fastest way to lose an audit.
  Fail closed, always.
- **One mega-tier for every write.** L1 everywhere is burnout; L2
  everywhere is a blank check. The tiers exist because the tools are
  not uniform.
- **Hiding tools behind the agent instead of removing them.** "The
  system prompt says don't use `kubectl delete`" is not a control.
  The tool must not be registered.
- **No pre-computed reversal plan.** A reversal plan computed in the
  moment is a reversal plan the operator discovers doesn't work.
- **Allowing silent scope escalation.** If a session can grant itself
  a new scope mid-run, the per-call approval is undermined.
- **L2 batches without caps.** "Approve all restarts in `ns=billing`"
  with no N-cap or Σ-cost cap is a batch that eats a cluster.
- **Forgetting the kill switch overrides approval.** A valid approval
  must *not* execute if ops has engaged the kill switch. The hierarchy
  is: kill switch > session scope > approval token.
- **Approver pool drift.** Departed staff retaining signing keys is a
  classic finding. Tie the approver pool to SCIM.
- **Chat-style approvals.** An approval expressed as "yes" in a chat is
  not a signed artifact, cannot be verified later, and is socially
  engineerable.
- **Treating L3 as cosmetic.** L3 is where most enterprise agentic
  work lives today. Treating "draft-and-send" as a demo mode means the
  pilot has no path to day-two.

## 6. References

- **NIST AI Risk Management Framework** — "Manage" function requires
  demonstrable ability to oversee, override, and recover. The tier
  model is the oversight surface; the kill switch is the override;
  the reversal plan is the recovery.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — HITL mitigates several
  categories (model manipulation, output handling); map each tier's
  mitigation claim to a specific item.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — relevant tactics include indirect prompt injection
  (which the "live current state" rule defends against) and payload
  mutation between approval and execution.
  <https://atlas.mitre.org/>
- **Anthropic — "Building Effective Agents"** — informs which patterns
  require which tiers (autonomous agents concentrate L1 writes;
  prompt chains lean L3).
  <https://www.anthropic.com/engineering/building-effective-agents>
- Companion exercises: exercise-01 (patterns that drive tier mix),
  exercise-02 (approval token implementation), exercise-04 (tool
  inventory), exercise-05 (threat model the tiers mitigate).
