# exercise-04 — Author Per-Customer Cost & Latency SLOs · Reference Solution

## 1. Solution overview

The deliverable is a **per-customer SLO document** that specifies the
cost and latency behavior the Lantern feature promises for *this*
customer, with the signals, budgets, review cadence, and consequences
written down. An SLO without a signal is a hope; an SLO without a
consequence is a slogan.

Two things make LLM SLOs different from generic web-service SLOs:

- **Cost is a first-class SLI.** Spend per request / per workflow / per
  tenant moves fast and is tied to model choice, caching, and retrieval.
  Treat it like latency — define a target, a budget, an alert, and a
  review cadence.
- **Latency has structure.** Most LLM traffic is non-interactive
  background inference with a different target than user-facing
  chat. The SLO document distinguishes request classes rather than
  conflating them.

A good SLO document is three pages, specific to one customer, and
operable: someone on-call can tell at 2am whether the system is
in-budget and in-latency, and what to do if it is not. The common FDE
mistake is to write latency SLOs that look like a generic web-app SRE
template and ignore cost, retrieval, and the LLM-specific tail.

## 2. Implementation — SLO for a worked customer

### 2.1 Customer context (one paragraph, always included)

```
Customer: Acme Underwriting
Feature: "Invoice Intake Assist" — extract line items from scanned PDFs;
         feeds the underwriter's review queue. Not customer-facing.
Volume:  ~8,000 extraction requests/day; p95 ≈ 12k on peak Mondays.
Business metric (from mod-102 ex-01): minutes saved per application,
         target 15 min/app. SLO breach must map back to this.
Pilot phase: production, 90 days in; design-partner contract renewing
         in 2026-Q2.
```

Rules:

- The customer section is grounded in artifacts from mod-102 (discovery,
  pilot scope). Numbers that cannot be sourced are flagged with an
  inline HTML comment so a reviewer can see exactly which figures still
  require customer discovery before the SLO goes live.
- The business metric link is explicit. "p95 latency < 4s" without "so
  underwriter review doesn't stall" is a vanity target.

### 2.2 Request classes

The feature's traffic splits into classes the SLOs treat differently:

| Class | Example | p99 user wait tolerance | Streaming? |
|---|---|---|---|
| **Interactive** | Underwriter opens a page, waits for the extracted fields to appear | ≤ 5s | Partial streaming of fields acceptable |
| **Background** | Overnight batch of newly-received PDFs | n/a (wall-clock SLO on job duration) | No |
| **Agentic / multi-step** | Draft-then-check-then-score chain | Measured end-to-end | Yes between steps |

SLOs are defined **per class**, then aggregated. Conflating them produces
either over-provisioned (expensive) or under-provisioned (missed)
promises.

### 2.3 SLIs (what we measure)

| SLI | Definition | Source |
|---|---|---|
| `latency_gateway_ms` | Wall-clock latency from gateway ingress to gateway egress, per request | OTel span `gateway.chat` (exercise-02) |
| `ttft_ms` | Time-to-first-token (streaming) | Gateway span attribute `ttft_ms` |
| `schema_valid_rate` | % of structured outputs that validate against the prompt's output schema | Prompt runtime (exercise-03) |
| `cache_hit_rate` | % of requests served fully/partially from cache | Gateway `usage.cache_hits` |
| `cost_per_request_usd` | Provider cost summed across providers used per request | Gateway cost accounting (exercise-02) |
| `cost_per_business_unit_usd` | Cost divided by number of applications successfully processed | Joined with product metric |
| `error_rate` | 5xx + schema-invalid + policy-refusal / total | Gateway + prompt runtime |
| `fallback_rate` | % of requests that used the secondary provider | Gateway `route.provider = secondary` |

Every SLI is tenant-scoped (`tenant=acme`). Cross-tenant SLIs are used
for platform health, not customer SLOs.

### 2.4 SLOs (what we promise, and where)

Pick numbers from measured baseline, not vibes. For a feature in
production for 90 days, baseline = last 28 days excluding incident
windows.

