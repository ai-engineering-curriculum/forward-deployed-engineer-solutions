# exercise-02 — Deploy a Model Gateway Inside a Customer VPC · Reference Solution

## 1. Solution overview

The deliverable is a **reference design + runnable artifacts** for a model
gateway that lives *inside the customer's VPC* and brokers every call the
Lantern app makes to an LLM provider. The gateway is the one place where
the hard enterprise concerns consolidate:

- **Egress control** — only the gateway is allowed to reach a provider
  endpoint; app pods have no outbound internet route.
- **Credentials** — the provider API keys live in the gateway's secrets
  tier (KMS/HSM-backed), not in app code or env files.
- **Policy** — rate-limit per tenant, per model, per user; redact
  known-sensitive fields before egress; enforce output policy.
- **Observability** — token counts, latency, cost, cache hits per tenant;
  traces correlated with the app's request ID.
- **Fallback and routing** — primary provider, secondary provider, and the
  "degrade gracefully" response are a config toggle, not a code change.
- **Multi-provider interface** — the app talks one API shape to the
  gateway, not N shapes to N providers.

A good gateway is *boring from the app's perspective*: one SDK, one base
URL, one auth header. All the mess (keys, regions, retries, redaction,
routing, caching, cost) stays behind the gateway. The common FDE mistake
is to let app services call provider SDKs directly, then try to bolt
policy on in N places later — this never converges.

The gateway sits in the customer VPC. The provider traffic leaves via a
**single, allow-listed egress point** (NAT gateway or forward proxy),
with PrivateLink / VPC endpoints used where the provider supports them.
Nothing in the gateway requires the customer to open inbound internet.

## 2. Implementation

### 2.1 Reference architecture

```
+---------------------------------- customer VPC ---------------------------------+
|                                                                                 |
|   +------------+   mTLS    +----------------+   mTLS/HTTPS   +------------+     |
|   | Lantern    | --------> | Model Gateway  | -------------> | Egress NAT |---->| Provider A
|   | app pods   |           | (stateless,    |                | or         |     | (PrivateLink
|   | (K8s)      |           |  HPA'd, N=3+)  |                | PrivateLink|     |  if offered)
|   +------------+           +----------------+                +------------+---->| Provider B
|        |                        |      ^                                        |
|        |                        v      |                                        |
|        |                   +---------+ +--------+                                |
|        |                   | Redis   | | Secrets|  (KMS/HSM-backed)              |
|        |                   | (cache, | | mgr    |                                |
|        |                   |  rate-  | +--------+                                |
|        |                   |  limit, |                                           |
|        |                   |  idem.) |                                           |
|        |                   +---------+                                           |
|        v                                                                         |
|   +-------------+        +--------------+                                        |
|   | OTel col.   | -----> | Observability|  (customer-owned sink)                 |
|   +-------------+        +--------------+                                        |
|                                                                                 |
+---------------------------------------------------------------------------------+
```

- The gateway is stateless; horizontal scale is the only scaling strategy.
- Redis holds short-lived state: rate-limit counters, idempotency keys, and
  response cache entries (see exercise-03). Redis is in-VPC; Elasticache /
  Memorystore / managed Redis is fine if the customer already uses it.
- Secrets live in the customer's secrets manager (AWS Secrets Manager,
  Azure Key Vault, GCP Secret Manager, or Vault). The gateway mounts them
  at runtime via workload identity. No `.env` files.
- OTel exports traces and metrics to the **customer's** observability
  stack. Nothing goes to vendor dashboards.

### 2.2 Public API shape (the one the app sees)

