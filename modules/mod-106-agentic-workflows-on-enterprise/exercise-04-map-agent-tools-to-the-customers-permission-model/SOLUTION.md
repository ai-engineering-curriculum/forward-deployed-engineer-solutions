# exercise-04 — Map Agent Tools to the Customer's Permission Model · Reference Solution

## 1. Solution overview

The deliverable is a **tool-to-permission matrix** that binds every agent
tool in the pilot to a concrete scope in the customer's existing IAM /
permission model, names the identity under which the tool actually runs,
and shows how the end user's entitlement is carried through the agent
to the downstream system without the agent becoming a privilege-escalation
path.

Three design principles separate a defensible permission mapping from a
"the agent has an API key" slide:

1. **The agent is not a principal.** The *user* on behalf of whom the
   agent acts is the principal. The agent is a delegation surface. Every
   tool call must be attributable to a human (or a named service) in the
   customer's IAM.
2. **Prefer token exchange over shared service accounts.** The pattern
   from mod-103 ex-02 (RFC 8693 OAuth 2.0 Token Exchange) carries the
   user's identity and scopes through the agent into the downstream
   system. A shared "agent service account" that can read every ticket
   is the single largest exfiltration lever in an enterprise LLM
   deployment.
3. **Least privilege is a design constraint, not an afterthought.** The
   matrix starts from the tightest scope that *could* satisfy the tool
   and widens only when a specific intent demonstrably requires it.
   "What's the minimum scope?" beats "what scope do we already have."

> This exercise maps the **permission surface**. The checkpoint model
> that gates writes lives in exercise-03; the SDK code that enforces
> scopes lives in exercise-02; the threats this mapping mitigates
> live in exercise-05.

## 2. Implementation — the permission matrix

### 2.1 Four identity shapes an agent tool can run under

Every tool call executes under exactly one of these:

- **User-delegated (OBO).** The agent exchanges the end user's token
  (SSO-issued, mod-103 ex-01) for a downstream token with
  `aud=downstream-system`. The downstream enforces the user's own
  entitlement. **Default for user-facing surfaces.**
- **Service account (impersonated).** The agent uses a service account
  scoped to a single downstream role; actions are attributed in the
  downstream to that service account, with the user ID recorded in a
  structured custom header. **Fallback when the downstream has no
  token-exchange support** (common with legacy SOAP/SFTP systems from
  mod-103 ex-05).
- **Shared service account (bounded).** A narrow, well-known identity
  used for genuinely shared reads (e.g., public runbook store). Only
  for data with no per-user ACL distinction.
- **Agent-only identity.** A principal used only inside the agent's
  own plane — for its own audit log, its own session store, its own
  observability. **Never a downstream call under this identity.**

### 2.2 The matrix

Columns: tool, side effect, identity shape, scope at downstream, how
delegation is proven, HITL tier (from exercise-03), audit attribution.

| Tool | Side effect | Identity shape | Downstream scope | Delegation mechanism | HITL tier | Audit attribution |
|---|---|---|---|---|---|---|
| `log_search` | read | User-delegated | `logs:read` restricted to clusters/services the user has `view` on | OIDC → RFC 8693 token exchange, `aud=loki` | L0 | user ID + session ID |
| `metric_query` | read | User-delegated | `metrics:read` scoped to user's dashboards | Same | L0 | user ID + session ID |
| `runbook_lookup` | read | Shared service account | `runbook:read` on public store | Agent SA with narrow role | L0 | agent SA + user ID in header |
| `docs_search` (Confluence) | read | User-delegated | user's space entitlements only; ACL filter applied on retriever (mod-105 ex-02) | SSO SAML/OIDC → tenant-scoped API token | L0 | user ID |
| `jira.search` | read | User-delegated | user's project permissions | OIDC → Jira delegated | L0 | user ID |
| `jira.create_ticket` | write | User-delegated | `create` on the one project named in the tool call | OIDC → Jira delegated | L3 (draft) / L1 (auto-file) | user ID |
| `jira.transition_ticket` | write | User-delegated | `transition` on the ticket | OIDC → Jira delegated | L1 | user ID |
| `kubectl rollout-restart` | write | User-delegated | RBAC: `patch deployments` in `ns ∈ user.allowed_namespaces` | SSO → Kubernetes OIDC (projected audience) | L1 | user ID (via `impersonate-user`) |
| `kubectl scale` | write | User-delegated | RBAC: `patch deployments/scale` in `ns ∈ user.allowed_namespaces` | Same | L1 | user ID |
| `kubectl cordon/drain` | write | User-delegated | RBAC: node-level, prod-restricted | Same | L1 + two-person | user ID × 2 |
| `terraform.plan` | read | Service account (impersonated) | read-only on the state backend | Short-lived IAM role assumed with user ID in session name | L0 | SA + user ID in session |
| `terraform.apply` | write | Service account (impersonated) | write on one namespace of the state | Same | L1 + two-person | SA + two user IDs |
| `audit.file_evidence_packet` | write | User-delegated | user is a named control owner | OIDC → audit-system token | L3 only | user ID (control owner) |
| `email.send_customer` | write (irreversible) | **not exposed** | — | — | — | — |
| `payments.issue_refund` | write | User-delegated | `refund:create` scoped to amount ≤ user cap | OIDC → payments-service token | L1 + two-person | user ID × 2 |
| `chatops.page_secondary` | write | User-delegated | user has `page` scope on the team | OIDC → PagerDuty delegated | L1 | user ID |

