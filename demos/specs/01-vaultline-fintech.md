# VaultLine — The Bank You Connect To Your Other Banks

> Tagline: "Your money in one tab. Your money on the open internet."
> Series role: Headline episode. The first thing a prospect watches when they ask, "does Apex actually work?"

## Premise & Vibe

VaultLine is a personal-finance aggregator and merchant-data platform pitched as "Plaid plus Mint, with a Series B haircut." End users sign up, link external bank accounts via a simulated Open Banking flow, and watch their transactions stream in, get categorized, and roll up into budgets and net-worth charts. On the other side of the same backend, a B2B "Aggregation API" sells normalized, opt-in transaction data to merchant tenants — think loyalty platforms, BNPL underwriters, SMB accounting tools.

The vibe of the demo target is "credible Series B fintech": clean React 19 dashboard, JWT-bearer mobile-style API, an OpenAPI-documented merchant API behind a per-tenant API key, a Spring Cloud Gateway in front, OpenTelemetry traces in Tempo, Kafka pumping `transactions.raw` into `transactions.categorized`. It looks like a company that just hired its first security engineer and hasn't onboarded them yet. Every "small" shortcut a real fintech takes when shipping fast is in here, deliberately.

The headline episode has to land hard: we are pointing Apex at money. Every finding the audience sees needs to feel like something they could screenshot and send to their CISO with the words "we have this."

### Name Candidates

- **VaultLine** (primary) — implies a money pipe and a vault, fintech-y without being on-the-nose.
- **Ledgerly** — softer brand voice, feels Mint-adjacent. Use if "VaultLine" trademarks poorly.
- **Northwire** — leans more "Plaid for B2B"; reserve for a future infra-fintech episode if we split the demo.

## Why This Stack

We chose Kotlin + Spring Boot 3.3 deliberately. It is the single most common "boring serious money" stack in fintech: most US neobanks, half of European challenger banks, and nearly every BNPL underwriter ship Spring services in production. That makes the bug classes economically real. SpEL injection, Jackson polymorphic deserialization, Actuator exposure, and broken JWT filters are not academic — they recur in fintech disclosures every year, and Apex needs to demonstrate it can find them with the same clarity a senior AppSec engineer would.

React 19 + Vite + TypeScript on the front end gives us a modern SPA with realistic auth flows (token in memory, refresh in HttpOnly cookie) and gives Apex's browser/Playwright tools something to actually drive. Kafka in the loop lets us show Apex reasoning about an async data path, not just request/response. PostgreSQL + Flyway gives us migration archaeology — Apex can read the migration history and infer historical schema decisions, which matters for the cross-tenant isolation finding.

Spring Cloud Gateway sits up front because real fintechs put a gateway up front, and because gateway misconfiguration (header forwarding, route stripping, upstream trust) is one of the most reliable sources of high-severity findings. OpenTelemetry is included because (a) Apex can read traces to infer the request graph in whitebox mode, and (b) it gives us a realistic "the heapdump is in S3 because Actuator was on" moment.

## Stack Details

**Backend**

- Kotlin 1.9.24, JVM 21 (Temurin)
- Spring Boot 3.3.2 (deliberately one minor behind current; pinned to allow the Jackson and SpEL stories to land cleanly)
- Spring Security 6.3.x
- Spring Cloud Gateway 4.1.x (reactive)
- spring-kafka 3.2.x, Kafka 3.7 (single-broker via Redpanda for the demo)
- Jackson 2.15.x with `enableDefaultTyping` enabled in one service (intentional)
- jjwt 0.11.5 used incorrectly on purpose (see Auth Model)
- Flyway 10.x against PostgreSQL 16
- HikariCP, default pool sizes
- Resilience4j for the merchant API rate limiter (mis-keyed on purpose)
- OpenTelemetry Java agent 2.5.x exporting OTLP to a local Tempo + Grafana stack
- Micrometer + Prometheus scrape on `/actuator/prometheus`
- Gradle 8.7, Kotlin DSL

**Frontend**

- React 19.0, Vite 5.4, TypeScript 5.5
- TanStack Query 5, React Router 6
- Zustand for client state
- Recharts for the net-worth chart
- A bespoke `apiClient` that stores the access JWT in `localStorage` (intentional smell — the audience should notice)

**Infra**

- Docker Compose for the demo: `gateway`, `auth-svc`, `accounts-svc`, `transactions-svc`, `merchant-api`, `categorizer-worker`, `postgres`, `redpanda`, `tempo`, `grafana`, `web`
- A second compose profile, `prod-like`, that flips `management.endpoints.web.exposure.include=*` and binds Actuator to `0.0.0.0:8081` — this is the "we deployed staging config to prod" mistake
- Seed scripts that simulate ~18 months of transaction history for 12 demo users and 3 merchant tenants

