# exercise-02 — Workflow Process Mining · Reference Solution

## 1. Solution overview

The exercise asks the trainee to take a target customer workflow — surfaced from the
exercise-01 interview — and produce a **process map** that is specific enough to scope
an AI intervention against. "Process mining" here means the field discipline: watch the
work, count the steps, measure the steps, name the systems, and name the inputs and
outputs — not the academic sub-field of event-log mining with specialised tooling.

The output is a single artifact with four parts:

1. A **step-by-step map** of the current ("as-is") workflow.
2. A **volume × duration table** per step, with evidence citations.
3. A **variant tree** of the top deviations (the exceptions that dominate real time).
4. A **candidate intervention list** mapped to specific steps — not to the workflow in
   aggregate.

The map is the shared object that lets the FDE, the SME, and the sponsor argue about
scope in weeks 2–3 with the same picture in front of them. Ambiguous maps produce
ambiguous pilots.

## 2. Worked answer — process map for an example workflow

The worked example below uses a **generic, sanitised workflow**: *"Underwriter
reviews a commercial loan application."* The trainee should replace this with their
own customer's workflow. Numbers are illustrative placeholders tied to a notional
engagement — in a real artifact every number cites an interview quote, ticket ID,
dashboard screenshot, or screen recording timestamp.

### 2.1 Current-state map (BPMN-lite)

```
┌─────────────────┐
│ 0. New app lands│  Input: PDF + XLSX bundle via email
│  in shared inbox│  System: Outlook shared mailbox "uw-intake@"
└────────┬────────┘  Freq: ~120/week
         │
         ▼
┌─────────────────┐
│ 1. Triage       │  Who: ops coordinator
│                 │  Action: renames files, drops into SharePoint folder
│                 │  Systems: Outlook → SharePoint
│                 │  Duration (median / p90): 3 min / 8 min
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. Create case  │  Who: underwriter (UW)
│  in LOS         │  Action: manual re-keying of 14 fields from PDF
│                 │  System: LoanIQ (web UI)
│                 │  Duration: 11 min / 22 min
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 3. Pull credit  │  Who: UW
│  + collateral   │  Systems: Experian API (manual), CRE comps tool
│  data           │  Duration: 9 min / 18 min
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 4. Build risk   │  Who: UW
│  spreadsheet    │  System: Excel template "RM_CRE_v4.xlsx"
│  (cash flow,    │  Duration: 32 min / 95 min  <-- the fat tail
│  DSCR, LTV)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 5. Narrative    │  Who: UW
│  memo           │  System: Word, saved into SharePoint
│  (1-2 pages)    │  Duration: 24 min / 60 min
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 6. Peer review  │  Who: senior UW
│                 │  Duration: 14 min / 40 min (plus wait time 4h / 36h)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 7. Submit to    │  Who: UW
│  committee      │  Duration: 3 min / 5 min
└─────────────────┘
```

### 2.2 Volume × duration table (one row per step)

| Step | Task                   | Freq (per wk) | Human-minutes p50 | Human-minutes p90 | Weekly load p50 (hours) | Evidence                                  |
|------|------------------------|---------------|-------------------|-------------------|-------------------------|-------------------------------------------|
| 1    | Triage                 | 120           | 3                 | 8                 | 6.0                     | Shadowed Ops lead 2026-11-10, screen rec. |
| 2    | Case creation          | 120           | 11                | 22                | 22.0                    | Interview UW-A, shadow 2026-11-11         |
| 3    | Pull credit/collateral | 120           | 9                 | 18                | 18.0                    | Shadow 2026-11-11                         |
| 4    | Risk spreadsheet       | 120           | 32                | 95                | 64.0                    | Interview UW-A, UW-B; 10 sampled files    |
| 5    | Narrative memo         | 120           | 24                | 60                | 48.0                    | Interview UW-A, UW-B                      |
| 6    | Peer review            | 120           | 14                | 40                | 28.0                    | Interview sr-UW                           |
| 7    | Committee submit       | 120           | 3                 | 5                 | 6.0                     | Shadow                                    |
|      | **Totals**             |               |                   |                   | **192.0 h/wk ≈ 4.8 FTE**|                                           |