```yaml
# slos/acme/invoice-intake-assist.yaml
customer: acme
feature:  invoice_intake_assist
window:   28_days_rolling
classes:
  interactive:
    latency_gateway_ms:
      target: "p95 <= 4000 AND p99 <= 7000"
      budget: 2% of requests outside target per window
    ttft_ms:
      target: "p95 <= 1200"
      budget: 2%
    schema_valid_rate:
      target: ">= 99.0%"
      budget: 1%
    error_rate:
      target: "<= 1.0%"
      budget: 1%
  background:
    job_duration_min:
      target: "p95 <= 45 minutes per 10k-PDF batch"
      budget: 1 missed batch per week
    schema_valid_rate:
      target: ">= 99.5%"
cost:
  cost_per_request_usd:
    target: "<= $0.012"       # measured baseline + 10%
    hard_cap: "$0.030"
    window: 7_days_rolling
  cost_per_business_unit_usd:
    target: "<= $0.18 per processed application"
  daily_spend_usd:
    soft_alert: "$180"
    hard_cap:   "$300"
  action_on_hard_cap:
    - throttle_tenant: true           # gateway enforces
    - page: team-finance-fde
quality:
  f1_line_items:
    target: ">= 0.90"
    source: eval set in prompt artifact (exercise-03)
    cadence: weekly rerun
fallback_rate:
  informational_alert_at: "> 5% in any 1h window"
review:
  weekly: FDE + customer product owner, 15 min
  monthly: FDE + customer VP + engineering sponsor, 30 min; review budgets
  quarterly: renegotiate targets based on observed baseline
```

Rules:

- **Target vs. hard cap.** The target is where we steer; the hard cap
  is where we enforce. A cost hard cap is a throttle (via exercise-02's
  quota mechanism), not just an alert.
- **Budget as a number, not "best effort."** "2% of requests may exceed
  the latency target per 28-day window" is operable; "we try" is not.
- **Fallback rate is informational, not enforced.** Falling back is a
  feature of the gateway; a high fallback rate is a signal to
  investigate the primary, not a customer-visible fault.
- **Quality SLOs are tied to the eval set.** A customer SLO that says
  "answers are accurate" without an eval cadence is not reviewable.
- **Review cadence is written in.** SLOs rot without weekly / monthly /
  quarterly review. Specify who meets when.

### 2.5 Error budget policy — the consequence

The budget is what you spend on risk. When it burns, the policy changes
behavior — this is the document's teeth.

| Budget consumption (rolling window) | Response |
|---|---|
| ≤ 50% | Normal cadence. Feature changes ship normally. |
| 50% – 80% | Elevated. New model changes require a shadow run and sign-off. |
| 80% – 100% | Freeze non-essential changes to the feature until the burn clears. |
| > 100% | Breach. Incident ticket. Customer notified within 24h. Postmortem required. If two consecutive windows breach: pause new feature rollouts to this tenant and reopen the SLO at the next quarterly review. |

For **cost** specifically:

- Soft alert = page the team; investigate top prompts by spend.
- Hard cap = gateway throttles the tenant per policy; customer
  notified; emergency review scheduled.

### 2.6 Dashboard and on-call runbook

Every SLO row has:

- A **dashboard panel** (OTel + metrics backend; see mod-110) with the
  SLI, the target line, the budget-burn meter, and the top-5
  contributing prompts/tenants.
