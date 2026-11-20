# exercise-01 — SME Discovery Interview Script · Reference Solution

## 1. Solution overview

The deliverable is a **reusable, agenda-driven interview script** a forward-deployed
engineer runs with a customer subject-matter expert (SME) in the first 1–2 weeks of a
pilot. The script must do four jobs, in order:

1. Elicit the current "day in the life" of the SME (so we can observe the real
   workflow, not the idealised one the buyer sold us).
2. Surface **evidence**, not opinions: artifacts, screenshots, ticket IDs, SQL, spreadsheet
   rows. Discovery that returns only paraphrased opinions is almost always wrong.
3. Separate **what is painful** from **what is AI-amenable**. Painful ≠ AI-amenable;
   exercise-03 drills this explicitly.
4. Produce a written artifact (the filled-in script) that the FDE, the product team, and
   the customer can all read back without re-interviewing.

A good script is **time-boxed to ~45 minutes**, uses open questions early and closed
questions late, and has an explicit "parking lot" for scope the SME wants to litigate
but we are not scoping today.

## 2. Worked answer — the script

Copy this template and fill one per SME. One SME ≠ one interview; expect 2–3 passes per
role as trust builds.

```
Interview: <SME name · role · team>
Interviewer(s): <FDE names>
Date / channel: <YYYY-MM-DD · Zoom/Teams/onsite>
Recording consent: <yes/no — ask on the call, note who agreed>
Pilot name: <engagement codename>
```

### Pre-work (do before the call — 15 min)

- Read the SME's LinkedIn, two recent tickets or documents they authored, and any prior
  call notes. Interviews where the FDE did no pre-work waste the SME's goodwill budget.
- Write **one hypothesis** you want to confirm or kill in this interview. Example:
  *"Underwriters spend >30% of their day copying fields between the LOS and the risk
  spreadsheet."* The interview exists to update that hypothesis.

### Opening — 3 min

1. One-line purpose: *"We're scoping a 6-week pilot. My goal today is to understand how
   you actually work, where the friction is, and what success would look like — not to
   demo anything."*
2. Consent to record and to quote back to the team under their name.
3. Logistics: time-box, who the notes will be shared with, where parked items go.

### Section A — Role and day shape (8 min, open questions)

- "Walk me through yesterday from the time you opened your laptop. What did you do first?"
- "Which of those tasks happen every day vs. once a week vs. rarely?"
- "If you had to name the three tasks that consume the most of your week, which three?"
- "Which of those do you dread, and why?"

**Interviewer behaviour:** do not propose solutions. Do not introduce the word "AI" or
"LLM." Note verbatim phrasing — the SME's wording becomes your copy later.

### Section B — Deep dive on one workflow (18 min, with artifacts)

Pick ONE workflow from Section A, usually the one with highest frequency × dread.
Ask the SME to screen-share and *perform* it with a real recent example.

- "Can you pull up a real ticket / case / document from last week and walk through it
  while I watch?"
- "Where did the input come from? Who sent it, in what format?"
- "What do you check first? What do you look up, and where?"
- "What is the output artifact? Who consumes it next?"
- "Where do you keep context between steps — memory, sticky notes, a spreadsheet?"
- "What makes a case 'easy' vs. 'hard'? Give me one example of each from last week."
- "When was the last time this went wrong, and what happened?"
- "Which step, if removed, would you miss least?"