**Auth provider**

- Self-hosted `auth-svc` issuing HS256 JWTs (intentional — see the JWT finding). No external IdP, no SSO. Refresh tokens stored in Postgres, rotated on use, with a deliberately weak invalidation path.

**Observability**

- OTel traces in Tempo, metrics in Prometheus, dashboards in Grafana
- Logs to stdout, structured JSON, with a log line that accidentally includes raw `Authorization` headers from one filter (Low finding)

## Architecture

```
                          +-----------------------------+
                          |   React 19 SPA (Vite)       |
                          |   web.vaultline.local       |
                          +--------------+--------------+
                                         |
                                         | HTTPS, Bearer JWT in header
                                         v
                          +-----------------------------+
                          |  Spring Cloud Gateway       |
                          |  - JWT pre-filter (broken)  |
                          |  - rate limit per IP        |
                          |  - forwards X-User-Id (!)   |
                          +------+----------+-----------+
                                 |          |
            +--------------------+          +-----------------+
            |                    |                            |
            v                    v                            v
   +-----------------+  +-------------------+      +----------------------+
   |   auth-svc      |  |  accounts-svc     |      |  merchant-api        |
   |  /auth/*        |  |  /accounts/*      |      |  /v1/aggregations/*  |
   |  HS256 JWTs     |  |  /transfers/*     |      |  per-tenant API key  |
   |  refresh table  |  |  Postgres: core   |      |  Postgres: shared    |
   +--------+--------+  +---------+---------+      +-----------+----------+
            |                     |                            |
            |                     |  Kafka: transactions.raw   |
            |                     +-------------+              |
            |                                   v              |
            |                       +------------------------+ |
            |                       |  categorizer-worker    | |
            |                       |  consumes raw,         | |
            |                       |  produces categorized  | |
            |                       |  uses Jackson default  | |
            |                       |  typing (intentional)  | |
            |                       +-----------+------------+ |
            |                                   |              |
            v                                   v              v
        +-----------------------------------------------------------+
        |                      PostgreSQL 16                        |
        |  schemas: auth, core, merchant                            |
        |  Flyway migrations show the schema's archaeology          |
        +-----------------------------------------------------------+
```

The gateway terminates TLS, parses the JWT, and (this is the bug) sets an `X-User-Id` header from the JWT's `sub` claim before forwarding upstream. Upstream services trust `X-User-Id` blindly because "the gateway already validated it." When the JWT validator is broken (see Critical findings), this trust collapses across every service.

`categorizer-worker` consumes `transactions.raw`, runs a small rule engine, and re-emits onto `transactions.categorized`, which `accounts-svc` consumes to update balances and feed the SPA over SSE. The rule engine accepts a JSON payload that includes a `@class` discriminator — this is the Jackson polymorphic deserialization vector.

The merchant API runs as a separate Spring Boot app with its own `RequestMappingHandlerMapping`. It shares the `core` Postgres schema with `accounts-svc` so merchants can pull aggregated transaction data — and that shared access is exactly where tenant isolation gets broken.

## Data Model

Three Postgres schemas, one database. Schema separation is real (different Flyway locations per service) but the database is single — so a "logical" tenant boundary that should require IPC actually only requires a `SET search_path`. That is itself part of the H3 story.

**`auth` schema**

- `users(id uuid pk, email citext unique, password_hash text, mfa_secret text null, created_at, last_login_at)`
- `refresh_tokens(id uuid pk, user_id uuid fk, token_hash text, revoked_at timestamptz null, expires_at)`
- `sessions(id uuid pk, user_id, ip inet, ua text, created_at)`

**`core` schema (the money)**

- `accounts(id uuid pk, user_id uuid fk, institution text, mask text, balance_cents bigint, currency char(3), linked_at)`
- `transactions(id uuid pk, account_id uuid fk, posted_at, amount_cents bigint, merchant_name text, raw_descriptor text, category text, mcc int, tenant_visibility text[])`
- `transfers(id uuid pk, from_account_id, to_account_id, amount_cents, status text, requested_at, settled_at)`
- `link_tokens(id uuid pk, user_id, expires_at, used_at)` — Open Banking link simulator
- `categorization_rules(id uuid pk, user_id uuid null, expression text)` — used by the SpEL bug

**`merchant` schema (B2B side)**

- `tenants(id uuid pk, name, api_key_hash, plan text, created_at)`
- `tenant_users(id, tenant_id, email, role text)` — roles: `owner`, `analyst`, `readonly`
- `aggregation_queries(id, tenant_id, dsl text, last_run_at)`
- `data_share_consents(id, user_id, tenant_id, scope text[], granted_at, revoked_at)`

