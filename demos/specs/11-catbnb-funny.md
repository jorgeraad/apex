# CatBnB
### Rent-a-cat by the hour. Earn purr points. Lose your data.

A delightfully absurd two-sided marketplace where humans rent out their cats by the hour to other humans who, for reasons unexamined, want to spend an afternoon with someone else's cat. Underneath the whiskers, it's a textbook Supabase + Stripe Connect SaaS with all the classic foot-guns developers actually ship to production.

---

## Premise & Vibe

CatBnB ("the Airbnb of cats") lets cat owners list their cats for hourly rental. Renters browse profiles, book slots, leave ratings, and accumulate "purr points" redeemable for... more cat time. Owners get paid via Stripe Connect (Express accounts). Photos uploaded via Supabase Storage. Importable from any URL (because owners love sharing the perfect Instagram shot).

The product copy is unapologetically pun-heavy: "Pawsitively booked," "Litter-ally the best," "Whisker-sharp ratings." The marketing site has a hero image of a tabby in a tiny suitcase. The tone is light. The bugs are not.

This demo's core anchor: **Apex finds a Supabase RLS misconfiguration in under 60 seconds**, then walks the operator agent through chained exploitation (RLS leak -> IDOR -> price tampering -> Connect account hijack). The final report is visually delightful (cat-themed chrome) but the findings table is stone-cold professional.

**Vibe:** SF-startup-Notion-aesthetic landing page, Y-combinator-hopeful copy, RLS policies written at 2am.

---

## Why This Stack

Supabase + Next.js + Stripe Connect is the modal 2024-2026 indie SaaS stack. It's what gets shipped on Twitter launches, what shows up in YC batches, and what populates HackerOne reports week over week. RLS misconfigurations have been the dominant Supabase bug class since 2023; high-profile incidents at startups using `auth.uid() IS NOT NULL` policies are frequent enough that the Supabase team itself publishes warnings about them. This stack is also where Apex shines: Postgres-backed APIs auto-exposed by PostgREST give the swarm a huge attack surface to enumerate quickly.

The Stripe Connect angle adds genuine money-flow risk (price tampering, webhook replay, OAuth state) without requiring us to build a real PSP integration.

---

## Stack Details

- **Framework:** Next.js 15.0.x, App Router, React 19, Server Components default, Server Actions for mutations.
- **Language:** TypeScript 5.6, strict mode, `noUncheckedIndexedAccess: false` (deliberate footgun).
- **Database / Auth / Storage:** Supabase (Postgres 15, GoTrue auth, Storage v3, Edge Functions on Deno).
- **Styling:** Tailwind CSS 3.4, shadcn/ui components, Lucide icons, custom cat illustrations (SVG).
- **Payments:** Stripe Connect Express, `stripe` Node SDK 17.x, webhook handler in `/app/api/stripe/webhook/route.ts`.
- **Hosting:** Vercel (Edge + Node runtimes mixed), preview deploys per PR.
- **Image handling:** `next/image`, Supabase Storage public bucket `cat-photos`, signed URLs unused.
- **Email:** Resend for transactional (signup, booking confirm).
- **Analytics:** PostHog (also a leak vector — public posthog key is fine; we accidentally also expose a server key).
- **Maps:** Mapbox GL JS for "cats near me" (token in `NEXT_PUBLIC_MAPBOX_TOKEN`, scope-overprovisioned).

Package highlights: `@supabase/ssr@0.5`, `@supabase/supabase-js@2.45`, `stripe@17.1`, `zod@3.23`, `resend@4.0`.

---

## Architecture

```
                  +--------------------------+
   browser -----> |  Vercel Edge / Node      |
                  |  Next.js 15 App Router   |
                  |   - RSC pages            |
                  |   - /api/* route handlers|
                  |   - Server Actions       |
                  +-----------+--------------+
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
       +-------------+  +-----------+   +-----------+
       |  Supabase   |  |  Stripe   |   |  Resend   |
       |  Postgres   |  |  Connect  |   |  Email    |
       |  + GoTrue   |  |  + Webhook|   +-----------+
       |  + Storage  |  +-----------+
       |  + EdgeFns  |
       +------+------+
              |
              v
   browser <----- direct PostgREST: https://<proj>.supabase.co/rest/v1/*
                  (anon key embedded in client bundle)
```

