# exercise-05 — Design Partner Commercials Kick-Off Brief · Reference Solution

## 1. Solution overview

A **design partner engagement** is a specific commercial shape distinct from a
proof-of-concept and distinct from a paid deployment. The customer provides access,
SME time, and a real workflow; the vendor provides engineering, product shaping,
and (usually) a reduced-or-waived pilot fee in exchange for feedback, co-design, and
a reference. The kick-off brief is the artifact that aligns the FDE lead, the vendor's
GTM owner, the customer's business sponsor, and the customer's procurement/legal
contact on *what kind of deal this is* before any contract is drafted.

The brief sits **upstream** of the pilot scope document (exercise-04). Scope answers
*what will we build and how will we measure it*; the commercials brief answers *why
we are doing this together, what each side gets, and what happens after the pilot*.

A complete brief has six parts:

1. Deal type and intent — what this engagement is and is not.
2. Value exchange — what each side gives and receives, including the non-money
   trades (SME time, data, references, co-marketing, product influence).
3. Commercial terms — fees, in-kind, IP, data rights, publicity.
4. Exit paths — what happens at the end of the pilot in each of continue / expand /
   stop / fix.
5. Governance and escalation — who signs what, and who to escalate to.
6. Risks and mitigations — specific to the partnership shape, not to the technical
   pilot.

## 2. Worked answer — design partner kick-off brief

Replace all placeholders (`<...>`) with the real engagement's details. The example
below uses the ACME commercial-underwriting engagement from exercises 02–04.