A few rows carry a specific message:

- **`email.send_customer` is not exposed.** No tool at all is a valid
  entry; the matrix captures what the agent *cannot* do just as
  explicitly as what it can.
- **`runbook_lookup` is the exception that proves the rule.** Shared SA
  is acceptable *because* the runbook store has no per-user ACL.
  Everywhere else, the default is delegation.
- **Terraform rows run under a service account** because Terraform's
  provider stack rarely accepts OBO tokens. The user ID is carried in
  the assumed-role session name and in audit emission; the two-person
  rule covers the "SA has god mode on state" risk.

### 2.3 Scope naming conventions

Scopes follow the customer's existing convention — do not invent your
own. If the customer uses OAuth scopes (`logs:read`, `tickets:write`),
use those; if the customer uses Azure AD app roles, use roles; if the
customer uses Snowflake RBAC, use Snowflake roles. The matrix maps
*from the tool* *to the scope the customer already named* — never the
reverse.

### 2.4 Delegation mechanism, concretely

Reuse mod-103 ex-02's token-exchange spine. For each downstream:

- **Modern (OIDC/OAuth): RFC 8693 Token Exchange.** The gateway
  (mod-104 ex-02) accepts the end user's bearer token and exchanges
  it for a downstream-audience token with a reduced scope. The agent
  attaches the exchanged token to the tool call.
- **Modern (Microsoft stack): OAuth on-behalf-of.** The same shape in
  Entra ID vocabulary; different grant type name, same audit model.
- **Kubernetes:** project the user's OIDC ID token with
  `audience=https://kubernetes.default.svc` and use OIDC auth on the
  API server. RBAC is then tied to the user's groups.
- **Legacy (SOAP/SFTP, LDAP-fronted systems):** service account with
  the user ID pinned into a structured header or into the connection
  session (e.g., SFTP `Note:` field or a wrapping `X-End-User:` header
  applied by the gateway). The audit then joins the SA call with the
  gateway's user context.

### 2.5 The three use cases, mapped

#### U-01 — Contract-intake triage

- User identity: the operations user who kicked off the triage run.
- All tool calls (docs read, queue enqueue) use **user-delegated** tokens.
  Queue enqueue scope is `queue:write` on *the specific queue* the
  rubric selected, not the whole queue namespace.
- Shared SA is not acceptable for `docs_search` because document ACLs
  matter (see mod-105 ex-02).

#### U-02 — SOC 2 evidence collection

- User identity: the control owner who initiated the packet build.
- Reads from Confluence, Jira, SharePoint use **user-delegated** tokens
  scoped to the control owner's entitlements; this prevents the
  evidence packet from including pages the control owner cannot
  actually see (which would be a finding on its own).
- The packet draft is stored in a staging area under the control
  owner's ownership.
- `audit.file_evidence_packet` is L3 — only the control owner can
  publish.

#### U-03 — On-call triage copilot

- User identity: the on-call SRE (SSO session).
- All reads and writes delegated. Kubernetes RBAC is keyed to the
  SRE's groups.
- When SRE pages a secondary, the page carries *both* identities in
  the audit: who paged and who was paged.

### 2.6 What the matrix does *not* allow

- No tool runs under "the agent" as a nameable principal with its own
  wide scope. "Agent reads tickets" is never acceptable; "user reads
  tickets they can see" is.