```
POST /v1/chat.completions
Headers:
  Authorization: Bearer <lantern-app-jwt>      # from IdP (see mod-103 ex-01)
  X-Lantern-Tenant: <tenant-id>                # tenant scope
  X-Lantern-Request-Id: <uuid>                 # app-side request correlation
  X-Lantern-Idempotency-Key: <optional>        # for safe retries

Body:
{
  "model": "chat-standard",                    // logical name, not provider model
  "messages": [...],
  "response_format": {"type":"json_schema", "schema":{...}},
  "max_tokens": 1024,
  "temperature": 0.2,
  "cache": {"enabled": true, "ttl_s": 900},    // optional per-call override
  "fallback_policy": "degrade_ok"              // or "strict"
}

200:
{
  "id": "gw_...",
  "model": "anthropic/claude-3-5-sonnet@2025-10",  // concrete model actually used
  "choices": [...],
  "usage": {"input_tokens": 523, "output_tokens": 180, "cache_hits": 0},
  "cost": {"amount_usd": 0.0071, "tenant": "acme"},
  "route": {"provider": "primary", "attempt": 1}
}
```

Rules:

- The app never specifies a provider-native model name. The gateway maps
  logical names (`chat-standard`, `chat-fast`, `chat-reasoning`, `embed-small`)
  to a concrete model per tenant.
- The gateway owns retry policy; the app treats a non-2xx as final unless it
  passes an idempotency key for safe retry.
- Streaming is supported via SSE on the same endpoint with
  `Accept: text/event-stream`.

### 2.3 Routing and fallback policy

The routing table is tenant-scoped config, loaded from the control plane:

```yaml
# tenant: acme
models:
  chat-standard:
    primary:
      provider: anthropic
      model: claude-3-5-sonnet
      region: us-east-1
      zero_retention: true
    secondary:
      provider: openai
      model: gpt-4o-mini   # concrete model names come from provider docs
      region: us-east-1
      zero_retention: true
    fallback_policy:
      on_5xx: secondary
      on_rate_limit: secondary
      on_content_filter: strict         # bubble up; do not re-ask
      on_hard_timeout_ms: 20000
    degrade:
      response_when_both_fail:
        role: assistant
        content: "This feature is temporarily unavailable; please retry."
  embed-small:
    primary:
      provider: voyage
      model: voyage-3
      region: us-east-1
rate_limits:
  per_tenant_rpm: 600
  per_user_rpm:   60
  per_model_tpm:  200000      # tokens per minute
quotas:
  daily_usd_soft: 500
  daily_usd_hard: 1000
```

Rules:

- Rate-limiting is enforced at the gateway using a token-bucket in Redis,
  not in the provider SDK. Primary + secondary share the same per-tenant
  bucket so a fallback does not silently double the spend.
- Quotas publish a header (`X-Lantern-Quota-Used`) so the app can show the
  customer their burn rate.
- `on_content_filter` does *not* fall back to a different provider by
  default — a refused output on one provider usually means the input is
  the problem, not the model.

### 2.4 Security posture

- **Transport:** TLS 1.3 everywhere; mTLS between app pods and the
  gateway; TLS to provider endpoints. No cleartext.
- **Auth to the gateway:** short-lived JWT issued by the customer IdP
  (mod-103 ex-01). Audience-restricted to the gateway. The tenant ID in
  the token must match the `X-Lantern-Tenant` header, which must match
  the routing-table tenant — three-way check or reject.
- **Auth from the gateway to providers:** provider API keys loaded from
  the customer's secrets manager at process start, refreshed via workload
  identity. Keys are *provider-scoped*, not Lantern-scoped — rotation is
  independent per provider.
- **Egress:** only the gateway's egress identity can reach provider CIDRs.
  Enforced in the VPC with route tables, security groups, and (where
  available) PrivateLink/VPC endpoints to the provider. App pods have
  **no** outbound internet route.
- **Redaction:** before egress, the gateway runs a pluggable redactor over
  messages for known-sensitive patterns (PII per customer's data
  classification, PHI if HIPAA-scoped, secrets heuristics). Redaction is
  recorded in the trace, not silent.
- **Prompt-injection posture:** per OWASP LLM Top 10 and MITRE ATLAS, the
  gateway enforces a system-prompt envelope the app cannot override, and
  marks tool-calling fields as separate from user content so downstream
  consumers can honor the boundary.