Capture artifacts as you go: file names, system names, URLs, exact field names,
approximate volumes ("~80 of these a week"), handoffs ("passes to the compliance
queue"), SLAs ("must clear within 2 business days").

### Section C — Measurement reality (6 min, closed questions)

- "How does your manager measure your team today? What's the KPI on the dashboard?"
- "Where does that KPI live? (dashboard URL / report name / spreadsheet)"
- "What was the number last quarter? Last month?"
- "If this workflow got 30% faster, which of those numbers would move, and by how much?"
- "Who would notice first — you, your manager, or a downstream team?"

This is the **operational-metric anchor** for the pilot. If the SME cannot name a
measurable number, the pilot does not have a credible success criterion yet — flag it
and plan a follow-up with their manager (see exercise-04).

### Section D — Constraints and non-goals (6 min)

- "What systems is this data **not** allowed to leave?" (data residency, PHI, PII,
  export controls)
- "Have you tried to automate any of this before? What happened?"
- "Who would veto a change to this workflow?" (the hidden stakeholder question)
- "If I came back in 6 weeks with a tool that did step 3 for you, who else would need
  to say yes before you could use it?"

### Section E — Close (4 min)

- "What did I not ask that I should have?"
- "Who else on your team should I talk to, and what will they tell me that you can't?"
- "Would you be willing to screen-record 2–3 real cases this week so I can watch
  asynchronously?" (gold — you want this)
- Confirm next step and date.

### Parking lot

Anything the SME wants to debate but we won't scope today: list it verbatim, name
the owner, assign to a future conversation.

## Implementation

A trainee runs the script like this in a real pilot:

1. **Pick the SME and co-pilot.** The business sponsor nominates 2–3 SMEs; the FDE
   picks one to interview first and pairs with a second interviewer (notes-only). Two
   people in the room produces a better artifact than one.
2. **Timebox the prep.** Spend no more than 15 minutes on pre-work per interview; the
   output is a one-sentence hypothesis in the shared pilot workspace.
3. **Create the artifact up front.** Copy the template in §2 into the pilot's
   `discovery/` folder as `sme-<name>-<date>.md` *before* the call. Having the empty
   artifact open forces you to fill the sections in order.
4. **Record with consent, transcribe after.** If the SME agrees, record the call and
   transcribe later; the live note-taker focuses on artifacts and verbatim quotes, not
   stenography. Delete the recording after the transcript is approved.
5. **Synthesise within 24 hours.** Fill the template end-to-end while the memory is
   fresh; mark any unresolved claim with an explicit follow-up tag and an owner. Share
   the filled artifact back to the SME for a 48-hour quiet-period correction.
6. **Expand breadth.** Schedule two more SMEs from different sub-teams within one
   week. Single-SME discovery is a known pilot-killer (see §5).

## 3. Validation steps

A finished interview artifact passes the review below. The FDE should self-check before
sharing.

| # | Check                                                                                      | Pass if …                                                                                 |
|---|--------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| 1 | Hypothesis updated                                                                         | The pre-work hypothesis is marked **confirmed / killed / refined**, with the evidence.    |
| 2 | ≥1 real artifact captured                                                                  | File name, URL, or screenshot reference is in the notes — not a paraphrase.               |
| 3 | At least one operational metric with a current value                                       | "X is Y per Z, measured in <dashboard>." Not a vibe.                                      |
| 4 | Workflow steps are numbered and have system names                                          | Each step names the system, the input field, and the next handoff.                        |
| 5 | A failure example from the last 30 days                                                    | Recent, concrete, with the SME's own words on what broke.                                 |
| 6 | Data-boundary question answered                                                            | What leaves the system and where; if the SME doesn't know, that's flagged as open.        |
| 7 | At least one follow-up named person                                                        | "Talk to <name> about <topic>" — otherwise you're still single-threaded on one SME.       |
| 8 | No solution talk in the notes                                                              | If the notes describe the solution the FDE wants to build, redo the interview.            |

## 4. Rubric / review checklist (for a trainer or peer reviewing a trainee's script)

Score 0–2 on each dimension. 10/14 is the pass bar for shipping to a real customer.

| Dimension                          | 0 — Missing                              | 1 — Partial                                            | 2 — Strong                                                                                 |
|------------------------------------|------------------------------------------|--------------------------------------------------------|--------------------------------------------------------------------------------------------|
| Pre-work hypothesis                | None stated                              | Stated but vague                                       | Specific, falsifiable, tied to an SME-owned number                                         |
| Open-then-closed structure         | Mixed or closed-first                    | Mostly ordered                                         | Clean Section A→E progression, no "jumping to solution"                                    |
| Artifact-driven deep dive          | No real example used                     | SME describes but doesn't show                         | Screen-share with a real recent case; artifacts captured by name                           |
| Operational metric capture         | No metric                                | Metric named, no current value or source               | Metric + source + current value + plausible delta                                          |
| Constraints and data-boundary ask  | Not asked                                | Asked but SME uncertain, no follow-up noted            | Asked; answers captured; unknowns escalated to named owner                                 |
| Stakeholder expansion              | No one else named                        | One other person named                                 | Multiple named people with topics; a veto-holder identified                                |
| Written artifact quality           | Freeform notes, no structure             | Filled template, missing sections                      | Filled template, verbatim quotes, parking lot, next step confirmed                         |

## 5. Common mistakes

- **Pitching, not listening.** The FDE spends >25% of the air time describing the
  system. The symptom in the notes: lots of "we can do X" phrases. Fix: interviews
  precede any demo; demos go in a separate meeting.
- **Confirming your hypothesis.** The FDE asks leading questions (*"so you'd save a lot
  of time if …?"*). Fix: run Section B strictly on a real past case; let the facts
  update the hypothesis.
- **Opinion instead of evidence.** Notes read like *"SME says it takes a long time."* No
  number, no artifact. Fix: every claim gets an artifact or an explicit follow-up tag.
- **Interviewing only enthusiasts.** The buyer introduces you to the SME who already
  likes AI tools. Fix: ask the enthusiast who on the team is **skeptical** and talk to
  them too — they'll name the real failure modes.
- **Skipping data-boundary and compliance.** Discovery that ignores where data is
  allowed to flow produces a scope that security kills in week 4. The NIST AI RMF
  "Map" function (identifying context and risks) applies here.
- **One SME, one meeting.** Pilots scoped on a single 45-minute interview collapse
  when the SME's manager has different priorities. Plan for breadth.

## 6. References

- NIST AI Risk Management Framework — "Map" function (context, use, and risk
  identification) and Generative AI Profile. Use the Map function as the checklist for
  Section D (constraints, non-goals, downstream stakeholders).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — relevant to Section D when data-boundary
  questions touch training data or model inputs.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — reference for Section D when the SME's workflow is adversarially
  exposed (fraud, content moderation, security ops).
  <https://atlas.mitre.org/>
