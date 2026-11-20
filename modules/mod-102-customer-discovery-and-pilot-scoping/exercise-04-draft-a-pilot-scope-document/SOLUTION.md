# exercise-04 — Draft a Pilot Scope Document · Reference Solution

## 1. Solution overview

A pilot scope document is the **written contract between the FDE team and the
customer** for a time-boxed engagement, usually 4–8 weeks. It is not a statement of
work (legal) and not a product spec (engineering); it is the artifact the steering
committee reads at kickoff and then uses to decide, at the end, whether the pilot
succeeded.

A good scope document has three properties:

- **Falsifiable success criteria.** The sentence "we will declare the pilot successful
  if …" can be objectively true or false after the pilot runs. No "improve efficiency"
  without a number and a method.
- **Explicit non-goals.** What we are *not* building. Non-goals are how you survive
  weeks 3–5 when a stakeholder tries to expand the pilot into a product.
- **Named owners.** Every commitment has one name on each side. "The data team" is not
  an owner.

The worked example below uses the same notional commercial-underwriting engagement from
exercises 02 and 03. Replace with your customer's engagement.

## 2. Worked answer — pilot scope document

```markdown
# Pilot scope: Underwriter AI assist · ACME Bank + <vendor> · 2026-Q4

Owners
- ACME: J. Patel (Head of Commercial UW) — business owner
- ACME: S. Rao (Platform Security) — security/risk
- ACME: M. Chen (IT integrations) — systems access
- Vendor: FDE lead <name>, backend eng <name>, eval lead <name>
Steering: biweekly, Thursdays 09:00 PT · 30 min · ACME + vendor owners above
Pilot dates: 2026-11-25 (kickoff) → 2027-01-20 (readout). 8 calendar weeks.
Pilot clone of this doc is versioned; changes require steering acknowledgement.
```

### 2.1 Problem statement (one paragraph)

> Commercial underwriters at ACME spend ~192 hours/week of team time producing credit
> memos for ~120 new applications/week. Steps 2 (case creation) and 4 (risk
> spreadsheet) together are ~45% of that time, with p90 durations 2–3× the median due
> to variant-driven exceptions. ACME wants to pilot AI assistance that reduces
> underwriter time-on-task on those two steps without reducing credit-decision
> quality or expanding regulatory risk.

Evidence: 6 SME interviews, 4 shadow sessions, 20 recent files sampled
(see `discovery/` folder, version 2026-11-18). Baseline numbers tied to the Ops
Dashboard metric "UW cycle time" (owner: J. Patel's team).

### 2.2 In-scope deliverables

| ID   | Deliverable                                                                 | Pattern            | Human in loop? | Target volume (pilot) |
|------|-----------------------------------------------------------------------------|--------------------|----------------|-----------------------|
| D-01 | LOS field extraction for the 14-field intake panel                          | Doc extraction     | Yes — per field | 100% of new intake    |
| D-02 | Risk-spreadsheet pre-fill for variant 4b (clean financials)                 | Structured extract | Yes — per sheet | ~20% of cases (variant 4b only)        |
| D-03 | Clarification-email draft for variant 4c (missing/stale financials)         | Assisted draft     | Yes — UW sends | All variant 4c cases  |
| D-04 | Eval harness and labelled set covering D-01..D-03                           | —                  | —              | ≥150 labelled cases   |
| D-05 | Weekly metrics report to steering (acceptance, edit distance, cycle time)   | —                  | —              | 8 reports             |

Each deliverable's technical design is a separate artifact (see mod-103 and mod-104
in the wider curriculum); this scope lists outcomes, not architecture.

### 2.3 Non-goals (explicit)

- **Not** fully automated underwriting. All outputs are human-reviewed before any
  downstream action. This is a product rule, not a phase.
- **Not** peer-review automation (step 6). Wait time is organisational — see
  exercise-02 recommendation I-05.
- **Not** memo drafting (step 5). Variant 4b and 4c cover the fat tail; step 5 is
  deferred to a potential follow-on engagement after the memo rubric exists.
- **Not** the auto-decline flow. Any decision that reduces credit availability stays
  with a human.
- **Not** fine-tuning on ACME customer data. We will use retrieval + prompting on a
  base model (data-handling rationale below).

### 2.4 Success criteria (how we declare the pilot pass/fail)

The steering committee will declare the pilot **succeeded** if, measured over the
last two weeks of the pilot on cases that entered the pilot flow:

- **S-1 Field extraction acceptance:** UWs accept ≥90% of the 14 extracted fields
  per case without edit, over the labelled 100-case acceptance set. Source: eval
  harness report.
- **S-2 Spreadsheet pre-fill accuracy (variant 4b):** per-cell match rate ≥95% on
  the labelled 40-case set; UW override rate ≤10% per sheet on live cases.
- **S-3 Email draft usefulness:** UWs send the draft email with ≤15% edit distance
  on ≥70% of variant-4c cases.
