# exercise-03 — "AI Wins" vs. "AI Breaks" Decision Drill · Reference Solution

## 1. Solution overview

The drill forces the trainee to stop asking *"could AI do this?"* (answer: usually
"some of it, somehow") and start asking *"should we put AI on this step of this
workflow for this customer in this pilot?"* It is a **gating judgment** that sits
between the process map (exercise-02) and the pilot scope (exercise-04).

The deliverable is a **decision table** that classifies each candidate step/variant
from the exercise-02 map into one of four buckets:

- **A — AI Wins:** build it; ship it in this pilot.
- **B — AI Assists:** AI drafts, human approves. Build it, but only with human-in-the-loop.
- **C — AI Later:** real AI value, but a precondition is missing (data access,
  integration, ground-truth labels, governance sign-off). Park it with the precondition
  and an owner.
- **D — AI Breaks:** do **not** put AI here in this pilot. Could be regulatory,
  adversarial, high-blast-radius, no human reviewer, or no measurable "right answer."
  Flag and send back as a process-or-engineering recommendation instead.

A good trainee makes **more D-decisions than they are comfortable with**. New FDEs
systematically over-classify into A. Senior FDEs classify into B and D more and ship
pilots that survive acceptance.

## 2. Worked answer — the decision drill

### 2.1 The seven gating questions

For each candidate step/variant, answer all seven. Any **"No"** on Q1–Q4 is a hard
stop (D-bucket unless the precondition is cheaply fixable, in which case C). Q5–Q7
move an item between A and B.

| # | Question                                                                                                 | Why it matters                                                                                                               |
|---|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| 1 | **Is there a reviewable, verifiable notion of "correct"** for this step's output?                        | If humans can't agree on right/wrong, model evaluation is impossible. NIST AI RMF Measure function requires this.            |
| 2 | **Do we have, or can we build in-pilot, a labelled or paired evaluation set** of ≥50 realistic cases?    | Without an eval set the pilot has no honest success criterion.                                                               |
| 3 | **Is the blast radius of a bad output bounded?** Either a human reviews before action, or the action is low-cost to reverse. | Autonomous AI on high-blast-radius actions is the #1 pilot-killer at acceptance.                                             |
| 4 | **Is the data allowed to flow** to the model (residency, PHI/PII, export control, contract)?            | Discovery that skips this produces scope that security kills in week 4. NIST AI RMF Map function covers this explicitly.     |
| 5 | **Is adversarial exposure acceptable?** Can the input be attacker-controlled? If yes, what does ATLAS say about this tactic? | MITRE ATLAS lists tactics (e.g., prompt injection, evasion, data poisoning) that change the architecture, not just the eval. |
| 6 | **Does an ML-amenable pattern fit?** Extraction, classification, summarisation, routing, assisted draft — not "AI thinks about it." | If no known pattern applies, you are proposing research, not a pilot.                                                        |
| 7 | **Is the time saved × frequency material** to the business metric from exercise-01 Section C?            | A step that's cheap to automate but doesn't move the number is a demo, not a pilot.                                          |

### 2.2 Worked classification (same example as exercise-02)

Using the candidate interventions I-01 through I-05:

