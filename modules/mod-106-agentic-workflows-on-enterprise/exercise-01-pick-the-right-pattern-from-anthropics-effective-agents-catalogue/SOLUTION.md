# exercise-01 — Pick the Right Pattern from Anthropic's Effective-Agents Catalogue · Reference Solution

## 1. Solution overview

The deliverable is a **pattern-selection worksheet** that classifies three
candidate enterprise use cases against the Anthropic "Building Effective
Agents" catalogue, picks the simplest pattern that solves each one, and
writes down why the more autonomous patterns were *rejected*.

Anthropic's catalogue (the December 2024 engineering post "Building
Effective Agents") draws a sharp line between **workflows** — orchestrated
chains of LLM calls with code-defined control flow — and **agents** — LLMs
that dynamically direct their own tool use in a feedback loop with their
environment. The post enumerates five workflow building blocks (prompt
chaining, routing, parallelization, orchestrator-workers,
evaluator-optimizer) and one agent pattern (autonomous agent). See
<https://www.anthropic.com/engineering/building-effective-agents>.

Three design principles separate a defensible pattern pick from a demo:

1. **Workflows first, agents last.** The Anthropic guidance is explicit:
   adopt the autonomous-agent pattern only when the extra flexibility pays
   for the extra cost, latency, and failure surface. In enterprise work the
   bar is higher, not lower — a workflow is auditable in a way an agent is
   not.
2. **Pick the pattern by the shape of the task, not by what's new.** Prompt
   chaining is older than LLMs; it is also the right answer for most
   enterprise tasks. "We chose agents" with no written justification is a
   smell.
3. **Write down the rejected patterns.** A pick is only defensible if the
   two neighboring patterns (one simpler, one more autonomous) were
   considered and ruled out on specific criteria.

The common FDE mistake is to reach for the autonomous-agent pattern because
it is the most technically interesting and demos well. Senior FDEs pick
prompt chaining or routing more often than feels ambitious and ship pilots
that remain within the customer's risk envelope.

> This exercise selects the **control-flow pattern**. It does not design
> the toolset (that is exercise-04), the HITL checkpoints (exercise-03),
> the implementation SDK (exercise-02), or the threat model (exercise-05).
> Name hand-offs explicitly.

## 2. Implementation — the pattern-selection worksheet

### 2.1 The catalogue (with when-to-use)

Reproduce the Anthropic catalogue in the trainee's own words, because
candidate picks will be scored against this table.

| Pattern | What it is | When to pick it | When to reject it |
|---|---|---|---|
| **Prompt chaining** | Split the task into fixed LLM calls, each consuming the previous output. Optional programmatic gates between steps. | Task decomposes cleanly into a fixed sequence (e.g., generate → validate → translate). You can write the steps on a whiteboard before building. | Steps are not knowable up front, or the branching exceeds a shallow if/else. |
| **Routing** | A classifier call directs the input to one of N specialized downstream prompts or pipelines. | Inputs fall into a small, enumerable set of handling classes (e.g., refund vs policy question vs account change). | Classes are not enumerable, or the "right" handler depends on multi-step reasoning. |
| **Parallelization — sectioning** | Decompose into independent subtasks run in parallel, then aggregate. | Task has independent dimensions (e.g., review for correctness, review for compliance, review for style) that can be reasoned about in isolation. | Subtasks share state or need to see each other's output. |
| **Parallelization — voting** | Run the same prompt N times with diverse seeds/angles, aggregate by majority/consensus. | High-stakes judgment where a single sample is unreliable (e.g., moderation calls, adversarial verification). | Cost/latency budget cannot absorb N× calls, or the task has an objective answer the first call usually gets right. |
| **Orchestrator-workers** | A planner LLM decomposes the task into sub-tasks on the fly, dispatches to worker LLMs, synthesizes. | Decomposition cannot be written in advance because it depends on the input (e.g., research across unknown-in-advance sources). | A fixed chain would cover ≥90% of inputs — don't pay the orchestrator overhead for a chain in disguise. |
| **Evaluator-optimizer** | One LLM produces, a second evaluates against criteria, the producer revises. Iterate until pass or budget exhausted. | There is a clear, mechanical evaluation signal (compiler errors, test results, rubric score). | Evaluation is subjective and you can't distinguish "good" from "plausible"; the loop will converge on plausible. |
| **Autonomous agent** | LLM chooses tools and iterates against the environment until a stopping condition, with observations fed back each turn. | Task is open-ended, requires exploration, and the toolset is small, safe, and reversible. | The toolset has irreversible side effects, the environment is a shared production system, or the customer needs a deterministic audit trail. |

