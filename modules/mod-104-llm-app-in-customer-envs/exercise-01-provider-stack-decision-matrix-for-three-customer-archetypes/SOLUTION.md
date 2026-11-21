# exercise-01 — Provider Stack Decision Matrix for Three Customer Archetypes · Reference Solution

## 1. Solution overview

The deliverable is a **decision matrix** that picks a model-provider stack for
three distinct customer archetypes you will meet in the field, and defends
each choice against the standards the customer's security and risk function
will hold you to (NIST AI RMF Map/Govern, OWASP ML Top 10, MITRE ATLAS).

A provider stack is more than "which model" — it is a tuple:

- **Hosting surface** — public SaaS API (vendor cloud), customer-cloud
  managed (e.g., Azure OpenAI Service, AWS Bedrock, Google Vertex AI,
  Databricks Model Serving), or self-hosted open-weights on customer
  compute.
- **Model family** — closed-weights frontier (Anthropic, OpenAI, Google),
  open-weights (Llama-class, Mistral-class, Qwen-class), or a mix.
- **Data posture** — zero-retention, logging opt-out, BYOK/HYOK, tenant
  isolation, residency region pinning.
- **Access shape** — direct provider SDK vs. a gateway (see exercise-02).
- **Fallbacks** — a second provider wired for degradation, a cheaper or
  smaller model for cost-tier requests, and an offline/air-gapped answer
  when the customer's network pulls the plug.

A good matrix does **not** name one "best" stack — it names three defensible
stacks, each one picked for *this customer*, with the trade-offs written down.
The common FDE mistake is to pick the model the FDE likes and rationalize
backward. The exercise forces you to start from the customer constraints.

## 2. Implementation — the decision matrix

### 2.1 Three archetypes

Pick customers that span the realistic constraint space for FDE work.