**Sensitive fields**: `password_hash` (Argon2id, work factor deliberately conservative `m=19456,t=2,p=1` so we can argue about it on screen if needed), `mfa_secret` (TOTP base32, stored unencrypted at rest — Medium-grade smell flagged by Apex but not promoted to a finding because key management is out of scope for the headline), `refresh_tokens.token_hash`, `accounts.balance_cents`, the full `transactions` row including `raw_descriptor` (which contains merchant identifiers and partial card masks), `tenants.api_key_hash`, and all PII columns on `users` (`email`, `last_login_at`, `last_login_ip`).

**Indexing**: there is a `btree(account_id, posted_at desc)` on `transactions` and a partial index `where revoked_at is null` on `refresh_tokens`. Mention is in scope because Apex's whitebox mode reads Flyway migrations and will surface that the `tenant_visibility` column is `text[]` with a GIN index — material to H3, because the `LIKE` filter does not use the index and the query plan reveals the bypass shape.

**Multi-tenancy model**: end-users live in the `core` schema and are scoped only by `user_id` on each row. Merchant tenants live in the `merchant` schema and are scoped by `tenant_id`. The merchant aggregation queries SELECT from `core.transactions` filtered by a JOIN through `data_share_consents`. The intentional bug is that the JOIN predicate is built with a string `LIKE` on `tenant_visibility` and is bypassable.

