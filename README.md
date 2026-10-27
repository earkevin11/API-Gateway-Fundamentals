# API-Gateway-Fundamentals

# API Gateways Explained (AWS API Gateway and Azure API Management)

A learning guide for cloud security engineers and incident responders.

---

## 1. What an API gateway is

An **API gateway** is a managed front door that sits between callers (mobile apps, websites, partners, other services) and the backend systems that do the work. Every request goes through it first. The gateway decides whether the request is allowed, where to send it, how fast callers may go, and it records what happened.

**Analogy:** the front desk of an office building.

| Front desk | API gateway |
| --- | --- |
| Checks visitor ID | Authenticates the caller (JWT, API key, IAM, mTLS) |
| Knows which office is on which floor | Routes `GET /claims/{id}` to the claims service |
| Enforces visiting hours and visitor limits | Throttling, quotas, rate limits |
| Signs everyone in and out | Access logs (who, what, when, result) |
| Turns away people on the banned list | IP filtering, WAF integration, resource policies |
| Can close the building's front entrance | Containment: block, throttle, or remove the endpoint |

| Cloud | Service |
| --- | --- |
| AWS | **Amazon API Gateway** (REST API, HTTP API, WebSocket API) |
| Azure | **Azure API Management (APIM)** |
| Other common products | Apigee (Google), Kong, MuleSoft, Tyk, NGINX-based gateways |

---

## 2. Core building blocks

| Concept | AWS API Gateway | Azure APIM | What it does |
| --- | --- | --- | --- |
| The API | REST API / HTTP API resource | API inside an APIM instance | Group of routes for one product or service |
| Route | Resource + method (`POST /upload`) | Operation | One callable action |
| Backend | Integration (Lambda, HTTP, AWS service) | Backend (App Service, Function, any URL) | Where the request is sent |
| Authentication | Authorizer: IAM, Cognito, Lambda, JWT | `validate-jwt` policy, client certs | Proves who the caller is |
| Caller identification and metering | API keys + usage plans | Products + subscription keys | Who is calling and how much they may call |
| Environment | Stage (`dev`, `prod`) | Revisions, or separate instances per env | A deployed, callable version |
| Traffic limits | Stage and method throttling | `rate-limit`, `quota` policies | Protect backends from overload and abuse |
| Network restrictions | Resource policy, private APIs (VPC endpoint) | `ip-filter` policy, VNet integration | Who can reach it at the network level |
| Logs | CloudWatch execution and access logs | Diagnostics to Log Analytics | Request-level evidence |
| Config change audit | CloudTrail | Azure Activity Log | Who changed the gateway |

**Important:** API keys (AWS) and subscription keys (APIM) identify and meter callers. They are **not** strong authentication. A key in a mobile app can be extracted. Real authentication comes from authorizers or `validate-jwt`.

---

## 3. One gateway per app, or one for many?

**Short answer:** it depends on the platform, but most organizations **share gateway infrastructure across many apps** and separate by environment and sensitivity.

### AWS API Gateway

AWS API Gateway is fully serverless and billed per request, so there is no "instance" to share. Each API is its own resource.

| Pattern | How it looks | When used |
| --- | --- | --- |
| **One API per app or microservice** (most common) | `claims-api`, `member-api`, `provider-api` each a separate REST API | Teams own their APIs; clean blast radius and permissions |
| **Shared custom domain** | `api.company.com` with base path mappings: `/claims` → claims-api, `/members` → member-api | One public hostname, many APIs behind it |
| **Central platform team API** | One big API fronting many backends | Smaller orgs or tightly governed environments |

### Azure APIM

APIM is a **provisioned instance** (Developer, Basic, Standard, Premium, v2 tiers) with real cost, so it is almost always shared.

| Pattern | How it looks | When used |
| --- | --- | --- |
| **Shared instance per environment** (most common) | `apim-prod` hosts 50 APIs from many teams; `apim-dev` separate | Cost efficiency, central governance |
| **Separate instances by sensitivity** | `apim-internal` vs `apim-external`, or a dedicated instance for PHI APIs | Compliance, isolation of internet-facing APIs |
| **Federated / workspaces** | One instance, teams manage their own APIs in isolated workspaces | Large orgs balancing central control and team autonomy |

### Security implications of sharing

| Shared gateway | Dedicated gateway per app |
| --- | --- |
| One place to enforce baseline policies (auth, logging, WAF) | Smaller blast radius if misconfigured or compromised |
| One place to investigate and contain | Containment affects only one app |
| Misconfiguration or outage hits every app | Policy drift: each team configures security differently |
| Admin role is very powerful; protect it like Global Admin | More resources to monitor and audit |

**IR takeaway:** before containing at the gateway level, know **what else shares it**. Deleting a shared stage or blocking a shared domain can take down unrelated apps.

---

## 4. What organizations used before API gateways

| Era | Approach | Problems |
| --- | --- | --- |
| Early web (1990s-2000s) | Apps exposed directly to the internet; each app did its own auth, logging, rate limiting | Inconsistent security, every team reinvented auth, no central visibility |
| SOA era (2000s) | **Enterprise Service Bus (ESB)** and XML gateways (IBM DataPower, CA/Broadcom Layer7, Oracle) | Heavy, expensive, slow to change, SOAP/XML focused |
| Reverse proxy era | NGINX, Apache, F5 BIG-IP in front of apps | Good at routing and TLS, but API-aware features (keys, quotas, JWT) needed custom scripting |
| Cloud and microservices (2010s+) | **API gateways** (Apigee, Kong, AWS API Gateway, Azure APIM) | Purpose-built for REST/JSON APIs, managed, policy-driven |
| Today | API gateways plus service meshes (Istio, Linkerd) for service-to-service traffic inside clusters | Gateway handles north-south traffic (in/out); mesh handles east-west (between services) |