| Archetype | Example industry | Key constraints |
|---|---|---|
| **A — Regulated enterprise, US-centric** | Large US bank or health-insurance payer | SOC 2 + HIPAA or FFIEC guidance applies; data stays in US; vendor-risk committee must sign off on every new SaaS; inference logs are subject to subpoena; "no-training-on-our-data" is a contract clause, not a checkbox. |
| **B — EU-regulated or data-residency-locked** | EU manufacturer, EU public-sector adjacent, or non-US financial institution | GDPR applies; data must stay in-region; EU AI Act obligations in scope (per customer's internal counsel); some workloads classified as high-risk under Annex III; prefers European or sovereign hosting options. |
| **C — Air-gapped / sovereign / offline-tolerant** | Defense, intelligence, utilities, classified-adjacent research | No egress to vendor cloud; sometimes no internet at all; the model must run on customer-owned GPUs; weights and runtime must be audit-trailed end-to-end; model updates ship on removable media or through a review gate. |

These three are enough to exercise the full space: closed-weights SaaS,
closed-weights-in-customer-cloud, and open-weights-on-customer-metal.

### 2.2 The matrix

Rows are the archetypes; columns are the dimensions a defensible stack has
to answer. Every cell has to cite a real constraint or a standard.

| Dimension | Archetype A (Regulated US) | Archetype B (EU residency) | Archetype C (Air-gapped) |
|---|---|---|---|
| **Primary model family** | Closed-weights frontier (Anthropic Claude or OpenAI GPT) via customer-cloud managed service. | Closed-weights frontier via **in-region** cloud offering (e.g., AWS Bedrock EU region, Azure OpenAI EU region) **or** sovereign-hosted provider the customer's legal has already blessed. | Open-weights (Llama-class, Mistral-class, Qwen-class, or a comparably licensed model) served on customer GPUs via vLLM / TGI / TensorRT-LLM. |
| **Hosting surface** | Vendor-cloud managed, same cloud the customer already uses for prod (Azure/AWS/GCP). Keeps the data plane inside the customer's existing SOC-2 scope and under their existing enterprise agreement. | In-region managed offering; residency region pinned at account level; redundancy to a second in-region cell, never cross-region. | Customer data center or classified enclave. No vendor-cloud traffic. |
| **Data posture** | Zero-retention mode on; no abuse monitoring sampling; BYOK/CMK on the storage and logging tier; prompt and completion logs encrypted at rest and tenant-scoped. | Zero-retention mode on; data processing agreement (DPA) with standard contractual clauses; BYOK/CMK; transfer-impact assessment on file. | No off-site telemetry. All logs on-prem. Air-gapped update path for weights. |
| **Access shape** | Lantern gateway in customer VPC (exercise-02) in front of the managed service. Direct SDK is *not* allowed from app code. | Same gateway pattern; residency enforced at gateway (reject requests whose routing would hop regions). | Gateway is local-only; routes to the on-prem inference endpoint; no fallback to vendor cloud. |
| **Fallback / second vendor** | Second closed-weights provider in the same cloud, wired but dark; quarterly failover drill. Cheaper small model for cost-tier requests. | Second in-region provider or a smaller model hosted in-region; never fall back across borders. | Second open-weights model on a second node class; no cloud fallback. If GPUs fail, degrade to a smaller local model or an explicit "unavailable." |
| **Fine-tune / adapter story** | PEFT (LoRA-class) adapters on a managed small model if needed; see exercise-05 for the gate. | Same, in-region only. | Full open-weights adapter training on customer compute; adapters checked into the customer's artifact repo. |
| **Observability** | Traces and token counts to customer-owned observability (exercise-02, mod-110); never to vendor dashboards. | Same; log sink pinned in-region. | Same; log sink on-prem. |
| **Compliance hooks** | NIST AI RMF **Govern/Map/Measure** used as the documentation spine for vendor-risk sign-off. SOC 2 CC-mapped controls for change management, access, and audit. | Same spine; EU AI Act obligations added when the use case falls under Annex III (confirm with customer counsel). GDPR Art. 28 processor terms in the DPA. | NIST AI RMF spine plus customer-specific regime (e.g., FedRAMP High if applicable; defer to customer compliance owner). |
| **Risk mitigations (OWASP ML Top 10 / MITRE ATLAS)** | Explicit mitigations for prompt-injection (ATLAS Initial Access / Execution via prompt-injection), training-data leakage (OWASP ML02), and model-output manipulation (ATLAS LLM Jailbreak tactic). Gateway enforces output policy. | Same mitigation menu; EU AI Act transparency obligations added for any user-facing generative output. | Same mitigations; plus adversarial robustness checks on the open-weights model before promotion (ATLAS Evade ML Model tactic). |
| **Commercial posture** | Usage via customer's existing enterprise cloud commit; new vendor paper is the hardest block to clear, so prefer to ride existing agreements. | In-region commit through existing cloud EA; sovereign provider added only if existing EA can't cover residency. | License-and-support for open-weights plus GPU capex; no per-token SaaS billing. |
| **Rollback** | Flip gateway routing back to prior provider; zero code change in app. | Same. | Pin to prior weights image; the artifact repo retains N-2 versions; promotion is gated. |

### 2.3 Rationale — why these three stacks

- **A (Regulated US)**: the binding constraint is *vendor-risk paperwork*, not
  model capability. The fastest path to production is a closed-weights
  provider the customer's cloud team already has paper with. You lose
  nothing by routing through the customer cloud, and you gain contractual
  zero-retention, CMK, and a shared responsibility matrix the vendor-risk
  team can review. Open-weights on customer compute would clear faster on
  *data* but slower on *ops* (GPU procurement, serving stack ownership).
- **B (EU residency)**: the binding constraint is *where the bytes live and
  under which clauses*. Use the same closed-weights family as A if, and only
  if, there is an in-region offering with a signed DPA and SCCs. Otherwise
  the customer's legal team blocks even a POC. The EU AI Act layer is
  additive: if the use case is Annex III high-risk, you add transparency,
  human-oversight, and record-keeping obligations to the data posture. Do
  not confuse Annex III risk classification with the hosting decision.
- **C (Air-gapped)**: the binding constraint is *no egress ever*. Closed-
  weights is off the table because the weights live in the vendor's cloud.
  Open-weights served locally is the only option. The trade-off is model
  capability per token and the operational burden of owning a serving
  stack, so the FDE must be honest with the customer about what tasks are
  achievable at their deployment and which require a managed stack.

### 2.4 Negative choices — things explicitly rejected

Writing down what you rejected is as important as the matrix. A defensible
answer lists the stacks that look attractive but are wrong for the archetype:

- **Rejected for A:** public SaaS API hit directly from app servers. Even
  with zero-retention, the data path crosses a vendor boundary the SOC 2
  scope has to account for every audit. The managed-in-customer-cloud
  option is strictly easier to defend.
- **Rejected for B:** the same closed-weights model on a US region "just for
  the POC." Region leakage during POC poisons the audit trail; the customer
  will then need a greenfield POC in-region anyway. Start in-region.
- **Rejected for C:** anything that phones home for telemetry, license
  checks, or auto-updates. Even "offline-capable" SaaS that batches calls
  home weekly fails the air-gap test. Weights and runtime must be shippable
  on removable media with cryptographic provenance.

## 3. Validation steps

Before the matrix is customer-ready:

1. **Three real customers behind the archetypes.** Each archetype must map
   to an actual customer in your pipeline (or a close proxy from mod-102
   discovery). An archetype with no customer attached is a demo, not a
   deliverable.
2. **Every cell has a citation or a constraint.** No hand-wavy cells. "Zero
   retention on" must cite the provider's documented zero-retention mode.
   "EU AI Act applies" must cite which Annex. "FedRAMP High applies" must
   cite who in the customer's compliance org asserted that.
3. **The data-flow diagram per archetype matches the matrix.** Draw where
   the prompt, completion, logs, and evals physically live for each row.
   A matrix that says "in-region" but a diagram that shows cross-region
   log shipping is wrong — fix one or the other before review.
4. **Fallback is wired, not asserted.** The row says "second provider
   warm"; the fallback must have a tested path in the gateway config. If
   it's "we will add later," label it in the matrix.
5. **Rollback is one toggle.** For every archetype, the gateway config
   change (or on-prem weights pin) that reverts to the prior stack must be
   named and dry-run-able.
6. **The matrix survives a security review tabletop.** Walk a customer
   security lead through the matrix, archetype by archetype, and confirm
   they would sign off on the posture. If they ask for a control you don't
   have, add it or pick a different stack.

## 4. Rubric / review checklist

Score 0–2 per row. 14/22 passes before sending to a customer.

| Dimension | 0 — Missing | 1 — Partial | 2 — Strong |
|---|---|---|---|
| Three archetypes, not three vendors | One archetype, repeated | Three but one is a thin variant of another | Three distinct constraint shapes (regulated, residency, air-gapped) |
| Binding constraint named per archetype | Not stated | Vague ("security") | Specific — paperwork, residency, no egress |
| Hosting surface matches constraint | Public API for a regulated row | In-region but not scoped | Managed-in-customer-cloud, in-region, or on-prem, each defended |
| Data posture explicit | "Enterprise plan" | Retention off, no BYOK | Retention off, BYOK/CMK, DPA/SCC filed, residency pinned |
| Model family defensible | Picked for familiarity | Picked for benchmark | Picked for the constraint — closed vs. open-weights argued |
| Fallback stack named and wired | None | Named only | Named, wired, drill scheduled |
| Fine-tune story deferred to ex-05 or justified | "We'll fine-tune" | Mentioned in passing | Explicit gate (see exercise-05) or an explicit "no" |
| Compliance spine | None | NIST mentioned generically | NIST AI RMF Govern/Map/Measure cited per row; vertical regime named per archetype |
| OWASP ML Top 10 / MITRE ATLAS mitigations | None | Mentioned generically | Specific risks named per row with the countermeasure |
| Negative choices | No rejection section | One rejection | Each archetype lists ≥1 rejected stack with reasoning |
| Customer-ready diagram | Absent | Box-and-arrow | Data-flow diagram per archetype that matches the matrix |

## 5. Common mistakes

- **Starting from "which model is best."** The exercise is customer-first.
  Capability differences between frontier models are smaller than the
  deployability differences between hosting surfaces at a given customer.
- **Treating "zero retention" as a universal checkbox.** Zero-retention
  mode varies by provider and by endpoint; logging and abuse monitoring
  sometimes retain data even when training does not. Cite the specific
  documented behavior for each provider on each endpoint you plan to use.
- **Mixing residency and hosting.** "In the vendor's EU region" is not the
  same as "processed by employees in the EU"; "data stays in-region" is
  not the same as "logs stay in-region." Separate the two in the matrix.
- **Assuming the EU AI Act affects every EU customer the same way.**
  Obligations attach to the use case, not the customer. Classify the use
  case (prohibited, high-risk per Annex III, limited-risk, minimal-risk)
  with the customer's counsel before writing obligations into the matrix.
- **Picking open-weights for A or B "to be safe."** Running GPUs the
  customer doesn't own requires a serving-ops competence the customer
  may not have. Open-weights trades data risk for operational risk; make
  the trade explicitly, do not assume it is free.
- **No rollback story.** A matrix without a tested rollback is a plan to
  stay with one vendor. Pilots rarely die on day 1; they die on day 90
  when the vendor is unavailable and there is no second path.
- **Citing standards generically.** "NIST AI RMF applies" without naming
  the function (Govern / Map / Measure / Manage) is a smell. Each
  mitigation should sit under one of the four.
- **No commercial row.** The matrix is a technical document that will
  travel through procurement. Not naming where the money flows (existing
  EA, new paper, GPU capex) hides the slowest step in the deployment plan.
- **Ignoring the gateway.** Direct-from-app calls to a provider SDK make
  rotation, logging, and policy centrally hard; see exercise-02 for the
  gateway pattern that keeps all three archetypes manageable.
- **Confusing POC with production stack.** A POC may run on a public API
  with synthetic data; the matrix is for the production stack. Say which
  stack you are specifying, not both at once.

## 6. References

- **NIST AI Risk Management Framework** — Govern, Map, Measure, and Manage
  functions. Use Govern for vendor-risk sign-off, Map for data-flow and
  residency, Measure for evaluation of the stack choice, and Manage for
  fallback/rollback readiness. <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — use the current project
  list for threats specific to ML/LLM systems (data leakage, model
  manipulation, supply chain). Cite per-row rather than globally.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — matrix of adversarial tactics against ML and LLM
  systems. Use to name specific tactics (e.g., prompt injection, model
  evasion) in the risk-mitigation row per archetype. <https://atlas.mitre.org/>
- Customer-specific regulatory references (GDPR, EU AI Act Annex III, HIPAA,
  FFIEC guidance, FedRAMP) — cite the specific obligation the customer's
  compliance owner has asserted applies; do not paraphrase from memory.