Two clients hit the database: the Next.js server (using `service_role` for some routes, `anon` for others) and the **browser directly** via the Supabase JS SDK using the `anon` key. RLS is the only thing standing between the browser and full table reads. That's the joke. That's the whole demo.

Edge Functions handle two tasks: `import-cat-photo` (URL fetch + store), and `compute-purr-points` (cron). Both ship with the `service_role` key as an env var, fine — except the photo importer is invokable by any signed-in user with attacker-controlled URL input.

---

## Data Model

Postgres schema, all in `public` (yes, that's part of the problem).

**users**
| col | type | notes |
|---|---|---|
| id | uuid PK | matches `auth.users.id` |
| email | text unique | |
| display_name | text | |
| stripe_account_id | text | Connect Express account |
| stripe_onboarded | bool | |
| role | text | `renter` \| `owner` \| `admin` |
| purr_points | int | |
| created_at | timestamptz | |

**cats**
| col | type | notes |
|---|---|---|
| id | uuid PK | |
| owner_id | uuid FK -> users.id | |
| name | text | |
| breed | text | |
| hourly_rate_cents | int | |
| bio | text | |
| primary_photo_url | text | Storage public URL |
| is_listed | bool | |
| location_geog | geography(Point) | PostGIS |

**cat_photos**
| col | type | notes |
|---|---|---|
| id | uuid PK | |
| cat_id | uuid FK | |
| storage_path | text | `cat-photos/{cat_id}/{uuid}.{ext}` |
| mime_type | text | client-supplied (bug) |

**bookings**
| col | type | notes |
|---|---|---|
| id | uuid PK | |
| cat_id | uuid FK | |
| renter_id | uuid FK -> users.id | |
| owner_id | uuid FK -> users.id | denormalized |
| start_at | timestamptz | |
| end_at | timestamptz | |
| total_cents | int | |
| stripe_payment_intent_id | text | |
| status | text | `pending`\|`paid`\|`cancelled`\|`completed` |
| renter_notes | text | free-text, sometimes contains addresses |

**ratings**
| col | type | notes |
|---|---|---|
| id | uuid PK | |
| booking_id | uuid FK | |
| stars | int | |
| body | text | rendered as markdown (XSS risk if naive) |

**webhook_events**
| col | type | notes |
|---|---|---|
| id | uuid PK | |
| stripe_event_id | text | NOT unique (bug) |
| received_at | timestamptz | |
| payload_json | jsonb | |

Storage buckets: `cat-photos` (public), `id-verification` (private, for owner KYC).

---

## Key Routes / Surfaces

### Pages

| Path | Purpose |
|---|---|
| `/` | Marketing, hero, featured cats |
| `/browse` | Search/filter cats |
| `/cats/[id]` | Cat profile, booking widget |
| `/book/[catId]` | Slot picker -> Stripe Checkout |
| `/dashboard` | Renter: bookings, purr points |
| `/owner` | Owner dashboard, listings, payouts |
| `/owner/onboard` | Stripe Connect Express onboarding |
| `/owner/cats/new` | Create listing, upload photos, "import from URL" |
| `/admin` | Admin-only ops (gated client-side only — bug) |
| `/login`, `/signup` | Supabase auth |

### API / Server Actions

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/checkout` | session | Create Stripe Checkout Session for a booking |
| POST | `/api/stripe/webhook` | sig | Stripe webhook receiver |
| GET | `/api/stripe/connect/start` | session | Begin Connect OAuth |
| GET | `/api/stripe/connect/callback` | none | OAuth callback (state validation broken) |
| POST | `/api/cats/import-photo` | session | Edge fn proxy: fetch URL, store as photo |
| POST | `/api/bookings` | session | Server Action wrapper: create booking |
| POST | `/api/admin/refund` | session+role | Admin refund (role check via JWT claim — but RLS not enforced) |
| GET | `/api/me` | session | Returns current user incl. `purr_points` |
| POST | `/api/auth/signup` | none | Wraps Supabase signup, no rate limit |

### Direct PostgREST (browser-reachable, anon key)

| Resource | Methods exposed | Notes |
|---|---|---|
| `/rest/v1/cats` | GET, POST, PATCH | broad |
| `/rest/v1/bookings` | GET, POST, PATCH | RLS misconfig |
| `/rest/v1/users` | GET, POST | anon insert allowed |
| `/rest/v1/ratings` | GET, POST | |
| `/rest/v1/cat_photos` | GET, POST | |

---

## Auth Model

- Supabase GoTrue email/password + magic link.
- JWT issued by GoTrue, stored as Supabase `sb-*-auth-token` cookie (SSR via `@supabase/ssr`).
- Server-side: `createServerClient` w/ cookie store; some routes shortcut to `createClient(url, SERVICE_ROLE)` for "admin tasks."
- Role check: a `role` column on `public.users` and a JWT custom claim (`app_role`) populated by an `auth.on_auth_user_created` trigger... that doesn't update the claim if the role changes later (stale claim bug, not exploited but flavorful).
- Admin gate on `/admin`: client-side `if (user.role !== 'admin') redirect('/')`. Server route `/api/admin/refund` checks JWT claim but the underlying tables have no RLS scoped to admin only — service_role key bypasses RLS entirely.
- Stripe Connect OAuth `state` is generated as `crypto.randomUUID()` and stored in a cookie that is **not HttpOnly and not bound to the session** — see vuln below.

---

## Intentional Vulnerabilities

>=12 bugs, mapped to anchors plus organic flavor. CVSS:3.1 vectors are illustrative.

### Critical

| # | ID | Title | Where | CVSS |
|---|---|---|---|---|
| 1 | CAT-001 | Permissive RLS on `public.bookings` (`auth.uid() IS NOT NULL`) | Supabase policy `bookings_select_authenticated` | 9.1 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N) |
| 2 | CAT-002 | `service_role` key referenced from a code path bundled to the client | `lib/supabaseAdmin.ts` imported by `app/cats/[id]/page.tsx` via shared util | 9.8 |
| 3 | CAT-003 | Price tampering on `/api/checkout` (client supplies `total_cents`, server doesn't refetch from DB) | `app/api/checkout/route.ts` | 8.6 |
| 4 | CAT-004 | Stripe webhook timestamp tolerance set to 3600s + no event-id dedupe | `app/api/stripe/webhook/route.ts` `stripe.webhooks.constructEvent(..., 3600)` | 8.1 |

### High

| # | ID | Title | Where | CVSS |
|---|---|---|---|---|
| 5 | CAT-005 | IDOR via direct PostgREST: `/rest/v1/bookings?id=eq.<uuid>` returns any booking due to CAT-001 | client-side anon key | 7.5 |
| 6 | CAT-006 | `anon` role has `INSERT` on `public.users` (allows arbitrary row insertion -> account-shaped rows w/ chosen role) | RLS policy `users_anon_insert` | 8.2 |
| 7 | CAT-007 | File-upload MIME spoof: client-declared `mime_type` trusted; Storage object served with `Content-Type: text/html` -> stored XSS / one-click HTML-on-CDN | `/api/cats/import-photo` + `cat-photos` bucket config | 7.6 |
| 8 | CAT-008 | SSRF in `import-cat-photo` Edge Function: no scheme/host allowlist, follows redirects, reads `http://169.254.169.254/...` | Edge Fn `import-cat-photo` | 8.6 |
| 9 | CAT-009 | Stripe Connect OAuth `state` not bound to session; attacker-initiated state -> victim links attacker's Connect account | `/api/stripe/connect/start` + `/callback` | 7.4 |

### Medium

| # | ID | Title | Where | CVSS |
|---|---|---|---|---|
| 10 | CAT-010 | No rate limit on `/api/auth/signup`; response distinguishes "email already exists" vs "ok" -> enumeration | `app/api/auth/signup/route.ts` | 5.3 |
| 11 | CAT-011 | Public Storage bucket `cat-photos` has overly broad `getPublicUrl` access; private filenames guessable (`{cat_id}/{seq}.jpg`) | bucket config | 5.8 |
| 12 | CAT-012 | Markdown rating bodies rendered with `dangerouslySetInnerHTML` after naive sanitization (allows `javascript:` URIs in links) | `components/RatingBody.tsx` | 6.1 |
| 13 | CAT-013 | Mapbox token scope = `*:write` instead of `*:read`; usable to mutate styles | `NEXT_PUBLIC_MAPBOX_TOKEN` | 5.0 |

### Low

| # | ID | Title | Where | CVSS |
|---|---|---|---|---|
| 14 | CAT-014 | Verbose error in `/api/checkout` leaks Stripe secret key prefix in stack trace under bad input | error handler | 3.7 |
| 15 | CAT-015 | `X-Powered-By: Next.js` not stripped; no CSP; `Strict-Transport-Security` absent on apex | `next.config.js` | 3.1 |
| 16 | CAT-016 | `purr_points` updatable via direct PATCH to `/rest/v1/users?id=eq.<self>` (RLS allows self-update of any column) | RLS policy `users_self_update` | 4.3 |

### Detail notes for the anchor bugs

**CAT-001 (RLS).** Policy literally:

```sql
create policy "bookings_select_authenticated"
on public.bookings for select
to authenticated
using (auth.uid() is not null);
```

Should be `using (auth.uid() = renter_id or auth.uid() = owner_id)`. Apex's swarm finds this by enumerating PostgREST, signing up two accounts, and observing that account B can read account A's bookings.

**CAT-002 (service_role leak).** `lib/supabaseAdmin.ts` exports `supabaseAdmin = createClient(url, process.env.SUPABASE_SERVICE_ROLE_KEY!)`. A helper `formatCatPrice` lives in the same file and is imported by a Client Component. Next.js bundler pulls the module; the env var is inlined at build because someone wrote `NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY` in `.env.production` "to make it work." Apex grep finds the prefix `eyJhbGciOi...` in `_next/static/chunks/*.js`.

**CAT-003 (price tampering).** Request body: `{ catId, hours, total_cents }`. Server creates Checkout with `unit_amount: body.total_cents`. Apex flips `total_cents` to `1` and books a $200 cat for 1 cent.

**CAT-004 (webhook replay).** `constructEvent(payload, sig, secret, 3600)` plus `webhook_events.stripe_event_id` is not unique-indexed. Replay a `payment_intent.succeeded` 30 minutes later -> booking double-marks `paid`, refund flow becomes confused.

**CAT-008 (SSRF).** Edge Fn does `await fetch(userUrl)` directly. Reaches Supabase's internal metadata endpoints from a Deno isolate, also `http://localhost:54321` if self-hosted. Demoed via `http://169.254.169.254/latest/meta-data/iam/security-credentials/` (or the GCP equivalent on the actual host).

**CAT-009 (Connect OAuth state).** `state` written to a non-HttpOnly cookie, not bound to user id. Attacker generates a state, gets victim to click a crafted URL with that state, victim onboards, **attacker's** Connect account ends up bound to victim's row.

---

## Real-World Parallels

- **Supabase RLS leaks:** see the long string of public disclosures since 2023, including the [Supabase team's own blog post on RLS pitfalls](https://supabase.com/blog/postgres-row-level-security) and HackerOne reports against Supabase-backed startups citing `auth.uid() IS NOT NULL` policies. The pattern shows up almost monthly.
- **service_role key in client bundle:** echoes Vercel's [public guidance](https://vercel.com/guides/how-can-i-use-environment-variables-in-vercel) on `NEXT_PUBLIC_*` mistakes; multiple disclosed indie-SaaS incidents have leaked admin keys this way.
- **Stripe price tampering:** classic. See HackerOne reports against shopify-style apps; Stripe's own docs explicitly recommend [creating Checkout Sessions server-side from a product/price ID, never a client amount](https://stripe.com/docs/payments/checkout/migrating-prices).
- **Stripe webhook replay / signature handling:** mirrors [CVE-2024-XXXX](https://github.com/stripe/stripe-node/security/advisories) class advisories around tolerance windows and the well-known guidance from Stripe to keep tolerance at 300s and dedupe on `event.id`.
- **IDOR via PostgREST:** equivalent to many disclosed Hasura/PostgREST bugs where row filters were assumed but absent.
- **SSRF on URL importers:** see GitLab CVE-2023-2825 (path traversal-adjacent), and the long lineage of Capital One-style SSRF -> IMDS attacks.
- **OAuth state CSRF:** OWASP A01; many Connect/PayPal/GitHub integrations have shipped this exact bug.
- **MIME spoof on file upload -> stored HTML/JS on a CDN origin:** see Dropbox, Slack, and several bug-bounty disclosures where Storage buckets served user content with attacker-controlled `Content-Type`.
- **Email enumeration on signup:** OWASP ASVS V2.2; persistent in Auth0/Supabase/Clerk-backed apps.

---

## Apex Features Showcased

- **`/pentest` blackbox + whitebox.** Run blackbox first (just the URL); show Apex enumerate routes, find PostgREST, do the RLS read. Then run whitebox with the repo mounted and watch Apex pick up the `service_role` leak in seconds via static analysis.
- **Swarm parallelism.** RLS enumeration, price tampering, and webhook replay run as separate swarm agents in parallel; finalized findings merged in the unified report.
- **Findings + CVSS + judge.** Each bug above lands as a Finding with vector string, reproduction steps, and the judge's confidence score. Demonstrates dedupe (RLS bug and IDOR bug are linked, not duplicated).
- **Patching agent.** End demo with Apex generating a corrected RLS policy and a server-side checkout refactor as a unified diff against the repo.
- **Memory.** Second run: Apex remembers the two test accounts it created, jumps straight to the chained exploit.
- **Attack-surface + JS endpoint extraction.** Apex parses the Next.js client bundles, pulls every fetch URL and Supabase call, builds the surface map. Visually striking on screen.
- **Threat-model.** Pre-pentest, Apex emits a 1-page threat model identifying Stripe Connect, RLS, and Storage as top risks — and is then "right" when findings land.
- **Visually delightful report.** HTML report header is a tasteful cat-line-illustration banner; the Findings table itself is plain, professional, and high-contrast. Cat-themed copy strictly limited to chrome.

---

## Demo Storyline

Target: 60-second core cut + 3-minute long cut.

**60-second cut (the headline demo):**

1. (0:00-0:05) Cold open: CatBnB landing page, "Pawsitively booked since 2026." Operator types `apex pentest https://catbnb.demo`.
2. (0:05-0:15) Apex spins up: threat-model card pops, swarm fans out, attack surface map draws.
3. (0:15-0:30) First finding lands: **CAT-001 RLS misconfig**, with a live cURL replay against `/rest/v1/bookings?select=*` showing 1,247 rows from a freshly-signed-up account.
4. (0:30-0:45) Chained finding: **CAT-005 IDOR** -> **CAT-003 price tampering** -> attacker books a $400 Maine Coon for $0.01. Receipt email appears in the inbox sidebar.
5. (0:45-0:55) Patching agent emits the corrected RLS policy + server-side price refetch in a side-by-side diff view.
6. (0:55-1:00) Final report card: 16 findings, 4 critical. Tagline: "Apex found it before the cat did."

**3-minute long cut adds:**

- Whitebox phase finding **CAT-002** (service_role in bundle).
- Webhook replay (**CAT-004**) demonstrated with a Stripe CLI replay against a live tunnel.
- SSRF (**CAT-008**) hits a mock IMDS, exfils a fake credential.
- Connect OAuth state CSRF (**CAT-009**) shown as a sequence diagram in the report.

---

## Build Notes

**Day 1: Skeleton + Auth + Schema**
- `pnpm create next-app catbnb --typescript --tailwind --app`.
- Install `@supabase/ssr`, `@supabase/supabase-js`, `stripe`, `zod`, `resend`, `lucide-react`, `shadcn/ui`.
- Provision Supabase project (free tier). Run schema migrations (tables above).
- Write the **deliberately broken** RLS policies and commit them with a comment that says `// TODO: tighten before launch` (period flavor).
- Implement `/login`, `/signup`, `/api/auth/signup` (no rate limit).
- Seed: 25 cats with public-domain Wikipedia/Unsplash CC0 cat photos, 6 owners, 12 renters.

**Day 2: Listings, Bookings, Storage**
- `/browse`, `/cats/[id]`, `/book/[catId]`. Use server components + a small island for the slot picker.
- Photo upload: client uploads to Supabase Storage, then writes `cat_photos` row with **client-supplied** `mime_type`. Implement `/api/cats/import-photo` that calls the Edge Function (no allowlist).
- Bucket policy on `cat-photos`: public read; storage object metadata `content-type` derived from the `mime_type` column at signed-URL time (the bug).
- Plant `lib/supabaseAdmin.ts` and import it from a util that gets bundled to the client. Verify with `next build` that the key string lands in `.next/static/chunks/*.js`.

**Day 3: Stripe Connect + Webhooks + Checkout**
- `/owner/onboard` -> Connect Express onboarding link. Implement broken `state` handling (cookie not bound to session).
- `/api/checkout`: accept `{ catId, hours, total_cents }`. Create Checkout Session with `unit_amount: total_cents`. Do **not** refetch the cat's `hourly_rate_cents`.
- `/api/stripe/webhook`: `constructEvent(..., 3600)`, insert into `webhook_events` without unique constraint. Update booking status on `payment_intent.succeeded`.
- Add `/api/admin/refund` with a JWT-claim role check; underlying table writes use `service_role`.

**Day 4: Polish, Vulns, Seed Data**
- Implement ratings with `dangerouslySetInnerHTML` after a naive sanitize that strips `<script>` only.
- Add `purr_points` self-update via direct PostgREST.
- Wire up Mapbox with the over-scoped token.
- Seed 60 bookings across the test users so RLS exfil shows real-looking data.
- Add the cat-themed marketing copy and footer ("Made with `<3` and litter").
- Write `README.md` with "do not deploy this anywhere real" warning.

**Day 5: Recording prep + Apex dry runs**
- Run `apex pentest` end-to-end three times; tune timing.
- Capture a clean run for the 60-second cut; capture the long cut.
- Verify every finding above actually lands; adjust difficulty knobs (e.g., make sure the service_role key shows up in a chunk Apex can find via grep).
- Confirm patching-agent output compiles and re-running `apex pentest` post-patch reduces criticals to zero.

**Defensive flags / safety:**
- Domain `catbnb.demo` only; robots.txt disallow all; basic-auth gate on the live preview to prevent random crawlers.
- Real Stripe in test mode only. No real money.
- All test cats are CC0 stock images; no real cats are rented.

---

## Recording Notes

- **Resolution:** 1920x1080, 60fps capture, ScreenStudio or OBS.
- **Two terminals visible:** left = Apex CLI (TUI mode for the swarm fan-out animation), right = a tail of CatBnB server logs (so the audience sees the attacks land in real time).
- **Browser pinned to right side** showing the CatBnB UI; switch to Stripe dashboard for the price-tampering moneyshot.
- **Cursor highlighter on.** Keystroke overlay off (TUI animation is the star).
- **B-roll:** 2-3 seconds of the cat-themed report header rendering, then immediate cut to the findings table — do not linger on the cute chrome.
- **Audio:** no music for the 60-second cut; soft synth bed for the long cut. VO is optional; on-screen captions for each finding ID and CVSS score are required.
- **Lower-third banner** appears at the moment the first critical lands: "RLS policy: `auth.uid() IS NOT NULL`" — quote the actual SQL on screen.
- **End card:** "16 findings. 60 seconds. Zero cats harmed." Apex logo, install command.
- **Avoid in frame:** any real Stripe key prefixes, real emails, the actual Supabase project ref. Use a redaction overlay if needed.
- **Take count:** plan for 6-10 takes of the 60-second cut to nail timing on the swarm fan-out and the price-tampering reveal.