```markdown
# Design partner kick-off brief: ACME Bank × <vendor>
Date: 2026-11-18
Authors: <FDE lead>, <vendor GTM>
Reviewed by: J. Patel (ACME sponsor), S. Rao (ACME security), <vendor CRO>
Status: DRAFT — for steering review on 2026-11-20

## 1. Deal type and intent
This is a **design partner engagement**, not a paid pilot and not a free trial.

ACME is a design partner because:
- Our product for assisted commercial underwriting is not yet GA. ACME's workflow
  shape (steps 2 and 4 of their underwriting process, including variant 4b/4c —
  see discovery notes) is representative of three other prospects in-pipeline.
- We need concentrated co-design access to ACME's SMEs and real (de-identified or
  tenant-isolated) cases to close gaps our current eval set does not cover.

ACME is **not** a paid customer yet. We are not selling on a per-seat basis during
this engagement. Conversion to a paid deployment is explicitly an option at readout,
not a commitment.

## 2. Value exchange
### What ACME provides
- Access: 3 SMEs for up to 4 hours/week each for 8 weeks; 1 sr-UW for rubric work
  (~6 hours total).
- Data: tenant-isolated access to intake documents and the risk-spreadsheet
  templates. No fine-tuning (see pilot scope §2.3).
- Systems: VPC-isolated inference environment inside ACME; SSO integration hours
  from M. Chen's team.
- Feedback: documented weekly on D-01..D-03 quality; written readout at pilot end.
- Reference (conditional): one named reference call for a comparable prospect within
  12 months of readout, if the pilot meets S-1 through S-5 in the scope doc.

### What <vendor> provides
- FDE team: 1 lead engineer + 0.5 backend engineer + 0.3 eval lead for 8 weeks.
- Engineering deliverables D-01..D-05 per the pilot scope.
- Pilot fee: **waived** for the 8-week pilot window. (This is the design-partner
  discount; the standard pilot fee would otherwise be <$X> — see §3.)
- Priority roadmap input: ACME's feedback is reviewed in <vendor>'s product sync;
  two ACME-raised items committed to the next product milestone.
- Early access to post-pilot GA features for 90 days after readout at no cost.

## 3. Commercial terms (summary — legal is a separate doc)
| Item                              | Term (pilot)                                                            | Term (post-pilot, if continue)                                         |
|-----------------------------------|-------------------------------------------------------------------------|------------------------------------------------------------------------|
| Pilot fee                         | Waived                                                                  | Standard commercial list, less 20% for design-partner year 1           |
| Seat/usage fee                    | None                                                                    | Per-seat or per-case pricing, negotiated at readout                    |
| Minimum commitment                | None                                                                    | 12-month term on conversion                                            |
| Data ownership                    | ACME retains all customer and derived data                              | Unchanged                                                              |
| Model-training rights             | <vendor> has **no** right to train models on ACME data in pilot or after | Unchanged — opt-in only                                                |
| IP                                | Each side retains its background IP; foreground IP in <vendor>'s stack  | Unchanged                                                              |
| Publicity / reference             | None during pilot without written approval                              | Reference call as described in §2; case study optional, ACME approves  |
| Termination                       | Either party may terminate on 10 business days' written notice          | Standard commercial terms                                              |

Legal terms are governed by the design-partner MSA/order form currently with
ACME procurement (owner: <procurement contact>). The scope doc is operational; the
MSA is contractual; this brief is the plain-English bridge between them.

## 4. Exit paths — what happens at readout (2027-01-20)
| Exit                 | Trigger                                                                                  | Commercial consequence                                                                          | Owner at ACME     | Owner at <vendor>   |
|----------------------|------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|-------------------|---------------------|
| Continue (expand)    | Scope success criteria met; ACME wants scale-up                                          | Commercial negotiation begins within 2 weeks of readout                                         | J. Patel          | <vendor CRO>        |
| Continue (fix)       | Criteria mostly met; one or two gaps named                                               | 4-week extension at design-partner terms; written gap plan                                      | J. Patel          | FDE lead            |
| Stop (we're wrong)   | Technical: pattern doesn't fit; governance: risk unresolved                              | Clean termination; access revoked within 5 business days; data destruction certificate          | S. Rao            | FDE lead            |
| Stop (not for us)    | Criteria met, but ACME strategic priorities moved                                        | Clean termination; <vendor> may use anonymised lessons in roadmap                               | J. Patel          | <vendor CRO>        |

Each exit has a 1-page template (`exit/<kind>-template.md`) ready at kickoff. The
template is filled at readout, not drafted under pressure.

## 5. Governance and escalation
- Steering committee: as per pilot scope §2.0.
- Escalation path, ACME side: J. Patel → <ACME exec>.
- Escalation path, <vendor> side: FDE lead → <vendor CRO> → <vendor CEO>.
- Legal escalation: ACME procurement contact ↔ <vendor> counsel.
- Security/privacy escalation: S. Rao ↔ <vendor> security lead.
- Response SLA for escalations: 1 business day acknowledgement, 5 business days to
  written resolution plan.

## 6. Risks specific to the design-partner shape
These are commercial-partnership risks, separate from the technical pilot risks in
the scope doc.

| Risk                                                                                    | Severity | Mitigation                                                                                           | Owner             |
|-----------------------------------------------------------------------------------------|----------|------------------------------------------------------------------------------------------------------|-------------------|
| ACME SMEs over-committed, slipping availability                                         | High     | Weekly availability audit; escalate at 2-week lag to J. Patel                                        | FDE lead          |
| Scope creep driven by ACME product feedback (useful but off-pilot)                      | High     | Parking-lot process (scope §2.8); steering reviews at midpoint; no silent absorptions                | FDE lead          |
| Expectations mismatch on reference/case study                                           | Medium   | Reference conditions in §2 are binding; case study is separate written approval                      | <vendor GTM>      |
| Data flow out of ACME tenant                                                            | High     | Covered in scope §2.6 (NIST AI RMF Map); contractually, no training on ACME data                     | S. Rao            |
| ACME strategic re-prioritisation mid-pilot (merger, exec change, budget freeze)         | Medium   | Monthly sponsor check-in with J. Patel; early-warning conversation if sponsor leaves                 | <vendor CRO>      |
| Design-partner pricing precedent leaks to full-price prospects                          | Medium   | Terms are private; publicity requires written approval; case study language avoids pricing           | <vendor GTM>      |
| IP ambiguity on co-designed workflows                                                   | Medium   | Background/foreground IP split in §3; novel joint IP, if any, flagged to legal before implementation | <vendor> counsel  |

## 7. Approvals required at kickoff
- [ ] ACME business sponsor (J. Patel) — sign brief
- [ ] ACME procurement contact — confirm MSA/order form is in flight
- [ ] ACME security (S. Rao) — confirm §2.6 of scope is acceptable
- [ ] <vendor> CRO — approve waived pilot fee
- [ ] <vendor> FDE lead — confirm staffing commitment
```

## Implementation

How a trainee produces this brief in a real engagement:

1. **Write the brief before the MSA, not after.** The brief shapes the business
   intent; the MSA encodes it. Teams that let legal draft first spend weeks
   re-arguing deal type in markup. The FDE lead and vendor GTM own the first draft.
2. **Pair with the customer sponsor on §1 and §2.** Deal type (design partner vs.
   paid pilot vs. free trial) and value exchange are decided in a single joint
   session with the business sponsor. Customers who read the deal type for the
   first time at kickoff push back hard.
3. **Loop in procurement/legal on §3 and §4 early.** Share the draft with both
   procurement teams by week −2; their red-lines on IP, training rights, and
   termination take longer than the technical work. Exit path templates
   (`exit/<kind>-template.md`) are stubbed at this stage, not at readout.
4. **Collect approvals in writing.** The §7 checklist is tracked in a shared
   document with named approvers and dates; a thumbs-up emoji is not sign-off.
   Approvals land before kickoff — retrofitting them after week 1 is how renewals
   get stuck in procurement.
5. **Re-read at the midpoint.** Steering committee opens the brief at the midpoint
   review alongside the scope doc; any drift in deal type, reference expectations,
   or SME availability gets corrected while it is still cheap.
6. **Keep the brief versioned next to the scope doc.** Both live in the pilot
   workspace with a version history; changes to either require steering
   acknowledgement. The brief is the plain-English bridge, so it is the first
   thing a new stakeholder reads.

