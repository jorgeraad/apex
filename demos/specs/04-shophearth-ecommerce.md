# ShopHearth — E-commerce Storefront

> A Shopify-style multi-tenant storefront for SMB merchants, built on Laravel
> 11 + Livewire 3, with a Filament merchant admin, Stripe-backed checkout,
> and a webhook pipeline that, on close inspection, you would not bet your
> GMV on.

---

## Premise & Vibe

ShopHearth is the kind of platform a 12-engineer team ships in two years
and starts onboarding "real" merchants onto in year three. It is
deliberately mid-scale: large enough that the codebase has shape (multi-
tenant, queue workers, a real admin), small enough that several features
were obviously built by one person on a deadline and never re-reviewed.

The customer-facing storefront looks polished. Products, collections,
a cart, a checkout that hands off to Stripe, post-purchase emails,
order tracking, customer accounts. Each merchant gets their own subdomain
(`acme.shophearth.test`, `barkbox.shophearth.test`) with their own theme
and catalogue. Behind the storefront sits a Filament admin where merchants
manage products, coupons, orders, and a "reports" tab someone added in
sprint 14.

The vibe of the demo is serious. ShopHearth is not a toy app and the
bugs in it are not toy bugs. Every anchor vulnerability maps to a real
incident shape that has cost real merchants real money — coupon stacking
during a flash sale, a webhook spoof that marks unpaid orders as paid,
a price-tampering exploit that survives a YC company's first audit.

The demo's headline beat is `/operator`. Apex is not running fully
autonomous here; the operator chains three bugs by hand — coupon TOCTOU
into webhook spoof into IDOR — and the camera follows along while the
agent reasons out loud. The point of the episode is that `/operator` is
not "ChatGPT with a terminal." It is a pentest co-pilot that can hold
a multi-step exploit in its head and drive a browser while doing it.

---

## Why This Stack

PHP 8.3 + Laravel 11 + Livewire 3 is one of the most common SMB-SaaS
stacks shipping in 2026. It is over-represented in the long tail of
Shopify-alikes, BigCommerce-alikes, and white-label storefronts. If
ShopHearth's bugs land, the demo lands with thousands of teams who run
something architecturally identical.

Specific reasons for each component:

- **Laravel 11.** Modern enough that Eloquent, policies, validation, and
  the queue system are all in their current shape. Old enough that the
  ecosystem habits the demo punishes (raw DB::raw, mass-assignment laziness,
  `Auth::check()`-only middleware) are still in production code.
- **Livewire 3.** Realistic UI surface for a Laravel storefront. Lets the
  cart, the review form, and parts of the merchant admin be component-
  driven without a separate SPA. Livewire's server-rendered model also
  gives Apex a clean blackbox surface — every action is a POST to
  `/livewire/update`, easy to enumerate and replay.
- **Filament 3.** The de-facto admin for Laravel SaaS. Includes
  policy-driven authorization out of the box, which makes the *missing*
  policy on the import endpoint look that much worse.
- **MySQL 8.** Default. Used for both the central tenancy DB and per-tenant
  shop databases. The race-condition bug depends on InnoDB's default
  REPEATABLE READ + the lack of a `SELECT ... FOR UPDATE` on coupon redeem.
- **Redis + Horizon.** Queue + dashboard. Webhook deliveries, order
  confirmation emails, the SSRF-prone image importer, and Stripe webhook
  ingestion all flow through queues. Horizon is also one of the surfaces
  the swarm enumerates.
- **Stripe.** The most realistic checkout integration, and the webhook-
  signature bug we want to demo is straight out of the Stripe docs'
  "do not do this" section.
- **stancl/tenancy.** The dominant multi-tenant Laravel package. Database-
  per-tenant by default, but ShopHearth uses single-database with global
  scopes for some models — which is exactly where the cross-tenant IDOR
  on `/orders/{id}` hides.
- **nginx.** Routes wildcard subdomains to the Laravel app and serves
  static `/storage` paths. Configured to forward `Host` so tenant
  resolution works.

---

## Stack Details