- **S-4 Cycle-time impact:** median UW time-on-task for step 2 + step 4 (clean
  variant) drops ≥25% vs. the baseline measured in the two weeks before pilot
  start. Source: ACME Ops Dashboard metric "UW cycle time," filtered to pilot
  UWs.
- **S-5 Quality guardrail:** no increase in post-committee rework rate attributable
  to pilot cases vs. control cohort. Source: ACME QA sampling, same sample size
  before and during pilot.

The pilot is **not** a success if any of: S-1 or S-2 miss by >10 percentage points;
S-5 shows a measurable rework increase; or any security/governance item in §2.6 is
unresolved at readout.

### 2.5 Baseline + measurement plan

- Baseline window: 2026-11-04 → 2026-11-24 (three weeks before kickoff).
- Metric owner at ACME: J. Patel's analytics partner <name>.
- Dashboard queries and definitions frozen at kickoff and versioned in
  `metrics/queries.sql`. Any change requires steering acknowledgement.
- Weekly reports published every Friday at 17:00 PT to the steering committee.

### 2.6 Risk, governance, and data

Governance is anchored to the **NIST AI Risk Management Framework** functions:

- **Govern:** Steering committee is the approving body; readout is the go/no-go
  gate. Decisions recorded in the steering log.
- **Map:** Data flows are limited to ACME's SharePoint tenant and the vendor's
  deployed inference environment inside ACME's VPC. No application data leaves
  ACME's tenant. DPIA review completed before kickoff (owner: S. Rao).
- **Measure:** Eval harness D-04 is the primary measurement surface. Acceptance
  review is a documented procedure; eval set is versioned.
- **Manage:** Kill-switch — any UW can disable AI assist for their session; ACME
  Platform Security can disable globally without vendor involvement.

Relevant ML-specific risks reviewed (OWASP ML Top 10):

- Training-data risks — N/A in pilot (no fine-tuning; see non-goals).
- Input-handling risks — intake documents are internal, not attacker-controlled;
  retrieval sources are internal ACME documents only. No external user input
  reaches the model.
- Output-handling risks — all outputs are human-reviewed; no autonomous action.

Adversarial exposure (MITRE ATLAS) reviewed and documented in `security/atlas-
review.md`; no public-internet surface in this pilot.

### 2.7 Timeline and gates

| Week | Dates                 | Milestones                                                                 | Gate                                   |
|------|-----------------------|----------------------------------------------------------------------------|----------------------------------------|
| 0    | 11-25 → 11-28         | Kickoff, accounts, VPC access, baseline freeze                             | Access working on both sides           |
| 1–2  | 11-29 → 12-12         | D-04 eval harness up; D-01 first version against labelled set              | ≥60% field accuracy on dev set         |
| 3–4  | 12-13 → 12-26         | D-01 to ≥90% dev; D-02 against variant-4b set                              | Steering review midpoint               |
| 5–6  | 12-27 → 01-09         | D-03 pilot with 3 UWs; D-01/D-02 live with full UW cohort                  | S-1, S-2 passing on last-week data     |
| 7–8  | 01-10 → 01-20         | Full-cohort live; weekly reports; readout prep                             | Readout on 01-20, go/no-go on scale    |

### 2.8 Out-of-scope escalation path

New ideas that arise mid-pilot land in `scope/parking-lot.md`. The steering committee
reviews the parking lot at midpoint and at readout. No in-flight scope changes without
steering acknowledgement — this is the single most common reason pilots overrun.

### 2.9 Readout

A 30-minute readout on 2027-01-20 covering: S-1 through S-5 with numbers;
governance items; parking lot; and a recommendation (continue / fix / stop /
expand). The recommendation is the vendor's; the decision is ACME's.

## Implementation

How a trainee produces this scope doc in a real pilot:

1. **Collect the inputs.** Pull the discovery artifacts from exercise-01, the
   process map from exercise-02, and the decision table from exercise-03 into a
   single `scope/` folder. Every number in the scope doc should trace to one of
   those three sources — if it doesn't, it's a guess.
2. **Draft in one sitting, with both owners.** The FDE lead and the customer
   business sponsor sit together (same room or shared document) and draft §2.1
   through §2.5 in one 90-minute block. Drafts assembled asynchronously drift on
   non-goals almost every time.
3. **Loop in security before the success criteria are frozen.** §2.6 (risk,
   governance, data) is written with the customer's security owner in the room,
   not reviewed as an appendix. Any unresolved item in §2.6 must either be fixed
   or rolled into the "pilot is not a success if …" clause.
4. **Freeze the baseline before kickoff.** Owner agrees the dashboard query and
   the three-week baseline window; both are committed to `metrics/queries.sql`
   and never edited without steering acknowledgement.
5. **Get written sign-off.** Business sponsor, security owner, and vendor lead
   each write "approved" on the versioned file (or sign a PDF export) before
   kickoff. A Slack thumbs-up is not sign-off.
