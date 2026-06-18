# GetDue — Engineering Context Brief (for agents)

A distilled, single-file brief of the GetDue platform, synthesized from the canonical docs in
the **`getdue-docs`** repo. Every agent reads this for grounding. When the `getdue-docs` repo is
present in the workspace, treat **it** as the source of truth and this file as the index/summary —
cite the specific doc (`getdue-docs/phase-0/04-api-design.md`, etc.) rather than paraphrasing from
memory.

> **Doc map.** Product/architecture: `getdue-docs/phase-0/00-overview` … `10-dashboard-analytics`.
> Engineering process: `getdue-docs/engineering/01-repositories`, `02-versioning`,
> `03-testing-standard`, `04-secure-sdlc`. The two authoritative rulesets are
> **`phase-0/09-security-standard.md`** (application/runtime controls, RFC-2119 MUST/SHOULD) and
> **`engineering/03-testing-standard.md`** (coverage + mutation gates).

---

## 1. What GetDue is

A personal **"home bank in the cloud"**: an iPhone app (SwiftUI), a personal web cabinet (Next.js),
and a monitoring system, over **C#/.NET 10 stateless microservices** on Kubernetes. A household
records its financial life — bank accounts, debts, real estate, mortgages, stocks, goals — and the
platform **aggregates, tracks, visualizes, and monitors** it.

**Phase 0 is a foundation slice.** Everything is **user-authored**; the platform stores/aggregates/
visualizes/monitors. Hard guardrails (enforced by architecture tests):

- **Money never moves** — no payment/transfer/payout endpoints. The system is a *ledger of record*,
  not a processor.
- **No real bank/brokerage/market/FX connectivity** — no outbound calls to financial-institution,
  market-data, or FX vendors exist in the build. Multi-currency *is* supported, but FX rates are
  user-sourced/seeded (`source = MANUAL | SEED`).
- Operational egress **is** allow-listed: transactional email, push (APNs/FCM), breached-password
  check (HIBP k-anonymity). The guardrail targets *financial feeds*, not all egress.

**Personas:** Account owner (web + iPhone), household co-member (web, read or co-edit), platform
operator (monitoring).

## 2. Architecture (invariants)

- **One API surface, many clients.** A **YARP** gateway/BFF fronts all services; clients hold no
  business logic and never address services directly.
- **Stateless microservices**, HA floor **≥2 pods**, autoscale **2→10** (ceiling is a Phase-0 cost
  cap). No in-pod state — JWTs validated locally; refresh tokens, rate-limit counters, read caches
  live in **Redis**; durable data in **Postgres**; in-flight events in **RabbitMQ**.
