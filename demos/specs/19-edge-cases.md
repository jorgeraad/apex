# Edge-Case and Hostile Targets — Apex Under Pressure

> Series role: A six-scenario combined spec covering the demos where Apex stops being a happy-path tool and starts being a credible practitioner. Every scenario here exists to defuse one specific objection a viewer or buyer is likely to raise after watching the headline episodes.
>
> Read order: top to bottom is fine on first pass; the scenarios are independent and you can jump to whichever objection your conversation actually surfaces. Scenario 5 (the hardened Rust target) is the most-quoted single scenario in sales conversations because it directly answers "does this thing fabricate findings to look useful."
>
> Audience: this doc is written for the demo-production team, the technical marketing lead, and security-engineering reviewers vetting the scaffolds. It is not customer-facing.

## Why this combined doc

The first run of the Pensar Apex demo series (VaultLine, MedVault, Atlas, PipelineIQ, PeoplePort, ThreadOps) leans toward target apps that are realistic but cooperative. They have real bug classes, but they are not actively trying to mislead an attacker, and they are not hardened to a degree that would make a junior pentester give up. That choice was correct for the headline run: viewers needed to see Apex find serious bugs, fast, and explain them well.

The follow-up problem, which this combined doc answers, is that buyers and senior practitioners now ask the next questions:

- "What happens when there is a WAF?"
- "What happens when half the logic is in the browser bundle?"
- "What happens when SSO and MFA are enforced?"
- "Will it hammer my production target into oblivion?"
- "Will it invent findings on a clean codebase to justify itself?"
- "Can it tell a real bug from something that just looks like one?"

These are the six scenarios in this spec, in that order. Each one is a short-form demo (8-14 minutes of finished video) intended to be released between the larger headline episodes. Each is built on top of an existing scaffold from the series — we are not building six new apps from scratch — and each one spotlights a specific subset of Apex features under conditions that the headline episodes did not stress.

The combined release proves four things together that no single headline episode can:

1. **Apex adapts.** WAFs, rate limits, CAPTCHA, and SSO are not show-stoppers; they are inputs the agent reasons about.
2. **Apex restrains itself.** It paces against rate limits, it respects scope, and it does not blindly retry past hCaptcha.
3. **Apex does not fabricate.** On a well-built target it returns a near-empty report and says so plainly.
4. **The judge agent works.** Findings that look real but are not get filtered out before they reach the human reviewer.

Each scenario also functions as a sales-cycle artifact for a different objection persona:

| Scenario | Objection it addresses | Audience persona |
|---|---|---|
| 1. WAF | "Our edge will block whatever you throw at it." | Network/infra security |
| 2. SPA | "Most of our app is in the browser; you can't model it." | Frontend platform leads |
| 3. SSO/MFA | "We're enterprise; you'll never get past the login wall." | IAM / identity engineers |
| 4. Rate limits | "We can't have you hammering production." | SRE, platform ops |
| 5. Hardened | "AI tools fabricate findings to look useful." | Senior AppSec, CISOs |
| 6. Planted-herring | "Static analysis is a noise machine; you'll be the same." | Security engineers |

Recording cadence, base-scaffold reuse, and shared infra notes are at the bottom of this doc.

---

## Scenario 1 — WAF-Fronted ShopHearth

### Premise & Stack

ShopHearth is the Laravel 11 + Livewire e-commerce scaffold from the series' Shopify-shaped episode: storefront, cart, checkout, admin, a coupon engine, a product-search endpoint, and a few REST/JSON endpoints used by the mobile app. For this demo we deploy ShopHearth unchanged behind a real Cloudflare zone with the WAF enabled at "high" sensitivity, plus a self-hosted ModSecurity reverse proxy running OWASP CRS 4.x in front of the origin. Cloudflare rate limiting is set to 100 req/min/IP on `/login`, `/checkout/*`, and `/api/*`. The origin Laravel app still has the same intentional bugs as the original ShopHearth episode (a couple of stored XSS vectors, a SSRF in the import-from-URL tool, an IDOR on order PDFs, a SQLi on a less-trafficked admin search).

Why a layered edge: real e-commerce shops rarely run only one defense. They run a CDN WAF for volumetric and signature-based attacks, then a self-hosted reverse proxy with CRS for finer-grained policy, then app-level mitigations underneath. We mirror that exact structure so the demo is recognizable to anyone who has shipped a production storefront.

| Layer | Choice |
|---|---|
| Origin | Laravel 11, PHP 8.3, Livewire 3, MySQL 8 |
| Edge | Cloudflare (free tier WAF + Bot Fight Mode off, Managed Ruleset on) |
| Reverse proxy | nginx + ModSecurity 3 + OWASP CRS 4.3 (paranoia level 2) |
| Rate limit | Cloudflare rules: 100 rpm /api, 30 rpm /login, 60 rpm /checkout |
| TLS | Cloudflare-managed cert, origin pull cert |

### Adversarial Twist

Every classic payload Apex would normally try first — `' OR 1=1 --`, `<script>alert(1)</script>`, `../../etc/passwd`, `{{7*7}}` — gets a 403 from Cloudflare or a 406 from ModSecurity before the origin ever sees it. The demo is not about defeating the WAF. It is about Apex correctly reading the situation: distinguishing "the WAF blocked me" from "the app is not vulnerable," then choosing whether to (a) try a different encoding, (b) move to a less monitored endpoint, or (c) note the WAF as a defense-in-depth control and move on.

A subtle secondary point: a WAF can produce false confidence. The audience should leave understanding that "we have a WAF" is not a substitute for fixing the bugs underneath, and Apex's report makes the underneath/edge distinction explicit by listing each finding's status as `bypassed`, `blocked`, or `partially-bypassed`.

### Routes or Surfaces of Interest

| Route | Why Apex cares |
|---|---|
| `POST /login` | Heavily WAF-monitored; tight rate limit. Test ground for "do I retry?" |
| `GET /search?q=` | First place classical XSS/SQLi payloads die at the edge |
| `POST /admin/products/import-url` | SSRF candidate; less common path, lighter CRS coverage |
| `GET /orders/{id}/invoice.pdf` | IDOR candidate; pure ID enumeration, no WAF signature |
| `GET /api/v1/products?filter=` | JSON API; CRS rule set behaves differently on JSON bodies |
| `POST /coupons/redeem` | Race-condition candidate; rate limits make naive parallel attacks fail |
| `GET /sitemap.xml`, `robots.txt`, `/build/manifest.json` | Recon surfaces the WAF doesn't touch |
| `GET /livewire/livewire.js`, `/livewire/update` | Livewire-specific endpoints; CRS does not have specific rules |
| `POST /api/v1/customer/profile` | Mass-assignment candidate on a low-traffic JSON endpoint |

### Intentional Vulnerabilities

| ID | Class | Reachability under WAF |
|---|---|---|
| SH-WAF-1 | SSRF in admin import-URL tool | Reachable; CRS does not block outbound URL submission |
| SH-WAF-2 | IDOR on `/orders/{id}/invoice.pdf` | Fully reachable; no payload signature to match |
| SH-WAF-3 | Stored XSS via product review (admin-rendered) | Reachable with double-URL-encoded `<svg onload>` (CVE class CWE-79) |
| SH-WAF-4 | Blind SQLi in `admin/search?q=` | Reachable only via time-based payloads with comment obfuscation |
| SH-WAF-5 | Race condition in coupon redemption | Constrained by rate limits; partial exploit |
| SH-WAF-6 | Mass assignment on `/api/v1/customer/profile` | Reachable; JSON bodies are less aggressively inspected by CRS at PL2 |