- No tool has a scope wider than the LLM needs to *do its job for
  this user*. Narrower-than-the-user's-own-scope is fine; wider is
  not.
- No tool call bypasses MFA requirements the downstream system
  enforces. If the user cannot do the action manually without MFA,
  the agent cannot do it under their delegated token either.
- No tool uses long-lived tokens in-process. Exchange on each call;
  cache with short TTL per-user keyed by principal + audience.
- No tool is exposed to multiple tenants from the same agent process
  without per-tenant isolation (per-tenant signing keys, per-tenant
  cache, per-tenant rate limits).

### 2.7 Wiring the scopes into the SDK

Exercise-02 ships the enforcement surface; the matrix is the input
to it.

```python
# agent/tools/registry.py — generated from the matrix (CSV / YAML)
TOOLS = [
    ToolSpec(
        name="log_search",
        side_effect="read",
        identity_shape="user_delegated",
        downstream_scope=("logs:read",),
        token_audience="loki",
        hitl_tier="L0",
        two_person=False,
    ),
    ToolSpec(
        name="kubectl_action",
        side_effect="write",
        identity_shape="user_delegated",
        downstream_scope=("k8s:patch_deployments",),
        token_audience="kubernetes",
        hitl_tier="L1",
        two_person=False,
        extra_rules=[TwoPerson.when(verb_in={"cordon", "drain"})],
    ),
    # ...
]
```

The SDK registration code iterates this spec and *refuses to register*
any tool whose downstream_scope is empty or whose identity_shape is
not one of the four allowed shapes. Humans forget to fill scope
columns; the registry won't.

### 2.8 Deprovisioning

When a user is offboarded (mod-103 ex-01 SCIM):

- The user's SSO session is invalidated → delegation stops producing
  valid downstream tokens → tool calls fail.
- Any pending approvals signed by the user are invalidated (the
  approver pool from exercise-03 is refreshed from SCIM).
- Any in-flight sessions the user owned are terminated.
- Audit retains the user ID on past calls. SCIM removal must not
  anonymize historical attribution.

### 2.9 Edge cases worth writing down

- **Shared mailboxes / team accounts.** If a tool runs on behalf of a
  team mailbox, the identity is still a nameable team account with a
  known membership; no tool runs on behalf of "someone on the team"
  without naming which human initiated the session.
- **System-of-record overrides.** Some customers have "break-glass"
  roles. Agent tools never run under break-glass. The break-glass
  path is a documented manual procedure, not a tool call.
- **Elevated roles with time-boxed JIT.** If the user is in a JIT role
  at session start, carry that role into the delegation and encode
  the JIT expiration into the session timeout.
- **Cross-tenant tools.** A tool that needs to hit two tenants' data
  (merger scenarios, parent-child org structures) requires a
  separate design — do not fold into the same row.
- **Rate limits are permission.** If the downstream rate-limits the
  user at N requests/minute, the agent must share that budget, not
  stack on top of it. Otherwise an agent burns the user's budget.

## 3. Validation steps

Before the matrix is customer-ready:

1. **Every tool in exercise-02's registry has exactly one matrix row.**
   No orphan tools; no matrix row without a running tool.
2. **No tool runs under a shared SA unless the matrix explicitly
   justifies it** (e.g., `runbook_lookup` because the store has no
   per-user ACL). Default is user-delegated.
3. **For each delegation row, point at the specific IdP flow** — RFC
   8693 grant, OAuth OBO, or OIDC projection. Not "SSO somehow."
4. **Scopes use the customer's naming**, confirmed with the customer's
   IAM lead. Invented scopes are a red flag.
5. **At least one tool proves token-exchange end-to-end in staging.**
   A real user, a real IdP, a real downstream call, an audit record
   attributed to the user — not a mocked token.
6. **Deprovisioning drill.** Remove a staging user from the IdP; any
   subsequent tool call under their session fails within the IdP's
   propagation SLA. Measured, not assumed.
7. **Scope-widening attempts fail.** A test harness mutates a tool
   call to request a wider downstream scope than the user has; the
   gateway refuses, not the LLM.
8. **MFA pass-through tested.** A write that would require MFA
   manually fails when attempted via the agent without a recent MFA
   assertion.
9. **Audit joins work.** For a scripted run, query the customer's
   audit system; the agent's call, the gateway's token exchange, and
   the downstream's action are joinable by session ID and user ID.