| Layer            | Choice                                       | Notes |
| ---------------- | -------------------------------------------- | ----- |
| Language         | PHP 8.3                                      | OPcache + JIT enabled |
| Framework        | Laravel 11.x                                 | First-party validation, policies, queues |
| Frontend         | Livewire 3 + Alpine.js + Tailwind            | Storefront and parts of admin |
| Admin            | Filament 3                                   | Resource-driven; policies registered for most resources (not all) |
| Multi-tenancy    | stancl/tenancy v3                            | Single-DB w/ tenant_id column for storefront models; per-tenant DB only for analytics |
| DB               | MySQL 8.0                                    | InnoDB, REPEATABLE READ |
| Cache / Queue    | Redis 7 + Laravel Horizon                    | Five queues: default, webhooks, mail, imports, reports |
| Payments         | Stripe PHP SDK                               | Checkout Sessions + webhook endpoint |
| Storage          | S3-compatible (MinIO in dev)                 | Product images, theme assets |
| HTTP             | nginx 1.27                                   | Wildcard `*.shophearth.test` |
| Browser          | Chromium (image importer worker)             | Headless puppeteer-php for some imports |
| Container        | docker-compose for dev; k8s-style for staging| Single Dockerfile, Horizon as separate process |
| PHP libs of note | guzzlehttp/guzzle, league/flysystem-aws-s3, stripe/stripe-php, intervention/image, league/csv |

The `composer.json` is intentionally believable — no obviously-abandoned
packages, no eye-watering version pins. All anchor bugs are application-
level, not "outdated dependency" bugs. Apex should be finding logic flaws,
not running `composer audit`.

---

## Architecture

```
                     ┌───────────────────────────────┐
                     │  nginx (wildcard *.shophearth) │
                     └──────────────┬────────────────┘
                                    │
                ┌───────────────────┴────────────────────┐
                │                                        │
       ┌────────▼────────┐                     ┌─────────▼────────┐
       │ Laravel app     │                     │ Filament admin   │
       │ (storefront +   │                     │ (same app, /admin │
       │  Livewire)      │                     │  prefix)         │
       └────────┬────────┘                     └─────────┬────────┘
                │                                        │
       ┌────────▼────────────────────────────────────────▼────────┐
       │                      MySQL 8                              │
       │   central:  tenants, domains, users, plan_subscriptions   │
       │   shared:   products, orders, line_items, coupons,        │
       │             reviews, webhook_events, audit_logs           │
       │   (tenant_id column on all shared tables; global scope    │
       │    in TenantScope.php — applied to most models)           │
       └────────┬──────────────────────────────────────────────────┘
                │
       ┌────────▼────────┐         ┌────────────────────────────┐
       │ Redis (cache +  │◄────────┤ Horizon (queue dashboard)  │
       │ queues)         │         └────────────────────────────┘
       └────────┬────────┘
                │
   ┌────────────┼─────────────────────┬───────────────────────┐
   │            │                     │                       │
┌──▼──┐     ┌───▼───┐             ┌───▼─────┐            ┌────▼───────┐
│mail │     │webhook│             │image    │            │reports     │
│queue│     │queue  │             │import   │            │queue       │
└─────┘     │(Stripe│             │worker   │            │(DB::raw   │
            │ +     │             │(SSRF)   │            │ aggregates)│
            │outbnd)│             └─────────┘            └────────────┘
            └───────┘
                │
        ┌───────▼────────┐
        │ Stripe API     │  ← signs webhooks with Stripe-Signature
        └────────────────┘
```

Tenant resolution: an `InitializeTenancyByDomain` middleware reads the
`Host` header, looks up `domains.domain` in the central DB, sets the
current tenant, and bootstraps a global `TenantScope` on every model that
declares `BelongsToTenant`. The `Order` model declares `BelongsToTenant`.
The `Order` model's *route binding*, however, does not — the controller
that handles `/orders/{id}` calls `Order::find($id)` directly. That is
the IDOR.

Webhook flow: Stripe POSTs to `/webhooks/stripe`. A controller verifies
the signature, then dispatches a `ProcessStripeWebhook` job onto the
`webhooks` queue. The signature check is the bug. The job itself is
fine — it just trusts whatever the controller hands it.

---

## Data Model

Central database (single, shared across all tenants):

| Table                | Key columns                                                       |
| -------------------- | ----------------------------------------------------------------- |
| `tenants`            | id, slug, plan, stripe_account_id, theme_path, created_at         |
| `domains`            | id, tenant_id, domain                                             |
| `users`              | id, tenant_id, email, password, role (`customer`/`merchant`/`admin`/`super`) |
| `plan_subscriptions` | id, tenant_id, stripe_subscription_id, status, current_period_end |