## 3. Validation steps

A brief is review-ready when:

1. **Deal type is named.** Design partner, paid pilot, free trial, PoC — they have
   different expectations. Writing the word out prevents mid-pilot drift.
2. **Value exchange is symmetric and specific.** Both columns list concrete items;
   "collaboration" is not a value.
3. **Reference commitments are conditional.** Unconditional references at the start
   of a pilot often unwind if the pilot has a rough patch.
4. **Model-training rights and data ownership are explicit.** These are the terms
   most likely to be contested post-pilot; write them at kick-off.
5. **Exit paths exist for all four states** — continue-expand, continue-fix, stop-
   wrong, stop-not-for-us. Each has owners on both sides.
6. **Escalation paths include names, not just roles.** Two levels of escalation on
   each side.
7. **Risks are partnership-specific**, not re-stated technical risks from the scope
   doc. If a risk from the scope doc appears here verbatim, it's probably the wrong
   document.
8. **The brief does not replace the MSA.** The last line of the brief should remind
   the reader that legal terms are in the MSA; the brief is the business bridge.
9. **Governance ties back to NIST AI RMF.** The commercial document references the
   governance regime in the scope doc; it does not invent a parallel one.
10. **Sign-offs are collected before kickoff**, not retrofitted.

## 4. Rubric / review checklist

Score 0–2 per row. 12/20 passes.

| Dimension                                | 0 — Missing                              | 1 — Partial                                        | 2 — Strong                                                                                      |
|------------------------------------------|------------------------------------------|----------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Deal type named explicitly               | Not named                                | Named casually                                     | Named, with reasoning for choosing design-partner vs. alternatives                              |
| Value exchange, both sides               | One-sided                                | Both sides but vague                               | Enumerated, specific, with volumes (SME hours, deliverables, fees)                              |
| Commercial terms table                   | Prose only                               | Table missing key terms                            | Table covers fees, IP, data, training rights, publicity, termination                            |
| Reference / publicity conditionality     | Unconditional                            | Mentioned                                          | Explicit conditions and approval mechanism                                                      |
| Exit paths × 4                           | 1–2 paths                                | All 4 named, uneven detail                         | All 4, with triggers, consequences, and named owners                                            |
| Escalation paths                         | Roles only                               | Names only one side                                | Named on both sides, with SLA for response                                                      |
| Partnership-specific risks               | Technical risks re-used                  | Mix                                                | Partnership risks named (SME capacity, reference leak, strategic shift) with owners             |
| Integration with scope doc + MSA         | None                                     | Mentioned                                          | Scope-doc section numbers cited; MSA owner named                                                |
| Approvals checklist                      | Absent                                   | Partial                                            | Checked at kickoff with signatures or written confirmations                                     |
| Governance alignment (NIST AI RMF)       | Not referenced                           | Mentioned generically                              | Points to scope doc's RMF mapping instead of duplicating                                        |

## 5. Common mistakes

- **Confusing the brief with the MSA.** The brief is business intent in plain
  English; the MSA is enforceable. Don't try to make the brief contractual, and
  don't let the MSA be the first time the customer sees the deal shape.
- **Paid-pilot phrasing on a design-partner engagement.** If the customer thinks
  this is a paid pilot and you think it's a design partner, you will disagree
  about references, roadmap influence, and pricing precedents at readout. Write
  the deal type out.
- **Unconditional reference commitments.** Customers who agree to be a reference
  at kickoff often regret it mid-pilot. Condition it on the pilot meeting its
  success criteria.
- **No exit for "stop — not for us."** This is the most emotionally awkward exit;
  having it pre-written removes the awkwardness and preserves the relationship
  for future opportunities.
- **Missing data-training rights language.** "We will not train on customer data"
  is table stakes to be written at kick-off, not negotiated later under pressure.
- **Governance duplication.** Re-writing the AI RMF mapping here invites
  divergence between the brief and the scope doc; cite the scope doc instead.
- **Treating the sponsor change as unlikely.** Twelve months after kick-off, a
  non-trivial share of named sponsors have moved roles. The monthly sponsor
  check-in is cheap insurance.
- **One-sided value exchange.** If the brief enumerates what <vendor> gives but
  says only "feedback and references" from the customer, the customer's finance
  team will under-invest; be specific about SME hours and data access.

## 6. References

- NIST AI Risk Management Framework — the brief points back to the scope doc's
  NIST AI RMF mapping (Govern/Map/Measure/Manage) rather than duplicating it.
  Governance consistency across the two documents matters more than restating it.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — relevant to the data-training-rights
  and data-handling clauses in §3; cited here only where those clauses bind.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — referenced from the scope doc's security review; the commercial
  brief notes the dependency rather than re-doing the threat model.
  <https://atlas.mitre.org/>

Note on the commercial terms in §3: fees, discounts, and term lengths vary per
vendor and region; the worked example uses illustrative placeholders. Replace with
vendor-approved numbers (confirmed with the vendor's CRO and procurement) before
any customer-facing use.
