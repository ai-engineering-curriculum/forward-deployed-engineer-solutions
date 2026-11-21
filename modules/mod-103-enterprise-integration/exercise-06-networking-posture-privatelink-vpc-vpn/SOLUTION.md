# exercise-06 — Networking posture: PrivateLink, VPC, VPN · Reference Solution

## 1. Solution overview

The deliverable is a **network architecture document** for a Lantern
deployment inside a regulated customer's cloud footprint. The design
answers four questions security review will ask on day one:

1. **Where does data flow, who sees it, and does any of it traverse
   the public internet?**
2. **How is Lantern reachable *to* the customer?** (Private endpoint?
   Public with allow-list? Site-to-site?)
3. **How does Lantern reach *out* to customer-owned services?** (Egress
   through the customer VPC? Through a shared services account?
   Through the internet with a managed proxy?)
4. **What is the blast radius if a Lantern component is compromised?**
   (Network segmentation, lateral-movement controls, DNS boundary.)

Three building blocks cover most enterprise shapes:

- **PrivateLink / Private Service Connect / Private Endpoints** — the
  default for "customer talks to Lantern's SaaS but data must stay off
  the public internet." Lantern exposes a service via a cloud provider
  private endpoint; the customer consumes it from inside their VPC.
- **VPC with peering / Transit Gateway** — for "Lantern deploys inside
  the customer's cloud account or inside the customer's VPC," with
  routing to other customer workloads.
- **VPN (site-to-site IPsec)** — for on-prem customers without a cloud
  footprint or when regulatory controls forbid cloud-to-cloud private
  peering. Also used as a bootstrap path before PrivateLink lands.