Shared application tables (filtered by `tenant_id` global scope where the
model has the trait):

| Table             | Key columns                                                                              |
| ----------------- | ---------------------------------------------------------------------------------------- |
| `products`        | id, tenant_id, slug, title, description, price_cents, currency, sku, theme_html_snippet  |
| `product_images`  | id, product_id, url, alt, position, source_url (for imported)                            |
| `collections`     | id, tenant_id, slug, title                                                               |
| `coupons`         | id, tenant_id, code, kind (`percent`/`fixed`), value, max_uses, uses, expires_at         |
| `coupon_redemptions` | id, coupon_id, user_id, order_id, redeemed_at                                         |
| `carts`           | id, tenant_id, user_id (nullable), token, currency, created_at                           |
| `cart_items`      | id, cart_id, product_id, quantity, unit_price_cents (← server should re-derive; doesn't) |
| `orders`          | id, tenant_id, user_id, status, total_cents, stripe_payment_intent_id, paid_at           |
| `order_items`     | id, order_id, product_id, quantity, unit_price_cents                                     |
| `reviews`         | id, tenant_id, product_id, user_id, rating, body (rendered as raw HTML in the theme)     |
| `webhook_events`  | id, tenant_id, source, type, payload_json, signature, processed_at                       |
| `audit_logs`      | id, tenant_id, actor_id, action, target_type, target_id, ip, ua, created_at              |

Note the two columns the demo lives or dies on:

- `cart_items.unit_price_cents` — written by the cart-update endpoint
  from the *request body* in the JSON API, not re-derived from the product.
- `coupons.uses` — incremented after the eligibility check in
  `Coupon::redeem()` with no row-level lock. The check and the increment
  are two separate statements.

Per-tenant analytics DBs exist (one per tenant, created on signup) but
they are read-only mirrors used by the reports tab. They are not the
target of the demo bugs.

---

## Key Routes / Surfaces

Storefront (per-tenant, on `*.shophearth.test`):

| Method | Path                                | Purpose                                          | Auth        |
| ------ | ----------------------------------- | ------------------------------------------------ | ----------- |
| GET    | `/`                                 | Storefront home (themed)                         | public      |
| GET    | `/products/{slug}`                  | Product detail, includes reviews                 | public      |
| GET    | `/collections/{slug}`               | Collection listing                               | public      |
| GET    | `/cart`                             | Cart page (Livewire)                             | session     |
| POST   | `/api/cart/items`                   | Add line item — accepts `unit_price_cents`       | session     |
| PATCH  | `/api/cart/items/{id}`              | Update line item — accepts `unit_price_cents`    | session     |
| POST   | `/api/cart/coupon`                  | Apply coupon                                     | session     |
| POST   | `/checkout`                         | Create Stripe Checkout Session                   | session     |
| GET    | `/checkout/return`                  | Post-Stripe return                               | session     |
| GET    | `/orders/{id}`                      | Order detail page                                | login req'd |
| POST   | `/products/{slug}/reviews`          | Submit review                                    | login req'd |
| GET    | `/account`                          | Customer account                                 | login req'd |
| PATCH  | `/account`                          | Update profile (mass-assignment vector)          | login req'd |

Merchant admin (Filament, mounted at `/admin` per tenant):

| Method | Path                                | Purpose                                       | Policy?     |
| ------ | ----------------------------------- | --------------------------------------------- | ----------- |
| GET    | `/admin`                            | Dashboard                                     | yes         |
| CRUD   | `/admin/products`                   | Filament resource                             | yes         |
| CRUD   | `/admin/coupons`                    | Filament resource                             | yes         |
| CRUD   | `/admin/orders`                     | Filament resource                             | yes         |
| GET    | `/admin/reports/sales`              | Aggregate sales (uses DB::raw)                | yes (read)  |
| POST   | `/admin/import-products`            | CSV import (Auth::check only — no role check) | NO          |
| POST   | `/admin/products/{id}/import-image` | Pull image from URL (SSRF)                    | yes         |
| GET    | `/admin/themes/preview`             | Render theme with sample data                 | yes         |

Webhook + system:

| Method | Path                  | Purpose                                                  | Auth                |
| ------ | --------------------- | -------------------------------------------------------- | ------------------- |
| POST   | `/webhooks/stripe`    | Receive Stripe events; signature compared with `==`      | Stripe-Signature    |
| GET    | `/horizon`            | Horizon dashboard                                        | gated by Gate::define (works) |
| GET    | `/livewire/update`    | Livewire HTTP transport                                  | session             |

Central / super-admin (on the apex domain `shophearth.test`):

| Method | Path                    | Purpose                                |
| ------ | ----------------------- | -------------------------------------- |
| GET    | `/`                     | Marketing site                         |
| POST   | `/signup`               | Create tenant                          |
| GET    | `/super`                | Super-admin (gated by role==super)     |

The demo's three chained bugs all fire on the per-tenant storefront and
admin. The signup flow is in scope for enumeration but not for the chain.

---

## Auth Model

Three guards:

- **`web`** — session-based, used by the storefront and Filament admin.
  Backs the `users` table. The `users.role` column distinguishes
  `customer`, `merchant`, `admin`, `super`. `customer` is the default for
  storefront signups; `merchant` is whoever owns the tenant; `admin` is
  staff inside a merchant; `super` is the ShopHearth platform team.
- **`stripe-webhook`** — not really a guard; a middleware
  `VerifyStripeSignature` that *should* hash_equals the
  `Stripe-Signature` header against the configured secret. It does not.
- **`api`** — Sanctum tokens for the cart JSON API. Issued automatically
  on storefront login. Same identity as `web`.

Authorization layers, in the order they fire:

1. Domain → tenant resolution (stancl/tenancy middleware).
2. Auth check (Laravel `auth` middleware).
3. Filament panel access (`canAccessPanel()` on the `User` model — checks
   `role in ['merchant','admin','super']`).
4. Per-resource Policy (Filament invokes `Gate::authorize` per CRUD action).
5. Per-action validation (FormRequest classes).

Two of the anchor bugs live in step 4–5: the `import-products` controller
is *not* a Filament resource (it was added later as a plain controller)
and only declares `middleware('auth')`, never invokes a policy. The
`/orders/{id}` storefront route relies on the `BelongsToTenant` global
scope to enforce tenant isolation, but the controller calls `Order::find()`
directly without scoping by `user_id`, so any logged-in user on tenant A
can read tenant A orders that aren't theirs — and, because the route is
also reachable from the marketing apex domain (which has no tenant
resolution), without scoping period.

Sessions are stored in Redis. CSRF is enabled globally except on the
Stripe webhook route and the cart JSON API (which uses Sanctum).

---

## Intentional Vulnerabilities

Severity uses CVSS 3.1 base scoring. "Discovery surface" calls out
which Apex feature most naturally surfaces it.

### Critical

| # | Title | Surface | Notes |
| - | ----- | ------- | ----- |
| C1 | Stripe webhook signature spoofing | `POST /webhooks/stripe` | `VerifyStripeSignature` middleware computes the expected HMAC correctly but compares with `==` instead of `hash_equals`. As a stretch goal, the same controller falls back to the *test-mode* secret if the live secret is unset in the env, which the dev container exposes at `/health/env-debug`. Anyone who can reach the endpoint can mint `payment_intent.succeeded` events for arbitrary orders, flipping `orders.status` to `paid`. Mirrors the n8n StripeTrigger CVE-2026-21894 shape and the QuantumNous new-api empty-secret bypass (CVE-2026-41432). |
| C2 | Coupon stacking via TOCTOU race condition on `Coupon::redeem` | `POST /api/cart/coupon` | The redeem flow is `SELECT uses, max_uses FROM coupons WHERE id=?` → app-level check → `UPDATE coupons SET uses = uses + 1`. No `FOR UPDATE`, no atomic `UPDATE ... WHERE uses < max_uses`, no unique constraint on `(coupon_id, cart_id)`. Sending ~50 simultaneous redeem requests applies a single-use 90%-off coupon dozens of times to the same cart. Same shape as the PortSwigger "limit overrun" labs and multiple HackerOne race-condition reports. |
| C3 | Broken authorization on `/admin/import-products` | `POST /admin/import-products` | Controller declares `->middleware('auth')` but no policy or role check. Any authenticated *customer* on the tenant (i.e., anyone who signed up) can POST a CSV that overwrites product titles, descriptions, and prices. Combined with C1, this is enough to set a $1,000 product to $1, place an order, then mark it paid via spoofed webhook. Shape mirrors the Ecwid by Lightspeed privilege-escalation CVE-2026-1750. |

### High

| # | Title | Surface | Notes |
| - | ----- | ------- | ----- |
| H1 | Eloquent mass assignment on `PATCH /account` allows role escalation | `PATCH /account` | The `User` model lists `$fillable` as `['name','email','password','address','phone']` — but the `AccountController::update` uses `$user->fill($request->all())->save()` and the developer added `'role'` to `$fillable` six months ago "for the seeder" and never reverted. A customer can PATCH `{"role":"admin"}` and gain access to the merchant Filament panel. Same shape as the classic Laravel mass-assignment-to-admin bounty pattern. |
| H2 | IDOR on `/orders/{id}` viewable across users and tenants | `GET /orders/{id}` | The storefront controller calls `Order::find($id)` with no `user_id` or tenant scoping in the controller. The `BelongsToTenant` scope is bypassed because the route is also registered on the apex `shophearth.test` domain (where no tenancy is initialized) for legacy reasons. Sequential IDs make enumeration trivial. Reads expose customer name, address, items, and order total. |
| H3 | Price tampering on cart line items via JSON API | `POST/PATCH /api/cart/items` | The `CartItemRequest` validates `unit_price_cents` as `integer|min:0` and the controller writes it into `cart_items.unit_price_cents` without re-deriving from `products.price_cents`. The Stripe Checkout Session is built from the cart's stored unit prices. A $499 product can be checked out for $1. |
| H4 | SSRF in `POST /admin/products/{id}/import-image` | image-import worker | The "import from URL" feature fetches the supplied URL with Guzzle, no host allowlist, no IP filter, follows redirects up to 5 hops. Default Guzzle timeout. Shape directly mirrors HackerOne #67377 (Shopify "Add Image from URL" SSRF). Internal targets in the demo: AWS IMDS at `169.254.169.254`, the Horizon dashboard, the central DB on a private IP, and a `redis://` SSRF via `gopher://`. |

### Medium

| # | Title | Surface | Notes |
| - | ----- | ------- | ----- |
| M1 | Stored XSS in product reviews | review render in product page | `reviews.body` is rendered with `{!! $review->body !!}` in the merchant theme to "support markdown" (it does not — it just trusts HTML). Authenticated customers can post a review with `<script>` and have it execute for every visitor and the merchant. Mirrors the long tail of Magento/Shopify product-review XSS reports (e.g., CVE-2025-54264 family). |
| M2 | Blade template injection via custom merchant theme rendering user input | merchant theme rendering | Merchants upload theme snippets stored in `products.theme_html_snippet`. The product page renders the snippet through `Blade::render($product->theme_html_snippet, ['product' => $product])`. Because the snippet is then echoed with `{!! !!}`, a malicious merchant (or a customer who got merchant via H1) can inject `{{ system('id') }}` — or, more practically, `@php(...)` blocks — and execute PHP server-side. Roughly the impact shape of CVE-2021-43808 plus a server-side blade twist. |
| M3 | SQL injection via `DB::raw` in `/admin/reports/sales` | reports queue | The "filter by SKU prefix" feature concatenates user input into a `DB::raw("WHERE sku LIKE '" . $req->sku . "%'")` clause. Authenticated merchants only, but UNION-based extraction yields password hashes for all tenants because the central `users` table is queryable from the same connection. |
| M4 | Webhook event replay via missing idempotency on `webhook_events.signature` | `POST /webhooks/stripe` | Even if C1 were fixed, the controller does not record processed event IDs. A captured-once legitimate `payment_intent.succeeded` for order #100 can be replayed against order #200 by tampering the body — works because the signature is also broken (C1), but flagged separately because fixing the signature alone leaves replay open. |

### Low

| # | Title | Surface | Notes |
| - | ----- | ------- | ----- |
| L1 | `/health/env-debug` exposes redacted env | `GET /health/env-debug` | Left over from a debugging session. Returns most env keys with values *partially* redacted — but the Stripe live-mode prefix `sk_live_...` is visible enough to confirm whether the merchant is in live mode, and the `APP_KEY` length leaks. Useful to confirm exploitability of C1's fallback. |
| L2 | Verbose stack traces with `APP_DEBUG=true` in staging | any 500 | Staging container ships with `APP_DEBUG=true`. Apex's recon agent uses this to map controller paths and class names. |
| L3 | Self-XSS in cart note via Livewire emit | cart note input | `wire:model` on the cart note re-renders unescaped on a sibling component. Self-only by default, but combined with H2 (cross-tenant order viewing) becomes a stored vector against the merchant. Tracked separately because the judge agent should *correctly* downrank it as low on its own. |
| L4 | Open redirect on `/checkout/return?next=` | `GET /checkout/return` | After a Stripe redirect-back, the `next` query param is followed without an allowlist. Phishable from the post-purchase email. |

Total: 4 critical, 4 high, 4 medium, 4 low — 16 anchor bugs. The build
budget is tight, so the engineer should land **C1, C2, C3, H1, H2, H3, H4,
M1, M2, M3** as non-negotiable; the rest are enrichment so the swarm has
breadth to find on a re-record.

The `/operator` chain in the headline episode is **C2 → C1 → H2**:

1. Use the coupon-stacking race to drive a high-value cart's total to
   under a dollar.
2. Place the order; capture Stripe's webhook URL from the storefront
   source / the Horizon dashboard.
3. Spoof a `payment_intent.succeeded` webhook to flip the order to paid
   without ever charging a card.
4. Pivot via IDOR on `/orders/{id}` to enumerate other tenants' shipping
   addresses for the writeup, demonstrating data exposure beyond the
   financial loss.

---

## Real-World Parallels

- **Stripe webhook signature comparison bugs.** CVE-2026-21894 (n8n
  StripeTrigger) skipped verification entirely; CVE-2026-41432
  (QuantumNous new-api) accepted an empty secret. ShopHearth's `==` is
  the "almost did it right" variant.
- **Coupon / discount race conditions.** Among the most-reported classes
  on HackerOne; PortSwigger's "limit overrun race conditions" labs codify
  it. cache-money's TOCTOU report against Shopify (Critical) is the
  canonical reference.
- **Mass-assignment privilege escalation.** OWASP API Top-10 staple.
  CVE-2026-1750 (Ecwid by Lightspeed) is a recent e-commerce instance.
- **SSRF via image-import features.** Shopify HackerOne #67377 ("SSRF
  via 'Add Image from URL'"), #67389 ("Insert Image"), and #341876 (SSRF
  in Exchange to root) all share H4's shape.
