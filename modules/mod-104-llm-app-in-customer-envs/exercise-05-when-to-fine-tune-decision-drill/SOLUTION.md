# exercise-05 — When to Fine-Tune Decision Drill · Reference Solution

## 1. Solution overview

The drill forces the trainee to stop asking *"can we fine-tune this?"*
(answer: usually "yes, technically") and start asking *"should we spend
pilot weeks on fine-tuning this task at this customer, versus the
non-fine-tune alternatives?"* It is a **gating judgment** that sits
between prompt architecture (exercise-03) and model deployment
(exercise-02).

The deliverable is a **decision table** that classifies each candidate
task/variant from the customer's pilot into one of four buckets:

- **A — Don't fine-tune, prompt+retrieval is enough.** Default for most
  tasks. Keep the capability cheap and swappable.
- **B — Light adapter (LoRA / PEFT) on a small model.** The task is
  high-volume, pattern-stable, latency- or cost-sensitive, and
  non-fine-tuned performance is close-but-not-close-enough. Adapter is
  owned by the customer, trained on customer-permitted data.
- **C — Fine-tune later (parked).** Real potential win, but a
  precondition is missing: no labelled data, no eval, no capacity to
  retrain, unresolved data-rights question.
- **D — Do not fine-tune.** Judgment-heavy, low-volume, rapidly
  changing, regulatorily sensitive, or there is no stable "right
  answer" to teach. Stay prompt-driven or pick a non-ML path.

A good trainee makes **more A- and D-decisions than feels ambitious**.
New FDEs systematically pick B because fine-tuning is the most
technically interesting lever. Senior FDEs pick A and D more and ship
pilots that don't melt down on first model rev.

## 2. Worked answer — the decision drill

### 2.1 The eight gating questions

For each candidate task, answer all eight. Any **"No"** on Q1–Q4 is a
hard stop (→ D unless the precondition is cheaply fixable, then C).
Q5–Q8 move an item among A, B, and C.

| # | Question | Why it matters |
|---|---|---|
| 1 | **Is the task a known fine-tuning pattern?** (extraction, classification, routing, style transfer, structured generation) | Fine-tuning is strong for narrow pattern-fitting and weak for new-skill acquisition. If your task is "reason better," you want a stronger base model, not a fine-tune. |
| 2 | **Is there an eval set that can decide pass/fail?** (≥100 labelled examples; held-out set; metric the customer agrees with) | Without a decisive eval, fine-tuning is a time sink: you cannot tell whether a change helped. Mod-107 is the deep dive; the gate is "is the eval here." |
| 3 | **Do you have rights to train on the data?** (customer contract, consent, residency, anonymization status) | Many customers' contracts bar training on their data, even for an adapter they own. This is a lawyer question; answer it before the data question. |
| 4 | **Is the task volume high and pattern-stable enough that marginal cost/latency savings compound?** | Fine-tuning pays off on hot paths. One-off tasks never recoup the training + maintenance cost. |
| 5 | **Has prompt + retrieval been tried seriously** (good system prompt, structured output, 5–20 shots, retrieval over the right corpus)? | Most "we need to fine-tune" conclusions come from a weak prompt or missing retrieval. Rule that out first. |
| 6 | **Is there a measurable prompt+retrieval gap on the eval** that is (a) consistent and (b) not closed by a stronger base model? | If a larger closed model closes the gap, prefer that over owning an adapter. If no base model closes it, fine-tuning may help. |
| 7 | **Can the customer own the retraining loop** (labelling cadence, data pipeline, retraining compute, promotion gate)? | An adapter you can't retrain decays. If the customer can't own the loop, stay prompt-driven or park (C). |
| 8 | **Is the regulatory posture compatible with a trained artifact** (OWASP ML Top 10 risks: training-data leakage, adversarial examples, model theft; NIST AI RMF Measure/Manage)? | Some domains require model provenance, red-teaming, and explainability that a fast-moving pilot can't yet produce. |

### 2.2 Worked classification (same example as exercise-01's underwriting archetype)

Assume the Lantern "Invoice Intake Assist" feature at Acme
Underwriting, with candidate fine-tune tasks T-01..T-05.

