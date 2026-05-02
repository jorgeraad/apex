# UnboundShelf
### Goodreads for books they don't want you to read.

## Premise & Vibe

UnboundShelf is a "Goodreads, but for banned, restricted, samizdat, and otherwise legally-awkward titles." The product positions itself as a polite, slightly winking literary community for people who like marginalia in their margins and footnotes in their footnotes. There are reading lists ("The 1955 Postmaster Index", "Things Confiscated at JFK", "Currently Banned in at least 3 jurisdictions"), star ratings, reviews, and curated shelves maintained by self-styled "underground librarians." Paid subscribers receive a monthly "samizdat" newsletter with hand-typeset PDFs.

The vibe is independent-bookstore-meets-public-radio: serif type, kraft-paper backgrounds, an inexplicable amount of letterpress ornaments, and a footer that says "Read responsibly. Cite always." Underneath the chamomile-tea aesthetic, the stack is a single Go binary plus a static SvelteKit frontend, glued together with PocketBase collection rules that someone wrote on a Tuesday and never came back to.

The bugs in this demo are not exotic. They are the boring, recurring failure modes of small-stack, "I shipped it on a weekend" backends: collection rules with the wrong quoting, form actions that forgot CSRF, a seed admin password that survived to production, and `{@html}` rendered straight from user input because the dev wanted markdown to "just work."

## Why This Stack

PocketBase is having a moment. It's a single Go binary that embeds SQLite, ships its own Admin UI, exposes a REST + Realtime API, and lets you write authorization as per-collection rule expressions. For a solo dev or a 2-person team, it collapses "backend, database, auth, file storage, realtime, admin panel" into one `./pocketbase serve`. Combined with SvelteKit on the frontend (deployed as a static or edge-rendered bundle on Vercel or Cloudflare Pages), it is one of the cheapest-to-run stacks that still feels modern.

That convenience is precisely why it shows up in Apex demos. PocketBase rule expressions are deceptively easy to get wrong: a stray quote, an `||` where you wanted `&&`, or copy-pasting `@request.auth.id != ""` from the docs and forgetting that this means "any logged-in user can do this." The rules are stored in `pb_data/data.db` and visible in `pb_schema.json` exports, which means Apex's whitebox mode can statically diff them against a policy and produce findings without ever sending a request. This demo is built to exercise exactly that pathway, while still leaving classic web bugs (XSS, CSRF, IDOR, weak crypto on share links) for the blackbox `/pentest` mode to catch.

## Stack Details

- **Frontend**: SvelteKit 2 (Svelte 5 runes), TypeScript strict, Tailwind v3, deployed via `@sveltejs/adapter-vercel` (edge functions for SSR + form actions). Static assets on Cloudflare R2 via Pages.
- **Backend**: PocketBase 0.22.x, single Go binary, no custom Go hooks (rules-only, with a few JS hooks via `pb_hooks/`).
- **Database**: SQLite, embedded in the PocketBase binary, persisted to `pb_data/data.db`. WAL mode.
- **File storage**: PocketBase local filesystem driver (`pb_data/storage/`). Cover images and "samizdat" PDFs.
- **Auth**: PocketBase built-in auth collections (`users`, plus a separate `librarians` auth collection for moderators). JWT in cookies on the SvelteKit side.
- **Realtime**: PocketBase `/api/realtime` SSE endpoint, used by the "live shelf activity" sidebar.
- **Email**: Postmark via PocketBase SMTP settings; templates live in `pb_hooks/`.
- **Payments**: Stripe Checkout for the $4/mo "samizdat" newsletter subscription. Webhook handled by a SvelteKit `+server.ts` route.
- **Deploy**: Backend runs on a Hetzner CX22 behind Caddy, fronted by Cloudflare. Frontend on Vercel. Deploy script is a single `deploy.sh` that scp's the binary and `pb_migrations/` up.

Versions pinned in `package.json` and `go.mod`; PocketBase upgrade is manual and lags by 2-3 minor versions, which is realistic.

## Architecture