### 2.2 Three candidate use cases (reuse the archetypes from mod-104/105)

| ID | Use case | Archetype | Volume / cadence | Risk envelope |
|---|---|---|---|---|
| **U-01** | Contract-intake triage: classify, extract key terms, route to the correct queue | Regulated US (A) | ~2k docs/day, business hours | Mid — wrong queue costs an SLA miss, not a legal filing |
| **U-02** | SOC 2 evidence collection: given an audit control, gather evidence from Jira, Confluence, SharePoint, and the ticketing system, and assemble a reviewer-ready packet | EU residency (B) | ~40 controls/quarter, bursty | High — missing evidence is an audit finding; fabricated evidence is a terminal event |
| **U-03** | On-call triage copilot: a human SRE drives; the system proposes next steps during an incident, cites runbook links, and offers to execute read-only queries | Air-gapped (C) | Spiky, 5–20 incidents/week | High — wrong action in production blast radius; SRE is always in the loop |

Each use case is a real shape an FDE sees in the field; none are novel
science.

### 2.3 Pick + rejected patterns

#### U-01 — Contract-intake triage · **Pattern: Routing + Prompt chaining**

- **Why routing:** inputs fall into a small set of document archetypes (MSA,
  SOW, NDA, DPA, renewal, amendment). A classifier call routes to a
  per-archetype extraction prompt — this is exactly the Anthropic routing
  use case.
- **Why chaining under each route:** per archetype, the task decomposes
  into (1) classify, (2) extract fields with structured output, (3)
  validate against a per-archetype rubric, (4) hand off to the queue.
  Write the four steps down; there are no mystery branches.
- **Rejected simpler:** a single prompt that tries to handle every
  archetype. Rejected because per-archetype rubrics diverge sharply
  (indemnification clauses vs renewal terms); one prompt means one
  compromised rubric and worse recall on every type.
- **Rejected more autonomous:** orchestrator-workers. Rejected because the
  chain is knowable in advance — paying for an orchestrator LLM to
  rediscover "classify then extract then validate" every call wastes cost
  and latency without improving recall. Also reduces auditability: the
  compliance reviewer wants a fixed step list.
- **Pattern diagram:**

```
input doc
  └─► classifier (routing) ──► {MSA, SOW, NDA, DPA, renewal, amend}
                                       │
                                       ▼
                 per-archetype prompt chain:
                   classify_confidence ─► extract_fields (structured out)
                     ─► rubric_validate ─► emit_to_queue
```

- **Hand-offs:** HITL (exercise-03) sits on the `rubric_validate` step for
  low-confidence rows. Tools (exercise-04) are read-only against the
  document store plus a single write tool `queue.enqueue`.

#### U-02 — SOC 2 evidence collection · **Pattern: Orchestrator-workers**

- **Why orchestrator-workers:** the decomposition depends on the control.
  Control CC6.1 pulls different evidence than CC7.2; new controls are added
  every audit cycle. A planner LLM reads the control text, decomposes into
  per-source sub-fetches, dispatches worker calls (each scoped to one
  source), and synthesizes a packet. This is Anthropic's example of when
  orchestrator-workers is correct — the sub-tasks "can't be [predicted]
  ahead of time."
- **Why not an autonomous agent:** the toolset is write-sensitive (filing
  evidence into the audit folder has an audit trail), the sources are
  shared production systems (SharePoint, Jira), and auditors require a
  fixed, reviewable packet structure. An agent that freely explores would
  produce non-repeatable packets across audits.
- **Rejected simpler:** prompt chaining. Rejected because the chain is not
  stable — the shape of a CC6.1 evidence packet has three Confluence
  pulls and one Jira query; a CC7.2 packet has one Confluence pull and
  two SharePoint pulls plus a ticket join. Encoding this in fixed code
  branches creates an unmaintainable `if control == "CC6.1"` ladder.
- **Rejected more autonomous:** autonomous agent. Rejected on scope: the
  auditor wants "this control, these sources, these artifacts, review."
  An agent that proposes new sources is a scope-expansion risk during
  audit season.
- **HITL coupling:** every packet is a drafted artifact a human owner
  signs before it goes to the auditor (exercise-03 L3 — advisory).
- **Hand-offs:** Tools (exercise-04) are read-only against Confluence,
  Jira, SharePoint, plus one write tool `audit.file_evidence_packet` that
  is HITL-gated.

#### U-03 — On-call triage copilot · **Pattern: Autonomous agent (scoped, HITL-dominant)**