- A **runbook link** in the alert text, with a known-good first
  response (e.g., "If cost p95 breaches, run the top-spend query,
  compare to prior 7d, check for cache-hit-rate drop, check for
  retrieval-token growth").
- A **"who to call"** pointer — the FDE owning this customer plus the
  platform on-call.

## 3. Validation steps

1. **Every SLI has a source.** Each row in §2.3 points to a span,
   metric, or table that can be queried today. No aspirational SLIs.
2. **Every SLO target is baselined.** The number was measured over the
   last 28 days and is explicitly `baseline × (1 + headroom)` or
   `baseline × (1 − headroom)`. Show the baseline in the doc.
3. **Hard cap is enforced in code.** A test asserts that pushing traffic
   past `daily_spend_usd.hard_cap` causes the gateway to throttle the
   tenant (exercise-02 quota mechanism).
4. **Latency SLO respects class.** Interactive and background have
   different targets; no single p95 line averaged across them.
5. **Alert text references a runbook.** Trigger a fake alert; the
   paging body includes a link to the step-by-step response.
6. **Error-budget math is reproducible.** A notebook computes current
   burn from the raw metrics; two FDEs run it and get the same number.
7. **Customer has seen and signed off.** The document is co-signed by
   the customer's product owner. Unsigned SLOs are internal targets,
   not customer SLOs.
8. **Review meetings exist on calendars.** Weekly, monthly, and
   quarterly reviews are scheduled with the right attendees before the
   document is "live."
9. **Business-metric link holds.** A breach of the latency SLO can be
   shown (even hypothetically) to move the business metric from
   mod-102, ex-01; if not, the SLO is measuring the wrong thing.

## 4. Rubric / review checklist

Score 0–2 per row. 18/28 passes.

| Dimension | 0 — Missing | 1 — Partial | 2 — Strong |
|---|---|---|---|
| Customer context grounded | Generic template | Named customer, no numbers | Customer + feature + volume + business metric cited |
| Request classes distinguished | One target for all | Two classes, unclear criteria | Interactive / background / agentic with distinct targets |
| SLIs sourced | Named without source | Named with vague source | Each SLI points to a specific span/metric queryable today |
| SLOs baselined | Round numbers | Baseline mentioned | Target = baseline × known headroom; data shown |
| Cost as SLI | Absent | Named globally | Per-request + per-business-unit + daily spend; hard cap enforced |
| Latency includes TTFT | Only end-to-end | TTFT noted | TTFT target per class; streaming semantics defined |
| Quality SLO | None | Mentioned | Tied to eval set with cadence (mod-107) |
| Error-budget policy | None | Alert thresholds only | Burn levels with explicit behavior changes and breach consequences |
| Hard-cap enforcement | Alert only | Manual throttle | Automatic throttle via gateway quota mechanism, tested |
| Runbook | None | Prose only | Alerts link to a step-by-step response with dashboard links |
| Review cadence | None | Weekly only | Weekly + monthly + quarterly with named attendees and agenda |
| Customer sign-off | None | Verbal | Co-signed document stored with the pilot artifacts |
| Fallback-rate signal | Missed | Alert, overweighted | Informational signal, not enforced; investigated weekly |
| Business-metric linkage | Missed | Named but not operationalized | Any SLO breach maps to the mod-102 ex-01 operational metric |

## 5. Common mistakes

- **One p95 latency line.** Averaging interactive and background
  traffic makes the number meaningless for both. Separate classes.
- **Cost as "we'll keep an eye on it."** Spend is the fastest-moving
  risk in an LLM feature. If it isn't an SLO with a budget and an
  enforcement, it is not operable.
- **No baseline.** Pick numbers from the data, not from the slide
  deck. "p95 < 4s" with no evidence of 90 days of traffic is a wish.
- **Target without a budget.** Pure thresholds page constantly and
  teach the on-call to ignore the alert. A budget absorbs noise.
- **Budget without a consequence.** 80%+ burn should change behavior
  (freeze non-essential changes, shadow before promotion). "We'll
  notice" is not a consequence.
- **Cost hard cap that doesn't throttle.** A hard cap that only alerts
  is a fire alarm without a sprinkler — the spend continues while
  someone wakes up and reads the ticket. Wire it to the gateway.
- **TTFT ignored for streaming UX.** Users feel TTFT far more than
  end-to-end latency. For interactive UIs, TTFT is often the real SLO.
- **Quality SLO without an eval.** "Answers are accurate" without a
  labelled set and a rerun cadence is not a commitment. Point at the
  eval artifact from exercise-03 and mod-107.
- **Cross-tenant p95.** Platform health and customer SLO are different
  questions. Customer SLOs are tenant-scoped.
- **No review cadence.** The number stops being right as traffic,
  model, retrieval, or the business shifts. Weekly check, monthly
  review, quarterly renegotiation.
- **Unlinked to business metric.** A pretty SLO document whose breach
  cannot be shown to affect the mod-102 business metric is a
  technical artifact, not a customer commitment.
- **SLO as retention lever.** Renegotiating targets quarterly is
  healthy; shrinking the scope mid-window to avoid a breach is not.
  Breaches are learning events, not bookkeeping errors.

## 6. References

- **NIST AI Risk Management Framework** — Measure and Manage functions
  cover the measurement, review, and response discipline the SLO
  document encodes. <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — threats whose impact
  may show up as SLI deviations (e.g., prompt-injection-induced error
  spikes, model-drift-induced quality drops).
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — tactics whose operational signatures (increased
  fallback rate, schema-invalid spikes, cost anomalies) the dashboard
  should surface. <https://atlas.mitre.org/>
- **Google SRE Book — Service Level Objectives** — canonical pattern
  for SLI/SLO/error-budget vocabulary used above. <https://sre.google/sre-book/service-level-objectives/>
- Local exercise context: mod-102 ex-01 (business metric baseline),
  mod-104 ex-02 (gateway cost/limit mechanism), mod-104 ex-03
  (prompt artifact and eval set), mod-107 (eval cadence), mod-110
  (observability pipeline).