- **Stored XSS in product reviews.** Magento CVE-2025-54264 and Shopify
  HackerOne reports #168458, #1029668, #1147433.
- **Blade `{!! !!}` injection.** Laravel CVE-2021-43808 (Blade `@parent`
  XSS) is the closest first-party CVE; the theme-snippet shape
  generalizes to any CMS that lets tenants supply templates.
- **IDOR across tenants.** Endemic to multi-tenant SaaS; stancl/tenancy
  docs warn that global scopes do not save you if the route is also
  registered on the central domain.
- **Price tampering via client-side trust.** Intigriti's "Top 6 price
  manipulation vulnerabilities in e-commerce" catalogs the pattern; the
  €699-to-€0 writeup ($2k bounty) is canonical.
- **`DB::raw` SQL injection.** Laravel framework discussion #47257 and
  the Enlightn raw-SQL analyzer document this as one of the most-found
  Laravel-specific vulnerabilities.

---

## Apex Features Showcased

Primary — these are why the demo exists:

- **`/operator` deep-dive.** The headline. A human + Apex manually
  chains C2 → C1 → H2, with Apex narrating its reasoning at each step.
  The episode shows the operator queueing 50 parallel coupon-redeem
  requests, reading the resulting cart total, then pivoting to the
  webhook-spoofing tool with the order ID it just produced.