- **Clean Architecture + DDD per service**, four projects, dependencies point **inward only**:
  `*.Api` → `*.Application` → `*.Domain` (pure C#, no deps); `*.Infrastructure` (EF/Redis/broker)
  depends inward, never the reverse.
- **Database-per-service.** Each service owns its schema; **no service reads another's tables** —
  integrate via its API or its events. (Phase-0 topology: 8 DBs on one Postgres Flexible Server,
  each with its own least-privilege role.)
- **Event-driven integration** via RabbitMQ (Rebus) + **transactional outbox**; sync calls only
  when a request truly needs another service's current state (typed `HttpClient`/gRPC, Polly-wrapped).
- **Event-sourced valuations** (not full ES): every monetary change appends a `VALUATION_SNAPSHOT`,
  giving net-worth history for free. Net-worth/dashboard are **eventually-consistent projections**
  (recompute-lag SLO **<5s**; responses carry `asOf` + `projectionVersion`; rebuildable by an
  idempotent reconcile job; drift-monitored).
- **Observable by construction** — OpenTelemetry from line one; traces span service hops.

**Services (one bounded context, one DB, ≥2 pods each):** Identity (`User`,`Household`), Accounts
(`BankAccount`), Debts & Mortgages (`LoanDebt`,`MortgageLoan`), Real Estate (`Property`), Stocks
(`Portfolio`,`Holding`), Goals (`FinancialGoal`), Net Worth/Aggregation (`ValuationSnapshot`,
`ExchangeRate`, dashboard read models + FX conversion), Insights (`Insight`).

## 3. Tech stack (Phase-0 pins)

**Backend:** .NET 10 · ASP.NET Core minimal APIs · YARP gateway · REST+OpenAPI, CQRS internally ·
**Mediator** (`martinothamar/Mediator`, source-gen — *not* MediatR) · **EF Core 10** (Npgsql) ·
**FluentValidation** · **Mapperly** (source-gen mapping) · ASP.NET Core Identity + stateless JWT/
refresh · **Rebus over RabbitMQ** + transactional outbox · Serilog→OTLP · OpenTelemetry .NET ·
**Polly**. **Postgres 16** (one DB/service), **Redis 7**, **RabbitMQ 3.13**.

**Web:** Next.js 15 (App Router) + React 19 · TypeScript strict · TanStack Query + `openapi-typescript`
generated client · Tailwind + shadcn/ui · Recharts · React Hook Form + Zod · HttpOnly cookie (refresh)
+ in-memory access token.

**iPhone:** Swift + SwiftUI (iOS 17+) · `swift-openapi-generator` client · Keychain (refresh) ·
certificate pinning + Face ID.

**Platform:** Azure · Kubernetes (AKS) · HPA floor 2/ceiling 10 · ingress-nginx + cert-manager
(Let's Encrypt, **DNS-01** via Cloudflare) · Cloudflare edge (**Full strict** TLS, WAF, mandatory) ·
distroless/chiseled non-root images · Terraform · GitHub Actions · Argo CD (GitOps) · Azure Key Vault ·
**Linkerd** automatic mTLS. Monitoring: OpenTelemetry → Prometheus / Grafana / Loki / Tempo /
Alertmanager.

## 4. API conventions (`getdue-docs/phase-0/04-api-design.md`)

| Topic | Rule |
|---|---|
| Base URL | `https://api.getdue.com/v1` — URL-path major; additive changes stay in `/v1` |
| Format | JSON, UTF-8, **`camelCase`** fields |
| Auth | `Authorization: Bearer <access_jwt>`; refresh via HttpOnly cookie |
| IDs | **UUID v7** strings |
| Money | `{ "amount": "1234.56", "currency": "EUR" }` — amount as **string** (no float loss) |
| Dates | ISO-8601, UTC (`Z`); calendar dates as `date` |
| Pagination | `?page&pageSize` → `{items,page,pageSize,total}`; **append-only ledgers use keyset/cursor** |
| Filter/sort | `?filter[field]=` / `?sort=field,-field2` over **allow-listed** fields only |
| Errors | **RFC 9457** Problem Details (`application/problem+json`) |
| Idempotency | **`Idempotency-Key` REQUIRED** on every state-changing/creating POST + snapshot writes |
| Concurrency | mutable resources return **`ETag`**; `PATCH/PUT/DELETE` require **`If-Match`** → stale = **412** |
| Correlation | W3C `traceparent` propagated; `X-Request-Id` echoed |

**Status codes:** 400 malformed · 401/403 unauth / not-your-household · 404 · 409 conflict (dup key /
version) · 412 If-Match failed · 422 domain/validation · 429 rate-limited (+`Retry-After`) · 5xx.
**Rate limits:** edge per-IP 20rps; per-user 10rps/burst40/600rpm; login 5/min; reset & verify-resend
3/min. **Idempotency:** key scope `(householdId,userId,method,route,key)`, 7d TTL; replay returns the
stored response verbatim; different body under same key → 409. Money writes store the idempotency row
**in the same transaction** as the domain change + outbox event. **Consumer idempotency:** event
consumers dedupe by event id (at-least-once delivery).

## 5. Domain model essentials (`getdue-docs/phase-0/03-domain-model.md`)

- **Money** is `{amount:decimal, currency}` — `numeric(19,4)` in PG, **never float**; arithmetic only
  within the same currency; cross-currency requires explicit `Convert(rate)`. The `Money` type
  **refuses to add two currencies**.
- **APR / interest = `Rate`**, a decimal **fraction** (`0.0599` = 5.99%), `numeric(9,6)`, monthly =
  `apr/12` — **never** stored as 0–100. `Percentage` (0–100) is display/derived only (progress %,
  allocation %).
- **FX rates** (`numeric(19,8)`): convention `rate = quote_per_base`; **append-only** (corrections
  insert a new row; latest `created_at` wins); base→base = 1.0; cross-rates triangulated through base;
  missing rate → entity flagged `unconverted` (never assume 1.0).
- **ExchangeRate / `source` / `createdAt` are server-set** — request bodies including them → 422.
- **Valuation snapshot** stores native `amount`+`currency`, `fx_rate_to_base`, `amount_in_base`
  (computed at write time) and `as_of`; snapshots are **immutable** (FX corrections don't rewrite
  history). Net worth = Σ latest-snapshot-per-subject `amount_in_base`, assets `+` / liabilities `−`.
- **Loans/mortgages:** immutable `PaymentSchedule` versions; payments (`SCHEDULED | PARTIAL_PREPAYMENT
  | FULL_PAYOFF | EXTRA`) record a ledger entry, `amount = principal + interest` in the loan's
  currency, `principal ≤ outstanding`; prepayment re-amortizes into a new version (`REDUCE_TERM` |
  `REDUCE_INSTALLMENT`). No money moves — payments are user-recorded facts.
- **Goal cold-start:** with no contributions or <30 days history, `projectedCompletionDate = null`,
  `onTrack = false`; never assume the user will start paying `requiredMonthly`.
- **Tenant = Household.** Every business row carries `household_id`; one `OWNER`; one `base_currency`;
  a user belongs to one household in Phase 0. Roles `OWNER | MEMBER` are **server-set**, never
  client-supplied.

**Domain events:** `BankAccountBalanceChanged`, `PropertyValued`, `LoanScheduleImported`,
`LoanPaymentRecorded`, `LoanBalanceChanged`, `LoanPaidOff`, `HoldingPriced`, `ExchangeRateChanged`,
`GoalContributed`, `GoalStatusChanged` — all via the transactional outbox, carry `schemaVersion` +
event id.

## 6. Security standard — MUST highlights (`getdue-docs/phase-0/09-security-standard.md`)

Zero trust · least privilege · defense in depth · secure by default · assume breach · money never moves.

- **AuthN:** JWT access ≤15 min + rotating refresh (Redis, revocable, never `localStorage`); **every
  service validates the JWT locally** (sig/iss/aud/exp/scopes) — gateway check doesn't exempt it.
  Passwords **Argon2id** (m=19456,t=2,p=1 min) + breached-password check; MFA on by default. Asymmetric
  signing keys (vault, JWKS rotation).
- **AuthZ / tenant isolation:** authorize at the **service/handler** layer; **`householdId` EF Core
  global query filter on every entity** (CI arch test asserts it); roles server-side; **object-level
  check on every `{id}` route (no IDOR)**.
- **Network:** mTLS everywhere (Linkerd); default-deny NetworkPolicy; public ingress only via
  edge→ingress-nginx→gateway behind a **mandatory WAF**; per-user+per-IP rate limits; broker TLS +
  per-service creds + idempotent consumers; **default-deny egress** (financial feeds blocked, ops egress
  allow-listed).
- **Secrets:** none in source/history/images/logs; managed vault; owned + rotated (app ≤90d, JWT keys
  ≤180d, DB creds ≤30d).
- **Data:** encryption in transit + at rest; data classification; **money as `decimal`/`numeric(19,4)`**;
  **no RESTRICTED/CONFIDENTIAL data (incl. monetary amounts) in logs/metrics/traces/errors**; hashed
  subject ids; GDPR export (`/me/export`) + erasure; synthetic data only in non-prod.
- **Logging:** immutable append-only **audit log** (authn, authz denials, role/data export/deletion,
  admin, secret access) to WORM/SIEM ≥1yr — **not Loki** (Loki = ops logs).
- **Resilience:** ≥2 replicas, probes, PDB(minAvailable 1), rolling deploys, HPA; Polly on sync calls;
  idempotency keys (7d TTL) + event-id dedupe.

## 7. Versioning (`getdue-docs/engineering/02-versioning.md`)

**SemVer everywhere**; back-compat default; **immutable artifacts** (a built image/package/tag is
never re-pushed). API: URL major `/v1`, additive stays in place, breaking → new major + ADR +
≥6-month deprecation (RFC 8594 `Deprecation`/`Sunset`). **Contracts** (`getdue-contracts`, SemVer):
breaking = MAJOR + ADR; **schema-diff CI gate** fails a breaking change without a MAJOR bump; services
pin **published, tagged** versions (never branch/local). **Events:** explicit `schemaVersion`, additive-
only within a major, breaking = new event version with dual-emit. **DB:** ordered EF Core migrations,
**expand → migrate → contract** for zero-downtime rolling deploys (never combine a destructive change
with the code that needs it); forward-only in prod. Every service exposes `/health` + version
(`{service,version,gitSha,builtAt,apiVersions}`).

## 8. Testing standard (`getdue-docs/engineering/03-testing-standard.md`)

- **Backend (.NET) `Domain`+`Application`: 100% line AND branch coverage — release-blocking** (Coverlet);
  no ratchet-down. **Mutation score ≥85%** (Stryker.NET), **100% for money-critical** modules (net-worth
  aggregation, FX, amortization, idempotency); a surviving mutant in money/authz/tenant code is a
  **release blocker**.
- **Web/mobile/infra: ≥80% line target, measured & reported, non-blocking**; no mutation gate.
- **Pyramid:** Unit (xUnit + FluentAssertions / Vitest / XCTest) · Integration against **real**
  Postgres/Redis/RabbitMQ via **Testcontainers** (no mocking the DB/broker/cache) · Contract (Pact-style/
  schema) · Architecture (**NetArchTest**) · E2E (Playwright / XCUITest, not counted toward coverage).
- **Quality (MUST):** assert behavior not implementation; no assertion-free tests; determinism (inject
  clock/RNG/ids, no flaky tests); property-based/boundary tests for money & multi-currency (rounding,
  currency-mismatch, missing-FX-rate); bug fixes ship a failing-first regression test.
- **Coverage exclusions** are a reviewed allow-list only (generated code, pure DI wiring, plain DTOs,
  trivial framework glue); anything with a branch/calculation/side-effect is never excluded.

**Architecture tests (CI, per service):** dependency direction; `householdId` filter on every entity;
no cross-service DB access; stateless guard (no static mutable request state); **no money movement**
(block `/transfers/initiate`,`/payouts`,`/withdrawals`, payment-rail SDKs; recording user-entered
loan/mortgage payments is allow-listed); **no outbound finance/market/FX** (HttpClient `BaseAddress`
host must be on the egress allow-list).

## 9. Repositories & workflow (`getdue-docs/engineering/01-repositories.md`, `04-secure-sdlc.md`)

**Polyrepo** in the **public** `getdue-dev` org — one repo per service + shared repos
(`getdue-contracts`, `getdue-buildingblocks`, `getdue-platform`, `getdue-deploy`, `.github`). Each
service repo shape:

```
getdue-<service>/
├─ src/{Api,Application,Domain,Infrastructure}/
├─ tests/{Unit,Integration,Architecture}/
├─ deploy/k8s/        # Deployment(replicas 2, HPA max 10), Service, HPA, PDB, probes
├─ .github/workflows/ # reusable CI from .github
├─ Dockerfile         # multi-stage, distroless/chiseled, non-root
└─ SECURITY.md        # → phase-0/09 + engineering/04
```

**GitHub Flow on protected `main`:** work on a feature branch → PR → **green CI to merge**; no direct
push, no force-push, linear history (squash/rebase), signed commits, conversation resolution required,
auto-delete merged branches. Solo phase: maintainer self-merges once CI is green. **Every change is a
PR with passing CI and a ticket reference.**

**CI pipeline (per service):** build → unit+integration (Testcontainers) → coverage gate → mutation
gate → architecture tests → SAST (Semgrep) + secret scan (gitleaks) + SCA + IaC scan + container scan
→ SBOM + cosign sign → publish image → bump tag in `getdue-deploy` → Argo CD rollout. Security gates
**fail the build** on High/Critical.

**`getdue-buildingblocks`** provides cross-cutting primitives — `Money`, transactional outbox + relay,
**idempotency-key middleware**, OTel bootstrap, JWT handlers, Polly policies, problem-details
middleware. **No business/domain code** there. Consume contracts + buildingblocks as **pinned** NuGet/
npm/Swift packages, never copy-paste, never a floating range.

### Agent workflow rules (mandatory)
- **Shipping code is done only through the `commit-feature` skill** — never raw `git commit`/`git push`/
  `gh pr create`. It branches off fresh `main`, commits, pushes, opens the PR, and returns to clean `main`.
- **Creating a repository is done only through the `create-new-repo` skill** — never create/configure a
  repo by hand; it applies the polyrepo conventions (public, protected `main`, `SECURITY.md`, shared CI).