- **Why an autonomous agent:** the state of an incident is open-ended and
  not knowable up front. The copilot's value is iterating against
  observations (`kubectl get pods`, `log search`, `dashboard query`) until
  it has enough signal to propose a next action. This matches the
  Anthropic criterion: "the task requires an unknown number of steps and
  you can't hardcode a fixed path."
- **Why it is still safe:** the tool surface is read-only by default
  (log/metric/CMDB reads, runbook lookups); write tools require
  explicit human confirmation (exercise-03 L1 — interactive). The SRE
  is always in the loop. The agent is an "autonomous reasoner, manual
  actuator."
- **Rejected simpler:** evaluator-optimizer. Rejected because there is no
  automatic evaluator for "is this the right next step in an incident."
  The evaluator-optimizer loop would converge on plausible-looking
  commentary.
- **Rejected more autonomous:** autonomous agent with write tools
  unsupervised. Rejected on blast radius: a wrong `kubectl delete pod`
  in a payment-processing cluster is a terminal event. The agentic
  pattern is correct; the autonomous-*actuation* bit is not.
- **Hand-offs:** HITL (exercise-03) on every write, with time-boxed
  session reviews. Permission model (exercise-04) scopes each read tool
  to the SRE's least-privilege role. Threats (exercise-05) include tool
  misuse, prompt-injected log content, and session hijack.

### 2.4 Negative choices — patterns we did *not* mix in

- **No evaluator-optimizer anywhere.** Each use case has a human reviewer
  or a hard eval set already; a self-grading inner loop would learn to
  game the grader. Evaluator-optimizer is reserved for use cases with a
  mechanical signal (unit tests, compiler, structured validator) that
  cannot be gamed cheaply.
- **No parallel voting on U-02.** Evidence collection is an
  exact-recall task, not a judgment task. Voting over four independent
  samples does not make a missing Jira ticket appear.
- **No prompt chaining on U-03.** The triage path is not knowable in
  advance; a chain would misfit most incidents.

### 2.5 Pattern-to-SLO quick map

The pattern constrains the SLO envelope. This is what you tell the
customer in the pilot scoping conversation.

| Pattern | Typical latency | Typical cost | Audit shape | Failure mode |
|---|---|---|---|---|
| Prompt chaining | Sum of step latencies | Sum of step tokens | Full per-step transcript | Any step fails → whole chain fails |
| Routing | Classifier + 1 downstream | Classifier + 1 downstream | Route decision + downstream transcript | Misroute → wrong downstream prompt handles it badly |
| Parallelization — sectioning | Max(step latencies) + aggregate | Sum of all branches | Per-branch transcript + aggregator | One branch stale/missing makes aggregate incoherent |
| Parallelization — voting | Max(sample latencies) + reduce | N × single-sample cost | N transcripts + vote record | Consensus below quorum → escalate |
| Orchestrator-workers | Variable; planner adds a round-trip | Planner + Σ worker calls | Plan + per-worker transcripts | Planner hallucinates a sub-task that has no worker |
| Evaluator-optimizer | Variable; loop count × single-call latency | Loop count × single-call cost | Per-iteration producer/evaluator pair | Loop oscillates; set a hard budget |
| Autonomous agent | Unbounded by default; cap by step count and wall-clock | Unbounded; cap by token budget | Full trace per session, with tool calls | Sidewalk-looping; stuck tool; prompt-injected observation |

### 2.6 Common composition — enterprise picks are usually hybrid

Real pilots rarely pick exactly one row. The defensible answer composes
patterns:

- U-01: **routing → chaining**.
- U-02: **orchestrator-workers wrapping chained workers**.
- U-03: **autonomous agent whose tool surface is itself a routing +
  chaining stack**, with write tools gated by exercise-03.

Document the composition; do not pretend a hybrid is a single pattern.

## 3. Validation steps

Before the worksheet is customer-ready:

1. **Three real use cases behind the picks.** Each use case maps to an
   actual customer intent captured in mod-102 discovery. Fictional use
   cases produce fictional patterns.
2. **Simpler-pattern rejection is written and specific.** "We didn't use
   chaining because the task is dynamic" is not sufficient. Name the
   branch that would have exploded in `if/else`.
3. **More-autonomous-pattern rejection is written and specific.** Reject
   on risk (blast radius, auditability, cost/latency) not on
   "agents are complicated."
4. **A pattern diagram exists for each use case** and names every LLM
   call, every tool, and every human checkpoint. If the diagram fits on
   a napkin, the pattern is probably simple enough for pilot.