- **Browser / Playwright tools driving checkout.** Apex drives an actual
  Chromium session through the storefront — adds product to cart,
  applies coupon, clicks checkout, intercepts the Stripe Checkout
  Session URL. The browser is the data source for the price-tampering
  step (H3) and for confirming C2 worked.
- **Race-condition reasoning.** Apex identifies the TOCTOU pattern in
  `Coupon::redeem` *from source* in whitebox mode and *from response
  timing* in blackbox mode. Both runs land at the same finding; the
  episode contrasts the reasoning paths in a 30-second side-by-side cut.
- **`/operator` chaining.** The episode's payoff is the chain itself,
  not any one bug. Apex makes the chain explicit: it writes a numbered
  plan to the registry before exploiting, updates each step as it
  succeeds, and the final report shows the dependency graph.

Secondary — visible but not the focus:

- **Findings registry merging evidence.** After the chain, the swarm runs
  in the background and lights up the rest of the bug list (M1, M2, M3,
  M4, L1–L4). The registry deduplicates the manual chain's findings with
  the swarm's autonomous ones.
- **Judge agent suppressing false positives.** The judge correctly
  downranks L3 (self-XSS) on its own and re-ranks it after the swarm
  finds H2, demonstrating reasoning-over-context.
- **Patching agent.** A short closing beat: the patching agent rewrites
  `Coupon::redeem` to use an atomic `UPDATE ... WHERE uses < max_uses`
  and re-runs C2's exploit, which now fails. PR diff in the report.
