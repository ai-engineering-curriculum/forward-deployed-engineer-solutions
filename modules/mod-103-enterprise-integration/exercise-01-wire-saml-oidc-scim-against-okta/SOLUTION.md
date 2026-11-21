# exercise-01 — Wire SAML, OIDC, and SCIM against Okta · Reference Solution

## 1. Solution overview

The deliverable is a **working federation + provisioning design** for a notional
LLM app ("Lantern") that an enterprise customer runs inside their Okta tenant.
You wire three protocols that each do one job:

- **SAML 2.0** — browser SSO to the Lantern web console. The customer's Okta
  tenant is the Identity Provider (IdP); Lantern is the Service Provider (SP).
  POST-binding only; signed assertions.
- **OpenID Connect (OIDC)** — SSO for the Lantern CLI and for programmatic
  flows (device-authorization, authorization-code with PKCE). Okta is the
  OIDC Provider; Lantern is the Relying Party (RP).
- **SCIM 2.0** — just-in-time and pre-flight provisioning. Okta pushes user
  and group lifecycle events to Lantern's `/scim/v2/` endpoint so access is
  revoked within minutes of HR deactivating the account.

A good design keeps these three protocols **non-overlapping**: SAML never
becomes an API token store, OIDC tokens never grant SCIM scope, and SCIM is
never the authn path. The common FDE mistake is to use SAML for machine
accounts or OIDC `id_token`s as API bearer tokens — see §5.

Two deployment realities shape the design:

- Many customers insist on SAML for the browser even though OIDC would be
  simpler, because their SOC-2/ISO auditors graded them on SAML coverage. Fight
  this only when you must; otherwise ship SAML and layer OIDC underneath.
- SCIM is the lever that satisfies the auditor's "access revoked within N
  minutes of termination" control. Without SCIM, you're asking the customer's
  IT admin to run manual deprovisioning scripts and the pilot will stall at
  security review.

## 2. Implementation — the design

### 2.1 Roles and trust directions

```
            +------------------+        SAML POST (assertion)       +------------------+
browser --> | Okta tenant      | -------------------------------->  | Lantern web SP   |
            |  IdP             |        OIDC auth-code + PKCE       |                  |
CLI     --> |  OP              | -------------------------------->  | Lantern CLI RP   |
            |  SCIM client     |                                    |                  |
            |                  |   SCIM 2.0 (Bearer, mTLS optional) | Lantern /scim/v2 |
            |                  | -------------------------------->  |                  |
            +------------------+                                    +------------------+
                     ^                                                        |
                     |                      HR system (Workday) ---> Okta Universal Directory
                     |                                                        |
                     +--------------- deactivation event ---------------------+
```

- Okta is authoritative for identity (`sub`, `email`, `groups`).
- Lantern is authoritative for Lantern-scoped roles and tenant membership.
- The HR → Okta → Lantern chain is the deprovisioning path; test it end-to-end
  before kickoff, not at audit time.

### 2.2 SAML wiring (browser SSO)

The SP metadata Lantern publishes:

```xml
<EntityDescriptor xmlns="urn:oasis:names:tc:SAML:2.0:metadata"
                  entityID="https://app.lantern.example.com/saml/metadata">
  <SPSSODescriptor AuthnRequestsSigned="true"
                   WantAssertionsSigned="true"
                   protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
    <KeyDescriptor use="signing">
      <KeyInfo xmlns="http://www.w3.org/2000/09/xmldsig#">
        <X509Data><X509Certificate>MIIC...SP signing cert...</X509Certificate></X509Data>
      </KeyInfo>
    </KeyDescriptor>
    <SingleLogoutService
       Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
       Location="https://app.lantern.example.com/saml/slo"/>
    <NameIDFormat>urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress</NameIDFormat>
    <AssertionConsumerService index="0" isDefault="true"
       Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
       Location="https://app.lantern.example.com/saml/acs"/>
  </SPSSODescriptor>
</EntityDescriptor>
```

Okta-side configuration (SAML 2.0 app):

- Audience URI / SP Entity ID: `https://app.lantern.example.com/saml/metadata`
- Single sign-on URL: `https://app.lantern.example.com/saml/acs`
- Name ID format: `EmailAddress`; Application username: `Email`
- Response signed; assertion signed; algorithm RSA-SHA256
- Attribute statements:

