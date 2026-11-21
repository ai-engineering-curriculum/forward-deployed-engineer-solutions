# exercise-02 — Service-to-service auth with OAuth, mTLS, and token exchange · Reference Solution

## 1. Solution overview

The deliverable is a **machine-to-machine (M2M) authentication design** that
covers three realistic enterprise shapes a forward-deployed engineer meets in
the field:

- **Pure OAuth 2.0 client-credentials** — simplest case, two services, one
  authorization server. Good default when both sides speak OAuth and nothing
  else constrains the design.
- **Mutual TLS (mTLS)** — either as the OAuth client-authentication method
  (`tls_client_auth` / RFC 8705), or as a standalone transport-layer identity
  when no authorization server exists (the "classic enterprise" case).
- **OAuth 2.0 Token Exchange (RFC 8693)** — on-behalf-of / impersonation
  flows: service A has a user context and needs to call downstream service B
  as that user, without re-authenticating. Also used for federated identity
  between trust domains.

The three are not alternatives; a serious deployment uses all three in
different places. The design job is choosing the right one per hop and
locking down the common failure modes (static secrets in config, long-lived
tokens, wildcard audiences).

Two framing principles:

- **Identity ≠ authorization.** Prove who you are (client certificate, mTLS,
  signed JWT client assertion) — then ask the AS for a short-lived token
  scoped to exactly what you need. Collapsing these two steps is where most
  outages and audit findings come from.
- **No long-lived bearer tokens in transit.** Target token lifetime for M2M
  is minutes, not days. Short-lived tokens shrink the blast radius of any
  leak and remove most of the "rotate the compromised secret" drill.

## 2. Implementation — three M2M shapes

### 2.1 Shape A — client-credentials, both services on the same AS