References: behavior models for CRS bypass align with [CVE-2023-38199](https://nvd.nist.gov/vuln/detail/CVE-2023-38199) (CRS body-parser quirks) and the broader CRS evasion research from Coreruleset's own test suite. For Cloudflare-specific bypass research, see the public test corpus at [https://github.com/coreruleset/coreruleset](https://github.com/coreruleset/coreruleset) and Cloudflare's own changelog for Managed Ruleset versioning.

### Apex Features Showcased

- **Adaptive payload encoding.** Apex notices repeated 403/406 responses with `cf-ray` or `Mod_Security` markers and shifts strategy: URL-encoding, mixed-case keywords, comment-splitting (`/*!50000UNION*/`), JSON unicode escapes, multipart-bypass tricks.
- **WAF fingerprinting.** It reads response headers and error pages to identify Cloudflare vs. ModSecurity vs. origin Laravel and adjusts.
- **Endpoint pivot reasoning.** When `/search` is a brick wall, Apex deprioritizes it and moves to `/admin/products/import-url` and the JSON API, both of which have lighter coverage.
- **Scope guard tool.** The demo deliberately includes an `*.cdn.shophearth.example` host that the agent could pivot to but must refuse because it is out of scope.
- **Findings + judge agent.** The judge downgrades the SQLi finding from "confirmed" to "suspected" because every probe was edge-blocked and the only signal is timing through a CDN, which is noisy.
- **Memory across runs.** Apex remembers, from a prior unprotected ShopHearth engagement, that the import-URL tool was historically vulnerable, and biases probing toward it earlier in the run when it sees the WAF on the front door.

### Demo Storyline

The episode opens with the operator pointing Apex at `https://shophearth.example` in blackbox mode with a one-line scope file. The first 30 seconds are the agent walking the public surface and immediately eating WAF blocks; we cut to the Apex log stream where it says, in essence, "the edge is Cloudflare, the inner proxy is ModSecurity CRS 4.x, classical payloads will not survive — switching to behavior-based probes and lower-signal endpoints." Over the next several minutes Apex finds the IDOR (no payload, just enumeration), the SSRF in the import tool (the WAF doesn't treat outbound URLs as hostile), the stored XSS (lands a payload that survives CRS through double encoding), and reports the blind SQLi as suspected with calibrated confidence.

The middle of the demo is structured around three on-screen "decisions" the agent verbally narrates: (1) "this 403 is the WAF, not the app; do not retry the same payload," (2) "this 406 includes a `Mod_Security` rule ID — encoding shift will likely succeed because the rule matches a literal string," and (3) "this endpoint has thinner CRS coverage than `/search`; pivot here." Each decision is a teachable moment for viewers learning how to think about WAF-fronted targets.

The last beat is the judge agent's pass: it confirms three of the findings, explicitly downgrades the SQLi to "investigate further with origin access," and marks the WAF and rate limiter as compensating controls in the report's "Defenses Observed" section. The closing shot is the report's "Bypassed" column versus "Blocked" column — concrete evidence that Apex distinguishes the two.

### Build Notes

- Reuse the existing ShopHearth Laravel scaffold in full; no app-side changes.
- New work: a `docker-compose.waf.yml` overlay that adds nginx + ModSecurity + CRS, and a Cloudflare zone setup script (`./scripts/cloudflare-up.sh`) using the API to apply the rule set against a demo subdomain.
- Effort: ~3 engineer-days, mostly Cloudflare rule tuning and CRS paranoia calibration so that the storyline beats land deterministically.
- Recording: pre-warm Cloudflare cache and CRS rule cache before each take to keep timing consistent.
- Have a deterministic "WAF-block fixture" suite under `tests/waf/` that asserts each of the five intentional bugs is still reachable and each of the canonical payloads is still blocked, so a CRS rule update doesn't silently change the episode.
- Document the exact CRS paranoia level and Cloudflare ruleset version used in the README so re-recordings six months later can reproduce the same behavior.
- Build a `compose.local.yml` that runs ModSecurity locally without Cloudflare for offline development; switch to the Cloudflare-backed compose only for the actual recording.
- Pre-stage a "no-WAF" run for B-roll so the editor can intercut "with WAF" and "without WAF" report screens for the closing montage.
- Capture both the Cloudflare ruleset audit log and the ModSecurity `audit.log` for each take and ship them as supporting artifacts alongside the video, for any viewer who wants to verify the blocks were real.

---

## Scenario 2 — Pure Client-Side SPA (Vue 3 + Pinia)

### Premise & Stack

"Cartographer" is a freight-routing dashboard for a fictional logistics SaaS. Almost all of the interesting product logic — pricing surcharges, role gating on which screens render, validation of shipment manifests, even some authorization decisions — lives in the Vue 3 SPA bundle. The backend is a deliberately thin Fastify + PostgreSQL REST API that mostly persists what the client tells it to persist, with shallow per-request JWT verification and not much else. The marketing pitch is "instant UI, edge-first." The reality is a fat JS bundle with source maps accidentally shipped to production and a backend that trusts the front end.

This is a deliberately archetypal modern stack. We see it constantly: a small team adopts Vue or React with Pinia/Zustand, scaffolds a thin Express/Fastify/Hono backend, and ships features by writing client logic that "calls the API." The role checks and the validations migrate, one feature at a time, into client stores. The backend never catches up. By Series B, half the authorization model lives in `src/stores/`. This demo is for that team.

| Layer | Choice |
|---|---|
| Frontend | Vue 3.5, Pinia 2, Vue Router 4, Vite 6 |
| Build | Vite with `build.sourcemap = true` (left on by mistake) |
| State | Pinia stores: `useAuthStore`, `usePricingStore`, `useFleetStore`, `useAdminStore` |
| Backend | Node 22, Fastify 5, Prisma 5, PostgreSQL 16 |
| Auth | JWT in `Authorization: Bearer`; backend verifies signature only, not claims-to-action mapping |
| Hosting | Vercel for SPA, Fly.io for API |

### Adversarial Twist

There is no server-rendered HTML to crawl. There are no `<a href>` links to most of the API. A naive web spider sees `/`, `/login`, and `/dashboard` and concludes the app has six routes. The real attack surface is in the JS bundle, in the source maps, and in tree-shaken-but-not-removed code paths that compile in but only fire if a Pinia store has a certain shape. The demo proves Apex's `extractJsEndpoints` plus source-map archaeology can recover endpoints that no UI navigation ever reaches.

A second, related twist: the SPA does *correct* client-side validation in many places — formats, ranges, regex checks. A reviewer skimming the bundle could mistake those for security controls. Apex must read them as UX hints, not enforcement, and probe the backend for whether the same constraints exist there.

### Routes or Surfaces of Interest

| Surface | Visible from UI? | How Apex finds it |
|---|---|---|
| `GET /api/shipments` | Yes | Standard nav |
| `POST /api/shipments` | Yes | Form action |
| `POST /api/admin/users/:id/role` | No | Found in `useAdminStore.ts` via source map |
| `POST /api/pricing/override` | No | Referenced only in dead-feature-flagged branch |
| `GET /api/internal/_debug/whoami` | No | String literal in bundle, never called |
| `POST /api/fleet/:id/decommission` | No | Behind a `v-if="user.role === 'super'"` guard, but the endpoint exists |
| `GET /api/exports/csv?token=` | No | Referenced in a worker chunk lazy-loaded for admins |
| `PATCH /api/shipments/:id` | Partially | Visible action, but accepts more fields than the form sends |
| `GET /api/_health/deep` | No | String in service-worker chunk only |

### Intentional Vulnerabilities

| ID | Class | Notes |
|---|---|---|
| CG-SPA-1 | Client-side-only role check on `super_admin` actions | Backend `/api/admin/users/:id/role` does not re-check role |
| CG-SPA-2 | Pricing override endpoint with no server-side authz | Discovered only via bundle analysis (CWE-602) |
| CG-SPA-3 | Source maps shipped to prod | Reveals internal module structure, comments, dev TODOs |
| CG-SPA-4 | Hardcoded internal API token | In `apiClient.ts` for the `/api/exports/csv` worker, scoped too broadly |
| CG-SPA-5 | JWT `aud` not validated | A token issued for the marketing site verifies on the API |
| CG-SPA-6 | Mass assignment via `PATCH /api/shipments/:id` | Backend spreads request body into Prisma `update.data` |

CWE references: CWE-602 (Client-Side Enforcement of Server-Side Security), CWE-540 (Inclusion of Sensitive Information in Source Code), CWE-798 (Use of Hard-coded Credentials), CWE-915 (Improperly Controlled Modification of Dynamically-Determined Object Attributes).

### Apex Features Showcased

- **`extractJsEndpoints` against a Vite bundle.** Walks the chunk graph, parses minified and unminified output, recovers fetch call sites and their literal/template URLs.
- **Source-map archaeology.** When `.map` files are present, Apex reconstructs the original `useAdminStore.ts`, reads developer comments, identifies endpoints referenced but not yet wired into UI.
- **Tree-shaken-but-included reasoning.** Apex notices that `pricingOverride()` is in the bundle but unreferenced from any router-visible component, and flags this as a "dead UI, live backend" case worth probing.
- **Pinia store inference.** Apex models the auth/role store and proposes payload-shape mutations that would convince components a user is `super_admin`, then tests whether the *backend* enforces the same thing (it doesn't).
- **Whitebox upgrade.** When the operator drops the source repo into the working dir, Apex correlates bundle artifacts back to source files and confirms findings with greatly increased confidence.
- **Attack-surface map output.** The final report includes a printable PDF appendix listing every recovered endpoint with provenance: which chunk it appeared in, which UI component (if any) calls it, whether it was reachable from any router state.

### Demo Storyline

The episode opens with the operator giving Apex a single URL: `https://cartographer.example`. The agent navigates the SPA and notes that there are only three router-visible pages, then immediately switches to bundle analysis. We cut to a split screen: the rendered app (six visible buttons) and Apex's discovered-endpoint table (twenty-three endpoints, several flagged "no UI path"). Apex picks `POST /api/admin/users/:id/role` first, signs in as a regular user, and demonstrates that the call succeeds with a flipped role — the front end never offered the button, but the back end never checked. It then chains to the pricing override (silently changes a freight rate by 80%), and finally pulls the source maps to retrieve a hardcoded internal token that grants CSV export across tenants.

A teaching beat in the middle of the demo: Apex shows its work on the bundle. We watch it parse `chunk-DEZ3.js`, identify the chunk as the admin lazy-loaded chunk via Vite's manifest, walk the source map back to `src/stores/admin.ts`, and surface a developer comment reading `// TODO: server-side enforce — for now FE only`. That single comment is the Pinia mistake distilled into one screenshot, and it is the moment the audience understands why client-heavy architectures are a distinct attack surface that pre-LLM scanners cannot reason about.

The closing beat reframes the whole demo: this is not an exotic class of bug, it is the most common modern SPA mistake, and traditional dynamic scanners that crawl rendered HTML cannot find any of it. The judge agent confirms all three findings as high-confidence because each was reproduced with a direct request, and the report includes a "Surfaces invisible to crawlers" section listing every endpoint Apex recovered from the bundle.

### Build Notes

- New scaffold; budget ~5 engineer-days (Vue + Pinia + Fastify + auth wiring + intentional bugs + seed data).
- Reuse the JWT helpers and Prisma seeding patterns from VaultLine to save time.
- Keep the bundle size realistic (300-500 KB gzipped) — too small and the "needle in a haystack" framing falls flat.
- Ship a `prod-broken` Docker profile with `build.sourcemap = true` so the source-map beat is reproducible.
- Include a second Vite config (`vite.config.locked.ts`) with source maps off and the role-check moved server-side, so the README can show the same target in "fixed" form for educational purposes.
- Seed at least two tenants and three roles (`viewer`, `dispatcher`, `super_admin`) so the cross-tenant CSV export finding has a concrete blast radius the report can quote.
- Add a unit test under `tests/bundle/` that asserts the bundle contains references to all six intentional endpoints; if a future code change tree-shakes one out, CI fails and the recording isn't accidentally invalidated.
- Optional follow-up: a "before and after" recap video that re-runs Apex on the *fixed* version (server-side role checks, no source maps, scoped JWT) and shows the report dropping to zero findings.

---

## Scenario 3 — SSO + MFA Enterprise Target (Spring Boot + Okta SAML + WebAuthn)

### Premise & Stack

"Northwind Procurement" is a fictional enterprise SaaS for B2B procurement workflows: vendor onboarding, RFP management, contract approvals. Every customer enforces SSO via Okta SAML, and end-user step-up auth uses WebAuthn (FIDO2) passkeys for any action that touches a contract. There is no password login at all once SSO is provisioned; service accounts use mTLS-bound API keys. This is the most enterprise-y target in the series: a Fortune-500-shaped login flow with branding, just-in-time provisioning, group-claim-based authorization, and a passkey requirement on sensitive actions.

We chose Spring Boot 3 deliberately because Spring Security's SAML 2 service-provider support is one of the most widely deployed enterprise SAML implementations in the world, and its quirks are economically real. WebAuthn4J was chosen because it is the actively maintained Java FIDO2 library with the cleanest test surface for virtual authenticators. Okta was chosen as the IdP because it is the most common enterprise IdP in mid-market SaaS and its dev tier is free.

| Layer | Choice |
|---|---|
| Backend | Spring Boot 3.3, Spring Security 6.3, Java 21 |
| SSO | Okta (real free-tier dev tenant) via SAML 2.0 |
| MFA | WebAuthn / FIDO2 via `webauthn4j` 0.25 |
| Frontend | Next.js 15 (App Router) |
| DB | PostgreSQL 16, Flyway migrations |
| Session | Spring Session + Redis |
| Authorization | Group claims from SAML mapped to Spring roles; method-level `@PreAuthorize` |

### Adversarial Twist

The login wall is real and unforgiving. Apex cannot brute-force its way past Okta — and shouldn't try, because Okta is out of scope and a serious customer-trust violation if it did. The demo is about Apex's `authenticationAgent` walking a *legitimate* auth flow with a test account: completing the SAML AuthnRequest/Response handshake, registering a passkey via the WebAuthn ceremony, persisting the resulting session/cookie state, and only *then* starting authenticated testing. We also need to be honest about what Apex cannot do — phishing the user, bypassing the IdP, or testing Okta itself.

A particularly thorny demo design challenge: WebAuthn assertions are bound to the relying-party origin and cannot be replayed across origins. Apex's virtual authenticator must register against the actual demo origin, not a proxy. We document this constraint explicitly because it is the single most common reason WebAuthn pentests "don't work" — and it is a feature of the spec, not a bug in the tool.

### Routes or Surfaces of Interest

| Route | Notes |
|---|---|
| `GET /` | Public marketing landing |
| `GET /sso/login` | Initiates SAML AuthnRequest to Okta |
| `POST /sso/acs` | SAML ACS endpoint — primary attack surface for SAML bugs |
| `GET /webauthn/register` | Passkey registration ceremony |
| `POST /webauthn/assert` | Passkey assertion for step-up |
| `POST /api/contracts/{id}/approve` | Step-up-required action |
| `GET /api/vendors` | Standard authenticated read |
| `POST /api/admin/groups/sync` | Privileged; group-claim-derived role |
| `GET /actuator/info` | Should be locked down; verify |

### Intentional Vulnerabilities

| ID | Class | Notes |
|---|---|---|
| NW-SSO-1 | SAML signature wrapping (XSW) | Legacy `OpenSAML` config accepts unsigned `Assertion` inside signed `Response`; analogous to [CVE-2017-11427](https://nvd.nist.gov/vuln/detail/CVE-2017-11427) class |
| NW-SSO-2 | Group claim trust without re-validation | If `groups` claim contains `procurement-admin`, server grants admin without checking JIT-provisioned mapping |
| NW-SSO-3 | WebAuthn challenge reuse window too long (5 min, no single-use) | Allows replay within window |
| NW-SSO-4 | Session not rebound on step-up | Step-up assertion accepted, but session-id rotation skipped, enabling fixation chain |
| NW-SSO-5 | `/actuator/info` exposed with build metadata | Low-severity info-disclosure |
| NW-SSO-6 | RelayState parameter unsigned and reflected on `/sso/acs` | Open-redirect after auth; CWE-601 |

References: [CVE-2017-11427](https://nvd.nist.gov/vuln/detail/CVE-2017-11427) (Shibboleth/OneLogin SAML XSW), [CVE-2024-6387](https://nvd.nist.gov/vuln/detail/CVE-2024-6387) is unrelated but commonly confused — do not cite. WebAuthn challenge handling: see W3C WebAuthn Level 3 §5.1.3.

### Apex Features Showcased

- **`authenticationAgent` driving real SAML.** Walks the redirect to Okta in a Playwright session, types the test-account credentials, handles the Okta verify push (or TOTP fallback for the demo account), follows the ACS POST back, and stores the resulting Spring Session cookie.
- **WebAuthn registration via the Playwright virtual authenticator.** Uses Chromium's WebAuthn DevTools protocol to register a software passkey, then exercises step-up flows.
- **Authenticated SAML response tampering.** Once Apex has a baseline working session, it captures a real signed Response and probes XSW variants in a controlled way against a *test-tenant* endpoint only.
- **Scope-aware restraint.** Apex explicitly refuses to test `*.okta.com` even though it has credentials; the report has a "Out of scope, not tested" section listing the IdP.
- **Limitations called out in the report.** A dedicated "Caveats" section lists what Apex did *not* test: IdP-side flows, real users' devices, recovery codes, Okta admin console.
- **Session re-binding awareness.** Apex captures the session cookie before and after step-up and checks for rotation; the lack of rotation is a finding the agent only catches because it is tracking session state across the whole flow.
- **Group-claim trace.** When Apex finds the group-claim trust bug, it traces the claim end-to-end: SAML attribute, JIT mapping, Spring `GrantedAuthority`, `@PreAuthorize` evaluator. The report includes the trace as evidence.

### Demo Storyline

The episode opens with the operator pointing Apex at Northwind and providing a single test-account credential file (Okta username + password + TOTP seed for the test account, plus a virtual-authenticator profile). Apex's `authenticationAgent` takes over: we watch a Playwright window navigate to `/sso/login`, redirect to Okta, complete the form, satisfy MFA, and bounce back to Northwind. In the second beat the agent encounters its first protected action (`/api/contracts/.../approve`), receives a 401-step-up, registers a passkey via the virtual authenticator, and continues. From here the demo runs like a normal whitebox/blackbox session: it finds the SAML XSW via tampering with a captured response on the test tenant, the group-claim trust bug via JIT manipulation, the challenge-replay issue, and the actuator info-disclosure.

A standout shot in the middle of the demo: a real Okta `verify push` notification arrives on the operator's test phone during the AuthnRequest, and Apex's log stream pauses with a status line reading `awaiting external MFA approval (timeout 60s)`. The operator approves on-camera. This single sequence does more for buyer trust than any technical claim — it concretizes the agent's relationship to human-in-the-loop authorization. Without that approval, Apex stops; with it, Apex proceeds; at no point does it pretend it can satisfy MFA on its own.

The closing two minutes are deliberately uncomfortable: we show Apex's "Caveats and Limits" section verbatim, in which it states that it did not and cannot test Okta itself, it did not phish anyone, it did not test recovery flows, and that a complete enterprise pentest must include the IdP via separate authorization. This honesty is the point — the demo is at least as much about what a responsible AI pentester *won't* do as about what it can.

### Build Notes

- New Spring Boot scaffold; budget ~6 engineer-days including Okta dev-tenant setup, SAML wiring, and WebAuthn4J configuration.
- Bring up a real Okta dev tenant (free) with a single test app and two test users. Document the seed in the repo so demos are reproducible.
- The XSW finding requires a careful, narrow vulnerability window — easy to make accidentally unexploitable. Add an integration test that confirms the bug is live before each recording.
- Recording: pre-record the Okta login screen interaction or use a stable test account; do not show real Okta admin UI.
- Include a `northwind-saml-fixtures/` directory with captured-and-redacted SAML responses so the XSW beat can be re-run offline for editing without re-driving Okta each time.
- Wire the WebAuthn virtual authenticator via Chrome DevTools Protocol's `WebAuthn.addVirtualAuthenticator`; document the exact `protocol`, `transport`, and `hasResidentKey` flags used so the recording is reproducible.

---

## Scenario 4 — Rate-Limited and CAPTCHA-Protected VaultLine Variant

### Premise & Stack

This demo reuses the entire VaultLine fintech scaffold from Episode 1, with two additions: hCaptcha (enterprise tier, with the "passive" mode for clean traffic) gating `/login`, `/register`, and `/transfers/initiate`; and Bucket4j-backed token-bucket rate limits on every authenticated endpoint, keyed by user ID with per-tenant quotas. The intentional bugs from Episode 1 are still there — JWT alg confusion, the Actuator heap-dump issue, the cross-tenant filter — but the path to them is now gated.

This is the most "boring" of the six episodes by design. The bugs are not new. The stack is not new. The audience has already met VaultLine. The whole point is to show that when a real fintech turns on the boring defenses everyone in the industry has agreed are table stakes — CAPTCHAs on auth surfaces, token-bucket rate limits on the API — Apex behaves like a polite citizen of that environment rather than a runaway scanner.

| Layer | Choice |
|---|---|
| Base | VaultLine scaffold from Episode 1 |
| CAPTCHA | hCaptcha Enterprise on `/login`, `/register`, `/transfers/initiate`, `/password/reset` |
| Rate limit | Bucket4j 8.x with Redis-backed buckets; 60 rpm authenticated, 10 rpm unauth |
| Edge | Spring Cloud Gateway sidecar applies rate limits before app filters |

### Adversarial Twist

This is not "can Apex defeat hCaptcha." Apex must not defeat hCaptcha — bypassing CAPTCHAs unsolicited would be a serious series-credibility hit. The demo is about restraint: Apex detects the CAPTCHA, recognizes 429 patterns from Bucket4j, paces itself, asks the operator for a session token if it needs authenticated probing past the wall, and explicitly refuses to brute force or solve CAPTCHAs without human authorization. The buyer fear we are addressing is "this thing will hammer my prod target into the ground."

The deeper adversarial twist is that Bucket4j here is mis-keyed in places (per Episode 1's original bugs), so naive request-rate observation produces inconsistent signals: some endpoints rate-limit per IP, others per user, others per tenant. A well-paced agent has to infer the keying scheme rather than assume one global limit, and the demo shows Apex doing that inference on camera.

### Routes or Surfaces of Interest

| Route | Gate | Apex behavior |
|---|---|---|
| `POST /login` | hCaptcha + rate limit | Single attempt with provided creds; no brute force |
| `POST /register` | hCaptcha + rate limit | Skip; flag as auth surface to test only with operator opt-in |
| `POST /transfers/initiate` | hCaptcha + step-up + rate limit | Test only with authenticated session; one transaction per minute |
| `GET /api/v1/accounts` | Rate limit only | Stay below 50 rpm conservatively |
| `GET /actuator/heapdump` | Rate limit + auth | Single request; back off if 429 |
| `POST /api/v1/merchant/data` | Per-tenant bucket | Detect tenant-scoped throttling; pace per-tenant |
| `POST /password/reset` | hCaptcha + rate limit | Skip; only operator-opt-in testing |
| `GET /api/v1/transactions` | Per-user bucket | Sample first; infer bucket from `X-Rate-Limit-*` |

### Intentional Vulnerabilities

The vulnerabilities themselves are unchanged from Episode 1 — JWT alg confusion, Actuator heapdump exposure, cross-tenant filter bypass, the merchant rate-limiter mis-keying, and the IDOR on transaction history. The full list and their CVE-class references can be read in `01-vaultline-fintech.md`. The point of *this* episode is not "what is wrong with VaultLine"; it is "how does Apex behave when reaching VaultLine's known bugs is gated by hCaptcha and Bucket4j."

### Scenario Variant

The variant is the *interaction model*:

- **CAPTCHA detection.** Apex parses hCaptcha widget markup and `siteverify` redirects, identifies the presence of a CAPTCHA without solving it, and flags the protected endpoint accordingly.
- **Rate-limit detection.** Apex distinguishes 429s from Bucket4j (`X-Rate-Limit-Remaining`, `Retry-After`), 403s from WAF, and origin 503s. It uses exponential backoff keyed on `Retry-After`.
- **Pacing.** Apex internally caps request rate to a configurable fraction of the observed limit (default 50% of `X-Rate-Limit-Limit`) to leave headroom for legitimate users during the test.
- **Operator-in-the-loop.** When Apex hits a CAPTCHA-gated endpoint that it would need to test, it pauses and asks the operator to either (a) provide a pre-authenticated session, (b) explicitly authorize a CAPTCHA-bypass test, or (c) skip the surface and note it.
- **Adaptive to per-tenant buckets.** Apex notices that hitting `/api/v1/merchant/data` quickly consumes `tenant_X` quota but not `tenant_Y` and shifts probes accordingly.

### Apex Features Showcased

- Pacing engine with per-host and per-endpoint budgets.
- 429/`Retry-After` honoring; exponential backoff with jitter.
- CAPTCHA fingerprinting (hCaptcha, reCAPTCHA v2/v3, Turnstile, FunCaptcha — even though only hCaptcha is in use here).
- Operator prompts for human authorization on CAPTCHA-gated surfaces.
- Final report's "Defenses Observed" and "Test Pacing" sections, with concrete metrics (avg requests/min, max sustained, total requests).
- Configurable "production-safe mode" that hard-caps sustained request rate at 10 rpm, disables any probe class flagged as `disruptive`, and refuses CAPTCHA-gated surfaces without explicit per-engagement opt-in.
- A pre-engagement "test plan" output that the operator can review before the run begins, listing every endpoint Apex intends to probe and the maximum request count budgeted for each. The operator can edit the plan and re-confirm.

### Demo Storyline

The episode opens with a graph: total requests Apex sent during the original VaultLine episode versus this run. The new run is one-third the volume and ten times slower wall-clock — and that is the point. We watch Apex hit `/login`, get a CAPTCHA, recognize it, back off, and ask the operator for a logged-in session. The operator pastes in a JWT from a test account. From there Apex resumes authenticated testing, but now we see it visibly pacing — the log stream shows "sleeping 8.4s for tenant_03 bucket refill" before each call to the merchant API.

The story climaxes with a moment that would be unflattering in any other demo: Apex hits a Bucket4j 429 storm because a probe accidentally matched a per-IP bucket. It backs off, lowers its sustained rate, retries successfully, and emits a self-correcting log line: "I exceeded the tenant rate budget at 14:22:08; reduced sustained rate from 40 rpm to 18 rpm." This is the single most important shot in the episode — the buyer sees the agent failing, noticing, and recovering without operator intervention.

The closing minute is a side-by-side of two reports: the original VaultLine report and this one. The findings are nearly identical (because the bugs are identical), but the "Test Pacing" appendix in this run shows: peak rate 22 rpm, sustained rate 14 rpm, total requests across the engagement 4,128, requests rejected by Bucket4j 12, CAPTCHAs encountered 4, CAPTCHAs solved 0, operator interventions for session re-auth 2. Those numbers are the deliverable a security engineering manager forwards to the SRE team to argue that automated pentesting is safe to point at staging, and possibly production-with-guardrails.

### Build Notes

- Reuse VaultLine in full; add a `compose.cap.yml` overlay that wires hCaptcha and Bucket4j.
- hCaptcha sitekey: use the hCaptcha sandbox sitekey (`10000000-ffff-ffff-ffff-000000000001`) for development so we never accidentally hit production hCaptcha quotas.
- Effort: ~2 engineer-days.
- Recording: capture the same target twice — once without limits, once with — and intercut the request-rate graphs.
- Include a `bucket4j-config-tour.md` that walks through the per-IP, per-user, and per-tenant bucket definitions, so viewers who pause the video can read the actual configuration.
- Add a regression test under `tests/pacing/` that asserts Apex's observed-from-outside request rate stays under a configured ceiling for the full recording window.
- Wire request-rate metrics into Prometheus and dashboard them in Grafana for the recording. The on-screen request-rate chart needs to be live and accurate, not a static screenshot.
- Stage two pre-authenticated session JWTs (one regular user, one merchant tenant admin) so the operator can paste the right one in without fumbling on camera.
- Optional bonus: a third take in which the operator deliberately raises the rate ceiling to "test what happens when production-safe mode is off" — a teaching moment for buyers who want to understand the difference between production-safe and full-throttle modes.

---

## Scenario 5 — Hardened "Honest" Rust + Axum Target

### Premise & Stack

"Lighthouse" is a fictional small-team SaaS for IoT device telemetry — a deliberately small surface, deliberately well-built. Two senior engineers wrote it in Rust over six months. They picked the right primitives, used them correctly, and didn't take shortcuts. There are essentially no real bugs. Apex's job is to confirm that and say so without inventing findings to justify itself. This episode is **critical to the series' credibility** — without it, every other report Apex produces is suspect.

| Layer | Choice |
|---|---|
| Language | Rust 1.80 (stable) |
| Web | Axum 0.7 + Tower middleware |
| DB | sqlx 0.8 with compile-time checked queries (no string concatenation) |
| Auth | argon2id password hashing (`argon2` crate, OWASP recommended params), session cookies bound to user-agent and IP CIDR |
| Authorization | A typed `AuthContext<Role>` extractor; handlers literally cannot compile without declaring required role |
| TLS | Caddy in front, modern TLS profile, HSTS preload-eligible |
| CSP | Strict CSP with nonces; no `unsafe-inline`, no `unsafe-eval`, `default-src 'self'` |
| Headers | All the boring ones done right: `Strict-Transport-Security`, `X-Content-Type-Options`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy: ()`, `Cross-Origin-Opener-Policy: same-origin` |
| Templating | `askama` (compile-time templates, auto-escape on by default) |
| Logging | `tracing` with structured fields; PII redaction filter |
| Deps | `cargo-audit` clean; `cargo-deny` configured |

### Adversarial Twist

There is nothing to find. Or rather — there is exactly one minor info-disclosure (a `/healthz` endpoint that returns the git SHA in plaintext, which is borderline acceptable in many shops). Everything else is genuinely correct. SQL is parameterized at compile time. Auth context is a Rust type that the compiler enforces. CSP is strict. Sessions rotate on auth-state change. Argon2id parameters meet 2025 OWASP guidance. A junior pentester running a checklist will turn in a report full of noise; Apex must not.

### Routes or Surfaces of Interest

| Route | Notes |
|---|---|
| `GET /` | Static landing |
| `POST /login` | argon2id verify; rate limited; constant-time response |
| `POST /devices` | Create device (auth required, scoped to org) |
| `GET /devices/:id/telemetry` | Read; sqlx-checked, RLS-enforced |
| `POST /api/v1/ingest` | Device-token-authenticated ingest |
| `GET /healthz` | Returns git SHA — minor info disclosure |
| `GET /metrics` | Behind `127.0.0.1:9000` listener, not exposed externally |

### Intentional Vulnerabilities OR Decoys

There is exactly one finding to report:

| ID | Class | Severity |
|---|---|---|
| LH-INFO-1 | `/healthz` returns build git SHA | Informational |

That is the entire findings table. Everything else Apex investigates must be marked *investigated and dismissed*, with reasoning. Examples of things that look promising but aren't:

| Surface checked | Why it's not a bug |
|---|---|
| Session cookie sent over HTTP locally in dev | Not the prod config; HSTS + Secure + SameSite=Strict in prod |
| `Authorization` header in some logs | `tracing` redaction filter strips before sink |
| `format!("...{}", user_input)` in a SQL-looking spot | It's a `tracing` span, not a query |
| A `Code.eval`-shaped function name | Rust has no eval; the function is a stub for a wasm sandbox in a feature-flagged module |
| Apparent open redirect on `/back?to=` | Strict allowlist of relative paths; rejects external |

### Apex Features Showcased

- **Negative-result reporting.** Apex emits a structured "Investigated and Dismissed" section with concrete reasoning per surface.
- **Judge agent suppression.** The judge gets a long stream of candidate findings from the swarm and rejects nearly all of them, with traceable rationale.
- **Whitebox source reading.** Apex reads Cargo.toml, dependency versions, Axum middleware stack, sqlx queries, and the `AuthContext` type to *prove* security properties rather than just probe them.
- **Honesty in the executive summary.** The summary literally says "this target is well built; the only finding is informational and at the operator's discretion."
- **Confidence calibration.** No "high" or "critical" rows. No "suspected" rows that the judge let through to pad the report.
- **Threat-model reasoning.** Apex's threat-model agent produces a clean diagram of trust boundaries (browser, edge, app, DB) and observes that the typed `AuthContext<Role>` makes one whole class of bug structurally impossible. That observation is in the report as a positive finding — not a vulnerability, but a defense worth crediting.
- **Dependency-aware audit.** Apex runs through the dependency graph and confirms there are no unpatched advisories at the time of the run; this section of the report can be regenerated on a schedule even if the app code does not change.

### Demo Storyline

The episode opens with a side-by-side: VaultLine's report (twelve findings, four critical) and Lighthouse's report (one informational). The operator points Apex at Lighthouse with both blackbox and whitebox modes enabled. We watch the swarm explore for ten minutes — extractJsEndpoints returns a tiny set, the SQLi probes all hit compile-time-checked sqlx with no observable signal, the XSS probes all die at the askama auto-escape, the auth-bypass probes all hit the typed AuthContext extractor and 403. Each negative result streams to the judge agent, which records it in the "investigated and dismissed" section.

A particularly important on-camera moment: Apex *does* surface several "I'd like to flag this" candidates from individual swarm agents — for example, the sight of `format!("...{}", user_input)` in a log line, or a `/back?to=` redirect parameter, or a session cookie observed without `Secure` in the local dev compose. The judge agent investigates each in turn: the `format!` is a `tracing::info!` macro arg, not a query; the `/back?to=` is gated by a relative-path validator; the missing `Secure` flag is a dev-profile artifact and the prod profile sets it. We watch each candidate get dismissed with a one-line citation. By the end of the run, the candidate-findings list has been four, then six, then nine, then back down to one as the judge clears them — and the one that remains is the `/healthz` git-SHA disclosure.

The closing two minutes are the executive summary: Apex states, in plain English, that the target is well built, lists what it tested, lists what it found (one informational item), and recommends the operator verify the `/healthz` git SHA exposure is acceptable for their threat model. There is no padding. There are no fabricated mediums. The voiceover lands the punch line: "If you ship Apex at a clean codebase, this is the report you get. We chose to record this one because the easiest way to lose your audience's trust is to invent findings that aren't there."

### Build Notes

- New scaffold; budget ~7 engineer-days. Most of the cost is doing it *right* — argon2id parameters, sqlx setup, typed auth contexts, strict CSP — because we cannot fake it.
- Have an external Rust security reviewer audit the scaffold and confirm there are no unintended findings before recording. Budget another 2 days for fixes.
- Pin all dependencies; run `cargo audit`/`cargo deny` in CI; fail if anything new appears, because a CVE in a dep would change the storyline.
- Keep the `/healthz` git-SHA exposure in deliberately — it gives the report exactly one row to populate, which is more honest than a fully empty report.
- Document the exact OWASP argon2id parameters used (`m=19456 KiB, t=2, p=1` as of the 2024 OWASP Password Storage Cheat Sheet) in the README so reviewers can verify them.
- Maintain a `docs/why-not-a-bug.md` file enumerating each surface a checklist scanner would flag and the precise reason it isn't a bug. The judge agent's "investigated and dismissed" output should match this file row-for-row; mismatch is a CI failure.

---

## Scenario 6 — Planted-Herring RelayChat (Judge Agent Showcase)

### Premise & Stack

This demo reuses the RelayChat Phoenix scaffold (Elixir/Phoenix-based real-time chat from a prior episode) and *adds* six deliberate constructs that look, on first inspection, like serious vulnerabilities — but are not, on careful inspection. The episode is a focused showcase of the judge agent: a swarm of probing agents will flag each construct as suspicious, and the judge will, one by one, investigate and dismiss them with cited evidence.

| Layer | Choice |
|---|---|
| Base | RelayChat Phoenix scaffold (Elixir 1.17, Phoenix 1.7, LiveView, Ecto, Postgres) |
| Plugs | Existing auth plug, plus a few new pipeline plugs that hide the safety mechanisms |
| Sanitizer | `html_sanitize_ex` 1.4 |
| JWT | `joken` 2.6 with HS256 |
| Frontend | LiveView + a sliver of plain JS for an admin console |

### Adversarial Twist

Every herring is *engineered* to look bad to a fast-reading agent or a junior pentester:

- The dangerous-looking call is right there.
- The safety mechanism is a few files away, in a plug or a pipeline, not co-located.
- The names of variables and modules are deliberately misleading: `raw_user_input`, `unsafe_eval`, `admin_panel`, `JWT_SECRET = "changeme"`.
- A grep-based scanner would file all six as findings. A senior human reviewer would dismiss all six in fifteen minutes. The judge agent must do the same.

### Routes or Surfaces of Interest

| Route or symbol | Looks like | Actually is |
|---|---|---|
| `RelayChat.Scripting.run/2` (calls `Code.eval_string`) | RCE | Behind `:authorize_admin` plug + AST allowlist via `Code.string_to_quoted` walker |
| `MessageController.preview/2` (renders `raw_user_input`) | Stored/reflected XSS | Pipeline plug `:sanitize_message_html` runs `HtmlSanitizeEx.basic_html` before controller |
| `MessageController.delete/2` (`Repo.delete!(Repo.get!(Message, id))`) | IDOR | `:require_message_owner` plug runs ahead of action |
| `MessageQueries.search/1` (uses `fragment("...")`) | Raw SQL injection | `fragment` with `?` placeholders + parameter binding |
| `Auth.JWT.secret/0` returns `"changeme"` literal at module-attribute level | Hardcoded secret | At runtime, `Application.compile_env/2` overlays from `RUNTIME_JWT_SECRET` env var; constant is a default for `mix test` only |
| `GET /admin/*` | Exposed admin | `:ip_allowlist` plug rejects everything outside `10.0.0.0/8` and the operator VPN range |

### Intentional Vulnerabilities (real bugs, clearly tagged)

We still need a small set of *real* findings so the episode has a concrete confirmed-vs-dismissed contrast:

| ID | Class | Severity | Real / Decoy |
|---|---|---|---|
| RC-REAL-1 | LiveView event handler on `phx-click="delete_room"` skips room-owner check | High | Real |
| RC-REAL-2 | Channel join authz allows pattern `room:*` for any authenticated user | Medium | Real |
| RC-DECOY-1 | `Code.eval_string` in `Scripting.run/2` | n/a | Decoy (auth + AST allowlist) |
| RC-DECOY-2 | `render(:html, raw_user_input)` in preview | n/a | Decoy (pipeline sanitizer) |
| RC-DECOY-3 | `MessageController.delete/2` IDOR shape | n/a | Decoy (ownership plug) |
| RC-DECOY-4 | `fragment("...")` in search | n/a | Decoy (parameterized) |
| RC-DECOY-5 | Hardcoded `JWT_SECRET = "changeme"` | n/a | Decoy (runtime ENV overlay) |
| RC-DECOY-6 | `/admin` exposed | n/a | Decoy (IP allowlist plug) |

CWE / CVE references for the *shapes* the decoys imitate: CWE-95 (eval), CWE-79 (XSS), CWE-639 (IDOR), CWE-89 (SQLi), CWE-798 (hardcoded secrets), CWE-284 (improper access control). Realistic counterpart CVEs include [CVE-2020-12717](https://nvd.nist.gov/vuln/detail/CVE-2020-12717) (Phoenix-related historical context) and the broader Elixir/Phoenix advisories tracked at [https://github.com/elixir-lang/elixir/security](https://github.com/elixir-lang/elixir/security).

### Apex Features Showcased

- **Judge agent end-to-end.** Six dismissals in a single episode, each with cited file:line evidence for *why* the finding was dropped.
- **Plug pipeline awareness.** Apex models Phoenix `pipeline` and `plug` declarations and recognizes that a controller action is gated by upstream plugs even if the action body itself looks dangerous.
- **Macro and AST awareness.** Apex sees `Code.string_to_quoted` plus an allowlist walker and reasons about the eval being constrained.
- **Parameterized-query recognition.** Apex distinguishes Ecto `fragment(...)` with `?`-placeholders from a true string-built query.
- **Runtime config tracing.** Apex follows `Application.compile_env`/`Application.get_env` through to runtime overlays and avoids reporting the placeholder literal as a hardcoded secret.
- **Network-control awareness.** Apex sees the `:ip_allowlist` plug and tries `/admin` from the configured "external" IP it has been told to test from, gets denied, and dismisses the "exposed admin" finding.
- **Confirmed real bugs preserved.** The two genuine bugs (RC-REAL-1 and RC-REAL-2) survive the judge and appear in the final report at correct severities.
- **Per-finding evidence packs.** Each dismissed finding gets a self-contained evidence pack (file path, line range, the exact request that was attempted, the response, the safety mechanism's rejection log line). This makes a human reviewer's spot-check fast — they can re-run any of the six dismissals and reproduce the result.

### Demo Storyline

The episode opens with a screen titled "Six bugs?" listing the six suspicious-looking constructs. The operator runs Apex; the swarm fires off candidate findings rapidly — within the first three minutes there are nine candidate findings on the table, eight of which look high-or-critical at a glance. We then cut to the judge agent's investigation phase. For each decoy, the judge produces a short paragraph: "this *looks* like X, but plug Y in pipeline Z runs first and constrains the input to W, therefore no exploit." Each dismissal cites the specific module and line. We watch the candidate-findings list shrink from nine to two as the judge works through them.

A particularly satisfying moment is the JWT decoy. A swarm agent flags `Auth.JWT.secret/0` returning `"changeme"` as a hardcoded credential. The judge investigates, opens `config/runtime.exs`, sees `config :relaychat, :jwt_secret, System.fetch_env!("RUNTIME_JWT_SECRET")`, traces the call site, and writes a one-paragraph dismissal that ends with "the literal in the source is a `mix test` default and is not the secret in use at runtime; verified by reading `runtime.exs` and confirming `RUNTIME_JWT_SECRET` is required at boot." This is exactly the kind of nuance a fast SAST tool gets wrong every time.

The closing beat is the final report: two real findings (the LiveView event handler bug and the channel-pattern authz bug), six "investigated and dismissed" entries with full reasoning, and a meta-section titled "Why this matters" that explicitly frames the credibility argument: an automated tool that flags every dangerous-looking line is a noise machine, not a pentester. The judge agent is what makes Apex's findings worth reading. As a bonus shot, we run the same target through a generic SAST tool that flags all six decoys and zero of the real bugs, just to drive the contrast home.

### Build Notes

- Reuse the existing RelayChat Phoenix scaffold; budget ~4 engineer-days to add the six decoys and the two real bugs.
- Each decoy needs a unit test that proves the safety mechanism works, committed alongside the decoy itself, so the spec stays honest.
- Add a `mix relaychat.verify_decoys` Mix task that exercises all six decoys against their safety mechanisms before each recording — a regression here would tank the episode.
- Recording: do two takes, one where the judge runs in default mode and one where it runs verbose, so the editor can intercut explanations.
- For the IP-allowlist decoy, run the recording from a network outside `10.0.0.0/8` so the deny is real on camera. Then re-record from inside the allowlist range to demonstrate the same admin route works as designed for legitimate operators.
- Keep a `decoys.md` file in the RelayChat repo that documents each decoy, the safety mechanism, and the file:line of both. The judge agent's transcript should be diff-comparable to this file at the end of the run; large diffs are a red flag for the recording.

---

## Cross-cutting Production Notes

### Reuse of base scaffolds

| Scenario | Base scaffold | New scaffold work |
|---|---|---|
| 1. WAF-Fronted ShopHearth | ShopHearth (Laravel) | None on app; new infra overlay |
| 2. Pure Client-Side SPA | New (Cartographer) | Full new app, ~5 days |
| 3. SSO + MFA | New (Northwind) | Full new app + Okta setup, ~6 days |
| 4. Rate-Limit / CAPTCHA | VaultLine | Infra overlay only |
| 5. Hardened Rust | New (Lighthouse) | Full new app, ~7 days + external review |
| 6. Planted-Herring | RelayChat (Phoenix) | Decoys + 2 real bugs, ~4 days |

Total new build: ~22 engineer-days, plus ~5 days infra/overlay work, plus ~4 days external review on the Rust target. Realistic calendar time with one engineer: ~8 weeks. With two: ~5 weeks.

### Recording cadence

- Release one of these episodes every 2-3 weeks, interleaved with the longer headline episodes.
- Order recommendation: (5) Hardened Rust first, because the credibility payoff is the highest and it raises the floor for every subsequent episode. Then (6) Planted-Herring, then (1) WAF, then (4) Rate-Limit, then (3) SSO/MFA, then (2) Pure SPA. This ordering front-loads the trust-building episodes and lets later ones lean on the credibility already earned.
- Each video: 8-14 minutes finished, with a tight 30-second cold open showing the report's headline section, then the run, then the report walkthrough.

### Shared infra

- Single `demos-edge/` directory under the repo root, with one subdirectory per scenario, each containing a `README.md`, a `docker-compose.yml`, and (where applicable) the overlay file that switches it into edge mode.
- A shared `tools/record/` directory with the recording-script generator, the request-rate graph renderer (used heavily in Scenario 4), and the report-diff tool (used in Scenario 5 to show the empty-vs-full report contrast).
- A shared `tools/verify/` directory with per-scenario "verify the bugs are still live" scripts; CI runs these on every PR so a refactor doesn't accidentally fix or break the demo bugs.
- A shared `tools/anonymize/` directory for stripping PII from any artifacts captured during recording (SAML responses, JWTs, log lines containing test-account emails) before publication.
- A shared editorial guide at `docs/editorial.md` covering tone, terminology consistency ("vulnerability" vs "finding" vs "issue"), severity rubric, and screen-recording cadence so the six episodes feel like a single series.

### Reporting consistency

All six episodes share a single report template so viewers can pattern-match across them. The template has the following sections, in this order:

1. Executive summary (3-5 sentences, written by Apex, reviewed by operator).
2. Findings table (severity, title, status, confirmed/suspected/dismissed).
3. Per-finding detail (description, evidence, reproduction, impact, remediation).
4. Investigated and dismissed (the judge agent's reasoning per non-finding).
5. Defenses observed (WAF, rate limits, CSP, typed auth, IP allowlists, etc.).
6. Caveats and limits (what was not tested and why).
7. Test pacing (request rates, total requests, operator interventions).
8. Out of scope (hosts, tenants, IdPs, third-party services).

Episodes 4 (rate limits) and 5 (hardened) lean heavily on sections 5 and 7. Episode 6 (planted-herring) leans heavily on section 4. Episode 3 (SSO/MFA) leans heavily on section 6. The template is intentionally over-specified for any single episode; the joint visual language across the series is the point.

### Series-level claims these episodes underwrite

By the end of these six episodes, the series can credibly claim — and demonstrate on tape — that Apex:

1. Adapts to edge defenses without bypassing them illegitimately (Scenario 1, 4).
2. Reasons about modern client-heavy architectures, not just request/response (Scenario 2).
3. Walks legitimate enterprise auth flows with operator-supplied credentials (Scenario 3).
4. Paces and restrains itself against rate limits and CAPTCHAs (Scenario 4).
5. Does not fabricate findings on a clean target (Scenario 5).
6. Filters look-alike vulnerabilities through a judge agent before reporting (Scenario 6).

Those six claims, each with a recorded episode behind it, are the basis on which a senior security buyer can recommend Apex internally without staking their own reputation on hand-wave marketing.

### Risks to manage during recording

- **CRS rule drift.** OWASP CRS publishes minor updates frequently; a paranoia level 2 ruleset today may behave differently in six months. Pin the version and document it in `compose.waf.yml`.
- **Okta dev-tenant policy changes.** Okta occasionally updates default password policies and MFA requirements on dev tenants. Re-verify the SAML test flow before each recording.
- **Cargo dependency CVEs.** A new CVE in a transitive dep on the Lighthouse target could turn the "clean target" episode into a "moderate finding" episode overnight. Run `cargo audit` immediately before each recording and snapshot the lockfile.
- **hCaptcha sandbox changes.** The sandbox sitekey occasionally changes its widget markup; re-verify CAPTCHA fingerprinting before each Scenario 4 take.
- **Phoenix or Ecto API changes.** A breaking change to `Code.string_to_quoted` or `fragment/1` semantics would invalidate the planted-herring decoys. Pin Elixir and Phoenix versions in the RelayChat repo and have CI fail on minor bumps.
- **Vite or Vue major-version changes.** A bundle-format change in Vite could break the chunk-graph parsing in Scenario 2. Pin Vite to a specific minor and document the bundle layout the demo depends on.
- **Apex agent behavior drift.** Across releases, the swarm's exact ordering of probes can change. The episodes should not depend on any specific *order* of finding discovery; storyline beats need to be robust to reordering.

### What this combined doc deliberately does not cover

- **Mobile app pentesting.** The series will get there, but it is its own thing — separate scaffolds, separate tooling, separate scope. Out of scope here.
- **Cloud-native infrastructure pentesting.** Kubernetes RBAC, cloud IAM, IMDS exploitation. Also a future series; out of scope here.
- **Internal-network pivots.** Apex is not, in any of these episodes, walking through internal networks. Every scenario assumes the operator runs Apex against an exposed-by-design target, with explicit scope.
- **Source-code-only static analysis as the primary mode.** Apex is a dynamic-first agent; whitebox is an enrichment, not a replacement. None of the six episodes lead with SAST behavior.