- **Audit:** every request writes a structured audit event (tenant, user,
  model, logical model, provider, token counts, cost, redaction flags,
  outcome). Retained per the customer's logging retention policy.

### 2.5 Deployment artifact (Kubernetes)

```yaml
# k8s/gateway.yaml — illustrative
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lantern-gateway
  namespace: lantern
spec:
  replicas: 3
  selector: { matchLabels: { app: lantern-gateway } }
  template:
    metadata: { labels: { app: lantern-gateway } }
    spec:
      serviceAccountName: lantern-gateway-wi   # workload identity to secrets mgr
      automountServiceAccountToken: true
      containers:
        - name: gateway
          image: registry.customer.example/lantern/gateway:1.4.2   # pinned; no :latest
          ports: [{ containerPort: 8443, name: https }]
          env:
            - name: LANTERN_TENANT_CONFIG_URL
              value: "https://control.lantern.svc/tenants"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.observability.svc:4317"
          readinessProbe:
            httpGet: { path: /healthz/ready, port: 8443, scheme: HTTPS }
          livenessProbe:
            httpGet: { path: /healthz/live,  port: 8443, scheme: HTTPS }
          resources:
            requests: { cpu: "500m", memory: "1Gi" }
            limits:   { cpu: "2",    memory: "4Gi" }
          securityContext:
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: { drop: ["ALL"] }
---
apiVersion: v1
kind: Service
metadata:
  name: lantern-gateway
  namespace: lantern
spec:
  type: ClusterIP
  selector: { app: lantern-gateway }
  ports: [{ port: 443, targetPort: 8443, name: https }]
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: lantern-gateway, namespace: lantern }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: lantern-gateway
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 60 }
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: gateway-egress-only, namespace: lantern }
spec:
  podSelector: { matchLabels: { app: lantern-gateway } }
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector: { matchLabels: { tier: app } }   # only app tier may call gw
      ports: [{ protocol: TCP, port: 8443 }]
  egress:
    - to:
        - namespaceSelector: { matchLabels: { name: observability } }
      ports: [{ protocol: TCP, port: 4317 }]
    - to:
        - ipBlock: { cidr: 10.0.0.0/8 }   # redis, secrets mgr proxy (adjust)
    - to:
        - namespaceSelector: {}
          podSelector: { matchLabels: { app: egress-proxy } }
      ports: [{ protocol: TCP, port: 3128 }]
```

Notes:

- Gateway egresses to providers through a *named* egress proxy pod set;
  the NetworkPolicy prevents it from reaching arbitrary CIDRs directly.
- `readOnlyRootFilesystem` + non-root + dropped caps = a baseline that
  survives a customer security review without extra conversation.
- Rolling updates use a surge of 1 and maxUnavailable 0 so a bad image
  does not drop capacity under load.

### 2.6 Minimal gateway handler sketch (Python; illustrative)

```python
# gateway/handler.py — shape only
from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel
from typing import Any

from auth import verify_jwt
from tenants import load_tenant_config
from redaction import redact_in_place
from providers import call_primary_or_secondary
from cache import try_cache, write_cache
from limits import check_and_consume
from audit import emit_audit
from tracing import start_span

app = FastAPI()

class ChatRequest(BaseModel):
    model: str
    messages: list[dict[str, Any]]
    response_format: dict | None = None
    max_tokens: int = 1024
    temperature: float = 0.2
    cache: dict | None = None
    fallback_policy: str | None = None

@app.post("/v1/chat.completions")
async def chat(req: ChatRequest,
               authorization: str = Header(...),
               x_lantern_tenant: str = Header(...),
               x_lantern_request_id: str = Header(...),
               x_lantern_idempotency_key: str | None = Header(default=None)):
    principal = verify_jwt(authorization, audience="lantern-gateway",
                           expected_tenant=x_lantern_tenant)
    tenant_cfg = load_tenant_config(x_lantern_tenant)
    route = tenant_cfg.models.get(req.model)
    if route is None:
        raise HTTPException(400, f"unknown logical model: {req.model}")

    with start_span("gateway.chat", attrs={"tenant": principal.tenant,
                                           "logical_model": req.model,
                                           "request_id": x_lantern_request_id}):
        check_and_consume(principal, req.model, tenant_cfg.rate_limits)
        redact_in_place(req.messages, policy=tenant_cfg.redaction)

        cached = try_cache(principal.tenant, req, x_lantern_idempotency_key)
        if cached is not None:
            emit_audit(principal, req, cached, cache_hit=True)
            return cached

        response, meta = await call_primary_or_secondary(req, route, principal)
        write_cache(principal.tenant, req, response, meta, req.cache)
        emit_audit(principal, req, response, meta=meta)
        return response
```