| ID | Task | Q1 Pattern? | Q2 Eval? | Q3 Rights? | Q4 Volume? | Q5 Prompt+RAG tried? | Q6 Gap persists? | Q7 Can own loop? | Q8 Reg OK? | **Bucket** | Reasoning |
|---|---|---|---|---|---|---|---|---|---|---|---|
| T-01 extract 14 line-item fields from invoice PDF | Yes — structured extraction | Yes — 400-case labelled set | Yes per contract | Yes — 8k/day | Yes, F1=0.91, p95=4.2s | Gap exists on vendor-specific formats; stronger base model does not close it | Yes — customer has labelling capacity | Yes — internal workflow, not customer-facing | **B — Light LoRA on a small model**, vendor-specific | Narrow pattern, strong evidence, loop owned. Adapter per top-10 vendor formats. |
| T-02 classify incoming email into underwriter queues | Yes — classification | Partial — labels exist but held-out set has drift issues | Yes | Yes | Prompt only, no RAG; F1=0.84 | Unknown — RAG over queue-routing rules has not been tried | Capacity exists | Yes | **C — Fine-tune later** | Prompt+RAG hasn't been tried properly. Park, re-score after mod-105 retrieval work. |
| T-03 summarise policy excerpts for underwriter | Partial — summarisation is pattern-adjacent | Hard — "good summary" is judgment | Yes | Medium | Yes; underwriters accept 82% of summaries | Stronger base model closes most of the remaining gap | Capacity exists | Yes | **A — Don't fine-tune** | Prompt change + stronger base model cheaper than owning an adapter for a judgment task. |
| T-04 draft underwriter's narrative memo | No — generative judgment work | Hard — pairwise rubric only | Yes | Medium | Yes | Base-model improvements track memo quality more reliably than fine-tunes | — | — | **A — Don't fine-tune** | Fine-tunes degrade on long-tail prose; hold prompt-and-retrieval + model revs. |
| T-05 rank peer-review panel suggestions | No — contested ranking, small volume | No — no rubric agreed | Partial — personnel data | No | No | — | — | No — HR-adjacent, no consensus definition | **D — Do not fine-tune** | Regulated, contested metric, low volume. Not a fine-tune problem. |

### 2.3 A second worked example — hostile territory

Not every "No" is obvious. Three stock cases the drill forces the
trainee to decline:

- *"Fine-tune a model to always refuse unsafe questions."* Q1 fails —
  refusal behavior is a policy surface, not a fine-tune. The right
  answer is an L1 policy envelope (exercise-03) + guardrails. A fine-
  tune here creates a brittle filter that breaks under distribution
  shift (OWASP ML Top 10 model manipulation risk). Bucket: **D**.
- *"Fine-tune on five anonymised customer emails."* Q2 fails — five
  examples is a few-shot, not a training set. Running SGD on it will
  memorise rather than generalise (OWASP ML Top 10 training-data
  leakage risk / NIST AI RMF Measure). Bucket: **A — use as few-shots
  in the prompt**.
- *"Fine-tune so the model always speaks in the customer's brand
  voice."* Possible with style-transfer data, but Q5/Q6 usually fail:
  a well-written system prompt + 10 style shots gets most of the win.
  Bucket: **A** unless the brand-voice gap is quantitatively
  measurable *and* persistent after a stronger base model is tried.

### 2.4 Implementation — how a trainee runs the drill

1. **List candidates.** Pull the candidate tasks from the pilot scope
   (mod-102 ex-04) and the prompt artifacts (exercise-03). Every task
   in scope must appear as a row; silent omissions are the common
   failure mode.
2. **Score Q1–Q8 with evidence.** Each cell links to a metric, an eval
   row, a contract clause, or an interview note. "Yes" without
   evidence is treated as "No."
3. **Pick the base-model lever first.** Before any B-decision, verify
   that a stronger or newer base model does not close the gap (via
   exercise-02's logical model map; swap `chat-standard` for
   `chat-reasoning` behind the gateway, re-run the eval).
4. **Pick the retrieval lever second.** If retrieval over the right
   corpus has not been tried, do not fine-tune; park in C.
5. **Pick the fine-tune lever last.** B-decisions pair with: who owns
   the labelling cadence, where the adapter is stored, how it
   promotes, how it rolls back.
6. **Involve legal before the data moves.** Q3 is a lawyer question;
   get it answered in writing. "We think it's fine" is not an answer.
7. **Hand off to exercise-04.** Any B-decision updates the SLO doc —
   fine-tuned paths are measured separately from prompt paths, with a
   quality SLO tied to a re-run cadence on the eval set.

## 3. Validation steps

1. **Every candidate task from pilot scope appears as a row.**
2. **Every "No" on Q1–Q4 produces a C or D, not a B.** An A-decision
   requires at least Q1 or Q5 to be strongly positive; a B-decision
   requires Q1–Q4 all positive plus Q6 persistent.
3. **At least one D row exists** or the trainee argues (in writing)
   why none applies. All-B/A tables are the symptom of over-eager
   trainees.