Scenario: Lantern's `indexer` service calls Lantern's `embeddings` service.
Both have identities in the same authorization server (customer's Okta
custom AS, or Lantern's own AS).

```
indexer -----(1) client_credentials grant---> AS
indexer <----(2) access_token (JWT, 10 min)-- AS
indexer -----(3) Authorization: Bearer ------> embeddings
                                               embeddings verifies JWT:
                                                 - iss  = AS issuer
                                                 - aud  = "lantern-embeddings"
                                                 - scope ⊇ "embeddings:write"
                                                 - exp, iat, jti (replay cache)
```

Request (RFC 6749 §4.4) with a JWT client assertion (RFC 7523) — **do not
use `client_secret_post`** with a shared secret for production M2M:

```
POST /oauth2/token HTTP/1.1
Host: auth.lantern.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=lantern-indexer
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJSUzI1Ni...
&scope=embeddings:write
&audience=lantern-embeddings
```

JWT client assertion claims (signed by indexer's private key):

```json
{
  "iss": "lantern-indexer",
  "sub": "lantern-indexer",
  "aud": "https://auth.lantern.example.com/oauth2/token",
  "jti": "a2d9f-...",
  "iat": 1731000000,
  "exp": 1731000120
}
```

Token lifetime: 10 minutes. No refresh tokens (there is no user). Indexer
caches the token until `exp - 60s`, then re-fetches.

### 2.2 Shape B — mTLS, no authorization server in the middle

Scenario: Lantern's gateway calls a customer-managed ERP API inside their
VPC. The ERP team doesn't operate an OAuth AS; they do mTLS.

Trust bootstrap:

- Customer issues a client certificate to Lantern from their internal CA.
- Customer pins Lantern's CN/SAN (`gateway.lantern.svc.customer`) and
  acceptable SANs.
- Customer runs OCSP or CRL; certs are 90-day with ACME-style renewal.

TLS handshake requirements (both directions):

- TLS 1.3 only; fallback to TLS 1.2 only with explicit exception.
- Server authenticates client with `Certificate` + `CertificateVerify`.
- Client validates server cert against customer's pinned root CA.
- Per-connection identity logged on both sides.

Application-layer identity:

- Gateway extracts the peer certificate subject and uses it as the
  authenticated principal — no additional bearer token on the HTTP layer.
- If the customer wants finer-grained authorization, pair mTLS with a
  scoped token in a header (shape C).

mTLS-only auth has one well-known weakness: **no scope**. Mitigate with
network segmentation (listener only reachable from the Lantern CIDR) and
server-side ACLs keyed by the cert SAN.

### 2.3 Shape C — OAuth client auth = mTLS (RFC 8705)

Scenario: Same as A, but the customer requires mTLS for every API call. Use
RFC 8705 — mTLS-bound OAuth tokens — so the access token is **sender-
constrained** and useless to anyone without the private key.

Token request:

```
POST /oauth2/token HTTP/1.1
Host: auth.customer.example.com
Content-Type: application/x-www-form-urlencoded
(TLS layer: client cert presented)

grant_type=client_credentials
&client_id=lantern-indexer
&scope=erp:read
```

AS authenticates the client by TLS certificate (`tls_client_auth` method) or
by self-signed cert (`self_signed_tls_client_auth`). The issued access token
includes the certificate thumbprint:

```json
{
  "iss": "https://auth.customer.example.com",
  "sub": "lantern-indexer",
  "aud": "erp-api",
  "scope": "erp:read",
  "exp": 1731000600,
  "cnf": { "x5t#S256": "9f8e...SHA-256 of cert..." }
}
```

ERP API verifies on each request:

1. The TLS peer cert's SHA-256 thumbprint matches the token's `cnf.x5t#S256`.
2. Standard JWT claim checks (`iss`, `aud`, `scope`, `exp`).

Key management:

- Private keys never leave Lantern's HSM / KMS. The signing operation for
  JWT client assertions and the TLS handshake happens inside the enclave.
- Rotation: 90 days, pre-staged. Both old and new keys registered at the AS
  for a 7-day overlap window.

### 2.4 Shape D — token exchange (RFC 8693) for on-behalf-of

Scenario: Lantern's web console receives an OIDC access token for an
end user (Alice). Lantern needs to call the customer's data warehouse as
Alice so row-level security applies.

```
console --(1) token exchange--> AS (customer)
   subject_token = Alice's access_token (from Okta)
   subject_token_type = urn:ietf:params:oauth:token-type:access_token
   requested_token_type = urn:ietf:params:oauth:token-type:access_token
   audience = https://warehouse.customer.example.com
   scope = warehouse:read
console <-(2) delegated access_token (acts as Alice, aud=warehouse)-- AS
console --(3) Authorization: Bearer--> warehouse
```

Token exchange request (RFC 8693 §2.1):

```
POST /oauth2/token HTTP/1.1
Host: auth.customer.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=eyJhbGciOi...  (Alice's AT)
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&requested_token_type=urn:ietf:params:oauth:token-type:access_token
&audience=https://warehouse.customer.example.com
&scope=warehouse:read
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOi...   (Lantern console client assertion)
```

Resulting token (JWT form) contains delegation metadata:

```json
{
  "iss": "https://auth.customer.example.com",
  "sub": "alice@customer.example.com",
  "aud": "https://warehouse.customer.example.com",
  "scope": "warehouse:read",
  "exp": 1731000900,
  "act": { "sub": "lantern-console" }
}
```

Important: the `act` claim records that Lantern is acting on Alice's behalf.
The warehouse logs both principals and can reject the exchange if
customer policy forbids a given actor.

### 2.5 Token lifetimes and rotation

| Token kind                          | Default TTL | Rotation                                              |
|-------------------------------------|-------------|-------------------------------------------------------|
| M2M access token (JWT)              | 10 min      | None — short TTL is the rotation                      |
| mTLS client cert                    | 90 days     | ACME renewal, 7-day overlap, auto-registered at AS    |
| JWT client assertion signing key    | 90 days     | KMS/HSM rotation, prior key kept 7 days               |
| Token-exchange delegated AT         | 5 min       | Not refreshable; re-exchange when needed              |
| SCIM/admin API tokens (see ex-01)   | 90 days     | Manual rotation, UI shows last-used                   |

### 2.6 Discovery and configuration

Every component reads config from one place. Example `auth.yaml`:

```yaml
authorization_server:
  issuer: https://auth.customer.example.com
  discovery: https://auth.customer.example.com/.well-known/openid-configuration
clients:
  indexer:
    client_id: lantern-indexer
    jwks_url: https://meta.lantern.example.com/jwks.json
    client_auth: private_key_jwt
    scopes: [embeddings:write]
    audiences: [lantern-embeddings]
    token_ttl_seconds: 600
resources:
  embeddings:
    expected_issuer: https://auth.customer.example.com
    expected_audience: lantern-embeddings
    required_scopes: [embeddings:write]
    cnf_required: false
mtls:
  trust_bundle: /etc/certs/customer-root.pem
  client_cert: /etc/certs/lantern-indexer.pem
  client_key_ref: kms://lantern/keys/indexer-tls
```

## 3. Validation steps

1. **Static-secret audit.** Grep the deploy manifest for `client_secret`,
   `api_key`, `password`. Any hit on an M2M path is a Sev-2.
2. **Token-TTL test.** Issue a token; wait 11 minutes; call the resource;
   must fail with 401.
3. **Scope negative test.** Request `scope=embeddings:write` and call a
   `queries:write` endpoint; must fail.
4. **Audience negative test.** Use a token with `aud=lantern-embeddings`
   against `lantern-search`; must fail.
5. **Replay test (JWT with jti).** Replay a valid client assertion within
   the same window; the AS must reject the second request (RFC 7523 §3).
6. **mTLS handshake test.** Connect without a client cert; TLS must fail.
   Connect with an expired cert; TLS must fail.
7. **mTLS binding test (RFC 8705).** Take a valid access token and attempt
   to use it from a TLS session that presents a *different* client cert; the
   resource must verify `cnf.x5t#S256` and reject.
8. **Token-exchange audience test.** Exchange for `warehouse`; use the
   returned token against `erp`; must fail.
9. **Delegation audit test.** Confirm the warehouse log line for an
   exchanged call records both `sub` (Alice) and `act.sub` (Lantern
   console).
10. **Key-rotation drill.** Rotate the JWT client assertion signing key
    with 7-day overlap; verify both keys accepted during the window, then
    the old key rejected after cutover.
11. **CA-outage drill.** Simulate the customer's CA rejecting an OCSP
    query; verify your mTLS clients have a documented soft-fail policy
    (and that the policy is what the customer approved).

## 4. Rubric / review checklist

Score 0–2 per row. 14/22 passes.

| Dimension                                  | 0 — Missing                                  | 1 — Partial                                        | 2 — Strong                                                                                   |
|--------------------------------------------|----------------------------------------------|----------------------------------------------------|----------------------------------------------------------------------------------------------|
| Client authentication method               | `client_secret_post` with shared secret      | `client_secret_basic`                              | `private_key_jwt` or `tls_client_auth`, no shared secrets                                    |
| Token lifetime policy                      | >1 hour M2M                                  | 30 min                                             | ≤15 min with documented rationale; no refresh for pure M2M                                   |
| Scope granularity                          | `*`                                          | Per service                                        | Per operation; minimum needed; rejected scopes tested                                        |
| Audience targeting                         | Wildcard or missing                          | Per service                                        | Per-resource audience enforced on both AS and resource                                       |
| mTLS posture                               | Opportunistic                                | Required one-way                                   | TLS 1.3, mutual, pinned CA, OCSP/CRL, rotation playbook                                      |
| Sender-constrained tokens                  | Bearer only                                  | Considered                                         | RFC 8705 cnf binding verified on each request                                                |
| Token exchange usage                       | Impersonation via shared secret              | Token exchange without `act` claim                 | RFC 8693 with `subject_token` + `act` audit trail                                            |
| Key / cert rotation                        | None                                         | Manual annual                                      | ≤90-day with pre-staged overlap, documented drill                                            |
| Config as code                             | Secrets scattered                            | One file                                           | Signed config, KMS-backed key refs, no plaintext secrets                                     |
| Observability                              | None                                         | Access logs                                        | Per-call: principal, scope, audience, `act.sub`, token id — traceable to issuing event       |
| Governance mapping                         | None                                         | Mentions standards                                 | Explicit NIST AI RMF Measure/Manage mapping for M2M surface                                  |

## 5. Common mistakes

- **Shared static client secrets.** `client_secret` strings in a deploy
  config spread across repos. Replace with `private_key_jwt` or mTLS client
  authentication; the private key lives in KMS and never leaves it.
- **Hour-long M2M tokens.** Nothing user-facing makes you do this. Ten
  minutes is plenty; shorter is fine. Long tokens exist to serve humans.
- **Wildcard audience / wildcard scope.** `aud: "*"` is a security review
  stop. So is `scope: "admin"` for routine traffic.
- **Bearer tokens on a public internet hop.** If a leaked access token
  lets an attacker call your API, you haven't limited blast radius enough.
  Add mTLS binding (RFC 8705) or move the hop inside the network.
- **Reusing JWT client assertions.** The AS must enforce `jti` replay
  protection. If your AS doesn't, patch it or move AS.
- **Token exchange without `act`.** You lose the audit chain — no record
  of which service impersonated the user. Every downstream log then shows
  "Alice" and nothing else, and incident response is blind.
- **Mutual TLS treated as authorization.** mTLS proves identity, nothing
  more. Pair it with scoped tokens for anything multi-tenant.
- **Trusting the client-side `scope` request.** A client that asks for
  `admin` must still be denied by AS policy. Scope is *requested* by the
  client and *granted* by the AS; forgetting the second half gives any
  client admin rights.
- **No overlap on cert/key rotation.** The customer's CA issues a new
  cert, you cut over, the old cert isn't trusted anymore, and a service
  running an older pod can't reach anything. Pre-stage; overlap; verify.
- **Hard-coding issuer URLs.** Discovery endpoints exist for a reason;
  reading `.well-known/openid-configuration` lets the customer change
  their IdP without a Lantern deploy.

## 6. References

- **RFC 6749** — OAuth 2.0 Authorization Framework.
  <https://www.rfc-editor.org/rfc/rfc6749>
- **RFC 7523** — JWT Profile for OAuth 2.0 Client Authentication and
  Authorization Grants. <https://www.rfc-editor.org/rfc/rfc7523>
- **RFC 7519** — JSON Web Token (JWT).
  <https://www.rfc-editor.org/rfc/rfc7519>
- **RFC 8693** — OAuth 2.0 Token Exchange.
  <https://www.rfc-editor.org/rfc/rfc8693>
- **RFC 8705** — OAuth 2.0 Mutual-TLS Client Authentication and Certificate-
  Bound Access Tokens. <https://www.rfc-editor.org/rfc/rfc8705>
- **RFC 9700** — Best Current Practice for OAuth 2.0 Security.
  <https://www.rfc-editor.org/rfc/rfc9700>
- **RFC 8446** — TLS 1.3. <https://www.rfc-editor.org/rfc/rfc8446>
- **RFC 6960** — OCSP. <https://www.rfc-editor.org/rfc/rfc6960>
- **RFC 8555** — Automatic Certificate Management Environment (ACME).
  <https://www.rfc-editor.org/rfc/rfc8555>
- **NIST AI Risk Management Framework** — Measure and Manage functions for
  machine-identity controls, rotation, and audit.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — relevant when M2M
  credentials front model inference or training surfaces.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — reference for adversary techniques against model APIs
  protected by M2M credentials. <https://atlas.mitre.org/>