| Attribute (SAML) | Okta value expression       | Lantern claim        |
|------------------|-----------------------------|----------------------|
| `email`          | `user.email`                | `email`              |
| `given_name`     | `user.firstName`            | `given_name`         |
| `family_name`    | `user.lastName`             | `family_name`        |
| `groups`         | `appUser.groups` (regex `.*`) | `groups` (array)    |
| `department`     | `user.department`           | `department`         |

Lantern verification rules (non-negotiable):

- Assertion must be signed; reject response-only signatures.
- `NotBefore` / `NotOnOrAfter` enforced with ≤30 s clock skew.
- `Audience` must equal Lantern's SP entity ID exactly.
- `InResponseTo` must match a stored, unused AuthnRequest ID; replay window
  ≤5 minutes; `Destination` must equal the ACS URL.
- Reject unsolicited assertions unless IdP-initiated is an explicit customer
  requirement (Okta ships this on by default — turn it off).

### 2.3 OIDC wiring (CLI and programmatic)

Two flows are enough for 95% of enterprise apps:

- **Web app (confidential client):** Authorization-Code + PKCE. Not SAML's
  replacement for the browser console; used for Lantern's web admin APIs
  called from a server-side session.
- **Native CLI (public client):** Authorization-Code + PKCE via the system
  browser (loopback redirect), with Refresh Tokens rotated on use. For
  headless CI, use **Device Authorization Grant** (RFC 8628).

Example OIDC discovery expectations (from Okta's well-known endpoint):

```
GET https://<tenant>.okta.com/.well-known/openid-configuration
→ authorization_endpoint, token_endpoint, jwks_uri, issuer
```

Lantern validates ID tokens per OIDC Core §3.1.3.7:

- `iss` equals the Okta issuer registered for this tenant.
- `aud` contains Lantern's client_id.
- `exp` not expired; `iat` within 5 minutes.
- `nonce` matches the one Lantern sent in the auth request.
- Signature verified against `jwks_uri` with key rotation cached ≤10 min.

Scope and claim shape Lantern requests:

```
scope = openid profile email groups offline_access
```

Lantern maps `groups` → Lantern roles via a tenant-scoped group-to-role table
stored in Lantern's DB. **Do not** hard-code Okta group names in app code;
the customer will rename them.

Access tokens vs. ID tokens — Lantern's rule:

- `id_token` is for authentication only. It never authorizes an API call.
- `access_token` is the bearer for Lantern's own APIs; introspect with
  Okta's `/introspect` endpoint or verify as a JWT (Okta issues JWT ATs for
  custom authorization servers).
- Refresh tokens are rotated on every use; prior refresh token is revoked.

### 2.4 SCIM 2.0 wiring (provisioning)

Lantern exposes RFC 7644 endpoints:

```
POST   /scim/v2/Users
GET    /scim/v2/Users/{id}
PUT    /scim/v2/Users/{id}
PATCH  /scim/v2/Users/{id}
DELETE /scim/v2/Users/{id}            (soft-delete; see below)
POST   /scim/v2/Groups
PATCH  /scim/v2/Groups/{id}
GET    /scim/v2/ResourceTypes
GET    /scim/v2/Schemas
GET    /scim/v2/ServiceProviderConfig
```

Lantern advertises `ServiceProviderConfig`:

```json
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:ServiceProviderConfig"],
  "patch":        { "supported": true },
  "bulk":         { "supported": false, "maxOperations": 0, "maxPayloadSize": 0 },
  "filter":       { "supported": true, "maxResults": 200 },
  "changePassword": { "supported": false },
  "sort":         { "supported": true },
  "etag":         { "supported": false },
  "authenticationSchemes": [
    { "type": "oauthbearertoken", "name": "OAuth Bearer Token",
      "description": "Issued by Lantern, rotated quarterly" }
  ]
}
```

Lifecycle semantics Lantern commits to:

- Okta `active: false` → Lantern soft-deactivates (session revoked within
  60 s, tokens revoked, SCIM status `active=false` persisted, user retained
  for audit).
- Okta `DELETE` → Lantern purges after a 30-day grace period; purge is
  audit-logged.
- Group membership changes propagate to Lantern role mapping within one
  SCIM poll cycle (Okta's default is ~40 min; customers can push on
  change for faster).

Bearer token:

- Lantern mints a SCIM-only bearer token on tenant setup. It's **scoped to
  `scim:provision`** and cannot be used for user-facing APIs.
- 90-day rotation; UI shows last-used timestamp; old token invalidated on
  rotation, no grace period.