There is no row-level security policy on `core.transactions` (we considered adding RLS that's misconfigured, decided against it — keeps the H3 reasoning crisp on camera). The merchant DB user has SELECT on `core.transactions` directly, with no view in between; this is exactly the kind of "we'll add the view in Q3" debt that ends up in real fintech postmortems.

## Key Routes / Surfaces

### End-user API (via gateway)

| Method | Path | Purpose | Auth | Notable params |
|---|---|---|---|---|
| POST | `/auth/register` | Create user | none | `email`, `password` |
| POST | `/auth/login` | Issue access + refresh | none | returns JWT (HS256), sets refresh cookie |
| POST | `/auth/refresh` | Rotate tokens | refresh cookie | reads cookie only |
| POST | `/auth/mfa/enroll` | TOTP enroll | bearer | returns secret in response body |
| GET | `/accounts` | List user's linked accounts | bearer | none |
| GET | `/accounts/{id}` | Account detail | bearer | `{id}` |
| GET | `/accounts/{id}/transactions` | Tx history (IDOR target) | bearer | `from`, `to`, `limit` |
| POST | `/accounts/link` | Begin Open Banking link | bearer | `institution_id` |
| POST | `/accounts/link/callback` | Finish link | link_token | `link_token`, `external_account_id` |
| POST | `/transfers` | Initiate internal transfer | bearer | `from`, `to`, `amount_cents` (race target) |
| GET | `/transfers/{id}` | Transfer detail | bearer | `{id}` |
| GET | `/categorization/rules` | List rules | bearer | none |
| POST | `/categorization/rules` | Create rule | bearer | `expression` (SpEL target) |
| GET | `/search/transactions` | Search | bearer | `q` (SpEL target) |

### Merchant API

| Method | Path | Purpose | Auth | Notable params |
|---|---|---|---|---|
| POST | `/v1/aggregations/query` | Run aggregation DSL | API key + tenant | `dsl`, `scope` (cross-tenant target) |
| GET | `/v1/aggregations/{id}` | Fetch result | API key + tenant | `{id}` |
| GET | `/v1/consents` | List consents | API key + tenant | filter |
| POST | `/v1/webhooks` | Register webhook | API key + tenant | `url` (SSRF-adjacent, Medium) |

### Internal / ops surfaces (should be private, intentionally aren't)

| Method | Path | Purpose | Auth | Notable |
|---|---|---|---|---|
| GET | `/actuator/env` | Spring env dump | none in `prod-like` | exposes secrets |
| GET | `/actuator/heapdump` | Heap binary | none | leaks tokens, hashes |
| GET | `/actuator/loggers` | Log level config | none | DoS via DEBUG flip |
| GET | `/actuator/mappings` | Route map | none | recon goldmine |
| POST | `/internal/categorize/replay` | Replay raw event | shared secret in header | Jackson default typing target |

## Auth Model

**Sessions**: stateless on the API, stateful refresh in Postgres. Access tokens are JWT HS256, signed with a 32-byte secret loaded from `app.jwt.secret`. Tokens are 15 minutes; refresh is 30 days, rotates on use. Refresh tokens stored as SHA-256 hashes — but the rotation path has a TOCTOU window where the old token isn't revoked until after the new one is issued.

**Token validation**: implemented in a custom `JwtAuthenticationFilter` in `gateway` and re-implemented in `auth-svc`. Three intentional flaws stacked:

1. The gateway's filter calls `Jwts.parser().setSigningKey(secret).parse(token)` (the unsafe `parse`, not `parseClaimsJws`) — on this jjwt version with this code path, an `alg: none` token whose signature segment is empty parses successfully and yields claims. The filter then trusts the `sub` claim.
2. The signing secret is the literal string `"vaultline-dev-secret-change-me"` baked into `application-prod.yml` and also dumped by `/actuator/env`. So even when the parser is fixed, the secret being world-readable means HS256 forgery is trivial.
3. The `iss` and `aud` claims are not checked. A token issued by `auth-svc` for the end-user audience is accepted by `merchant-api` and vice versa, which is part of the H3 chain.

**Roles**:

- End-user app: only `USER`. There is an undocumented `ADMIN` role that exists in the JWT claims schema and unlocks `/internal/*` if present.
- Merchant API: `owner`, `analyst`, `readonly`. Role is read from the JWT's `roles` claim by `merchant-api`, which trusts the gateway-forwarded `X-Tenant-Id` header without re-checking against the JWT.

**MFA**: TOTP via `mfa_secret`. The enroll endpoint returns the raw secret in the response body, and the verify endpoint accepts the secret itself as a fallback "recovery" path (Medium finding).

**SSO**: not implemented. We acknowledge this in the demo as "this is what real fintechs deploy six months later, and that migration is when the next batch of bugs lands" — sets up a future episode.

**API key model (merchant side)**: 32-byte random tokens, stored as SHA-256 hashes in `tenants.api_key_hash`. Sent by clients as `X-Tenant-Key`. The gateway validates the key, derives `tenant_id`, and forwards both `X-Tenant-Id` and `X-Tenant-Role` headers. `merchant-api` re-checks neither against any signed claim, which is the seed of H3.

**Password reset**: uses a JWT with a `purpose: reset` claim, signed with the same secret. Same secret reuse across purposes is itself a smell (Apex flags it Medium and it gets bundled under the C1/C2 narrative; it does not get its own findings row to keep the registry tight).

## Intentional Vulnerabilities

Eleven anchored findings across severities. Each is reproducible against the seeded demo data and each maps to public prior art so Apex's writeups feel grounded.

### Critical

| # | Bug class | Location | Exploit storyline | Real-world parallel | Expected Apex output |
|---|---|---|---|---|---|
| C1 | Broken JWT validation (alg confusion / unsigned accept) | `gateway/src/main/kotlin/.../JwtAuthenticationFilter.kt`, the `validate()` method using `Jwts.parser().parse()` | Attacker forges a token with `alg: none` and `sub` set to any user UUID; gateway accepts it, sets `X-User-Id`, every upstream service trusts it. Full account takeover of any user. | The 2015 jwt-none family of disclosures (Auth0 advisory) and the recurring "JWT parser accepts unsigned tokens" pattern that has hit multiple Spring shops on HackerOne. | Severity Critical, CVSS ~9.8. Evidence: filter source, a forged token cURL, response showing another user's accounts. |
| C2 | Spring Boot Actuator exposed in prod (`/env`, `/heapdump`) | `accounts-svc/src/main/resources/application-prod.yml` with `management.endpoints.web.exposure.include=*` and `management.server.address=0.0.0.0` | Unauthenticated `GET /actuator/env` leaks `app.jwt.secret`, DB creds, Kafka SASL. `GET /actuator/heapdump` returns a 200MB binary; Apex's heap-string scan finds live JWTs and refresh-token hashes. Combined with C1, attacker forges valid-signed tokens for anyone. | The well-documented Actuator-leaks-secrets pattern: many disclosed bug bounty reports against Spring shops, and Veracode/Snyk write-ups documenting `/heapdump` as a credential exfil vector. | Severity Critical, CVSS ~9.1. Evidence: 200 response on `/actuator/env`, key strings extracted from heapdump, chained TOCTOU showing forged signed JWT. |
| C3 | Jackson polymorphic deserialization | `categorizer-worker/.../EventDeserializer.kt` using `ObjectMapper().enableDefaultTyping()`; reachable via `POST /internal/categorize/replay` (shared-secret-only, secret leaked by C2) | Attacker posts a JSON event with `@class` set to a known gadget. RCE in the worker container, which has Postgres creds and Kafka publish rights — full data plane compromise. | CVE-2017-7525 and the long tail of Jackson default-typing CVEs (2019, 2020) that hit fintech and identity vendors. The pattern remains in the wild because teams keep `enableDefaultTyping` for "convenience." | Severity Critical, CVSS ~9.8. Evidence: code path trace, gadget class chosen, reverse shell or file-write proof in worker, blast-radius mapping. |
| C4 | SpEL injection in search/categorization | `accounts-svc/.../SearchController.kt` evaluating `q` via `SpelExpressionParser`; same parser used in `POST /categorization/rules` | Authenticated user submits `q=T(java.lang.Runtime).getRuntime().exec(...)`. Arbitrary code execution inside `accounts-svc`, which holds the master Postgres role. | CVE-2022-22963 (Spring Cloud Function SpEL) and CVE-2022-22965 (Spring4Shell) reasoning — different root causes, but the "user-controlled string reaches SpelExpressionParser" archetype is the same and continues to appear in Spring app bounty disclosures. | Severity Critical, CVSS ~9.8. Evidence: payload, code path from controller to parser, executed command output. |

### High

| # | Bug class | Location | Exploit storyline | Real-world parallel | Expected Apex output |
|---|---|---|---|---|---|
| H1 | IDOR on account transactions | `accounts-svc/.../TransactionsController.kt`, `GET /accounts/{id}/transactions` reads `id` from the path and queries by `account_id` without checking ownership against `X-User-Id` | Logged-in attacker iterates UUIDs (or pulls them from C2's heapdump) and reads any user's transaction history. | The Venmo/Robinhood-era IDOR disclosures and the long line of fintech HackerOne IDOR reports against `/accounts/{id}` style routes. | Severity High, CVSS ~7.5. Evidence: two-account replay showing victim's tx history returned, controller diff highlighted. |
| H2 | TOCTOU race on `/transfers` | `accounts-svc/.../TransferService.kt` reads balance, validates, then debits in two separate transactions with no `SELECT ... FOR UPDATE` | Attacker fires N concurrent transfers totaling >balance. Several settle. Net-negative balance; free money. | Public race-condition writeups against several neobanks and crypto exchanges; the "double-spend via concurrency" archetype is recurrent on bug bounty programs. | Severity High, CVSS ~8.1. Evidence: concurrent request burst, final balance < 0, DB log showing interleaved reads. |
| H3 | Cross-tenant isolation break in merchant API | `merchant-api/.../AggregationService.kt` builds the consent JOIN with a `LIKE` on `tenant_visibility` and trusts gateway-forwarded `X-Tenant-Id`; the `scope` body param is also concatenated into the visibility filter | Tenant A submits a query with a crafted `scope` that matches Tenant B's visibility tag; or rotates `X-Tenant-Id` directly through the merchant route, which the merchant service does not re-verify against the JWT. | The recurring "B2B data API leaks across tenants" pattern: Plaid-style fintech APIs have had disclosed incidents around consent scope misenforcement; aggregator APIs more broadly have had multiple HackerOne reports. | Severity High, CVSS ~8.2. Evidence: two tenants seeded, cross-read demonstrated, query log, JOIN predicate annotated. |
| H4 | Gateway header trust / `X-User-Id` spoof on internal port | upstream services bound on `0.0.0.0` in compose; if attacker reaches the upstream port directly (via SSRF in H5 or container egress), they can set `X-User-Id` themselves | Attacker uses webhook SSRF (M2) to reach `accounts-svc:8080` directly, sets `X-User-Id: <victim>`, reads anything. | The "trusted header injection" pattern documented in Spring Cloud Gateway hardening guides and in disclosed incidents where reverse proxies were the only line between clients and upstreams. | Severity High, CVSS ~7.5. Evidence: SSRF chain, direct upstream call, victim data. |

### Medium

| # | Bug class | Location | Exploit storyline | Real-world parallel | Expected Apex output |
|---|---|---|---|---|---|
| M1 | Refresh token rotation TOCTOU / replay window | `auth-svc/.../RefreshService.kt` issues new token before revoking old; revoke happens in a separate tx | Attacker who briefly captures a refresh cookie (e.g., from logs in L1) can rotate it twice in parallel, getting two valid lineages. | Public writeups of OAuth refresh rotation flaws at SaaS vendors; Auth0 has documented similar TOCTOU traps. | Severity Medium, CVSS ~6.5. Evidence: parallel refresh requests both succeed. |
| M2 | SSRF via merchant webhook registration | `merchant-api/.../WebhookController.kt` `POST /v1/webhooks` accepts any URL, then validates with a sync HEAD; allow-list missing | Tenant points webhook to `http://accounts-svc:8080/actuator/env` (in `prod-like`) or to the cloud metadata IP in cloud variant. | The classic Capital One SSRF-to-metadata pattern (2019 disclosure) and the recurring "webhook URL becomes SSRF" findings on fintech bounty programs. | Severity Medium, CVSS ~6.8. Evidence: HEAD response captured by victim service, internal data echoed. |
| M3 | MFA enroll leaks raw TOTP secret + bypass via "recovery" | `auth-svc/.../MfaController.kt` returns `secret` in body; `verify` accepts the secret as a one-time bypass | Attacker who reads the enroll response (e.g., via XSS or logs) bypasses MFA forever. | The "MFA recovery code is the secret" anti-pattern that's appeared in multiple disclosed audits. | Severity Medium, CVSS ~6.1. Evidence: response body shows secret, verify accepts it. |
| M4 | Mass assignment / Jackson `@JsonIgnore` missing on `User` | `auth-svc/.../UserController.kt` `PATCH /me` binds the request body straight to the `User` JPA entity; `role` field is settable | User patches their own row to `role: ADMIN`, gaining the latent admin claim used by `/internal/*`. | The recurring Spring/Jackson mass-assignment disclosures across SaaS vendors. | Severity Medium, CVSS ~6.5. Evidence: PATCH body, subsequent `/internal/*` access. |

### Low

| # | Bug class | Location | Exploit storyline | Real-world parallel | Expected Apex output |
|---|---|---|---|---|---|
| L1 | Authorization headers logged in plaintext | `gateway/.../LoggingFilter.kt` logs full request headers in `prod-like` profile | Anyone with log access (operators, log-shipping vendor) sees JWTs. | A long line of "we logged the bearer token" disclosures; common enough to appear in OWASP Logging guidance. | Severity Low, CVSS ~3.7. Evidence: log line with `Authorization: Bearer ...`. |
| L2 | Verbose error responses leak stack traces | global `@ControllerAdvice` returns full stack trace when `app.debug=true`, which is on in `prod-like` | Recon: stack traces reveal internal paths, library versions, package layout. | Standard CWE-209 finding; appears in nearly every pentest. | Severity Low, CVSS ~3.1. Evidence: induced 500 with full trace. |
| L3 | Front-end stores access JWT in `localStorage` | `web/src/lib/apiClient.ts` | XSS (none seeded, but pattern is risky) escalates to token theft. | Standard OWASP guidance violation; documented in countless React app audits. | Severity Low/Info, CVSS ~3.5. Evidence: storage screenshot, recommendation. |

That gives 4 Critical, 4 High, 4 Medium, 3 Low — fifteen real bugs, eleven anchored, all reachable in the seeded demo. The required anchor classes (IDOR, broken JWT, Actuator, race, Jackson polymorphic, SpEL, cross-tenant) are covered as C1, C2, C3, C4, H1, H2, H3.

**Chain map for the storyline.** The narrator should be able to draw arrows on screen between findings:

- C2 leaks `app.jwt.secret` → upgrades C1 from "alg: none forgery" to "fully signed forgery, indistinguishable from legitimate."
- C2 leaks the `internal.replay.secret` → unlocks C3 reachability from the public network.
- C1 → H4 (with a forged JWT, the attacker can also set `X-User-Id` directly, doubling their options).
- M2 SSRF → H4 by reaching an upstream port that trusts gateway headers.
- M4 mass-assignment → access to `/internal/*` even without C1.
- L1 logging → M1 refresh replay (logged refresh tokens become the seed material).

Apex's findings registry surfaces these chains explicitly; the judge agent uses them to bump severity on individual findings when the chain materially expands blast radius. This is one of the things the headline episode needs to make legible to viewers.

## Real-World Parallels

We deliberately want the demo's narration to be able to say "this is what happened to X, and Apex would have caught it." Five public incidents in the same industry, real, no fabrication:

1. **Capital One, 2019** — SSRF against EC2 metadata via a misconfigured WAF/proxy led to ~100M records exfiltrated. Mirrored by M2 + the gateway/upstream trust chain.
2. **Equifax, 2017** — unpatched Apache Struts (CVE-2017-5638) gave RCE on a public-facing app. Same family of "user-input string reaches an evaluator" as our SpEL story (C4) and Jackson (C3).
3. **Venmo public-by-default transaction feed (multiple researcher disclosures, 2018-2019)** — IDOR-adjacent: even authorization-correct systems can leak by design. Mirrors H1 in spirit.
4. **Robinhood and other neobank race-condition writeups (various, public on bug bounty disclosure feeds)** — concurrent transfer / withdrawal races leading to negative balances. Mirrors H2.
5. **Spring4Shell / CVE-2022-22965 and CVE-2022-22963** — SpEL/data-binding RCE in Spring; widely exploited. Mirrors C4.
6. **Jackson default-typing family, CVE-2017-7525 onward** — long tail of polymorphic deserialization RCEs across Java services. Mirrors C3.

## Apex Features Showcased

- `/pentest` end-to-end as the headline run. The video opens with a single command and ends with a reviewed findings registry.
- `--threat-model` flag pointed at a checked-in `THREAT_MODEL.md` in the target repo. The threat model lists "trusted gateway headers," "Open Banking link callback," and "merchant aggregation queries" as crown-jewel paths. Apex prioritizes those; we show in voiceover that the JWT and cross-tenant findings come out first because the model said so.
- Whitebox vs blackbox compare on the same target. We run blackbox first (no source mount) — Apex finds H1, H2, C2, M2 from the outside. Then we run whitebox with the repo mounted and Apex additionally finds C1, C3, C4, M4, M1 by reading source. Side-by-side findings registry on screen.
- Findings registry with CVSS scoring. We hover over each finding to show the vector string and the judge's notes; the judge suppresses two false-positive Jackson hits in modules that don't actually use `enableDefaultTyping`.
- Sub-agent swarm. Show three swarm members in parallel: one driving the merchant API, one fuzzing `/transfers`, one reading Flyway migrations to map the schema.
- Browser/Playwright tools. Used for the SPA login + linking flow before the API attacks start, and for the `localStorage` evidence in L3.
- JS endpoint extraction. Pulled from the React bundle to discover `/internal/categorize/replay` referenced in dev tooling left in the build (intentional artifact).
- Patching agent. Climax of the video: Apex proposes a patch for C1 (swap `parse` for `parseClaimsJws`, add explicit alg allowlist), re-runs, finding closes.
- Persistent memory across engagements (`~/.pensar/`). We mention that on a second run a week later, Apex remembers the Actuator port and the gateway header trust shape, and re-checks them first.
- Kali container tools. Show `nuclei` and `ffuf` invocations for the actuator discovery step, but framed as Apex orchestrating them rather than the operator typing them.
- Headless CLI for CI/CD. Closing shot: a GitHub Actions tab where the same `/pentest` runs on every PR with `--threat-model` and fails the build on Critical findings.

## Demo Storyline

**Cold open (0:00 - 0:30).** A clean shot of the VaultLine SPA. A user logs in, links a "Chase" account, and watches transactions stream in. Voiceover: "This is the kind of app that holds the financial life of millions of people. We didn't write the bugs — we wrote the kind of bugs every fintech ships and forgets." Cut to a single terminal: `apex /pentest https://vaultline.local --threat-model ./THREAT_MODEL.md --whitebox ./repo`.

**Reveal (0:30 - 3:00).** Apex spins up its swarm. Threat model is read first; we narrate the three crown-jewel paths it picks. Blackbox-only run finishes first in a side window — H1 (IDOR), H2 (race), C2 (Actuator), M2 (SSRF) lit up. Cut to the whitebox window: Apex is reading `JwtAuthenticationFilter.kt` and finds the unsafe `parse()` call. The finding lands with full CVSS, evidence, and a forged-token cURL. We watch the swarm pivot — once C1 is in the registry, every other auth-touching finding gets re-scored upward because the JWT can be forged.

**Climax (3:00 - 5:00).** The Jackson polymorphic deserialization chain. Apex chains C2 (leaks the internal shared secret from `/actuator/env`) into C3 (posts a crafted `@class` payload to `/internal/categorize/replay`) and gets code execution in the categorizer worker. Findings registry shows the chain with Apex's judge notes confirming reachability. Then the patching agent proposes the Jackson allowlist change, applies it in a sandbox branch, and Apex re-runs that single check — the finding closes on screen.

**Close (5:00 - 6:00).** Final findings registry: 4 Critical, 4 High, 4 Medium, 3 Low. The merchant cross-tenant break (H3) is highlighted last because it's the one most fintechs don't even test for. Voiceover: "This is one command, one app, one afternoon. Now imagine the engagement you've been postponing for two quarters." Cut to the GitHub Actions tab where the same run is now wired into CI on the demo repo. Hard out.

## Build Notes

**Effort estimate.** 4-5 engineer-days for one senior backend who has shipped Spring before. Day 1: scaffold Gradle multi-module, gateway + auth-svc + accounts-svc skeletons, Postgres + Flyway, JWT happy path. Day 2: transactions, transfers, Kafka categorizer, merchant-api skeleton. Day 3: React SPA, Open Banking link simulator, seed data, Docker Compose with both `dev` and `prod-like` profiles. Day 4: wire in every intentional vuln, write reproduction scripts for each, write `THREAT_MODEL.md`. Day 5: dress rehearsal — run Apex against it twice and tune until findings come out clean and ordered the way the storyline needs.

**Scaffolding approach.** One Gradle root with five subprojects: `gateway`, `auth-svc`, `accounts-svc`, `merchant-api`, `categorizer-worker`. Shared `common` module with DTOs and the (deliberately bad) JWT helper. React app in `web/` as its own Vite project. All wired together in `docker-compose.yml` with named volumes for Postgres and Redpanda. A `Makefile` with `make dev`, `make prod-like`, `make seed`, `make demo-reset` so the operator can rewind the target between takes.

**Key deps.** `spring-boot-starter-web`, `-security`, `-actuator`, `-data-jpa`, `spring-cloud-starter-gateway`, `spring-kafka`, `flyway-core`, `flyway-database-postgresql`, `jjwt-api/impl/jackson` 0.11.5, `jackson-databind` 2.15.4, `kotlin-reflect`, `resilience4j-spring-boot3`, `micrometer-registry-prometheus`, `opentelemetry-javaagent`. Front end: `react@19`, `react-router@6`, `@tanstack/react-query@5`, `zustand`, `recharts`, `vite@5`.

**Seed data.** 12 demo users with distinct realistic transaction streams (groceries, payroll, subscriptions, occasional large transfers); 3 merchant tenants (a loyalty platform, a BNPL underwriter, an SMB accountant). Six months of history pre-seeded, plus a live `seed-stream` job that pushes fresh transactions onto Kafka every 5 seconds during recording so the SPA looks alive. `transactions.tenant_visibility` deliberately includes overlapping tags between tenants for the H3 demonstration. Pre-create one user with `role=ADMIN` already set (for fallback) and one without (so we can show M4 escalation live).

**Infra.** Single-host Docker Compose; everything fits in 8GB RAM. Tempo + Grafana included so the whitebox run can show OTel traces in voiceover. A `prod-like` overlay (`docker-compose.prod-like.yml`) flips Actuator exposure and `app.debug=true`. The recording rig should run on the `prod-like` profile.

**Repo layout.**

```
vaultline/
  build.gradle.kts
  settings.gradle.kts
  gateway/                    # Spring Cloud Gateway, JWT pre-filter (intentionally broken)
  auth-svc/                   # users, sessions, refresh, MFA
  accounts-svc/               # accounts, transactions, transfers, search, categorization rules
  merchant-api/               # tenant aggregation API, webhooks
  categorizer-worker/         # Kafka consumer, Jackson default-typing target
  common/                     # shared DTOs and the bad JWT helper
  web/                        # React 19 + Vite SPA
  infra/
    docker-compose.yml
    docker-compose.prod-like.yml
    grafana/, tempo/, prometheus/
  seed/
    seed-users.sql
    seed-tenants.sql
    seed-stream.kt           # rolling Kafka producer for live takes
  THREAT_MODEL.md            # consumed by Apex's --threat-model flag
  Makefile
  README.md                  # innocuous-looking, no security warnings
```

A second top-level file `EXPECTED_FINDINGS.md` lives outside the repo (in `demos/notes/`) and lists each intentional bug, its location, and the exact reproduction Apex should produce. The recording engineer uses this as the rubric for whether a take is good. It is never visible on camera.

**Threat model file.** `THREAT_MODEL.md` should be ~80 lines, written in the voice of a fintech security engineer who knows what matters but is overworked. It identifies three crown-jewel paths: (1) the gateway's trust of forwarded headers, (2) the Open Banking link callback flow, (3) the merchant aggregation query path. It does not mention Actuator, Jackson, or SpEL — the demo lands harder when Apex finds those without being told to.

## Recording Notes

**Length target.** 6 minutes total. Cold open 30s, reveal 2:30, climax 2:00, close 1:00. We need to stay under 6:30 for the headline cut; a longer 10-minute "deep dive" recut will be produced from the same take for the docs site.

**Taglines.**

- "One command. Your money. Their bugs."
- "Apex finds what your last pentest missed before lunch."
- "We didn't write the bugs. We wrote the agent that finds them."

**Gotchas.**

- Run `make demo-reset` between takes. The Kafka topic and the transfers race state will drift; the race finding looks better on a fresh DB.
- Confirm clock sync between gateway and auth-svc containers before each take. JWT `exp` skew has bitten previous demos.
- The heapdump scan can take 10-15 seconds. Cut the audio around it or use a B-roll of the swarm.
- Do not let the SPA auto-refresh during the C1 forged-token scene — the SSE channel will reconnect with the real token and the demo will look confusing. Disable auto-refresh in the demo build flag `VITE_DEMO_FREEZE_SSE=1` for that scene.
- Make sure `/actuator/env` is reachable on `localhost:8081` from the recording host but firewalled from the open internet on the demo box. This is a money-app demo with leaky endpoints by design; we do not want it indexed.
- The patching agent scene needs a sandbox git branch ready. Pre-create `apex/patch-sandbox` in the demo repo so the patch lands without auth prompts on camera.
- Lower terminal font to 14pt and bump line height; the findings registry has a lot of rows in the close shot and we want them legible at 1080p.