Shape only. The real repo wires each dependency with tests.

## 3. Validation steps

1. **Egress boundary test.** Exec into an app pod and curl any provider
   endpoint. Expected result: connection refused by NetworkPolicy. Exec
   into a gateway pod: the same curl succeeds (via egress proxy).
2. **mTLS test.** Call the gateway from a client without the client cert.
   Expected result: TLS handshake failure.
3. **IdP + tenant three-way check.** Craft a JWT for tenant X and send a
   request with `X-Lantern-Tenant: Y`. Expected: 401.
4. **Rate-limit test.** Burst traffic past `per_tenant_rpm`; 10% of
   requests return 429 with `Retry-After`. Primary and secondary share the
   bucket — confirm the counter is not reset by falling back.
5. **Fallback test.** Inject a 5xx on the primary. Confirm the secondary
   handles the traffic within `on_hard_timeout_ms`, and the response
   `route.provider == "secondary"`.
6. **Degrade test.** Make both providers fail. Expected: the
   `degrade.response_when_both_fail` payload is returned with a 200 and a
   `route.provider == "degraded"` marker.
7. **Cost accounting test.** Send 100 requests of known size; sum the
   `usage.input_tokens + usage.output_tokens` against the provider's
   usage dashboard; difference < 1%.
8. **Redaction test.** POST a message containing a synthetic SSN; confirm
   the audit event records `redaction_applied: true` and the egress
   payload captured at the egress proxy does not contain the SSN.
9. **Rolling update test.** Push a new gateway image; confirm zero 5xx
   during the roll by driving load through the service endpoint.
10. **Observability test.** Query the customer-owned trace backend for a
    known `X-Lantern-Request-Id`; confirm the end-to-end trace includes
    the gateway span with tenant, model, provider, token counts, cost.
11. **Secrets rotation test.** Rotate the provider API key in the secrets
    manager; confirm the gateway picks up the new key within one refresh
    cycle without pod restart.
12. **Residency/PrivateLink test (where applicable).** Capture the gateway
    egress destination; confirm it resolves to the provider's in-region
    endpoint (or PrivateLink interface), not a cross-region or public IP.

## 4. Rubric / review checklist

Score 0–2 per row. 20/30 passes before customer handover.

| Dimension | 0 — Missing | 1 — Partial | 2 — Strong |
|---|---|---|---|
| Deployment location | Vendor cloud | Customer cloud but shared VPC | Customer VPC, dedicated namespace, NetworkPolicy enforced |
| Egress control | App can hit provider directly | Gateway and app both can | Only gateway egresses to providers; proven by test |
| Transport security | HTTP or TLS only app→gw | TLS app→gw, TLS gw→provider | TLS 1.3 everywhere + mTLS app→gw |
| IdP integration | Shared API key | JWT without tenant scoping | JWT with audience + tenant three-way check |
| Secrets handling | .env | Secrets manager without workload identity | Secrets manager + workload identity + rotation tested |
| Logical model abstraction | App names provider model | Partial map | Logical names only; tenant-scoped mapping |
| Rate limits | None | Per-tenant only | Per-tenant + per-user + per-model-TPM; bucket shared with fallback |
| Fallback routing | None | Secondary configured but untested | Secondary + degrade response + hard timeout; tested |
| Redaction | None | Regex-only, no audit | Policy-driven, logged, verified in egress capture |
| Observability | Logs only | Traces without tenant attributes | OTel with tenant/model/cost on every span, sinked customer-side |
| Cost accounting | Absent | Token counts only | Token counts + USD per tenant per model; <1% drift vs. provider |
| Audit log | Missing | Freeform logs | Structured per-request audit with required fields |
| Rollout safety | Single replica | Multi-replica, no HPA | HPA + rolling update surge 1 / unavailable 0, roll tested |
| Resource hygiene | Root container | Non-root but writable FS | Non-root + read-only FS + dropped caps |
| PrivateLink/VPC endpoint | Public internet egress | NAT only | PrivateLink/VPC endpoint where provider supports |
| Prompt-injection posture | Not considered | Mentioned | OWASP LLM Top 10 and ATLAS mitigations named and tested in gateway |