### 2.5 Role mapping (the piece that always drifts)

Store the group → role mapping as data, in Lantern's control plane:

```yaml
# tenant: acme.lantern.example.com
mappings:
  - okta_group: "okta-lantern-admins"
    lantern_role: "admin"
  - okta_group: "okta-lantern-analysts"
    lantern_role: "analyst"
  - okta_group: "okta-lantern-viewers"
    lantern_role: "viewer"
default_role: "viewer"   # anyone outside the mapped groups
deny_if_no_group: false  # set true for high-regulation tenants
```

Evaluation rules:

- Applied on every assertion (SAML) or ID-token (OIDC); roles are
  re-computed per login, not cached across sessions.
- SCIM group changes update the mapping cache for already-logged-in users
  on next request; an idle session doesn't keep a stale role.

### 2.6 Logout and session revocation

- SAML Single Logout via the IdP SLO endpoint; Lantern treats `LogoutRequest`
  as authoritative and invalidates the server-side session.
- OIDC: Lantern supports back-channel logout per the OIDC Back-Channel
  Logout spec; verifies the `logout_token` signature and the `sid` claim.
- SCIM: `active: false` invalidates sessions immediately (push, not pull) —
  use it as the belt to SAML/OIDC's suspenders. The auditor cares about this.

## 3. Validation steps

End-to-end testing before customer handover:

1. **SP metadata round-trip.** Upload Lantern's SP metadata into Okta; the
   Okta admin console imports it without manual editing.
2. **SAML assertion test.** Sign in as a test user; capture the SAML
   response (SAML-tracer or equivalent). Verify:
   - `Signature` element on both response and assertion
   - `Audience` matches SP entity ID
   - `NotBefore`/`NotOnOrAfter` present, ≤5 min window
   - All five attributes land in the Lantern session
3. **Clock skew test.** Push test machine clock +40 s; the login must fail.
   Restore clock.
4. **Replay test.** Capture a valid SAML response; POST it a second time to
   `/saml/acs`. Must fail with an "InResponseTo already used" error.
5. **OIDC ID-token test.** Walk the auth-code + PKCE flow from the CLI;
   decode the ID token (`jwt.io` locally, not a web paste). Verify issuer,
   audience, nonce, exp.
6. **Refresh rotation test.** Call the refresh endpoint with a valid refresh
   token; attempt to re-use the *previous* refresh token. Must fail.
7. **SCIM push test.** In Okta, assign a test user to the Lantern app.
   Lantern receives `POST /Users`; user visible in Lantern admin UI within
   one SCIM cycle.
8. **SCIM deactivation test.** Suspend the user in Okta; Lantern receives
   `PATCH /Users/{id}` with `active: false`; existing session is killed
   within 60 s; a fresh login attempt is denied.
9. **Group change test.** Move the test user from `okta-lantern-viewers`
   to `okta-lantern-admins`; on next request Lantern shows admin role.
10. **Logout test.** Trigger IdP-initiated logout; the Lantern session
    cookie is invalidated server-side (cookie-reuse attempt fails).

Every item above produces a line in the customer's runbook under
`federation/validation-log.md`.

## 4. Rubric / review checklist

Score 0–2 per row. 14/22 is the pass bar before a customer handover.

| Dimension                                     | 0 — Missing                                  | 1 — Partial                                        | 2 — Strong                                                                                   |
|-----------------------------------------------|----------------------------------------------|----------------------------------------------------|----------------------------------------------------------------------------------------------|
| SAML assertion signature policy               | Response-only or none                        | Assertion signed, no replay protection             | Signed assertion + replay cache + ≤30 s clock skew enforced                                  |
| SAML metadata lifecycle                       | Hand-edited certs                            | Metadata exchanged, no rotation plan               | Metadata URL, scheduled rotation, override for emergency                                     |
| OIDC flows chosen                             | Implicit flow or ROPC                        | Auth-code without PKCE                             | Auth-code + PKCE for all clients; device-code for headless                                   |
| ID-token vs. access-token discipline          | Confused                                     | Mostly separate                                    | id_token never authorizes; access_token verified every request                               |
| Refresh-token rotation                        | None or long-lived static                    | Rotated but prior not revoked                      | Rotated on every use, prior revoked, reuse detected                                          |
| SCIM endpoint completeness                    | Users only                                   | Users + Groups, no ServiceProviderConfig           | Full `/Users`, `/Groups`, `/Schemas`, `/ServiceProviderConfig` + PATCH                       |
| Deprovisioning latency                        | Manual                                       | <24h                                               | ≤60 s from Okta deactivation to Lantern session revoked, measured                            |
| Role mapping as data                          | Hard-coded group names                       | Config file in repo                                | Tenant-scoped mapping in control plane, no code change to update                             |
| SCIM bearer-token scope                       | Shared with user APIs                        | Separate token, no rotation                        | Scoped `scim:provision`, 90-day rotation, last-used shown                                    |
| Logout path                                   | None (session TTL only)                      | SP-initiated only                                  | SP + IdP SLO + OIDC back-channel + SCIM active=false, all tested                             |
| Governance mapping                            | None                                         | Mentions NIST                                      | NIST AI RMF Govern/Measure functions cited for access/audit controls                         |