5. **Composition is explicit.** Any hybrid (U-01's routing + chaining,
   U-02's orchestrator around workers) is drawn with its boundary
   clearly labeled so the implementation team (exercise-02) does not
   accidentally collapse them.
6. **Pattern-to-SLO row is filled in per use case** against the mod-104
   ex-04 SLOs. If the pattern latency exceeds the SLO ceiling, pick a
   simpler pattern or fix the SLO now.
7. **The worksheet survives a product-sense tabletop.** A product
   reviewer should be able to name, for each use case, which pattern
   and why in under 30 seconds after reading the worksheet.

## 4. Rubric / review checklist

Score 0–2 per dimension. 14/22 passes before customer review.

| Dimension | 0 — Missing | 1 — Partial | 2 — Strong |
|---|---|---|---|
| Three distinct use cases | One case repeated | Three but one is a demo | Three with real customer intent |
| Catalogue table filled in trainee's own words | Copied verbatim | Paraphrased but incomplete | Trainee can defend each when-to-use |
| Pattern pick per use case | Only one case solved | All three but one pick is unjustified | Each pick named with ≥2 sentences of why |
| Simpler-pattern rejection | None | Generic | Named branch/collapse that breaks simpler |
| More-autonomous rejection | None | "Too risky" only | Specific blast-radius / audit reason |
| Diagram per use case | None | One diagram | Three diagrams showing LLM calls, tools, HITL |
| Hybrid / composition explicit | Pretends single pattern | Composition noted but not drawn | Boundary labeled on diagram |
| Pattern-to-SLO row | Missing | Named generically | Matches mod-104 ex-04 SLO numbers |
| Hand-offs to ex-02/03/04/05 | Not named | One hand-off | All four hand-offs named |
| Negative-pattern list | None | One item | Explicit "we did not use voting because …" |
| Catalogue source cited | None | URL only | Catalogue post + which section applies |

## 5. Common mistakes

- **Starting from "we chose agents."** The Anthropic post leads with
  "workflows first." Starting from the most autonomous pattern means
  every simpler pattern becomes a retrofit.
- **Ignoring the composition.** Routing + chaining is the most common
  enterprise answer; calling it just "routing" drops the chained-worker
  design from the worksheet, which drops it from the implementation.
- **Confusing orchestrator-workers with autonomous agents.** The
  orchestrator pattern has fixed worker roles and a synthesizer; the
  autonomous pattern lets the LLM pick tools and iterate against
  observations. Mislabeling turns an auditable workflow into an
  un-auditable agent on the architecture diagram.
- **Picking voting for cost savings.** Voting *costs* N×; it does not
  save tokens. Its job is correctness-through-sampling.
- **Using evaluator-optimizer on subjective judgment.** Without a
  mechanical evaluator the loop will converge on plausible-but-wrong.
  Reserve it for code/structured validation.
- **Not naming a termination condition for agent patterns.** Autonomous
  agents without an explicit stop rule (max steps, max tokens, max
  wall-clock, success criterion) run until the API bill escalates.
- **Treating pattern as a product.** The catalogue is a design
  vocabulary, not a buying decision. The SDK (exercise-02) is
  orthogonal — Claude Agent SDK can implement any of these; OpenAI
  Agents SDK too.
- **Forgetting the human in U-03.** "Autonomous agent" does not mean
  "unsupervised actuator." The autonomy is in the reasoning loop; the
  side effects stay HITL.
- **No written SLO envelope per pattern.** Without one, the
  implementation team (exercise-02) discovers the latency ceiling in
  staging.
- **Letting the demo pattern survive.** The pattern that demos cleanly
  at trial is not always the pattern that passes change-management.
  Use the rejection list to catch this before the customer does.

## 6. References

- **Anthropic — "Building Effective Agents"** (December 2024). The
  canonical source for the catalogue and the workflows-first framing.
  <https://www.anthropic.com/engineering/building-effective-agents>
- **NIST AI Risk Management Framework** — apply Govern/Map/Measure/Manage
  to each pattern pick: Map includes the pattern diagram and tool
  surface; Measure includes pattern-specific metrics (step success,
  misroute rate, orchestrator-plan quality, agent-step budget).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — pattern-specific risks
  (training-data leakage affects evaluator-optimizer loops that
  memorize evaluator rationale; adversarial examples affect routing
  classifiers). Keep the current list as the source of truth.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — adversarial tactics against LLM systems; relevant
  per-pattern (indirect prompt injection via tool output affects
  autonomous agents disproportionately). <https://atlas.mitre.org/>
- Companion exercises in this module: exercise-02 (SDK implementation),
  exercise-03 (HITL checkpoints), exercise-04 (tool permission model),
  exercise-05 (OWASP agentic threat model).