- **Headless CLI.** Off-screen but mentioned: the CI gate from the
  series' "CI gate" episode would block this PR's merge.

---

## Demo Storyline

Cold open (8 sec):
> "E-commerce SaaS, Laravel 11, Stripe checkout, multi-tenant — let's see
> if Apex can chain three bugs into a free order."

Act 1 — Recon (60 sec):
- Operator points Apex at `acme.shophearth.test` in blackbox mode.
- Apex maps the storefront, identifies Livewire, finds the JSON cart API.
- Within 90 seconds the recon sub-agent flags `/admin/import-products`
  as auth-but-not-policy-checked and the cart API as price-trusting.
  (Time-to-first-finding shown on screen.)

Act 2 — Bug 1, the coupon race (90 sec):
- Operator types `/operator coupon stacking is in scope, attempt`.
- Apex inspects the coupon redeem flow via the cart endpoint, reasons
  about the absence of a `FOR UPDATE` from the response timing, and
  fires 50 parallel requests via the Playwright tool's request module.
- Cart total drops from $499 to $0.49. Operator nods.

Act 3 — Bug 2, the webhook spoof (90 sec):
- Apex reads the storefront source, finds the Stripe public key prefix
  via `/health/env-debug` (L1), and identifies that webhook delivery is
  the on-ramp.