```
                    +-------------------+
                    |   Cloudflare CDN  |
                    +---------+---------+
                              |
                +-------------+-------------+
                |                           |
        +-------v--------+         +--------v---------+
        |  Vercel Edge   |         |  Hetzner VPS     |
        |  SvelteKit SSR |         |  Caddy (TLS)     |
        |  /+page.server |  HTTPS  |     |            |
        |  form actions  +-------->+  pocketbase      |
        |                |         |  ./pb_data/      |
        +-------+--------+         |  ./pb_hooks/     |
                |                  +--------+---------+
                |                           |
        +-------v--------+         +--------v---------+
        |  Browser       |  SSE    |  SQLite (WAL)    |
        |  Svelte client +<--------+  /api/realtime   |
        +----------------+         +------------------+
```

SvelteKit owns: rendering, marketing pages, auth cookie handling, form actions for review/shelf creation, Stripe webhook, share-link signing.

PocketBase owns: collections (`users`, `librarians`, `books`, `shelves`, `reviews`, `subscriptions`, `newsletters`, `share_tokens`, `flags`), authentication, file storage, realtime fan-out, and the Admin UI at `/_/`.

The frontend never talks to SQLite directly. It either calls SvelteKit form actions (which then call PocketBase via the JS SDK with the user's JWT) or, for read-heavy paths like book lists, calls PocketBase REST directly from the browser using the same JWT.

## Data Model

PocketBase collections (`pb_schema.json` excerpt; types simplified):

| Collection | Type | Key fields | Notes |
|---|---|---|---|
| `users` | auth | `username`, `email`, `displayName`, `bio`, `avatar` (file), `isAdmin` (bool) | `isAdmin` is the bug. See V10. |
| `librarians` | auth | `handle`, `bio`, `verified` (bool), `region` | Curators. Separate auth collection. |
| `books` | base | `title`, `author`, `year`, `isbn`, `cover` (file), `synopsis`, `restrictionNotes`, `jurisdictions` (json) | Public catalog. |
| `shelves` | base | `name`, `slug`, `owner` (rel users), `curator` (rel librarians, optional), `description`, `books` (rel books, multi), `visibility` (select: public/unlisted/private) | |
| `reviews` | base | `book` (rel), `author` (rel users), `rating` (1-5), `bodyMarkdown` (text), `bodyHtml` (text), `flagged` (bool) | `bodyHtml` is rendered with `{@html}`. See V8. |
| `subscriptions` | base | `user` (rel), `stripeCustomerId`, `stripeSubId`, `status`, `currentPeriodEnd` | |
| `newsletters` | base | `month`, `title`, `pdf` (file), `excerptHtml`, `paywalled` (bool) | |
| `share_tokens` | base | `shelf` (rel), `token`, `expiresAt`, `creator` (rel users) | Token = MD5-based. See V7. |
| `flags` | base | `target` (text), `reason`, `reporter` (rel users) | Moderation queue. |

`pb_data/types.d.ts` is generated by `pocketbase-typegen` and committed; Apex reads it in whitebox mode to enumerate collection shape without ever hitting the server.

## Key Routes / Surfaces

### SvelteKit routes

| Route | Method(s) | Purpose | Notes |
|---|---|---|---|
| `/` | GET | Marketing + featured shelves | SSR'd, cached at edge |
| `/login` | GET, POST (action `default`) | User login | Sets `pb_auth` cookie |
| `/signup` | GET, POST (action `default`) | User signup | Calls PB `/api/collections/users/records` |
| `/books` | GET | Catalog index | PB list call from browser |
| `/books/[isbn]` | GET | Book detail + reviews | |
| `/books/[isbn]/review` | POST (action `submit`) | Create review | V2: no origin check |
| `/shelves` | GET | Shelf directory | |
| `/shelves/[slug]` | GET | Shelf detail | Supports `?expand=books,curator` (V3) |
| `/shelves/new` | GET, POST (action `default`) | Create shelf | |
| `/shelves/[slug]/share` | POST (action `default`) | Generate signed share URL | V7 |
| `/u/[username]` | GET | Public profile | |
| `/u/[username]/reading` | GET | "Currently reading" | |
| `/account` | GET | Settings | |
| `/account/avatar` | POST (action `upload`) | Avatar upload | V5: type confusion |
| `/subscribe` | GET | Plan picker | |
| `/subscribe/checkout` | POST (action `default`) | Stripe Checkout session | |
| `/api/stripe/webhook` | POST | Stripe webhook | `+server.ts` endpoint |
| `/api/share/[token]` | GET | Resolve share token | V7 |
| `/admin` | GET | Internal admin dashboard | Gated on `user.isAdmin` (V10) |

### PocketBase REST API (relevant subset)

| Endpoint | Auth | Purpose |
|---|---|---|
| `POST /api/collections/users/auth-with-password` | none | User login |
| `POST /api/collections/users/records` | createRule | Signup. V10. |
| `GET  /api/collections/users/records` | listRule | V1 (readers list rule). |
| `GET  /api/collections/books/records` | listRule | Public catalog. |
| `GET  /api/collections/shelves/records?expand=books,curator` | listRule | V3 IDOR via expand. |
| `POST /api/collections/reviews/records` | createRule | Submit review. |
| `GET  /api/collections/reviews/records?filter=...` | listRule | List reviews. |
| `POST /api/collections/users/records/:id/avatar` | updateRule | Avatar upload (multipart). V5. |
| `GET  /api/realtime` | subscriptionRule | SSE stream. V6. |
| `GET  /api/collections` | superuser | Schema dump. V9. |
| `POST /api/admins/auth-with-password` | none | Admin auth. V4. |
| `GET  /api/files/:collection/:recordId/:filename` | filesRule | File download. |

## Auth Model

Three principals:

1. **Anonymous**: can read marketing pages, public shelves, the book catalog, and public reviews.
2. **User** (`users` auth collection): authenticated via PocketBase password auth, JWT stored as `pb_auth` cookie (HttpOnly, SameSite=Lax — note "Lax", which is part of V2). Can create reviews, create shelves, follow librarians, subscribe.
3. **Librarian** (`librarians` auth collection): a separate auth collection. Logs in at `/librarian-login`. Can curate any shelf they're attached to. `verified=true` librarians get a checkmark.
4. **Admin** (PocketBase superuser, plus a `users.isAdmin=true` flag for app-level admin UI): superuser logs in at `/_/` (PB admin). App admins access `/admin`.

Tokens: PocketBase issues 1-week JWTs by default. The frontend stores them in a cookie set from the server-side form action so client JS can also read them via `pb.authStore.loadFromCookie()`. SvelteKit `hooks.server.ts` validates the cookie and populates `event.locals.user`.

## Intentional Vulnerabilities

Severities use CVSS 3.1 vectors. Twelve total: 4 Critical, 4 High, 3 Medium, 1 Low.

| ID | Severity | Class | Title |
|---|---|---|---|
| V1 | High | BOLA / Rule misconfig | `users` listRule allows any authenticated user to read all reader records |
| V2 | High | CSRF | `/books/[isbn]/review` form action has no origin/sameSite check |
| V3 | Critical | IDOR via `expand` | `shelves` list with `expand=curator` leaks unpublished librarian fields |
| V4 | Critical | Weak auth / hardcoded creds | Seed admin password `changeme123` left in `deploy.sh` |
| V5 | High | Unrestricted upload | Avatar field accepts `text/html` masquerading as image |
| V6 | Medium | Excessive data exposure | `/api/realtime` subscriptionRule allows any user to subscribe to any collection |
| V7 | High | Weak crypto | Share link signature is `md5(shelfId + 6-char secret)` |
| V8 | Critical | Stored XSS | Review body rendered via `{@html}` after naive markdown |
| V9 | Medium | Information disclosure | `/api/collections` returns full schema to unauthenticated callers |
| V10 | Critical | Mass assignment | `users` createRule allows `isAdmin` to be set on signup |
| V11 | Medium | SSRF (lite) | "Import from Goodreads URL" feature fetches arbitrary URLs server-side |
| V12 | Low | Verbose errors | PocketBase error responses include SQL fragments in dev-mode left enabled |

### V1. Reader enumeration via `users.listRule`

**Where**: `pb_schema.json` → `users` collection → `listRule`.
**The bug**: rule is `@request.auth.id != ""`. Cargo-culted from the PocketBase docs section on auth collections, where it appears as an example with a clear "do not use as-is" note. The intent was "logged-in users can list users." The effect is "any logged-in user can read every row of the readers table, including `email`, `bio`, and the boolean `isAdmin`."
**Reference**: see PocketBase issue tracker discussions on `listRule` defaults and Daniel Miessler / Detectify writeups on PocketBase rule misconfig (e.g., the recurring "PocketBase open list rule" finding pattern documented by independent researchers in 2023-2024).
**Severity**: High. CVSS 3.1: AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N — 6.5.
**Apex find path**: whitebox rule audit reads `pb_schema.json`, flags any auth collection with `listRule` that does not constrain by `id = @request.auth.id`.

### V2. CSRF on review submission

**Where**: `src/routes/books/[isbn]/review/+page.server.ts`, action `submit`.
**The bug**: form action processes `request.formData()` and calls PocketBase create on the user's behalf. There is no `Origin` header check, no CSRF token, and the auth cookie is `SameSite=Lax`. `Lax` blocks cross-site POST from a top-level form, but a `fetch()` with `mode: 'no-cors'` from an attacker-controlled site, or a redirect-driven POST through a 307, can still hit it under specific conditions; more importantly, `Lax` does *not* protect against same-site subdomain takeover scenarios, and the team has `*.unboundshelf.io` wildcard certs.
**Reference**: SvelteKit's own docs explicitly recommend origin checking on form actions; the `csrf.checkOrigin` config defaults to true but is disabled here ("we kept getting 403s in dev so we turned it off"). HackerOne report H1 #1313920-class pattern.
**Severity**: High. CVSS 3.1: AV:N/AC:L/PR:N/UI:R/S:U/C:N/I:H/A:N — 6.5.

### V3. IDOR via `expand` on shelves

**Where**: PocketBase `shelves.listRule` is correct (`visibility = "public" || owner = @request.auth.id`), but the `librarians` collection that `curator` expands into has `viewRule = @request.auth.id != ""`.
**The bug**: a shelf with `visibility=public` legitimately exposes its curator. But the expanded librarian record contains `email`, `region` (precise), `bio` (which sometimes contains personal addresses), and a `verified` flag. Worse, an attacker can set `?expand=curator,books.reviews_via_book.author` to chain through relations whose individual viewRules are each lax in isolation but combine into a full reviewer-de-anonymization graph.
**Reference**: PocketBase's own documentation warns about expansion bypass when relation-target rules are looser than parent rules; this is a recurring class also seen in Hasura permissions and PostgREST.
**Severity**: Critical. CVSS 3.1: AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N — 6.5; bumped to Critical (8.1) in our scoring because the chain enables doxxing of pseudonymous librarians, which is the platform's marketed selling point.

### V4. Default admin password

**Where**: `deploy.sh` line 47.
```sh
./pocketbase superuser create admin@unboundshelf.io changeme123 || true
```
**The bug**: the `|| true` makes the command idempotent across deploys, which means the first deploy creates the admin with that password and subsequent deploys silently succeed without rotating it. PocketBase admin is at `/_/` and has no rate limit by default.
**Reference**: classic CWE-798 (Hardcoded Credentials); see also the Snyk 2023 advisory on PocketBase admin endpoint exposure.
**Severity**: Critical. CVSS 3.1: AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H — 10.0.
**Apex find path**: blackbox `/pentest` tries the documented PB default-credential set; whitebox finds the literal in `deploy.sh`.

### V5. File upload type bypass

**Where**: `users.avatar` field; PocketBase MIME validation set to `image/png, image/jpeg, image/webp`.
**The bug**: PocketBase 0.22 validates MIME via the client-supplied `Content-Type` header on multipart parts, not via magic-byte sniffing. A request with `Content-Type: image/png` but body `<script>...</script>` is accepted. The avatar is then served from `/api/files/users/:id/avatar.png` with a `Content-Type: image/png` response header *but* if the original filename ends in `.html` and the field's `protected` flag is false, PB serves the raw bytes with sniffed content-type on some CDN paths. Combined with the Cloudflare proxy stripping `X-Content-Type-Options: nosniff` on cached HTML responses, you get stored HTML/JS execution under `unboundshelf.io`.
**Reference**: see PocketBase issue #4012-class discussion on MIME detection; classic CWE-434.
**Severity**: High. CVSS 3.1: AV:N/AC:L/PR:L/UI:R/S:C/C:H/I:H/A:N — 8.3.

### V6. Realtime over-subscription

**Where**: `pb_data` settings → realtime subscription rule (global) is `@request.auth.id != ""`.
**The bug**: any logged-in user can `pb.collection('subscriptions').subscribe('*')` and receive live events for every subscription record — including `stripeCustomerId` and `status`. They can also subscribe to `flags` and watch the moderation queue in real time.
**Reference**: PocketBase realtime auth uses the same rule expression syntax as collection rules, with the same misconfig patterns. The fix is per-collection subscription rules.
**Severity**: Medium. CVSS 3.1: AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N — 6.5.

### V7. Weak share-link signing

**Where**: `src/lib/server/share.ts`.
```ts
export function sign(shelfId: string) {
  return md5(shelfId + SHARE_SECRET); // SHARE_SECRET is 6 chars
}
```
**The bug**: MD5 is fast, the secret is 6 characters, the input space (shelfId) is enumerable, and `share_tokens.token` is checked with string equality. With ~20 known shelf IDs, `26^6` (~308M) candidates, and a $0.40-on-vast.ai GPU hour, the secret falls in minutes. Once recovered, an attacker generates valid share links for any private shelf.
**Reference**: CWE-327 + CWE-326. See bcrypt-vs-MD5 perennial advice; HashiCorp Vault audit reports for the "short shared secret" pattern.
**Severity**: High. CVSS 3.1: AV:N/AC:H/PR:N/UI:N/S:U/C:H/I:L/A:N — 7.1.

### V8. Stored XSS via `{@html}` in reviews

**Where**: `src/routes/books/[isbn]/+page.svelte`.
```svelte
{#each reviews as r}
  <article>{@html r.bodyHtml}</article>
{/each}
```
**The bug**: `bodyHtml` is generated server-side from `bodyMarkdown` using `marked` with `sanitize: false` (deprecated option that no longer sanitizes anyway in marked >=5). DOMPurify is not used. A review body of `<img src=x onerror=fetch('/api/collections/users/records').then(...)>` exfiltrates the entire reader list (chains with V1).
**Reference**: SvelteKit security docs explicitly call out `{@html}` as user-controlled-data-hostile. CVE-2023-26136 (tough-cookie) demonstrated the analog in a different ecosystem; for Svelte the canonical writeup is the Snyk "Svelte XSS via {@html}" advisory.
**Severity**: Critical. CVSS 3.1: AV:N/AC:L/PR:L/UI:R/S:C/C:H/I:H/A:N — 9.0.

### V9. `/api/collections` schema disclosure

**Where**: PocketBase global setting "Hide collections from non-superusers" left at default (off, in 0.22).
**The bug**: `GET /api/collections` returns the full schema, including rule expressions, to any caller. This is intended only for superusers but the default permits it. An attacker reads the rule strings, finds V1/V3/V6/V10 by inspection, and skips fuzzing.
**Reference**: this default has been discussed in PocketBase GitHub Discussions and was tightened in later releases; the demo runs an older minor version.
**Severity**: Medium. CVSS 3.1: AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N — 5.3.

### V10. Mass-assignment of `isAdmin` on signup

**Where**: `users.createRule` and field schema.
**The bug**: `createRule` is empty (open). The `isAdmin` field is in the schema with no `@request.body.isAdmin = false` guard and no separate "hidden from create" flag. PocketBase's create endpoint accepts arbitrary fields that exist in the schema. So `POST /api/collections/users/records` with body `{email, password, passwordConfirm, isAdmin: true}` mints an app-admin account. Combined with V4's lack of admin auth hardening, this gives `/admin` access.
**Reference**: classic mass-assignment, CWE-915. The PocketBase docs cover this in "Securing your fields" but it's easy to miss.
**Severity**: Critical. CVSS 3.1: AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N — 9.8.

### V11. SSRF via "Import from Goodreads"

**Where**: `src/routes/books/import/+page.server.ts`, action `default`.
**The bug**: form posts a URL, server-side `fetch()` retrieves it, parses HTML for OG tags, creates a `books` record. No allowlist; `http://169.254.169.254/latest/meta-data/` happily resolves on the Hetzner box (which has IMDS-equivalents on its own metadata service for cloud-init data, and adjacent reachable internal services on the host network).
**Reference**: Capital One 2019 SSRF; Detectify SSRF-via-import patterns.
**Severity**: Medium. CVSS 3.1: AV:N/AC:L/PR:L/UI:N/S:C/C:L/I:N/A:N — 5.0 (limited by Hetzner's lack of AWS-style IMDS, but lateral-internal reach is real).

### V12. Verbose errors with SQL fragments

**Where**: PocketBase env `PB_DEBUG=true` left set in systemd unit.
**The bug**: error responses include the failing SQL snippet and the offending parameter, which is useful for an attacker mapping the schema and crafting filter-injection payloads. PocketBase filter expressions are not raw SQL but the error text leaks enough to confirm field names that are otherwise hidden.
**Severity**: Low. CVSS 3.1: AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N — 5.3 (treated as Low here because it's an enabler, not a primary).

## Real-World Parallels

- **PocketBase rule misconfigs (V1, V3, V6, V10)**: independent security researchers have published recurring writeups on PocketBase deployments with overly-permissive `listRule`/`viewRule` defaults; the pattern echoes Firebase Firestore rules misconfig (e.g., the Appthority 2018 "open Firebase" report that catalogued 3,000+ exposed apps, and the 2023 "Firebase rules `if true`" wave covered by various bounty hunters).
- **SvelteKit CSRF (V2)**: SvelteKit's `csrf.checkOrigin` default-on was a direct response to a class of bounty reports; teams that disable it for "dev convenience" reproduce the original bug. See the SvelteKit changelog for v1.0.0-next.500-era hardening.
- **MIME upload bypass (V5)**: GitLab CVE-2020-13280, plus the long Imgur-style history of HTML-as-image confusion.
- **Default admin creds (V4)**: Mirai botnet (2016) and the recurring "Jenkins admin/admin" finding.
- **Weak share-link signing (V7)**: Slack's 2017 shared-channel link incident, Trello public-board enumeration via short tokens (multiple HackerOne reports).
- **Stored XSS via markdown (V8)**: GitHub's own 2017 markdown XSS (CVE-2017-9930-class), the recurring `marked` sanitize-removal advisories.
- **SSRF via URL import (V11)**: Capital One CVE-2019-* SSRF; SSRF-via-webhook patterns documented by Assetnote.
- **Mass assignment (V10)**: GitHub's 2012 Rails mass-assignment incident (Egor Homakov), still the canonical reference.

## Apex Features Showcased

- **Whitebox PocketBase rule audit**: Apex reads `pb_schema.json` and `pb_migrations/*.go|*.js`, parses each rule expression, classifies by collection sensitivity, and flags rules that match known anti-patterns (`@request.auth.id != ""` on a non-self-scoped collection, empty `createRule` on auth collections with privilege fields, missing `subscriptionRule`). Hits V1, V3, V6, V9, V10 statically.
- **Single-binary blackbox `/pentest`**: against `:8090`, runs an authenticated swarm that enumerates collections via `/api/collections` (V9), tries default admin (V4), exercises `expand=` chains (V3), and attempts mass-assigned signup (V10).
- **JS endpoint extraction**: from the SvelteKit edge bundle, extracts `/api/share/[token]`, the Stripe webhook path, and the `import` action, feeding V7 and V11 paths.
- **Attack-surface map**: produces a graph of "edge route → form action → PocketBase collection rule," letting the operator click from V2 (the action) to the underlying create call and the rule that did or didn't catch it.
- **Threat-model agent**: ingests the `Data Model` section of the README and proposes "if `isAdmin` is on `users` and `createRule` is open, signup is mass-assignable" — the kind of property-driven reasoning that finds V10 without any traffic.
- **Patching agent**: emits a unified diff for `pb_migrations/*_fix_rules.js` that tightens listRule/viewRule/subscriptionRule, plus a `+hooks.server.ts` snippet adding origin check for V2, plus a `marked` → `marked + DOMPurify` swap for V8.
- **Findings + CVSS + judge**: the judge sub-agent rejects a naive "V1 is Critical" claim and downgrades to High based on data sensitivity, demonstrating calibration.
- **Memory**: across runs, Apex remembers that this target is PocketBase-on-Hetzner and skips re-discovery, going straight to rule audit on subsequent invocations.

## Demo Storyline

A 12-minute recording, three acts.

**Act 1 — Whitebox (4 min).** Operator clones the repo, runs `apex /pentest --whitebox .`. Apex detects PocketBase by `pb_schema.json` and SvelteKit by `svelte.config.js`. Within 30 seconds the rule auditor returns 6 findings (V1, V3, V6, V9, V10) with collection names and rule strings. Operator opens V10, sees the proposed exploit ("POST to `/api/collections/users/records` with `isAdmin:true`"), and asks Apex to confirm dynamically.

**Act 2 — Blackbox confirmation (5 min).** Apex spins up the local PocketBase from the demo's `make dev` (single binary, no Docker), runs the swarm. The swarm:

1. Confirms V10 by signing up `pwn@example.com` with `isAdmin:true` and reading the response.
2. Logs in as that admin, hits `/admin`, screenshots the moderation queue.
3. Tries V4's `admin@unboundshelf.io / changeme123` against `/_/` — succeeds, dumps full collection list.
4. With a normal user token, exercises V3 by listing `shelves` with `?expand=curator,books.reviews_via_book.author` and shows the leaked librarian email.
5. Posts a review with the V8 payload and watches the second-browser session ship its `users` list to a webhook.

**Act 3 — Patch + judge (3 min).** Operator runs `apex /operator patch`. Apex emits:

- A `pb_migrations/1714600000_tighten_rules.js` migration that rewrites the offending rules.
- A `src/hooks.server.ts` patch enabling `csrf.checkOrigin` and adding an explicit origin allowlist.
- A `src/lib/server/markdown.ts` patch replacing `marked` raw with `marked` + DOMPurify, plus a Svelte change from `{@html}` to a sanitized `{@html sanitized}`.
- A `deploy.sh` patch removing the hardcoded password and reading from `${ADMIN_PASSWORD:?}`.

The judge re-runs the swarm against the patched binary, confirms 9 of 12 findings closed, leaves V11 and V12 open with notes ("SSRF allowlist out of scope for rules-only patch; debug flag needs ops change"), and the recording ends on the findings diff.

## Build Notes

- **Days 1**: scaffold SvelteKit, drop in PocketBase binary, write `pb_migrations/0001_init.js` with all collections. Wire `users` auth via SvelteKit form action. Tailwind + the kraft-paper aesthetic. Seed data: 200 books pulled from Project Gutenberg metadata, 30 of which we mark as "restricted" with plausible jurisdictions in `restrictionNotes`. 8 librarians, 12 shelves.
- **Day 2**: implement reviews + markdown rendering (deliberately broken per V8), shelf detail with expand support, share link generation (V7), avatar upload (V5). Deploy script (V4).
- **Day 3**: subscriptions + Stripe + newsletter, "Import from Goodreads" (V11), realtime activity sidebar (V6). End-to-end test the happy paths so the bugs are *latent*, not loud.
- **Day 4**: write `pb_data/types.d.ts`, finalize the README so it matches what Apex's threat-model agent will read, and dry-run the Apex commands. Verify every vuln triggers cleanly under the swarm without flake.
- **Day 5**: polish, record, edit. Cut to 12 minutes. Add a "post-patch" run as the closer.

Repo layout:
```
unboundshelf/
  apps/web/            # SvelteKit
  services/pb/         # pocketbase binary + pb_hooks/ + pb_migrations/
  scripts/deploy.sh
  scripts/seed.ts
  README.md
  Makefile
```

`make dev` boots both: PocketBase on `:8090`, SvelteKit on `:5173`, with `PUBLIC_PB_URL=http://127.0.0.1:8090`.

Realism nits to keep in: the dev disabled `csrf.checkOrigin` in `svelte.config.js` with a `// TODO: re-enable after staging` comment dated 14 months ago; the `pb_hooks/utils.js` file has a `console.log(record)` in the user-create hook that prints passwords to the systemd journal (bonus finding for the careful operator, not counted in the 12).

## Recording Notes

- Record at 1440p, 30fps, dark terminal theme (Catppuccin Mocha or similar). Keep the SvelteKit browser side in light mode for visual contrast — kraft-paper backgrounds film well.
- B-roll: the kraft-paper homepage with "Currently Banned in at Least 3 Jurisdictions" shelf scrolling. Soft jazz, 30 seconds, then drop to silence when terminal work begins.
- Lower-third for each finding: severity, CVSS, the one-line rule string. Keep it on screen for 4 seconds.
- For V8 (XSS), use a second browser window split-screen; the exfil endpoint is a local `nc -l 9999` so the bytes are visible in the terminal.
- For V4, briefly show the `deploy.sh` line in the editor before the dynamic confirmation — the audience should see the literal `changeme123` for half a second.
- Do not show real PII; all seeded librarians have @example.com addresses and Lorem-Ipsum bios.
- End card: "Apex found 12 issues in 4 minutes. The dev built it in 4 days. Sounds about right." Pause two seconds. Cut.