4. **Each B-row has a rollback story.** The fine-tune model has a
   previous version pinned; the gateway can revert without a code
   deploy (exercise-02's logical-model mapping).
5. **Each B-row has a retraining cadence** and an owner. "We'll retrain
   when it's bad" is not a cadence.
6. **Each C-row names the precondition and an owner.** ("Need
   queue-routing RAG — owner: data-eng A. Patel — by 2026-02-14.")
7. **Each D-row has an alternate recommendation.** "Fine-tuning breaks
   here *because of X*; the non-fine-tune path is Y."
8. **Q3 (data rights) is signed off in writing.** For every B and C
   row, the customer's contract owner has confirmed the training-data
   posture, preserved in the artifact.
9. **Adapter artifacts are provenance-tracked.** Base-model hash,
   training-data manifest, eval-run ID, promotion signature — mapped
   to NIST AI RMF Measure/Manage obligations.
10. **The decision is dated.** Fine-tune decisions age fast as base
    models improve; the table records a re-scoring date (e.g.,
    quarterly or at every base-model rev).

## 4. Rubric / review checklist

Score 0–2 per row. 12/18 passes.

| Dimension | 0 — Missing | 1 — Partial | 2 — Strong |
|---|---|---|---|
| Candidates scoped from pilot artifacts | Invented | Partial list | Full list traced to mod-102 scope + mod-104 ex-03 prompts |
| Q1–Q8 answered with evidence | Many skipped | Answered without evidence | Each cell cites eval/metric/contract/interview |
| Bucket consistent with answers | B with Q3=No | Mostly consistent | Every bucket follows deterministically from Q1–Q8 |
| Base-model lever tried before B | Skipped | Mentioned | Eval-based comparison across logical models in the gateway |
| Retrieval lever tried before B | Skipped | Mentioned | RAG tried on tasks where it fits, with eval delta shown |
| Rollback & retraining for B | Absent | One mentioned | Rollback via logical-model flip + retraining cadence with owner |
| C-row preconditions | Not named | Named | Named, owned, dated |
| D-row alternatives | Not named | One-line | Concrete non-fine-tune path with owner |
| Data-rights signoff | Verbal | Written but vague | Written, contract-section cited, preserved in artifact |
| Provenance (base hash, data manifest, eval run, promotion signature) | Absent | Partial | All four captured per NIST AI RMF Measure/Manage |
| Quality SLO linkage (ex-04) | Missed | Mentioned | Each B path has its own quality SLO + rerun cadence |
| Regulatory posture (OWASP ML Top 10 / ATLAS) | Not considered | Mentioned generically | Named risks tied to specific rows with mitigations |

## 5. Common mistakes

- **Fine-tuning instead of prompt engineering.** The first fix for a
  weak output is almost always a better prompt, better retrieval, or
  a stronger base model. Only commit to an adapter after those have
  been tried with evidence.
- **Fine-tuning for refusal/safety behavior.** Policy is a surface, not
  a training objective. Fine-tunes meant to enforce refusals are
  brittle under distribution shift (OWASP ML Top 10 model manipulation
  risk) and complicate audit.
- **Small-N fine-tunes.** "We have 20 examples" is a few-shot bank, not
  a training set. Running fine-tuning on it memorises rather than
  generalises.
- **Fine-tuning without an eval.** With no eval, you cannot tell
  whether the fine-tune helped or hurt. Fine-tune is the single
  biggest multiplier on eval-set quality; without one, you are
  gambling.
- **No retraining loop.** An adapter you can't retrain decays as the
  world moves. Specify labelling cadence, data pipeline, retraining
  compute, and promotion gate before approving a B.
- **Ignoring data rights.** Even an adapter the customer owns may be
  blocked by their contract, by GDPR consent, or by vertical
  regulation (HIPAA, FFIEC). Q3 is a lawyer question; answer it first.
- **Owning an adapter for a judgment task.** Judgment-heavy tasks
  (narrative drafting, nuanced classification) usually track base-
  model improvements more reliably than a frozen adapter. Fine-tuning
  on today's judgment data can lock in yesterday's taste.
- **No provenance.** A trained artifact without base-model hash, data
  manifest, eval-run ID, and a promotion signature is unauditable.
  NIST AI RMF Measure/Manage obligations apply.
- **Confusing "high-value" with "fine-tune-able."** The task that
  would most benefit a customer is sometimes a D because the volume
  is low, the data is contested, or the answer is judgment-heavy.
  Say so; do not stretch fine-tuning to fit the value.
- **No expiry on the decision.** Base models improve; a B-decision from
  Q1 can become an A-decision by Q3. Write a re-scoring date into the
  table.
- **One adapter for everything.** A single adapter trying to cover many
  vendor formats or many task variants underperforms narrow adapters
  per variant. If you B, scope the adapter narrowly.

## 6. References

- **NIST AI Risk Management Framework** — Measure (eval, drift,
  provenance) and Manage (retraining, rollback, governance) functions
  apply directly to the fine-tune lifecycle.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — training-data leakage,
  model manipulation, adversarial examples, and supply-chain risks
  that attach to any trained artifact.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — tactics (data poisoning, model evasion, model
  extraction) that shape red-teaming and promotion gates for a
  customer-owned adapter. <https://atlas.mitre.org/>
- Local exercise context: mod-102 ex-04 (pilot scope that supplies
  candidates), mod-104 ex-02 (gateway logical-model mapping for
  swap/rollback), mod-104 ex-03 (prompt + eval artifact), mod-104
  ex-04 (SLOs for fine-tuned paths), mod-107 (eval-engineering depth).
