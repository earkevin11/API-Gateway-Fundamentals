# API Gateways Explained

A beginner-friendly guide to AWS API Gateway and Azure API Management (APIM) for cloud security engineers and incident responders.

The companion guide, `API_Gateway_Investigation_and_Containment_README.md`, covers attacks, investigation, and containment.

---

## Table of contents

1. [What an API gateway is](#1-what-an-api-gateway-is)
2. [Key terms in plain English](#2-key-terms-in-plain-english)
3. [Walkthrough of one request](#3-walkthrough-of-one-request)
4. [API gateway vs load balancer vs WAF](#4-api-gateway-vs-load-balancer-vs-waf)
5. [One gateway per app or one for many](#5-one-gateway-per-app-or-one-for-many)
6. [What organizations used before API gateways](#6-what-organizations-used-before-api-gateways)
7. [Security benefits and what breaks them](#7-security-benefits-and-what-breaks-them)

---

## 1. What an API gateway is

**In one sentence:** an API gateway is a managed front door that every request passes through before it reaches your application's backend.

First, what is an **API**? An API (Application Programming Interface) is how one program asks another program for something. When a mobile app shows your bank balance, it sent a request like `GET /accounts/123/balance` to the bank's API and got data back.

The **gateway** sits in front of those APIs and handles the jobs every API needs, so each app team doesn't have to build them:

| The gateway... | Like a building's front desk that... |
| --- | --- |
| Checks who the caller is | Checks visitor ID |
| Sends the request to the right backend | Tells visitors which floor and office to go to |
| Limits how often a caller can make requests | Limits how many visitors come in at once |
| Logs every request | Keeps a sign-in sheet |
| Can block or shut off access | Can lock the front entrance |

```
Caller (mobile app, website, partner)
        │
        ▼
 ┌──────────────────┐
 │   API Gateway    │  ← checks identity, limits, logs, routes
 └──────────────────┘
        │
        ▼
Backend (Lambda function, server, database)  ← never exposed directly
```

| Cloud | Service name |
| --- | --- |
| AWS | **Amazon API Gateway** |
| Azure | **Azure API Management (APIM)** |
| Other vendors | Apigee (Google), Kong, MuleSoft |

---

## 2. Key terms in plain English

| Term | Plain-English meaning | AWS name | Azure APIM name |
| --- | --- | --- | --- |
| **Route** | One thing a caller can ask for, like `GET /claims/{id}` | Resource + method | Operation |
| **Backend / integration** | The code or service that actually does the work | Integration (Lambda, HTTP endpoint, AWS service) | Backend |
| **Authorizer** | The check that proves who the caller is, usually by validating a login token | Authorizer (IAM, Cognito, Lambda, JWT) | `validate-jwt` policy |
| **JWT** | A signed login token issued after a user signs in, like an Entra ID access token | Same | Same |
| **API key** | A code that identifies which customer or partner is calling. **Not a password.** It can be copied out of an app. | API key + usage plan | Subscription key + product |
| **Throttling** | A speed limit on requests | Stage / method throttling | `rate-limit`, `quota` policies |
| **Stage** | A published, live version of the API, such as `dev` or `prod`. Each has its own URL. | Stage | Revision, or a separate instance |
| **Deployment** | A saved snapshot of the API's configuration. A stage points to one deployment. | Deployment | Revision |
| **Resource policy** | Rules for which networks or accounts can reach the API at all | Resource policy | `ip-filter` policy, VNet integration |
| **Execution role** | The cloud identity the backend uses to touch other services (S3, databases) | IAM role | Managed identity |
| **Access log** | A record of each request: who, what, when, result | CloudWatch access logs | `ApiManagementGatewayLogs` |
| **Control plane** | Settings that define the API (who changes the config) | Logged in CloudTrail | Logged in the Activity Log |
| **Data plane** | The actual requests flowing through the API | Logged in access logs | Logged in gateway logs |

**Stage vs deployment, simply:** a deployment is like a saved draft of a document, and a stage is like publishing that draft to a public website. Deleting the stage takes the website down, but the drafts still exist.

---

## 3. Walkthrough of one request

A member opens a healthcare app to check a claim.

| Step | What happens | Which part does it |
| --- | --- | --- |
| 1 | App sends `GET /claims/123` with the member's login token | Caller |
| 2 | Request is checked for attacks like SQL injection | WAF (see section 4) |
| 3 | Token is validated. A bad or expired token gets **401 Unauthorized** and stops here. | Authorizer |
| 4 | Caller's request rate is checked. Too many requests get **429 Too Many Requests**. | Throttling |
| 5 | Request is sent to the `getClaim` Lambda function | Route + integration |
| 6 | Lambda reads the claim using a role that can access **only** the claims table | Backend + execution role |
| 7 | Response returns to the app with **200 OK**, and the request is logged | Gateway + access log |

**HTTP status codes you'll see constantly:**

| Code | Meaning |
| --- | --- |
| 200 | Success |
| 401 | Not authenticated (missing or bad token) |
| 403 | Authenticated but not allowed, or blocked by policy or WAF |
| 404 | Route doesn't exist |
| 429 | Throttled, too many requests |
| 5xx | Server or backend error |

---

## 4. API gateway vs load balancer vs WAF

**In one sentence:** they do different jobs and are usually used together. The WAF blocks attacks, the gateway knows who the caller is, and the load balancer spreads traffic across servers.

| Question | Load balancer | WAF | API gateway |
| --- | --- | --- | --- |
| Main job | Spread traffic across healthy servers | Block malicious requests | Manage and secure APIs |
| Knows *who* the caller is? | Barely (ALB can do basic sign-in) | No | **Yes**: tokens, keys, IAM |
| Blocks SQL injection and bots? | No | **Yes, its core job** | Only basic input checks |
| Per-customer limits (Partner A: 10,000 calls/day)? | No | No (only per-IP rate limits) | **Yes** |
| Routes by URL path? | Yes | No | Yes |
| Changes request or response format? | No | No | Yes |
| Handles non-web traffic (raw TCP/UDP)? | Yes (Network LB) | No | No |
| AWS service | ALB, NLB | AWS WAF | API Gateway |
| Azure service | Application Gateway, Azure Load Balancer | Azure WAF (Front Door or App Gateway) | APIM |

### Strengths and weaknesses of the gateway

| Gateway is better at | Gateway is worse at |
| --- | --- |
| Authenticating callers and enforcing per-customer quotas | Very high-volume, low-latency traffic (a load balancer is cheaper and faster) |
| Versioning, transforming, and governing APIs | Detecting attack payloads and bots (that's the WAF's job) |
| Giving one consistent log format for every API | Non-HTTP protocols |

### How they're layered together

```
Internet
   ▼
[ WAF ]                ← blocks attack patterns and bad IPs
   ▼
[ API Gateway / APIM ] ← checks identity, applies limits, routes, logs
   ▼
[ Load balancer ]      ← spreads load (skipped for serverless backends like Lambda)
   ▼
[ Backends ]           ← kept private
```

Azure version: Front Door with WAF → APIM → Application Gateway or internal load balancer → App Service, AKS, or Functions.

---

## 5. One gateway per app or one for many

**In one sentence:** most organizations share gateway infrastructure across many apps and separate it by environment (dev vs prod) and sensitivity.

| | AWS API Gateway | Azure APIM |
| --- | --- | --- |
| Billing model | Pay per request; nothing to provision | Pay for a running instance |
| Most common pattern | One API per app or microservice (`claims-api`, `member-api`) | One shared instance per environment (`apim-prod` hosting dozens of APIs) |
| How they share | A custom domain maps paths to APIs: `api.company.com/claims` → claims-api | Many teams publish APIs into the same instance, sometimes in separate workspaces |
| Extra isolation when needed | Separate AWS accounts for sensitive apps | Separate instances for internal vs internet-facing, or for PHI |

### Shared vs dedicated: the security trade-off

| Shared gateway | Dedicated gateway per app |
| --- | --- |
| ✅ One place to enforce auth, logging, and WAF | ✅ A mistake or incident affects only one app |
| ✅ One place to investigate and contain | ❌ Each team may configure security differently |
| ❌ A misconfiguration or outage hits every app | ❌ More things to monitor |
| ❌ Gateway admin role is very powerful | |

**Why IR cares:** before you block or delete anything at the gateway, find out what else shares it. Containing a shared domain can take down unrelated apps.

---

## 6. What organizations used before API gateways

| Era | Approach | Why it fell short |
| --- | --- | --- |
| 1990s-2000s | Apps exposed directly; every app built its own login, logging, and limits | Inconsistent security, no central view |
| 2000s | Enterprise service buses and XML gateways (IBM DataPower, Layer7) | Expensive, heavy, slow to change |
| 2000s-2010s | Reverse proxies (NGINX, Apache, F5) | Great for routing and TLS, but token checks and quotas needed custom scripting |
| 2010s-today | Purpose-built API gateways (AWS API Gateway, APIM, Apigee, Kong) | Managed, policy-driven, built for modern REST/JSON APIs |

Today, gateways handle traffic coming **into** an environment (north-south). Service meshes such as Istio often handle traffic **between** services inside a cluster (east-west).

---

## 7. Security benefits and what breaks them

Each benefit only holds if it's configured correctly. The right column is what attackers look for.

| Benefit | Example | Misconfiguration that breaks it |
| --- | --- | --- |
| **Central authentication**: backends never see unauthenticated traffic | Bad token gets 401; the Lambda never runs | A route with authorization set to `NONE` |
| **Smaller attack surface**: only defined routes are reachable | Backend sits in a private subnet | Default `execute-api` URL left open, bypassing the WAF on the custom domain |
| **Rate limiting**: slows brute force and scraping | Scraper gets 429 after 100 requests/second | No throttling, or limits set far too high |
| **Caller identification**: know which partner made each call | Each partner has its own key | Treating API keys as passwords; a leaked key equals full access |
| **Least-privilege backends**: limit damage from a bug | Lambda can read only the claims table | Execution role with `s3:*` or admin rights |
| **Consistent logging**: evidence for every request | Every call logged with IP, status, size | Access logs are **off by default** in AWS |
| **Safe error handling** | Generic error messages | Stack traces and internal names returned to callers |
| **Single containment point**: shut off access in one place during an incident | Throttle a stage to zero | No documented owner or approval path for prod changes |

**Azure parallel:** a route with no authorizer in AWS is the same risk as removing the `validate-jwt` policy from an API in APIM. Authentication silently disappears and the backend is exposed.

For detecting and responding to these issues, see `API_Gateway_Investigation_and_Containment_README.md`.