## 5. Common mistakes

- **Letting the app import provider SDKs.** Any direct SDK call from app
  code defeats the gateway. Treat the gateway as the only client. If a
  team "needs" a direct call, their need is really a missing gateway
  feature.
- **Baking provider names into app code.** Logical model names decouple
  app releases from provider changes. Hardcoded names mean a provider
  swap becomes an app deployment.
- **Shared rate-limit bucket not shared with fallback.** A fallback that
  doubles the bucket silently doubles the spend during an incident. The
  bucket must be per-tenant-per-logical-model, not per-provider.
- **Content-filter fallback.** Falling back to a second provider when the
  first refuses content hides policy from the audit trail. Bubble up and
  let the application decide, with the refusal recorded.
- **Keys in env vars.** `.env` files leak via dumps, images, and
  snapshots. Use the customer's secrets manager with workload identity.
- **Egress through the app's default NAT.** Co-tenants and other
  workloads will pollute the egress identity; the gateway needs a
  dedicated egress identity for provider allow-listing and audit.
- **No per-tenant cost attribution.** "Total spend was $X" cannot become
  invoice line items. Tag tokens by tenant on every call; mod-110 uses
  the same tag downstream.
- **One replica, no HPA.** The gateway is on the request path; a single
  replica turns every deploy into an outage. Three minimum, HPA tuned
  on CPU or request latency.
- **No PrivateLink / VPC endpoint where available.** If the provider
  offers a private interface in the customer's cloud, use it. Public
  egress is harder to defend in security review even if the data is
  identical.
- **Silent redaction.** A gateway that strips SSNs without marking it in
  the audit event hides a change to the model input. Redaction is a
  decision; record it.
- **No degrade response.** When both providers fail, the app must get
  *something*. A clean "unavailable" response is better than a 502 the
  app then treats as a crash.
- **Observability going to vendor dashboards.** Customer-paid traces to
  someone else's console is a data-governance finding. Export to the
  customer's stack only.

## 6. References

- **NIST AI Risk Management Framework** — Govern/Manage functions apply
  to the gateway as the operational control point for provider access,
  key rotation, fallback, and audit. <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — threats the gateway's
  redaction, output-policy, and audit paths mitigate (data leakage,
  model manipulation, supply chain). <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — adversarial tactics (prompt injection, model evasion,
  exfiltration via LLM output) that the gateway's system-prompt envelope
  and output filtering are designed to blunt. <https://atlas.mitre.org/>
- **RFC 8693** — OAuth 2.0 Token Exchange. Used when the gateway brokers
  a delegated identity down to a provider or downstream API.
  <https://www.rfc-editor.org/rfc/rfc8693>
- **RFC 9700** — OAuth 2.0 Security Best Current Practice. The gateway's
  token handling and refresh posture should conform.
  <https://www.rfc-editor.org/rfc/rfc9700>
- Kubernetes documentation for NetworkPolicy, HPA, and Pod Security
  Standards — baseline primitives used in the artifact. Cite the
  cluster version your customer runs. <https://kubernetes.io/docs/>