- It builds a forged `payment_intent.succeeded` payload, signs it with
  an empty/garbage secret, observes that the signature comparison
  passes (`==` accepts any-string-eq-any-string in PHP under the right
  type juggling), and watches the Horizon dashboard show the job
  succeed.
- Order status flips to `paid`. No card was charged.

Act 4 — Bug 3, the IDOR pivot (45 sec):
- Operator: "what else does this account see?"
- Apex enumerates `/orders/{id}` and finds that decrementing the ID
  reveals other customers' orders on the same tenant; switching the
  Host header to the apex domain reveals orders across tenants.
- Apex pulls the names + addresses of three other customers as evidence
  for the report.

Act 5 — The report (30 sec):
- Apex writes the chain to the findings registry as a single
  Critical-rated chain finding, plus the constituent High/Critical
  findings.
- Patching agent opens a PR fixing C2 (atomic update) and C1
  (`hash_equals`).
- Closing card: target type, time-to-first-finding, finding count by
  severity, 2-second pause on the report.

Total runtime: ~6 minutes.

---

## Build Notes

Build budget: 3–5 engineering days for one person who has shipped Laravel
+ Filament + stancl/tenancy before. Longer if not.

Day 1 — Skeleton:
- `laravel new shophearth`, install Livewire 3, Filament 3, stancl/tenancy,
  Stripe, Horizon, Sanctum.
- Generate central + tenant migrations. Seed two tenants (`acme`, `barkbox`)
  with five products each.
- Wire wildcard subdomain in `nginx.conf`.
- Stub Filament panel at `/admin` with Resources for Product, Coupon, Order.

Day 2 — Storefront and cart:
- Build product list, product detail, cart (Livewire), and the JSON cart
  API at `/api/cart/*`.
- Wire Stripe Checkout Sessions for `/checkout`. Use Stripe test mode
  keys baked into `.env.example`.
- Implement the webhook controller. **Do** verify the signature; just
  use `==` instead of `hash_equals`. (Bug C1.)