| ID   | Q1 Correct? | Q2 Eval set? | Q3 Blast bounded? | Q4 Data OK? | Q5 Adversarial OK? | Q6 Pattern? | Q7 Material? | **Bucket** | Reasoning                                                                                                                                             |
|------|-------------|--------------|-------------------|-------------|--------------------|-------------|--------------|------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| I-01 extract 14 LOS fields from PDF        | Yes — fields are verifiable against ground truth | Yes — can label 100 recent PDFs in <1 day | Yes — UW reviews each field before save | Yes per data-handling interview | N/A — intake is internal | Yes — document extraction | Yes — step 2 is 22 h/wk | **B — AI Assists** | High-confidence pattern, but every field is reviewed. Ship with per-field confidence surfaced in the UI.                                               |
| I-02 pre-fill risk spreadsheet, variant 4b | Yes — spreadsheet has defined cells | Yes — 40 clean-financial cases labelled | Yes — UW reviews full sheet | Yes             | N/A                | Yes — structured extraction + calc | Material only for 20% of cases, but p50 saving is large | **A — AI Wins** on variant 4b only | Narrow scope, high verifiability. The win is in picking the variant, not the whole step.                                                              |
| I-03 draft clarification email (4c)        | Partial — "good email" is judgment, not fact | Yes — 30 past emails + relationship-manager scoring rubric | Yes — human sends the email | Yes             | N/A                | Yes — assisted drafting | Yes — hits fat tail | **B — AI Assists**     | Humans send, humans edit. Measure acceptance rate + edit distance.                                                                                    |
| I-04 first-draft narrative memo (step 5)   | Partial — judgment-heavy | Harder — need rubric and pairwise comparisons | Yes — human edits before submit | Yes             | N/A                | Yes — assisted drafting | Yes — 48 h/wk | **B — AI Assists**     | Build only after memo rubric is written. If rubric is missing at kickoff, demote to **C — AI Later** until a sr-UW produces one.                      |
| I-05 reduce peer-review wait time          | — | — | — | — | — | No — scheduling problem | Yes, but not via AI | **D — AI Breaks** (in this pilot) | Queue depth and shift schedules. Send to ops as a process recommendation; do not re-badge as an AI deliverable.                                       |

### 2.3 A second worked example — hostile territory

Not every "No" is obvious. Here are three stock cases the drill forces the trainee
to decline:

- *"Auto-decline loan applications under a dollar threshold."* Q3 fails — this is an
  autonomous action with real customer harm if wrong. The right answer is "AI drafts
  the decline reason; a human presses the button." Bucket: **B**, not A. If the
  customer insists on fully autonomous, bucket: **D** for this pilot.
- *"Summarise external customer support tickets to a public status page."* Q5 fails
  — attacker-controlled input + outward-facing output is a prompt-injection surface
  (MITRE ATLAS tactic). Architecture needs guardrails and output review before this
  is eligible. Bucket: **C** until mitigations are specified.
- *"Rank job applicants."* Q1 and Q4 fail — "correct ranking" is contested, and
  depending on jurisdiction the data flow is restricted. OWASP ML Top 10 and local
  employment regulation apply. Bucket: **D** for a first pilot.

Before using the hiring example in a customer-facing deliverable, confirm the
jurisdictional rules that apply (e.g., NYC Local Law 144 for automated employment
decision tools, EU AI Act Annex III employment uses) with the engagement's legal
reviewer — the specific obligations shape whether the bucket stays at D or escalates
further.

## Implementation

How a trainee runs the drill in a real pilot:

1. **Start from the exercise-02 candidate list.** Copy the I-01..I-N rows into a
   decision-table artifact at `scope/decision-drill.md`. Every candidate from the
   map must appear as a row; silent omissions are the most common failure mode.
2. **Run a working session with the SME and the sponsor together.** Walk the seven
   gating questions row by row on screen share. The session surfaces disagreements
   between the SME's reality (data access, review capacity) and the sponsor's
   assumption (ambition, timeline) — both views have to agree on the bucket.
3. **Write the reasoning in the cell, not the chat.** The bucket (A/B/C/D) is a
   summary; the one-to-three-sentence reasoning in the row is what survives into
   the scope doc and the security review. Pair the reasoning with a citation
   (interview quote, map row, policy).
4. **Loop in security/governance before freezing.** Any row that touches regulated
   data (Q4), attacker-controlled input (Q5), or autonomous action (Q3) goes to the
   named security reviewer for sign-off on the bucket. Save their written OK in the
   same artifact.
5. **Hand off to exercise-04.** The frozen table becomes the input to the pilot
   scope document: A-rows become deliverables, B-rows become deliverables with a
   human-in-the-loop clause, C-rows become parking-lot items with owners, D-rows
   become explicit non-goals.

## 3. Validation steps

1. **Every candidate from the exercise-02 map has a row.** No silent omissions.
2. **Every "No" on Q1–Q4 produces either a C or a D bucket.** If a row says "No" on
   Q3 and still lands in A, the decision is wrong.
3. **At least one D row exists.** If every candidate is A or B, the trainee has not
   exercised judgment yet; push back and re-review.