## 5. Common mistakes

- **Using SAML attribute mapping as a permission system.** The IdP sends
  `groups`; Lantern copies them straight into roles. When the customer
  renames a group, every Lantern tenant breaks. Keep a mapping table.
- **Treating `id_token` as a bearer.** The `id_token` proves *who*; use
  `access_token` (JWT or opaque) for *what*. Mixing them creates audit
  findings and makes revocation impossible.
- **Skipping replay protection.** SAML without `InResponseTo` tracking is
  vulnerable to captured-response replay. The spec requires it.
- **Long-lived refresh tokens without rotation.** OAuth 2.0 Security BCP
  (RFC 9700) calls out refresh-token reuse detection; absence is a Sev-2
  finding at any serious customer.
- **Pull-only SCIM.** Okta's default is 40-minute polling. For a
  deprovisioning SLA tighter than that, enable push and test the SLA.
- **SCIM token reused for user APIs.** The SCIM token is the IdP's key to
  your SP. If it can also call customer-data endpoints, a compromised
  SCIM token exfiltrates data.
- **No rollback to local accounts.** The federation breaks occasionally
  (expired cert, Okta outage). A break-glass admin — scoped, audited,
  MFA-locked, hardware-key — is required.
- **Hardcoding issuer URLs in app code.** Tenants have different Okta
  tenants, custom domains, and sometimes migrate to a different IdP mid-
  pilot. Issuer is a tenant-level config, not a constant.
- **No audit of "group → role" drift.** The customer renames a group in
  Okta; Lantern role mappings silently de-reference. Nightly job to
  reconcile mapping table against live Okta groups.
- **Forgetting SCIM `externalId`.** Without it, you can't reconcile Okta
  users with existing Lantern users during onboarding; you end up with
  duplicate accounts.

## 6. References

- **SAML 2.0 Core** (OASIS Standard) — assertion structure, replay
  protection, signature rules. <https://docs.oasis-open.org/security/saml/v2.0/>
- **SAML 2.0 Bindings** and **Profiles** (OASIS) — POST / Redirect bindings
  and the Web Browser SSO Profile used above.
  <https://docs.oasis-open.org/security/saml/v2.0/>
- **OpenID Connect Core 1.0** — ID-token validation rules (§3.1.3.7),
  scopes, nonce semantics. <https://openid.net/specs/openid-connect-core-1_0.html>
- **OIDC Back-Channel Logout 1.0** — logout_token semantics.
  <https://openid.net/specs/openid-connect-backchannel-1_0.html>
- **RFC 6749** — OAuth 2.0 Authorization Framework.
  <https://www.rfc-editor.org/rfc/rfc6749>
- **RFC 7636** — Proof Key for Code Exchange (PKCE).
  <https://www.rfc-editor.org/rfc/rfc7636>
- **RFC 8628** — OAuth 2.0 Device Authorization Grant.
  <https://www.rfc-editor.org/rfc/rfc8628>
- **RFC 9700** — Best Current Practice for OAuth 2.0 Security.
  <https://www.rfc-editor.org/rfc/rfc9700>
- **RFC 7643** — SCIM 2.0 Core Schema.
  <https://www.rfc-editor.org/rfc/rfc7643>
- **RFC 7644** — SCIM 2.0 Protocol.
  <https://www.rfc-editor.org/rfc/rfc7644>
- **NIST AI Risk Management Framework** — Govern and Measure functions
  used above for access review, audit, and session-revocation SLAs.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — relevant when federation
  controls protect the training or inference surface.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — reference when evaluating identity-boundary attacks on
  model-serving endpoints. <https://atlas.mitre.org/>