6. **Re-open at the midpoint, not before.** Scope changes land in `scope/parking-
   lot.md` during weeks 1–4; the midpoint steering is the only forum in which the
   scope document itself gets edited. This cadence is the single highest-leverage
   habit for pilots finishing on time.

## 3. Validation steps

A scope document is review-ready when:

1. **Every "we will succeed if…" sentence is falsifiable.** If a reader cannot
   imagine the dataset and the query that would compute the number, it's not
   falsifiable.
2. **Every number has a source.** Baseline, target, and measurement cadence.
3. **Every deliverable row has an in-scope pattern** (extract, classify, assist,
   summarise, route). If a deliverable says "AI for X" with no pattern, it's not
   scoped.
4. **Non-goals section is at least as long as the deliverables section.** Pilots
   usually fail from over-reach, not under-reach.
5. **Governance maps to NIST AI RMF functions by name.** A scope with no
   cross-reference to Govern/Map/Measure/Manage is governance-by-vibes.
6. **Kill-switch and rollback are explicit.** Who disables it, how fast, without
   vendor involvement.
7. **Owners, not teams, are named.** Replace "the data team" with a person.
8. **The parking-lot process exists and is written down.** Every pilot has
   mid-course ideas; the question is whether they are tracked or absorbed silently.
9. **Readout date is a calendar date**, not "around end of pilot."
10. **The customer business owner has read it and said yes** in writing before
    kickoff.

## 4. Rubric / review checklist

Score 0–2 per row. 14/22 passes.

| Dimension                                          | 0 — Missing                              | 1 — Partial                                           | 2 — Strong                                                                                                  |
|----------------------------------------------------|------------------------------------------|-------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Problem statement tied to evidence                 | Vague                                    | Cites interviews but no numbers                       | One paragraph, cites discovery artifact and operational metric                                              |
| Deliverables — pattern and volume per row          | Generic "AI for X"                       | Pattern listed, no volume                             | Each row has pattern, HITL stance, and target volume                                                        |
| Non-goals explicitness                             | None                                     | A few generic ones                                    | Named non-goals that defend against known expansion pressure                                                |
| Success criteria falsifiability                    | "Improve efficiency"                     | Numbers but no measurement method                     | Each criterion has threshold + source + window + sample definition                                          |
| Baseline + measurement plan                        | Absent                                   | Baseline exists, method unclear                       | Baseline window, query ownership, queries frozen at kickoff and versioned                                   |
| NIST AI RMF alignment                              | Not referenced                           | Mentioned generically                                 | Explicit mapping per function (Govern/Map/Measure/Manage)                                                   |
| OWASP ML Top 10 / ATLAS applicability              | Not reviewed                             | Listed without reasoning                              | Each applicable risk named with pilot-specific treatment                                                    |
| Kill-switch / rollback                             | Not mentioned                            | Vendor-controlled only                                | Customer can disable without vendor; latency specified                                                      |
| Owners by name                                     | Teams only                               | One side named                                        | Named owners on both sides for each commitment                                                              |
| Timeline with gates                                | Dates only                               | Dates + milestones                                    | Dates + milestones + explicit pass/fail gates                                                               |
| Parking-lot / scope-change process                 | None                                     | Informal                                              | Written process, steering checkpoints, change log                                                           |

## 5. Common mistakes

- **Success criteria that aren't measurable.** "Reduce underwriter workload" is not
  a success criterion. "Median time-on-task for steps 2+4 drops ≥25% vs. baseline
  measured in query Q, over the last two pilot weeks" is.
- **No baseline.** Pilots that start measuring only after the tool ships can never
  prove the delta. Freeze the baseline window *before* kickoff.
- **Scoping to the workflow, not the variant.** If you scope "risk spreadsheet
  automation" instead of "variant 4b pre-fill," the customer will test you on
  variant 4c and you will miss. See exercise-02 and exercise-03.
- **Soft non-goals.** "We might also look at…" is not a non-goal. Non-goals must be
  written as declines.
- **Governance as an appendix.** NIST AI RMF alignment belongs in the body, and the
  risk register must be reviewed at the midpoint steering, not just at readout.
- **One owner from the customer, none from the vendor.** Named owners on *both*
  sides; otherwise accountability collapses to the FDE.
- **No readback.** The customer business owner has to say yes to the exact wording,
  not to a summary slide. Calendar it as its own meeting.
- **Confusing success criteria with product KPIs.** Pilot success criteria measure
  pilot success; product KPIs measure a shipped product. Don't borrow one for the
  other.

## 6. References

- NIST AI Risk Management Framework (AI RMF 1.0) — four functions used explicitly
  in §2.6 of the worked example: Govern, Map, Measure, Manage. See also the
  Generative AI Profile where generative tooling is in use.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — used in §2.6 to enumerate ML-specific
  risks in-scope for the pilot and name which ones are N/A for the chosen pattern.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — adversarial-ML tactic matrix. Used in §2.6 when the pilot has
  attacker-reachable surfaces; here it's referenced and found N/A for the internal
  intake flow.
  <https://atlas.mitre.org/>