The common FDE mistake is to pick a pattern by aesthetic ("we always
PrivateLink") rather than by the data flows and the customer's existing
hub-and-spoke model. The right pattern is whichever one slots into the
customer's **landing zone** without making it a one-off.

## 2. Implementation — three canonical topologies

### 2.1 Topology A — Lantern SaaS exposed via PrivateLink (customer is consumer)

Scenario: Lantern runs its service plane in `lantern-prod` on AWS.
Customer "ACME" wants their internal services to call Lantern without
the traffic leaving the AWS backbone.

```
+------------------------------------+       +---------------------------------+
|         Lantern AWS account        |       |          ACME AWS account       |
|                                    |       |                                 |
|   NLB  ──►  ECS/EKS Lantern API    |       |   App services ──► VPC endpoint |
|   │                                |  PL   |                         (eni)   |
|   └── PrivateLink service endpoint | <──── |                                 |
|                                    |       |   (dns: api.acme.privatelink)   |
+------------------------------------+       +---------------------------------+
```

Key design decisions:

- **Endpoint service** (Lantern side): associated with an NLB fronting
  the Lantern API; requires-acceptance set to true so Lantern can
  gate who consumes it.
- **Consumer-side VPC endpoint** (ACME side): an interface endpoint in
  ACME's VPC; private IP from ACME's CIDR; traffic stays on the AWS
  backbone.
- **DNS**: ACME points `api.lantern.acme.internal` at the VPC
  endpoint's private DNS name; no public DNS lookup from ACME
  workloads.
- **Mutual authentication**: PrivateLink gives you network reachability,
  not authentication. Layer mTLS or OAuth per ex-02 on top.
- **Access control at the service**: PrivateLink doesn't give granular
  authz; the Lantern API enforces per-tenant scoping of every call.

Equivalent patterns on other clouds:

- **Azure**: Private Link + Private Endpoint. Lantern publishes a
  Private Link Service tied to the Standard Load Balancer; customer
  creates a Private Endpoint. Approval-gated.
- **GCP**: Private Service Connect (PSC) with a published service
  attachment; customer creates a PSC endpoint.

When PrivateLink is the right call:

- The customer's security policy forbids outbound internet from
  workload VPCs.
- The customer has a landing zone with dedicated private-endpoint
  subnets and a documented naming convention.
- The customer's compliance framework (SOC 2, ISO 27001, HITRUST) has
  a control for "private connectivity to vendors."

When PrivateLink is **not** the right call:

- The customer is on-prem with no cloud footprint.
- Lantern's traffic is bidirectional and the customer is also a
  producer the Lantern side needs to consume from — PrivateLink is
  one-way per endpoint; you may need a pair, or shift to peering.
- The customer's team doesn't operate VPC endpoint DNS; the setup
  will fail at DNS time, not network time.

### 2.2 Topology B — Lantern deployed inside the customer's VPC

Scenario: Regulatory constraints (HIPAA, GLBA, a government agency)
require Lantern's compute to run inside the customer's AWS account.
Lantern deploys a workload into `acme-ai-prod` with a tightly scoped
IAM role and no outbound internet.

```
acme-ai-prod VPC (/22)
├── subnets: 10.42.0.0/23  app        (private)
├── subnets: 10.42.2.0/24  data       (private, no IGW route)
├── subnets: 10.42.3.0/24  egress     (private, routes to central NAT in shared services)
└── VPC endpoints for AWS services (STS, KMS, ECR, S3 gateway)

Transit Gateway attaches:
  - acme-ai-prod VPC
  - acme-shared-services VPC (egress, logging, perimeter)
  - acme-data VPC (customer-owned data stores)
```

Design decisions:

- **No IGW, no default route to 0.0.0.0/0** in the Lantern VPC.
- **Egress through a central NAT** or a managed egress proxy in
  shared services, with URL-level allow-list for model APIs Lantern
  must reach (Anthropic, OpenAI, Lantern control plane).
- **VPC endpoints for AWS services**: Lantern uses S3 gateway endpoint
  and interface endpoints for STS, KMS, ECR, Secrets Manager — no
  AWS API calls hit the public internet.
- **Security groups**: default deny; explicit rules per listener.
  App subnet can reach the data subnet on specific ports only.
- **IAM boundary**: Lantern's role carries a **permissions boundary**
  policy that caps what Lantern can do even if its task roles ask for
  more. Customer owns the boundary.
- **KMS**: customer-managed KMS keys (CMKs) encrypt Lantern-owned
  storage. Grants are tenant-scoped.
- **Logging**: VPC Flow Logs → central logging account; CloudTrail
  data events on S3 buckets Lantern touches; Lantern application
  logs → central Splunk/Datadog bucket.

Equivalent patterns:

- **Azure**: Lantern deploys into a customer-owned spoke VNet in
  their hub-and-spoke. Egress through Azure Firewall in the hub;
  Private Endpoints for Azure services; Entra workload identity.
- **GCP**: Shared VPC; Lantern project is a service project attached
  to the customer's host project. Egress through Secure Web Proxy or
  Cloud NAT with PSC for Google APIs.

When this is the right call:

- Data never leaves the customer environment; this is a hard legal
  constraint for some industries.
- The customer has an opinionated landing zone and requires
  tenant-scoped deployments inside it.
- Lantern's architecture already supports a **single-tenant deploy
  mode** with the control plane kept separate (see ex-07 and
  mod-104).

When this is **not** the right call:

- Customer doesn't operate a landing zone with deployment automation;
  you'll be the one running the account.
- The feature set depends on Lantern-side services that haven't been
  packaged for single-tenant deploy.
- The commercial model doesn't support the extra operational
  overhead.

### 2.3 Topology C — VPN (site-to-site IPsec) to an on-prem customer

Scenario: ACME's data center isn't in a cloud. They have a traditional
perimeter network with an IPsec termination point. Lantern's VPC
attaches via a managed VPN to the customer's edge router.

```
+-------------------+        IPsec IKEv2 (two tunnels, BGP)      +---------------------+
| Lantern VPC       | -------------------------------------->   | ACME edge firewall   |
|   Virtual Private |                                           | (IPsec terminator)   |
|   Gateway or TGW  |                                           |                      |
+-------------------+                                           +---------------------+
                                                                        │
                                                                        ▼
                                                                 ACME core network
                                                                 (10.0.0.0/8)
```

Design decisions:

- **IKEv2 + IPsec** (RFC 7296). AES-GCM, SHA-256, DH group 19/20 or
  stronger. PSK rotated via a scheduled change; or certificate-based
  peers where the customer's firewall supports it.
- **Two tunnels** (active/active or active/standby) terminating on
  distinct Lantern public endpoints; BGP on both for route
  propagation and failover.
- **IP overlap is the usual pain.** ACME uses RFC 1918 ranges overlapping
  Lantern's defaults. Plan the Lantern VPC CIDR up front
  (`172.31.192.0/20` is often free when `10.0.0.0/8` is customer-owned).
- **Routing table discipline**: Lantern VPCs route only the specific
  ACME CIDRs they need; no `0.0.0.0/0` through the tunnel.
- **Firewall rules on both sides** restrict ports/protocols; VPN
  isn't authorization.
- **Change windows**: customer firewall changes have a change calendar;
  tunnel re-keying is choreographed.

When VPN is the right call:

- Customer is on-prem; no cloud footprint.
- The integration is a bootstrap; PrivateLink or Direct Connect /
  ExpressRoute will land later.
- Regulatory controls accept IPsec over the internet but forbid
  public API endpoints.

When VPN is **not** the right call:

- The customer already has Direct Connect / ExpressRoute / Cloud
  Interconnect — use those; VPN is more operational overhead for
  less bandwidth.
- Lantern's traffic needs < 50 ms RTT and the customer is far from
  Lantern's cloud regions; VPN latency jitters more than private
  connectivity.

### 2.4 Direct Connect / ExpressRoute / Cloud Interconnect

For high-volume or regulated workloads, dedicated circuits are the
long-term answer. Lantern's job is to present the service over
PrivateLink/PSC/Private Endpoint; the customer's network team runs
the Direct Connect / ExpressRoute / Cloud Interconnect gateway and
attaches it to the appropriate VPC/VNet/VPC.

### 2.5 DNS

- **Split-horizon DNS** is the norm. Customer resolves
  `api.lantern.acme.internal` to the private endpoint IP; public
  resolution of the same name returns NXDOMAIN or a customer-chosen
  public endpoint.
- **Lantern's Route 53 / Azure DNS / Cloud DNS entries** for the
  endpoint service should include TTLs you can lower before
  announced changes.
- **No public DNS leakage of internal names** — the customer's
  perimeter should block external DNS for internal domains.

### 2.6 Observability

- **VPC Flow Logs → central logging account** at the customer side,
  with retention aligned to the compliance framework (e.g., ≥90 days,
  often 365).
- **CloudTrail/activity logs** for every control-plane API Lantern
  invokes against customer AWS/Azure/GCP.
- **Tunnel / endpoint health metrics**: PrivateLink endpoint accepts,
  rejects; VPN tunnel up-time; BGP neighbor flaps.
- **Alerting**:
  - PrivateLink endpoint request rejections > 0 for > 5 min
  - VPN tunnel down > 60 s
  - BGP flap count > 2 in 10 min
  - Unexpected destination IPs in Flow Logs (data-exfil signal)

### 2.7 Governance mapping

Tie the design to NIST AI RMF:

- **Map**: identify where data resides and flows (§2.1–2.4 of the
  design doc).
- **Measure**: tunnel health, endpoint rejection rate, flow log
  anomalies (§2.6).
- **Manage**: kill-switch runbooks — customer disables endpoint /
  pulls VPN / revokes IAM boundary — without vendor involvement
  (§3, §4).

For OWASP ML Top 10 and MITRE ATLAS, call out network controls that
specifically protect model-serving surfaces (TLS termination,
intrusion controls) in the security review appendix.

## 3. Validation steps

1. **Private-path test.** From an ACME VPC instance, resolve
   `api.lantern.acme.internal`; the IP must be a private address in
   ACME's VPC endpoint CIDR.
2. **Public-path denial.** From the same ACME VPC, confirm the
   service is **not** reachable at any public DNS name (NXDOMAIN or
   explicit block).
3. **No-internet-egress test (topology B).** From a Lantern pod in
   the customer VPC, attempt `curl https://example.com`; must fail.
   Attempt an allow-listed destination; must succeed.
4. **VPC Flow Log correlation.** Make a known request; find it in
   Flow Logs within the agreed latency; verify src/dst/port/bytes
   align with the application log.
5. **VPN failover.** Bring down tunnel 1; verify tunnel 2 absorbs
   within the BGP hold time; no application visible impact.
6. **CIDR overlap check.** Diff Lantern's VPC CIDR against the
   customer's RFC 1918 announcements; confirm no overlap.
7. **PrivateLink endpoint authorization.** Attempt consumption from
   an unapproved account; must be rejected at the endpoint
   acceptance stage.
8. **mTLS/OAuth layering.** With network reachability established,
   confirm the application auth from ex-01/ex-02 still gates calls.
   PrivateLink alone must not be sufficient to call the API.
9. **Kill-switch drill.** Customer revokes the PrivateLink endpoint
   acceptance; verify Lantern pods return a clear error; verify no
   retries hammer the customer endpoint.
10. **DNS leak test.** Capture Lantern's internal DNS queries;
    verify none of them hit the customer's public DNS.

## 4. Rubric / review checklist

Score 0–2 per row. 14/22 passes.

| Dimension                              | 0 — Missing                              | 1 — Partial                                        | 2 — Strong                                                                 |
|----------------------------------------|------------------------------------------|----------------------------------------------------|----------------------------------------------------------------------------|
| Topology selection with rationale      | "PrivateLink because always"             | One pattern, some rationale                        | Pattern justified against data flow, customer landing zone, and failure story |
| Data flow diagram                      | None                                     | Boxes and arrows                                   | Diagram + per-arrow labels (protocol, direction, data classification)        |
| Private connectivity                   | Public endpoint                          | PrivateLink / VPN exists                           | PrivateLink/PSC/VPN + private DNS + endpoint acceptance control               |
| Egress controls (topology B)           | Default route to IGW                     | NAT in place                                       | Allow-listed egress proxy, VPC endpoints for cloud services, no 0/0           |
| IAM / role boundary                    | Broad roles                              | Scoped roles                                       | Permissions boundary owned by customer, task roles scoped, STS-only           |
| Encryption in transit and at rest      | Transit only                             | Transit + CMK                                      | TLS 1.3 + customer-managed KMS + grants per tenant                            |
| Logging / Flow Logs                    | None                                     | VPC Flow Logs enabled                              | Flow Logs + CloudTrail/activity + retention aligned to compliance             |
| DNS plan                               | Public DNS only                          | Private DNS exists                                 | Split-horizon, TTL controlled, no internal-name leakage                       |
| Failover / resilience                  | Single endpoint, single tunnel           | Dual tunnels                                       | Dual tunnels + BGP + tested failover + documented RTO/RPO                     |
| Kill-switch / revocation               | None                                     | Vendor-controlled                                  | Customer-executable without vendor; latency specified                         |
| Governance mapping                     | None                                     | Mentions NIST                                      | Explicit NIST AI RMF Map/Measure/Manage mapping; OWASP/ATLAS cross-ref        |

## 5. Common mistakes

- **Treating PrivateLink as authentication.** PrivateLink is network
  reachability. You still need mTLS, OAuth, or signed requests at the
  application layer.
- **Default routes to the internet from the workload VPC.** Even with
  a NAT, a default route makes auditing egress hard. Explicit
  destinations, through a managed egress proxy.
- **Overlapping CIDRs discovered at tunnel time.** Plan the VPC CIDR
  from the customer's IPAM, not from your lab defaults.
- **Single VPN tunnel.** Guaranteed to drop at the worst moment. Two
  tunnels, BGP, tested failover.
- **No endpoint acceptance gate.** Any AWS account that knows the
  endpoint service name can request a connection. Require acceptance.
- **Public DNS for internal names.** The customer's internal names
  leak to the internet and are now a reconnaissance surface.
- **No Flow Logs.** The customer's SOC can't answer "what left my VPC
  last Tuesday?" — and they will ask.
- **Single IAM role with wildcards.** A Lantern task role that can
  `s3:*` on every bucket fails a security review every time.
- **No kill-switch.** The customer cannot stop the vendor without
  calling the vendor. Required for any serious deployment.
- **Forgetting the control plane.** You put the data plane inside the
  customer's VPC but the control plane still phones home over the
  internet with telemetry that includes tenant-identifying data.
  Design that traffic too.

## 6. References

- **AWS PrivateLink** developer guide.
  <https://docs.aws.amazon.com/vpc/latest/privatelink/>
- **Azure Private Link** documentation.
  <https://learn.microsoft.com/azure/private-link/>
- **Google Cloud Private Service Connect** documentation.
  <https://cloud.google.com/vpc/docs/private-service-connect>
- **AWS Transit Gateway** documentation.
  <https://docs.aws.amazon.com/vpc/latest/tgw/>
- **AWS VPN (Site-to-Site)** documentation.
  <https://docs.aws.amazon.com/vpn/latest/s2svpn/>
- **RFC 7296** — IKEv2.
  <https://www.rfc-editor.org/rfc/rfc7296>
- **RFC 4301** — Security Architecture for the Internet Protocol
  (IPsec). <https://www.rfc-editor.org/rfc/rfc4301>
- **RFC 1918** — Address Allocation for Private Internets.
  <https://www.rfc-editor.org/rfc/rfc1918>
- **RFC 8446** — TLS 1.3. <https://www.rfc-editor.org/rfc/rfc8446>
- **RFC 4271** — BGP-4. <https://www.rfc-editor.org/rfc/rfc4271>
- **NIST SP 800-207** — Zero Trust Architecture. Useful cross-reference
  for the "network reachability ≠ authorization" principle in §5.
  <https://csrc.nist.gov/publications/detail/sp/800-207/final>
- **NIST AI Risk Management Framework** — Map, Measure, Manage
  functions cited for network governance.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- **OWASP Machine Learning Security Top 10** — relevant when network
  controls protect model-serving surfaces.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- **MITRE ATLAS** — reference for adversary techniques against
  model-serving endpoints exposed through private connectivity.
  <https://atlas.mitre.org/>