10. **"Not exposed" list is reviewed.** The list of tools the matrix
    says are off-limits (e.g., `email.send_customer`, `kubectl
    delete`) has been reviewed with the customer and signed by their
    security lead.

## 4. Rubric / review checklist

Score 0–2 per dimension. 18/26 passes before security review.

| Dimension | 0 — Missing | 1 — Partial | 2 — Strong |
|---|---|---|---|
| Identity shape per tool | Not named | One for all | Four shapes defined; each tool labeled |
| Default is user-delegated | Shared SA by default | Mixed, unjustified | Delegated default; shared SA exceptions justified |
| Scope uses customer's naming | Invented | Loose | Exact strings the IdP uses |
| Delegation mechanism named | "SSO" | OIDC named | RFC 8693 / OBO / K8s audience specifically |
| Legacy fallback (SOAP/SFTP) | Ignored | Named | SA + user ID in header, audit joined |
| HITL tier per tool | Missing | Partial | Every row has an L0–L3 and matches ex-03 |
| Two-person rule applied | Missing | Named | Enforced on at least one irreversible write |
| "Not exposed" list | None | Named | Reviewed with security; signed |
| Deprovisioning path | Not specified | SCIM named | Drill done; expiration time measured |
| Rate-limit sharing | Agent gets its own | Named | Shared with user's budget |
| Cross-tenant isolation | Not stated | Named | Per-tenant keys/caches enforced in SDK |
| Audit attribution | Agent-only | SA + user in header | User ID primary; SA secondary where unavoidable |
| Scope-widening test | None | Discussed | Executed; gateway refuses |

## 5. Common mistakes

- **One service account to rule them all.** "The agent reads every
  ticket" is the fastest path to a leak that no one can attribute.
  Delegate by default.
- **Inventing scopes the customer does not have.** Writing
  `kb:search_everything` when the customer's IdP uses
  `confluence:space:read` means your matrix does not compile against
  their IAM and the integration slips.
- **Letting the LLM pick the audience.** The audience for token
  exchange is per-tool, per-code. The LLM must never be able to
  request a different audience in the tool call.
- **Caching exchanged tokens too long.** A 1-hour cache under a shared
  key is a scope-widening engine. Cache per-(user, audience) with a
  minute-scale TTL.
- **Forgetting the downstream's MFA.** Delegation preserves identity,
  not MFA state. If the downstream requires step-up MFA, the agent
  must step up the user, not bypass it.
- **Running tools in a process without tenant isolation.** A shared
  agent process serving two tenants with one in-memory token cache is
  a data-leak staging area.
- **Agent-attributed audit.** "Agent action at 02:14" without a user
  ID is unaccountable. The attribution must survive compliance
  review.
- **Wider-than-user scopes "for flexibility."** Agents do not need
  more than their user. If the user cannot do X, the agent cannot do
  X. Period.
- **No deprovisioning drill.** SCIM claims come to pieces when a
  departed user retains an active agent session. Drill it.
- **Treating `runbook_lookup`'s shared-SA pattern as a general
  pattern.** It is an exception because the store has no per-user
  ACL. Copying this pattern to `docs_search` leaks every document
  the agent can reach.
- **Mixing JIT and standing scopes silently.** A JIT-elevated role
  that doesn't expire with the session is a standing scope by another
  name.
- **Treating "not exposed" as implicit.** Explicit "we are not giving
  the agent `email.send_customer`" is a reviewable control; silence
  is a vulnerability.

## 6. References

- **NIST AI Risk Management Framework** — "Map" function requires an
  inventory of actors and permissions; "Measure" requires
  measurement of least-privilege adherence.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — map "insecure plugin
  design" / "excessive agency" risks onto the matrix; defense is
  delegation + narrow scopes.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — relevant tactics: credential theft via prompt
  injection, privilege escalation via over-scoped tools.
  <https://atlas.mitre.org/>
- **RFC 8693 — OAuth 2.0 Token Exchange** — the delegation spine the
  matrix references. <https://datatracker.ietf.org/doc/html/rfc8693>
- Companion exercises: exercise-01 (patterns determining tool mix),
  exercise-02 (SDK enforcement), exercise-03 (HITL tier per tool),
  exercise-05 (threat model targeting the permission surface).
- Companion module: mod-103 ex-01 (SAML/OIDC/SCIM baseline), ex-02
  (service-to-service auth and token exchange).