**Rule-of-thumb reading:** steps 4 and 5 together are 58% of total human time and both
have long p90 tails. These are the first two scope candidates.

### 2.3 Variant tree (top deviations)

Variants are the reason p50 and p90 diverge. Capture only the ones that recur.

```
Step 4 "Risk spreadsheet" — variants observed in a 20-file sample:
├── Variant 4a: borrower has existing relationship → analyst reuses prior memo (12 files)
├── Variant 4b: new borrower, clean financials → ~25 min (4 files)
├── Variant 4c: new borrower, missing/stale financials → back-and-forth with
│              relationship manager, 60–180 min, spans 1–3 days (3 files)
└── Variant 4d: shared collateral across multiple loans → manual cross-check
               against 2 other active files (1 file)
```

**Implication:** an AI assistant that helps with variant 4b (the clean happy path) is
not helping with the real cost driver, which is 4c. Scope decisions must pick the
variant, not the step.

### 2.4 Candidate intervention list

| ID   | Step(s) | Intervention sketch                                                               | Why here                                                                | AI-amenable? (see ex-03)                   |
|------|---------|-----------------------------------------------------------------------------------|-------------------------------------------------------------------------|--------------------------------------------|
| I-01 | 2       | Extract the 14 LOS fields from the intake PDF; UW accepts/edits in LOS.           | Deterministic structure, high volume, cheap to verify field-by-field.   | Yes — classic doc-extraction, verifiable.  |
| I-02 | 4b      | Pre-fill the risk spreadsheet for the clean-financial variant only.               | Narrow variant; success is easy to measure.                             | Yes, with human sign-off.                  |
| I-03 | 4c      | Draft the clarification email to the relationship manager when financials stale. | Hits the fat tail. Email draft is low-stakes if a human sends.          | Yes, assist pattern, **not** autonomous.   |
| I-04 | 5       | First-draft the narrative memo from the completed spreadsheet.                    | High time, structured input, human reviews before committee submit.     | Yes, assist pattern.                       |
| I-05 | 6       | Reduce peer-review wait time via queue dashboard — *not an AI intervention*.      | Wait time is organisational, not cognitive. Flag out of scope.          | No — this is a process change, not AI.     |

The I-05 row is deliberate: part of the exercise is **learning to say "this is not an
AI problem"** and send it back as a process recommendation.

## Implementation

Building the process map in practice:

1. **Seed from exercise-01.** Pull the one workflow the SME performed under screen
   share in Section B. That recording is the primary source of truth; everything in
   the map should be defensible against it.
2. **Shadow two more cases.** Schedule one more screen-share with the same SME and
   one with a second SME in the same role. Record and timestamp step boundaries;
   those timestamps become the p50/p90 evidence.
3. **Pull volumes from a system, not from memory.** Export counts from the LOS,
   ticketing system, or dashboard the SME named in Section C of the interview.
   Attach the CSV/export to the artifact folder.
4. **Draft the map as ASCII first.** The BPMN-lite block diagram in §2.1 is cheap to
   edit; a prettier visual tool slows down iteration in weeks 2–3 when the map still
   churns daily. Promote to a diagramming tool only after a readback has settled it.
5. **Hold a 30-minute readback.** Walk the SME (ideally a different SME than the one
   who co-authored it) through the map step by step; capture every "that's not how
   it usually goes" verbatim — those are the variants for §2.3.
6. **Freeze the artifact before intervention scoping.** The candidate list in §2.4
   should be written against a frozen map, not a map that's still moving; otherwise
   exercise-03's gating questions will target stale steps.

## 3. Validation steps

Before showing the map to the customer, verify:

1. **Every step has a system name.** "They do it in a tool" is not a step.
2. **Every step has a duration backed by an artifact.** Screen recording timestamp,
   dashboard export, or a direct observation note. If a number is a guess, mark it
   with a `<!-- follow-up: ... -->` HTML comment and chase it before scoping.