4. **Each A- or B-row has a one-line success metric** tied back to the exercise-01
   operational metric. ("UW accepts ≥90% of extracted fields without edit, measured
   against the labelled 100-case set.")
5. **Each C-row names the precondition and an owner.** ("Need memo rubric — owner:
   sr-UW J. Smith — by 2026-12-01.") C-rows without owners rot.
6. **Each D-row has an alternate recommendation.** AI breaks here *because of X*;
   the non-AI recommendation is Y.
7. **Governance references** (NIST AI RMF, OWASP ML Top 10, MITRE ATLAS) are cited
   on the rows where they drove the decision, not sprinkled generically.

## 4. Rubric / review checklist

Score 0–2 per row. 10/16 passes.

| Dimension                                  | 0 — Missing                      | 1 — Partial                                      | 2 — Strong                                                                                      |
|--------------------------------------------|----------------------------------|--------------------------------------------------|-------------------------------------------------------------------------------------------------|
| All seven gating questions answered        | Many skipped                     | Answered without evidence                        | Each answer cites an interview, artifact, or policy                                             |
| Bucket decision consistent with answers    | A-bucket with "No" on Q1–Q4      | One edge case misclassified                      | Every bucket follows deterministically from the Q1–Q7 pattern                                   |
| At least one D decision defended           | None — everything is A/B         | One D, thin reasoning                            | Multiple D/C decisions, each with an alternate                                                  |
| Blast-radius analysis (Q3)                 | Not asked                        | Treated as "add a human check" with no detail    | Names the reviewer, the review latency, and the reversal cost                                   |
| Adversarial / data-boundary (Q4, Q5)       | Generic                          | Named but not tied to standards                  | Specific ATLAS tactic or OWASP ML-Top-10 risk cited where it applies                            |
| Ties to operational metric (Q7)            | Absent                           | Vague "it'll save time"                          | Explicit number × frequency tied to exercise-01 Section C metric                                |
| Preconditions for C-rows                   | Not stated                       | Named but no owner                               | Named, owned, dated                                                                             |
| Alternatives for D-rows                    | Not stated                       | One-line                                         | Concrete non-AI recommendation with owner                                                       |

## 5. Common mistakes

- **Over-classifying A.** New FDEs put almost everything in A because "LLMs are good."
  Senior FDEs classify into B and D more. The classification is about *this pilot at
  this customer*, not about the state of the art.
- **Treating "add a human" as a universal escape hatch.** Human-in-the-loop only
  bounds blast radius if the human actually reviews each output and has the time to
  do so. If the proposed volume exceeds what reviewers can process, the human-in-the-
  loop is on paper only.
- **Skipping the eval question (Q2).** "We'll figure out evaluation later" produces
  a pilot that cannot be acceptance-tested. Mod-107 is the deep dive; mod-102 just
  needs the eval question answered honestly at scoping time.
- **Ignoring adversarial exposure.** Any input an outsider can shape is potentially
  adversarial. Consult MITRE ATLAS and name the tactic before deciding A vs. C.
- **Confusing "high-value" with "AI-amenable."** The highest-value step is sometimes
  a D because success isn't verifiable or the action is autonomous and irreversible.
  Say so; do not stretch the pattern to fit the value.
- **Classifying the whole workflow instead of individual steps/variants.** A
  workflow can contain both an A and a D. The exercise exists to break the habit of
  thinking at the workflow level.
- **No written reasoning.** An unjustified A-decision will not survive a security
  review or a steering committee. Reasoning is the artifact — the bucket is just the
  summary line.

## 6. References

- NIST AI Risk Management Framework — **Govern / Map / Measure / Manage** functions.
  Use Map for Q4 and parts of Q3; use Measure for Q1 and Q2; use Manage for Q3 and
  the C-row preconditions.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — cite on rows involving training data,
  model inputs/outputs, or adversarial inputs. (Reference the current list ID per
  OWASP's project page; version-specific IDs change across revisions.)
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — matrix of adversarial tactics against ML systems. Use on any Q5
  "Yes, attacker-controlled" row to name the specific tactic (e.g., prompt
  injection, model evasion) that shapes the mitigation.
  <https://atlas.mitre.org/>