---

## 5. Security benefits

| Benefit | What it gives you | Example |
| --- | --- | --- |
| **Centralized authentication** | Backends never see unauthenticated traffic | Invalid JWT gets 401 at the gateway; the Lambda never runs |
| **Reduced attack surface** | Only defined routes are reachable; backends stay private | Backend in a private subnet, only the gateway can call it |
| **Rate limiting and quotas** | Slows brute force, scraping, and DoS | 100 req/s per client; scraper hits 429 |
| **Consistent logging** | One log format for every API | Every call logged with IP, status, latency, request ID |
| **Input validation** | Reject malformed requests before they reach code | Request schema validation blocks unexpected fields |
| **Secrets kept out of clients** | Gateway adds backend credentials, not the caller | APIM policy fetches a Key Vault secret for the backend |
| **WAF integration** | Blocks injection and known bad IPs | AWS WAF or Azure Front Door/App Gateway WAF in front |
| **Single containment point** | Block, throttle, or remove access in one place | Throttle stage to 0 during an incident |
| **Governance** | Inventory of every exposed API | Find shadow APIs and routes with no authorizer |

### Common security mistakes

| Mistake | Risk |
| --- | --- |
| Route with authorization `NONE` | Anyone on the internet can call the backend |
| Treating API keys as authentication | Leaked key = full access |
| Logging disabled (default for access logs) | No evidence during an incident |
| Over-privileged backend role (`s3:*`) | One API bug exposes all data |
| Default `execute-api` endpoint left on when a custom domain with WAF is used | Attacker bypasses WAF via the raw endpoint |
| Verbose errors returned to callers | Leaks stack traces and internal names |

---

## 6. API gateway vs load balancer vs WAF

They solve different problems and are usually **used together**, not instead of each other.

| Capability | Load balancer (ALB / Azure App Gateway, Azure LB) | WAF (AWS WAF / Azure WAF) | API gateway (AWS API Gateway / APIM) |
| --- | --- | --- | --- |
| Main job | Spread traffic across healthy servers | Block malicious request patterns | Manage, secure, and govern APIs |
| OSI focus | Layer 4 (NLB/Azure LB) or Layer 7 (ALB/App Gateway) | Layer 7 inspection | Layer 7, API-aware |
| Routing | By host/path to target groups | No routing | By route, method, version, stage |
| Authentication | Limited (ALB can do OIDC) | None | Strong: JWT, OAuth, IAM, mTLS, custom authorizers |
| Per-client rate limits and quotas | No | Rate-based rules by IP only | Yes, per key, per user, per product |
| SQLi/XSS/bot protection | No | **Yes, its core job** | Basic validation only; relies on WAF |
| Request/response transformation | No | No | Yes |
| API keys, developer portal, versioning | No | No | Yes |
| Best for | High-throughput traffic to servers or containers | Filtering attacks before they reach anything | Exposing APIs safely to apps and partners |
| Cost model | Hourly + capacity units | Per rule + per request | Per request (AWS) or per instance (APIM) |

### Where each is better or worse

| Gateway is **better** than a load balancer at | Gateway is **worse** than a load balancer at |
| --- | --- |
| Authenticating callers and enforcing per-client quotas | Raw throughput and latency at very high volume |
| API versioning, transformation, developer onboarding | Long-lived connections, non-HTTP protocols (NLB handles TCP/UDP) |
| API-level logging and governance | Cost at massive scale (per-request pricing adds up) |

| Gateway is **better** than a WAF at | Gateway is **worse** than a WAF at |
| --- | --- |
| Knowing *who* the caller is | Detecting injection, XSS, bots, and known bad IPs |
| Business rules (Partner A gets 10k calls/day) | Managed threat-intel rule sets |

### Typical layered design

```
Internet
   │
   ▼
[ WAF ]               ← blocks attacks, bad IPs, bots
   │
   ▼
[ API Gateway / APIM ] ← authenticates, throttles, routes, logs
   │
   ▼
[ Load Balancer ]      ← spreads load across backend instances (if not serverless)
   │
   ▼
[ Backend services ]   ← Lambda, containers, VMs, databases (private)
```

Azure equivalent: Front Door (with WAF) → APIM → Application Gateway or internal LB → App Service / AKS / Functions.

---

## 7. Worked example: a healthcare member portal

1. The mobile app sends `GET /claims/123` with the member's login token.
2. **WAF** checks for SQL injection and known bad IPs. Clean request passes.
3. **API Gateway** validates the token with the authorizer. Invalid token gets 401.
4. Gateway checks throttling: this member is under 100 requests per minute.
5. Gateway routes to the `getClaim` Lambda, which runs with a role that can read **only** the claims table.
6. Lambda returns the claim; gateway logs IP, user, status 200, response size, latency.

**What an attacker would try:** steal a token, find a route with no authorizer, enumerate claim IDs (`/claims/1` through `/claims/100000`), or modify the gateway config if they compromise an admin identity. The companion README covers how to detect and contain each.

---

## 8. Quick reference

| Question | Answer |
| --- | --- |
| What is it? | Managed front door for APIs: auth, routing, throttling, logging |
| AWS service | Amazon API Gateway |
| Azure service | Azure API Management (APIM) |
| Shared or 1-to-1? | AWS: usually one API per app, often behind a shared domain. APIM: usually one shared instance per environment |
| Replaced what? | Direct exposure, ESBs, XML gateways, hand-scripted reverse proxies |
| Biggest security benefit | Central enforcement of authentication and a single containment point |
| Use with a WAF? | Yes. The WAF stops attack patterns; the gateway knows who the caller is |
| Most dangerous misconfig | A route with no authorizer in front of a privileged backend |