3. **The sum of p50 × frequency reconciles to a plausible headcount.** If the map says
   8 FTE but the team is 3 FTE, something is double-counted.
4. **At least one real case was observed end-to-end.** Interview-only maps omit
   silent steps (copy-pastes, re-reads, Slack lookups).
5. **Variants are from a real sample**, not invented. State sample size ("20 recent
   files, pulled on <date>").
6. **The candidate list ties interventions to specific steps and variants**, not to
   "the workflow."
7. **Data-boundary annotations exist** anywhere the work touches regulated data (see
   NIST AI RMF Map function and OWASP ML Top 10 for data-handling risks).
8. **The map has been read back** to one SME other than the one who authored the
   content with you — and they agreed or the disagreement is captured.

## 4. Rubric / review checklist

Score 0–2 per row. 12/18 is the pass bar.

| Dimension                       | 0 — Missing                               | 1 — Partial                                       | 2 — Strong                                                                                        |
|---------------------------------|-------------------------------------------|---------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Step granularity                | Steps collapse multiple actions           | Mostly atomic                                     | Each step is one actor, one system, one output                                                    |
| Durations backed by evidence    | Numbers absent or round-number guesses    | Some evidence, some guesses                       | Every number cites an observation, interview, or system export                                    |
| p50 **and** p90 captured        | Only averages                             | Only one of the two                               | Both present; fat-tail steps explicitly called out                                                |
| Variant tree                    | None                                      | Variants listed but no counts                     | Variants from a named real sample, with counts                                                    |
| Intervention list specificity   | "AI for the workflow"                     | Interventions tied to steps                       | Tied to steps **and** variants; non-AI recommendations named separately                           |
| Downstream risk & data boundary | Not addressed                             | Noted once                                        | Each sensitive step annotated; aligned to NIST AI RMF Map and relevant OWASP/ATLAS entries        |
| Reconciliation sanity check     | None                                      | Checked verbally                                  | Written: total load reconciles to headcount or the gap is explained                               |
| Readback                        | Not read back                             | Read back to original SME only                    | Read back to a second SME; disagreements captured in the artifact                                 |
| Artifact quality                | Freeform text                             | Structured but partial                            | Visible map + table + variant tree + candidate list in one document                               |

## 5. Common mistakes

- **Averages-only.** Reporting only p50 hides the variants where real time lives. The
  60-minute spreadsheet and the 200-minute spreadsheet are different *problems*; an
  intervention that only helps the median case has little business value.
- **Scoping to a step, not a variant.** "AI will speed up risk analysis" sounds right
  and will fail in acceptance. "AI will pre-fill the risk sheet for the clean-financial
  variant (~20% of cases)" is a scope you can actually deliver.
- **Mapping what SMEs say they do, not what they do.** Ask for a screen recording of a
  real case. Mental models are tidier than reality.
- **Including wait time as if it were work.** Wait-time problems are scheduling or
  queueing problems, not AI problems. Flag them; don't fold them into the AI case.
- **No evidence citations in the artifact.** The map must be defensible to a sceptical
  reviewer six weeks later; opinions without citations will not hold up in a steering
  meeting.
- **Ignoring compliance and data-residency.** A step that moves regulated data between
  systems may be an immediate out-of-scope flag regardless of its time cost.
- **One intervention per step.** The real pattern is often "intervene on variant 4c
  and step 5 together" — scope by value, not by diagram geometry.

## 6. References

- NIST AI Risk Management Framework — **Map** function (context, impact, intended
  use, dependencies). Use this as the top-level checklist when annotating steps for
  risk and data flow.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — reference for data-handling risks on any
  step that touches training-eligible or sensitive data.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — use for workflows where adversaries are in scope (fraud, abuse,
  trust & safety); maps tactics to the steps where an AI component could be attacked.
  <https://atlas.mitre.org/>