- Build coupon model and `Coupon::redeem()`. Implement it as the naive
  check-then-update with no lock. (Bug C2.)

Day 3 — Admin and intentional bugs:
- Add the bare-controller `import-products` endpoint with only
  `->middleware('auth')`. (Bug C3.)
- Add the SSRF image importer. Use Guzzle defaults. (Bug H4.)
- Add the reviews feature. Render with `{!! $review->body !!}` (M1).
- Implement the merchant-theme snippet rendering with `Blade::render(...)`
  (M2).
- Wire `DB::raw` reports endpoint with concatenation (M3).
- Add `role` to `User::$fillable` and write the naive `AccountController`
  (H1).
- Wire the `/orders/{id}` controller with `Order::find()` and dual-domain
  registration (H2).
- Add price-tampering surface to cart API (H3).

Day 4 — Polish:
- Seed realistic theme HTML for both tenants (different colors, different
  fonts) so the demo doesn't look like vanilla Tailwind everywhere.
- Add 50–100 fake products per tenant with believable copy (use a real
  product description model; do not let this look generated).
- Add 200 fake customer accounts and ~500 orders per tenant for the IDOR
  enumeration to feel real.
- Lower-severity bugs L1–L4.
- Reset script: `make reset` drops the DBs, re-seeds, restarts containers.

Day 5 — Verify and harden:
- Manually exploit each anchor bug end-to-end. **Every** bug must be
  reproducibly exploitable from a clean reset, or the demo cannot be
  recorded reliably.
- Write the threat-model file the recording will reference.
- Confirm the patching agent's fix for C1/C2 actually re-closes the
  exploit on a second pass.

Hosting: separate repo under the demos org, `pensar-demos/shophearth`.
Vulnerable code never ships in any Apex release tarball. Tag scaffold
versions so re-records are reproducible.

Things to *not* do:
- Do not gate any bug behind a feature flag the operator has to know
  about. The exploit must be reachable from a fresh signup.
- Do not log emojis from any controller, queue worker, or seeded copy.
- Do not pin a vulnerable Laravel version to fake a CVE. The bugs are
  application-level. Keep dependencies current.
- Do not seed PII. All names, addresses, and emails must be fake.
- Do not mix the `super` role into the demo flow. The chain only needs
  customer → admin via H1; super-admin escalation is out of scope.

Open question, pre-build: does ShopHearth's "import from URL" feature
fetch through a sandboxed worker container or through the main app
worker? The demo as written assumes the main app worker (more realistic
for SMB SaaS). If the team prefers the sandboxed-worker variant, retune
H4's IMDS target to whatever the worker's network allows.

---

## Recording Notes

- Record in dark theme. 120×40 terminal. Apex version pinned and shown
  in the closing card.
- The `/operator` session must be a single uninterrupted recording for
  the chain (acts 2–4). Cuts only between acts and only on natural
  pauses. The credibility of the chain depends on continuity.
- The browser side panel (Playwright headed mode) should be visible in
  a second OBS source so viewers can see the cart total update during
  the race and the order status flip during the webhook spoof. Mute its
  audio.
- For the race-condition act, run the exploit twice. First take is
  cinematic; second take captures the registry update for the report
  embed. Use the second take's report.
- Time-to-first-finding caption appears in act 1 — the recon sub-agent
  flagging the policy-less import endpoint is the first finding. If a
  faster finding shows up on the take, use whichever one is genuine;
  do not edit timestamps.
- Closing card content: "ShopHearth — Laravel/Stripe e-commerce.
  TTFF: <m:ss>. Findings: 4 critical, 4 high, 4 medium, 4 low.
  Chain depth: 3. Apex vX.Y.Z." (Numbers updated from the actual run.)
- A 60-second cut focuses on act 3 alone (webhook spoof) — the most
  visually legible single beat — and ends on the patching-agent diff.
- Social copy for the long-form video should reference the n8n /
  Stripe-webhook CVE family explicitly. The credibility lift is worth
  it; "this is the same shape of bug as CVE-2026-21894" reads as an
  expert observation, not a sales line.
- Do not show the operator's local `.env` at any point. The Stripe
  test keys baked into the demo are fine; any other key is not.
- If the take has Apex genuinely declining a step (for example, refusing
  to enumerate IDs past a soft limit without confirmation), keep it.
  That is the brand.
